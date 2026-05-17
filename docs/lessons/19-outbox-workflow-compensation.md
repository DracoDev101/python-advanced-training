# Lesson 19：事务 Outbox、任务编排与补偿

## 学习目标

- 理解 DB 与消息系统双写问题，以及为什么 `transaction.on_commit()` 不是完整 outbox。
- 设计 outbox 表、publisher、幂等 consumer、reconciliation job。
- 理解 Celery chain/group/chord 的失败语义与业务补偿边界。
- 用状态机和 Saga 思维设计订单工作流。

## 关键问题

1. DB commit 成功但 Kafka/Celery publish 失败怎么办？
2. outbox publisher 重复发布是否允许？前提是什么？
3. chain/group/chord 中某一步失败，已经完成的副作用如何补偿？
4. 如何发现 pending outbox 卡住？

## 核心结论

Outbox 把“业务状态变化”和“待发布事件”放进同一个 DB 事务：

```text
transaction.atomic:
  write business rows
  write outbox row
commit
publisher repeatedly publishes pending outbox
consumer handles events idempotently
```

它把不可控的跨系统双写，变成可恢复、可扫描、可重放的本地状态。

## 1. 双写问题

错误：

```python
with transaction.atomic():
    order = Order.objects.create(...)
    kafka.produce("order-events", event)
```

失败窗口：

| 窗口 | 结果 |
|---|---|
| DB 写成功，Kafka 失败 | 业务有订单，无事件 |
| Kafka 成功，DB 回滚 | 有幽灵事件 |
| Kafka 成功但响应超时 | 应用不知道是否发布成功 |

`transaction.on_commit(lambda: publish())` 避免了 DB 回滚后发布，但仍有 commit 成功后 publish 失败的窗口。

## 2. Outbox 表设计

```python
class OutboxEvent(models.Model):
    event_id = models.CharField(max_length=128, unique=True)
    aggregate_type = models.CharField(max_length=64)
    aggregate_id = models.CharField(max_length=128)
    aggregate_version = models.BigIntegerField()
    event_type = models.CharField(max_length=128)
    payload = models.JSONField()
    status = models.CharField(max_length=32, default="pending")
    attempts = models.IntegerField(default=0)
    next_attempt_at = models.DateTimeField(null=True)
    published_at = models.DateTimeField(null=True)
    last_error = models.TextField(null=True)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        indexes = [
            models.Index(fields=["status", "next_attempt_at", "id"]),
            models.Index(fields=["aggregate_type", "aggregate_id", "aggregate_version"]),
        ]
        constraints = [
            models.UniqueConstraint(fields=["aggregate_type", "aggregate_id", "aggregate_version"], name="uniq_outbox_aggregate_version"),
        ]
```

`aggregate_version` 支持同一聚合内顺序和幂等。

## 3. Publisher 抢占与并发

```python
with transaction.atomic():
    rows = (
        OutboxEvent.objects
        .select_for_update(skip_locked=True)
        .filter(status="pending", next_attempt_at__lte=now())
        .order_by("id")[:100]
    )
    events = list(rows)
```

`skip_locked=True` 让多个 publisher 并发处理不同 outbox row。

发布后更新：

```python
try:
    broker_publish(event)
except Exception as exc:
    event.attempts += 1
    event.next_attempt_at = backoff(event.attempts)
    event.last_error = repr(exc)
    event.save(update_fields=["attempts", "next_attempt_at", "last_error"])
else:
    event.status = "published"
    event.published_at = timezone.now()
    event.save(update_fields=["status", "published_at"])
```

仍可能 `broker_publish` 成功但 DB update 失败，所以 consumer 必须幂等。

## 4. Consumer 幂等与乱序

```python
class ProcessedEvent(models.Model):
    event_id = models.CharField(max_length=128, unique=True)
    consumer = models.CharField(max_length=128)
```

处理：

```python
with transaction.atomic():
    ProcessedEvent.objects.create(event_id=event_id, consumer="order-read-model")
    apply_event(event)
```

如果有聚合版本：

```text
expected next version = current_version + 1
if event_version <= current_version: duplicate/old, ack
if event_version > current_version + 1: gap, park/retry
```

## 5. Celery chain/group/chord 的边界

```python
chain(
    reserve_inventory.s(order_id),
    capture_payment.s(),
    send_confirmation.s(),
)()
```

问题：如果 capture 成功后 send 失败，Celery 不知道业务该如何补偿。

因此复杂业务不要只依赖 chain 表达一致性，应该有显式状态机：

```text
pending → inventory_reserved → payment_captured → confirmed
                 ↘ reserve_failed
                         ↘ compensation_required
```

每个 transition 是幂等的，并有补偿动作。

## 6. Saga / Compensation

示例：

```text
1. reserve inventory
2. capture payment
3. create shipment
```

如果 3 失败，补偿：

```text
void/refund payment
release inventory
mark order failed
```

补偿不是回滚。外部系统副作用已经发生，只能发起相反业务动作。

## 7. Reconciliation job

定时扫描异常状态：

```text
outbox pending age > 5m
payment captured but order not paid
inventory reserved but order cancelled
event published but read model missing
```

这是生产系统自愈能力的核心。

## 8. 指标

```text
outbox_pending_count
outbox_oldest_pending_age_seconds
outbox_publish_attempts_total
outbox_publish_failures_total
consumer_duplicate_total
consumer_gap_total
compensation_required_total
```

## 9. Runbook：outbox 堆积

```text
1. 查询 pending 数量和 oldest age
2. 看 publisher worker 是否在线
3. 看 last_error top N
4. 判断 broker/Kafka/Redis 是否故障
5. 扩 publisher 或暂停非关键事件
6. 对 poison event 标记 failed/manual_review
7. 验证 pending age 下降、consumer lag 恢复
```

## 作业

为 `order.paid` 设计 outbox：表结构、publisher 抢占、Kafka/Redis Stream 发布、consumer 幂等、补偿流程、reconciliation job 和指标。

## 评估标准

- 能解释双写问题所有失败窗口。
- 能设计 outbox 表和 publisher。
- 能说明重复发布为什么可接受。
- 能设计显式状态机和补偿流程。
