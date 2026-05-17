# Lesson 20：Celery 监控、队列堆积与排障

## 学习目标

- 建立 Celery 的队列、worker、task、broker、下游依赖五层观测模型。
- 使用 inspect/events/Flower/Prometheus 指标定位 active、reserved、scheduled、retry、lost worker。
- 区分 queue depth、oldest message age、runtime histogram、failure rate、retry rate。
- 写出 Celery 事故 runbook。

## 核心结论

Celery 排障不要只看 worker 日志。队列堆积是结果，不是根因。必须拆：

```text
arrival rate > processing rate?
worker online?
prefetch 囤任务?
task runtime 变慢?
downstream DB/Redis/API 慢?
poison message retry storm?
broker reconnect/visibility timeout?
```

## 1. 基础命令

```bash
celery -A config status
celery -A config inspect active
celery -A config inspect reserved
celery -A config inspect scheduled
celery -A config inspect stats
celery -A config inspect registered
celery -A config events
```

解释：

- active：正在执行。
- reserved：已被 worker 预取但未执行。
- scheduled：ETA/countdown 等待。
- stats：pool、prefetch、broker、rusage。

## 2. 关键指标

```text
queue_depth{queue}
oldest_message_age_seconds{queue}
task_runtime_seconds_bucket{task,queue}
task_success_total{task}
task_failure_total{task,exception_type}
task_retry_total{task,exception_type}
worker_online{worker,queue}
worker_active_tasks{worker}
worker_reserved_tasks{worker}
broker_publish_latency_ms
```

`queue_depth` 不够。一个队列长度只有 5，但 oldest age 2 小时，说明可能有 poison/stuck task。

## 3. Events 与 Flower

启用：

```python
worker_send_task_events = True
task_send_sent_event = True
task_track_started = True
```

代价：更多事件流量。生产可采样或只在关键队列开启。

Flower 适合人工查看，不应作为唯一监控系统。核心指标要进入 Prometheus/Grafana。

## 4. Worker lost / OOM

现象：

```text
WorkerLostError
Child process exited prematurely
signal 9 (SIGKILL)
```

证据：

```bash
journalctl -u celery-worker -n 200 --no-pager
dmesg -T | grep -i 'killed process'
ps -o pid,ppid,cmd,rss,%mem -C celery
```

常见原因：

- task 内存泄漏。
- 一次加载大 QuerySet。
- PDF/图片处理爆内存。
- hard time limit kill。
- 容器 memory limit 太低。

## 5. Broker reconnect

broker 短暂不可用会导致：

- producer publish fail。
- worker reconnect。
- late ack task 重投。
- Redis visibility timeout 后重复。

日志字段必须保留 broker URL family、exception type、duration。

## 6. 队列堆积定位矩阵

| 现象 | 可能原因 | 证据 |
|---|---|---|
| active 很多，runtime 高 | task 慢/下游慢 | runtime histogram、DB/API latency |
| reserved 很多，active 少 | prefetch 囤积/worker 卡 | inspect reserved、prefetch config |
| scheduled 很多 | ETA 滥用 | inspect scheduled |
| failure/retry 高 | 下游错误/poison | exception_type、retry count |
| worker 少 | deploy/崩溃 | status、orchestrator events |
| depth 高但 worker 空闲 | routing 错 | queue binding、registered tasks |

## 7. Runbook：订单 read model 延迟

```text
1. 看用户影响
   - order status page latency/read model lag
2. 看 Celery 队列
   - depth, oldest age, active/reserved
3. 看 task runtime
   - update_read_model P95/P99
4. 看下游
   - MongoDB latency/explain
   - Redis/Kafka latency
5. 看错误
   - retry_total by exception_type
   - failed_tasks open count
6. 止血
   - 扩 read_model worker
   - 降 prefetch
   - poison → DLQ
   - 熔断下游非关键调用
7. 验证
   - oldest age 回落
   - read model lag 回落
```

## 常见误区

- 只看 queue length，不看 oldest age。
- worker 全部订阅所有队列，关键任务被慢任务饿死。
- 没有 task runtime histogram。
- retry storm 时盲目扩 worker，打爆下游。

## 作业

为 Celery 设计 Grafana dashboard：队列、worker、task、broker、下游依赖五个区域，每个区域至少 4 个指标，并写出 3 个告警规则。

## 评估标准

- 能用 inspect 命令解释 active/reserved/scheduled。
- 能建立队列堆积定位矩阵。
- 能写出 worker lost/OOM 排障路径。
- 能设计 Celery dashboard 和告警。
