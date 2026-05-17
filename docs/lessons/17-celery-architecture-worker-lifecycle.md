# Lesson 17：Celery 架构、Broker 与 Worker 生命周期

## 学习目标

完成本课后，你应该能够：

- 解释 Celery 的 producer、broker、worker、pool、result backend、beat 之间的职责边界。
- 理解 Redis/RabbitMQ 作为 broker 时的投递语义差异和风险。
- 解释 prefork、threads、gevent、eventlet、solo worker pool 的适用场景。
- 用 `inspect active/reserved/scheduled`、events、worker 日志定位 worker 生命周期和队列堆积问题。

## 关键问题

1. Celery task 从 `delay()` 到 worker 执行，中间经过哪些状态？
2. `prefetch_multiplier` 为什么会导致某个 worker “囤任务”？
3. Redis broker 的 `visibility_timeout` 和 RabbitMQ ack 模型有什么差异？
4. worker crash、OOM、SIGTERM、deploy reload 时，任务到底会不会丢/重跑？

## 核心结论

Celery 是任务执行系统，不是业务一致性系统。它解决的是“把函数调用交给 worker 执行”，但不自动解决 DB commit、一致性、幂等、去重、补偿和可观测性。

生产上必须把 Celery 看成一个独立运行时：

```text
Django web producer
→ broker queue
→ worker main process
→ worker pool child/thread/greenlet
→ task ack/retry/result
→ monitoring/events
```

任何可靠 Celery 设计都要先回答：什么时候 ack？任务能否重复执行？worker crash 后如何恢复？队列堆积时如何隔离？


## 1. Celery 架构图

```text
producer: Django/Gunicorn
  send task message
       ↓
broker: Redis/RabbitMQ
  queue/routing/visibility/ack
       ↓
worker main process
  consumer, heartbeats, control commands
       ↓
pool: prefork/thread/gevent/eventlet/solo
  executes task function
       ↓
result backend: optional
  store result/state
```

关键边界：

- producer 只负责发布消息。
- broker 负责暂存和投递消息。
- worker 负责消费并执行。
- result backend 不是审计系统，不能长期保存业务结果。
- task 函数必须可重复执行或能识别重复。

## 2. Task message 生命周期

调用：

```python
send_order_email.delay(order_id=123)
```

等价于发布一条消息，里面包含：

```json
{
  "id": "task-uuid",
  "task": "orders.tasks.send_order_email",
  "args": [],
  "kwargs": {"order_id": 123},
  "retries": 0,
  "eta": null,
  "expires": null
}
```

worker 侧典型状态：

```text
reserved → active → succeeded/failed/retried/revoked
```

观察：

```bash
celery -A config inspect active
celery -A config inspect reserved
celery -A config inspect scheduled
celery -A config inspect stats
```

## 3. Broker 选择：Redis vs RabbitMQ

| 维度 | Redis broker | RabbitMQ broker |
|---|---|---|
| 运维复杂度 | 低 | 中 |
| 语义 | visibility timeout 模拟 ack | 原生 AMQP ack |
| routing | 基础 | 强 |
| 优先级 | 有限制 | 较成熟 |
| 大规模队列 | 一般 | 更适合 |
| 常见风险 | visibility timeout、Redis 内存、和 cache 混用 | queue/exchange 配置、连接/ack 管理 |

Redis broker 常见坑：

- Redis 同时做 cache/session/broker，cache eviction 影响任务。
- `visibility_timeout` 小于任务执行时间，任务可能重复投递。
- 大量 ETA/countdown task 可能给 broker/worker 带来内存压力。

## 4. Worker pool 深入

### prefork

```bash
celery -A config worker -P prefork -c 4
```

特点：

- 多进程。
- 适合 CPU-bound 和普通 Django ORM task。
- 每个进程有独立 Python interpreter 和 DB connection。
- 内存开销较高。

### threads

```bash
celery -A config worker -P threads -c 16
```

适合 I/O-bound，注意 GIL 对 CPU-bound 无帮助。

### gevent/eventlet

```bash
celery -A config worker -P gevent -c 100
```

风险：

- monkey patch 必须早。
- Django ORM、MySQL client、Redis client、HTTP client 兼容性要压测。
- CPU-bound task 会阻塞 event loop/greenlet 调度。
- 一旦某个库未 patch 或 C 扩展阻塞，高并发会退化。

### solo

调试用：

```bash
celery -A config worker -P solo -c 1
```

## 5. Prefetch：worker 囤任务问题

配置：

```python
worker_prefetch_multiplier = 4
worker_concurrency = 8
```

一个 worker 最多预取：

```text
4 × 8 = 32 tasks
```

如果 task 耗时差异大，某个 worker 可能预取了很多慢任务，其他 worker 空闲，造成不公平。

长任务建议：

```python
worker_prefetch_multiplier = 1
task_acks_late = True
```

但 `acks_late=True` 要求 task 幂等，因为 worker crash 后任务会重投。

## 6. Ack 时机

默认通常是执行前/早 ack 风险较低重复但可能丢任务；`acks_late=True` 是执行后 ack，worker crash 会重投。

```python
@app.task(bind=True, acks_late=True, reject_on_worker_lost=True)
def process_order(self, event_id: str):
    ...
```

选择：

- 不可重复、非关键任务：可接受早 ack。
- 关键业务 side effect：late ack + 幂等 + outbox/状态机。
- 外部支付/邮件：必须有 provider idempotency key 或业务去重。

## 7. Worker 部署与优雅退出

Celery worker 收到 SIGTERM 会尝试优雅停止：

```text
stop accepting new tasks
wait active tasks finish
exit
```

但容器平台可能有 termination grace period。若 task 超过 grace period，会被 kill。

配置建议：

```python
worker_cancel_long_running_tasks_on_connection_loss = True
worker_send_task_events = True
task_track_started = True
task_time_limit = 300
task_soft_time_limit = 240
```

## 8. 队列隔离

不要一个 default 队列跑所有任务。

```text
critical: 短、关键、用户可感知
email: 可延迟通知
heavy: CPU/长任务
webhook: 外部回调
maintenance: backfill/清理
```

路由：

```python
task_routes = {
    "orders.tasks.publish_outbox": {"queue": "critical"},
    "notifications.tasks.send_email": {"queue": "email"},
    "reports.tasks.generate_pdf": {"queue": "heavy"},
}
```

## 9. 生产证据字段

每个 task 日志必须包含：

```json
{
  "event": "celery_task_finished",
  "task_id": "...",
  "task_name": "orders.tasks.publish_outbox",
  "queue": "critical",
  "request_id": "req_1",
  "event_id": "evt_1",
  "retries": 1,
  "latency_ms": 1200,
  "status": "success"
}
```

指标：

```text
task_runtime_ms
task_failure_total
task_retry_total
queue_depth
oldest_message_age
worker_active_count
worker_reserved_count
```

## 10. Runbook：队列堆积

```text
症状：订单邮件/读模型延迟，Celery queue depth 上升

1. 看 worker 状态
   celery -A config inspect active
   celery -A config inspect reserved
   celery -A config inspect stats
2. 看队列维度
   broker queue length
   oldest message age
3. 判断
   - worker 不在线？
   - active task 慢？
   - reserved 太多 prefetch？
   - 下游 DB/Redis/API 慢？
   - poison task 反复 retry？
4. 处理
   - 扩对应队列 worker
   - 降 prefetch
   - 隔离慢任务
   - revoke/修复 poison task
   - 修复下游依赖
5. 验证
   queue depth 下降
   oldest age 下降
   task runtime/failure 恢复
```


## 作业

为 `Production Order Workflow` 设计 Celery 部署：队列划分、worker pool、concurrency、prefetch、ack 策略、time limit、日志字段和队列堆积 runbook。

## 评估标准

- 能画出 Celery producer/broker/worker/pool/result backend 边界。
- 能解释 prefetch、acks_late、visibility_timeout 的风险。
- 能根据 task 类型选择 worker pool。
- 能用 inspect/events/metrics 定位队列堆积。
