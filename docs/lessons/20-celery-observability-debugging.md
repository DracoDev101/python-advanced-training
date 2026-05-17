# Lesson 20：Celery 监控、队列堆积与排障

## 学习目标

- 建立 Celery 从 broker → worker → task → 下游依赖的证据链。
- 使用 inspect、events、broker metrics、结构化日志定位队列堆积。
- 区分 worker 不足、prefetch 囤积、poison task、下游慢、broker 故障。
- 设计 Celery dashboard 和告警。

## 关键问题

1. queue length 不高但用户仍然延迟，可能是什么问题？
2. active 很少、reserved 很多说明什么？
3. worker lost 与 task failed 的处理差异是什么？
4. 如何证明扩容 worker 有效，而不是让下游更崩？

## 核心结论

Celery 排障不能只看 worker 日志。必须分层：

```text
broker queue depth / oldest age
→ worker online/concurrency/prefetch
→ active/reserved/scheduled
→ task runtime/failure/retry
→ downstream DB/Redis/API latency
→ business lag
```

## 1. 第一批命令

```bash
celery -A config status
celery -A config inspect active
celery -A config inspect reserved
celery -A config inspect scheduled
celery -A config inspect stats
celery -A config inspect registered
```

含义：

- active：正在执行。
- reserved：已被 worker 预取但未执行。
- scheduled：ETA/countdown 等待执行。
- stats：pool、prefetch、broker、rusage。

## 2. 必备指标

```text
queue_depth{queue}
oldest_message_age_seconds{queue}
task_runtime_seconds_bucket{task,queue}
task_success_total{task}
task_failure_total{task,error_type}
task_retry_total{task,error_type}
worker_online{hostname,queue}
worker_active_count{hostname}
worker_reserved_count{hostname}
downstream_latency_ms{dependency}
```

`oldest_message_age` 比 queue length 更重要。低 length 但 oldest age 高，说明有少数老任务卡住。

## 3. 典型故障分类

### 3.1 worker 不在线

证据：

```bash
celery -A config status
```

修复：重启 worker、检查 broker URL、检查部署/secret。

### 3.2 reserved 很多 active 少

可能：prefetch 太高，worker 囤任务。

修复：

```python
worker_prefetch_multiplier = 1
```

长任务队列单独 worker。

### 3.3 active 很多且 runtime 高

下游慢或任务本身慢。看 task runtime 分布和下游 span。

### 3.4 retry 激增

可能下游故障，继续 retry 会放大。应启用 circuit breaker、限流、DLQ。

### 3.5 worker lost / OOM

证据：worker 日志、container exit code、dmesg/OOMKilled。

处理：降低 concurrency、拆大任务、加 soft/hard time limit、优化内存。

## 4. Events 与 Flower

启用：

```python
worker_send_task_events = True
task_send_sent_event = True
task_track_started = True
```

实时：

```bash
celery -A config events
```

Flower 适合现场观察，但生产告警应接 Prometheus/Grafana，不依赖人工看 UI。

## 5. 结构化日志串链路

同一订单链路：

```text
request_id=req_1
order_id=123
event_id=evt_1
task_id=task_1
queue=critical
```

task 开始/结束都打：

```json
{"event":"task_started","task_id":"...","task_name":"...","queue":"critical","order_id":123,"attempt":1}
{"event":"task_finished","task_id":"...","status":"success","latency_ms":834}
```

## 6. Runbook：队列堆积

```text
症状：critical queue oldest age > 60s

1. broker depth/oldest age
2. celery status / active / reserved / stats
3. top task runtime/failure/retry
4. 下游 DB/Redis/API latency
5. 判断：worker 数、prefetch、poison、下游慢
6. 动作：扩容/限流/隔离/DLQ/暂停非关键队列
7. 验证：oldest age 和 runtime 回落
```

## 作业

为 Celery 设计 Grafana dashboard：至少包含 queue depth、oldest age、runtime p95、failure/retry rate、worker online、reserved/active、downstream latency。

## 评估标准

- 能解释 inspect 各输出含义。
- 能用指标区分不同堆积根因。
- 能设计日志字段串起 HTTP→task→DB/API。
- 能给出队列堆积 runbook。
