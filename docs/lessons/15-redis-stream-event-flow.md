# Lesson 15：Redis Stream 与轻量事件流

## 学习目标

完成本课后，你应该能够：

- 解释 Redis Stream 的 stream id、consumer group、PEL、ack、claim 机制。
- 设计 Django outbox → Redis Stream → consumer 的轻量事件流。
- 处理 at-least-once 投递下的幂等消费、重放、poison message、pending 堆积。
- 判断 Redis Stream、Celery broker、Kafka 的边界和替代关系。

## 关键问题

1. Redis Stream 和 Redis list queue 的可靠性差异在哪里？
2. 消息被 consumer 读走但没 ack，会发生什么？
3. `XPENDING` 长期增长说明什么？
4. Redis Stream 什么时候足够，什么时候应该上 Kafka？

## 核心结论

Redis Stream 是 Redis 内置的 append-only-ish 事件流结构，适合 Django 内部轻量事件分发：

```text
XADD stream * event fields
→ XGROUP consumer group
→ XREADGROUP read new entries
→ process with idempotency
→ XACK
→ XAUTOCLAIM stale pending
```

它提供 consumer group 和 pending 机制，但不是 Kafka。生产上要明确：

- Redis 内存和 trim 策略。
- consumer group pending entries lifecycle。
- at-least-once 语义下的幂等。
- stream id 与业务 event_id 的区别。

## 1. 基础命令

创建 stream 并写入：

```bash
redis-cli XADD order-events '*' event_id evt_1 event_type order.created order_id 123
```

查看：

```bash
redis-cli XRANGE order-events - + COUNT 10
redis-cli XLEN order-events
```

创建 consumer group：

```bash
redis-cli XGROUP CREATE order-events order-read-model '$' MKSTREAM
```

从新消息读：

```bash
redis-cli XREADGROUP GROUP order-read-model c1 COUNT 10 BLOCK 5000 STREAMS order-events '>'
```

ack：

```bash
redis-cli XACK order-events order-read-model 1710000000000-0
```

## 2. Stream id vs event_id

Redis stream id：

```text
1710000000000-0
```

含义：毫秒时间 + 序列号。用于 Redis 内部排序和 ack。

业务 event_id：

```text
evt_01H...
```

用于幂等、追踪、跨系统关联。

不要把 stream id 当唯一业务幂等键，因为：

- 同一个业务事件可能重新发布到另一个 stream，stream id 变化。
- 事件从 Redis Stream 转发到 Kafka 后需要稳定业务 id。
- replay/backfill 时 stream id 可能完全不同。

## 3. PEL：Pending Entries List

consumer 读到消息但未 ack，消息进入 PEL。

查看 group：

```bash
redis-cli XINFO GROUPS order-events
```

查看 pending 摘要：

```bash
redis-cli XPENDING order-events order-read-model
```

查看 pending 明细：

```bash
redis-cli XPENDING order-events order-read-model - + 10
```

PEL 增长意味着：

- consumer 处理慢。
- consumer crash 后消息未 ack。
- poison message 反复失败。
- consumer 忘记 ack。

## 4. XAUTOCLAIM：接管超时 pending

```bash
redis-cli XAUTOCLAIM order-events order-read-model c2 60000 0-0 COUNT 10
```

含义：把 idle 超过 60s 的 pending entries 转给 consumer `c2`。

生产策略：

```text
normal consumer: read '>' new messages
reclaimer: periodically XAUTOCLAIM stale PEL
poison handling: attempts > N → write DLQ stream + XACK original
```

Redis Stream 没有内置 DLQ，要自己建：

```bash
XADD order-events-dlq '*' original_stream_id <id> event_id evt_1 reason validation_failed payload <json>
```

## 5. Django outbox → Redis Stream

业务事务内写 outbox：

```python
with transaction.atomic():
    order = Order.objects.create(...)
    OutboxEvent.objects.create(
        event_id=event_id,
        event_type="order.created",
        aggregate_id=f"order:{order.id}",
        payload=payload,
        status="pending",
    )
```

publisher：

```python
def publish_pending_outbox(batch_size=100):
    events = OutboxEvent.objects.filter(status="pending").order_by("id")[:batch_size]
    for event in events:
        stream_id = redis.xadd(
            "order-events",
            {
                "event_id": event.event_id,
                "event_type": event.event_type,
                "aggregate_id": event.aggregate_id,
                "payload": json.dumps(event.payload),
            },
            maxlen=1_000_000,
            approximate=True,
        )
        event.status = "published"
        event.published_stream_id = stream_id.decode()
        event.save(update_fields=["status", "published_stream_id"])
```

注意：上面仍有“xadd 成功但 DB save 失败”的窗口。更稳的做法：

- publisher 可重复发布，但 event_id 稳定。
- consumer 基于 event_id 幂等。
- outbox publisher 记录 attempts 和 last_error。
- 定期 reconcile：stream 与 outbox 状态不一致时可重发。

## 6. Consumer 幂等

```python
class ProcessedEvent(models.Model):
    event_id = models.CharField(max_length=128, unique=True)
    processed_at = models.DateTimeField(auto_now_add=True)
```

消费逻辑：

```python
def handle_event(stream_id: str, fields: dict):
    event_id = fields["event_id"]
    try:
        with transaction.atomic():
            ProcessedEvent.objects.create(event_id=event_id)
            apply_business_change(fields)
    except IntegrityError:
        # duplicate delivery, safe to ack
        pass
    redis.xack("order-events", "order-read-model", stream_id)
```

ack 时机：

- 成功处理后 ack。
- 已处理过的重复消息也 ack。
- 不可恢复错误写 DLQ 后 ack。
- 可恢复错误不 ack，等待 claim/retry，但必须有 attempts 限制。

## 7. Consumer loop 设计

```python
def consume_forever(consumer_name: str):
    while True:
        resp = redis.xreadgroup(
            groupname="order-read-model",
            consumername=consumer_name,
            streams={"order-events": ">"},
            count=50,
            block=5000,
        )
        for stream, messages in resp:
            for stream_id, fields in messages:
                try:
                    handle_event(stream_id.decode(), decode_fields(fields))
                except TransientError:
                    logger.exception("stream_event_transient_failed", extra={"stream_id": stream_id})
                    # 不 ack，等待重试/claim
                except Exception as exc:
                    write_dlq(stream_id, fields, exc)
                    redis.xack("order-events", "order-read-model", stream_id)
```

生产要加：

- graceful shutdown。
- batch size。
- block timeout。
- metrics。
- poison message attempts。
- reclaim worker。

## 8. Trim 策略

```bash
XTRIM order-events MAXLEN '~' 1000000
```

或 `XADD MAXLEN ~`。

风险：如果 stream 被 trim，但某 group PEL 里还有未处理消息，可能造成恢复困难。

监控：

```bash
XINFO STREAM order-events
XINFO GROUPS order-events
XPENDING order-events order-read-model
```

指标：

```text
stream_length
consumer_lag_estimate
pending_count
oldest_pending_idle_ms
processed_total
failed_total
dlq_total
trimmed_total
```

## 9. Redis Stream vs Celery vs Kafka

| 能力 | Redis Stream | Celery | Kafka |
|---|---|---|---|
| 核心定位 | 轻量事件流 | 任务执行系统 | 分区持久日志 |
| Consumer group | 有 | worker queue | 有 |
| Ack/Pending | 有 PEL | 有 ack/retry | offset commit |
| 长期保留 | 受内存/trim 限制 | 不适合 | 强 |
| 大规模 replay | 一般 | 弱 | 强 |
| 调度/ETA/retry | 手写 | 内置较多 | 通常手写/平台化 |
| 适合 | 单服务/小系统事件流 | 后台任务 | 多服务事件总线 |

选择建议：

- 需要执行函数、重试、ETA、rate limit：Celery。
- 需要轻量内部事件流和 consumer group：Redis Stream。
- 需要跨服务、高吞吐、长期 replay、多 consumer group：Kafka。

## 10. Runbook：Pending 堆积

```text
症状：订单读模型延迟，Redis Stream pending 增长

1. 看 group 状态
   XINFO GROUPS order-events
   XPENDING order-events order-read-model
2. 看 consumer 明细
   XINFO CONSUMERS order-events order-read-model
3. 判断原因
   - active consumer 少？
   - idle pending 很久？
   - 单条 poison message？
   - consumer 处理慢？
4. 处理
   - XAUTOCLAIM idle pending
   - 扩 consumer
   - poison → DLQ + XACK
   - 修复下游 DB/Redis 慢点
5. 验证
   pending_count 下降
   oldest_pending_idle_ms 下降
   read model lag 恢复
```

## 11. 常见误区

### 误区 1：读到消息就马上 ack

如果处理失败，消息丢失。ack 必须在业务处理成功或明确 DLQ 后。

### 误区 2：没有业务幂等

Redis Stream 是 at-least-once。重复投递是正常情况，不是异常。

### 误区 3：无限保留 stream

Redis 是内存系统。必须定义 maxlen/trim 与归档策略。

### 误区 4：用 Redis Stream 替代所有 Kafka

当需求是长期保留、大规模 replay、多服务事件总线时，Kafka 更合适。

## 12. 课堂讨论题

1. PEL 中 pending 很多，但 stream length 不大，说明什么？
2. `XAUTOCLAIM` 后，如果原 consumer 其实还在处理，会不会重复处理？如何保证安全？
3. Redis Stream consumer 的 DLQ 应该保存哪些字段？
4. Outbox publisher 为什么可以允许重复发布？前提是什么？

## 13. 作业

为 `order.created` 设计 Redis Stream 消费链路：

- stream name。
- consumer group。
- event fields。
- idempotency table。
- DLQ stream。
- pending reclaim 策略。
- trim 策略。
- metrics 和 runbook。

## 14. 评估标准

- 能解释 stream id、consumer group、PEL、XACK、XAUTOCLAIM。
- 能设计 Django outbox 到 Redis Stream 的发布和消费。
- 能处理 at-least-once 与幂等。
- 能判断 Redis Stream 与 Celery/Kafka 的边界。
