# Lesson 19：事务 Outbox、任务编排与补偿

## 学习目标

完成本课后，你应该能够：

- 解释 DB 与 broker/Kafka/Redis Stream 双写问题。
- 设计 transactional outbox 表、publisher、reconciliation job。
- 判断 Celery chain/group/chord 的失败语义，不把它误当业务事务。
- 用 saga/compensation 设计跨系统工作流。

## 关键问题

1. DB commit 成功但 Celery/Kafka publish 失败怎么办？
2. publish 成功但 outbox 标记 published 失败怎么办？
3. chain 第 3 步失败，前 2 步的 side effect 如何处理？
4. 补偿是 rollback 吗？什么时候补偿也可能失败？

## 核心结论

Outbox 解决的是“业务状态变化”和“待发布事件记录”在同一数据库事务里原子提交。它不保证消息只发布一次，所以 consumer 必须幂等。

```text
business transaction
  write order/payment/inventory
  write outbox_event(status=pending)
commit
publisher scans pending
  publish to Celery/Kafka/Redis Stream
  mark published
consumer idempotently applies side effect
reconciler repairs stuck states
```

## 1. 双写问题

错误：

```python
with transaction.atomic():
    order = Order.objects.create(...)
    kafka.produce("order-events", payload)
```

失败窗口：

| 窗口 | 结果 |
|---|---|
| DB 写成功，Kafka 失败 | 业务状态已变，但没有事件 |
| Kafka 成功，DB rollback | 下游收到不存在/未提交事件 |
| publish 成功，进程 crash | 无法知道是否要重发 |

## 2. Outbox 表

```python
class OutboxEvent(models.Model):
    event_id = models.CharField(max_length=128, unique=True)
    aggregate_type = models.CharField(max_length=64)
    aggregate_id = models.CharField(max_length=128)
    event_type = models.CharField(max_length=128)
    schema_version = models.IntegerField(default=1)
    payload = models.JSONField()
    status = models.CharField(max_length=32, default="pending")
    attempts = models.IntegerField(default=0)
    next_attempt_at = models.DateTimeField(null=True)
    published_at = models.DateTimeField(null=True)
    last_error = models.TextField(blank=True)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        indexes = [
            models.Index(fields=["status", "next_attempt_at", "id"]),
            models.Index(fields=["aggregate_type", "aggregate_id", "id"]),
        ]
```

## 3. Publisher 并发抢占

```python
with transaction.atomic():
    events = list(
        OutboxEvent.objects
        .select_for_update(skip_locked=True)
        .filter(status="pending", next_attempt_at__lte=now())
        .order_by("id")[:100]
    )
    for event in events:
        event.status = "publishing"
        event.save(update_fields=["status"])
```

`skip_locked` 适合多个 publisher 并发抢 pending event，避免互相等待。

发布后：

```python
try:
    producer.publish(event)
except Exception as exc:
    mark_retry(event, exc)
else:
    mark_published(event)
```

窗口：publish 成功但 mark_published 失败。解决：允许重发，consumer 幂等。

## 4. Reconciliation job

必须定期扫异常状态：

```text
pending 太久
publishing 太久
attempts 超限
published 但下游未处理
```

指标：

```text
outbox_pending_count
outbox_oldest_pending_age
outbox_publish_failure_total
outbox_republish_total
outbox_stuck_publishing_count
```

## 5. Celery canvas 不是事务

```python
chain(reserve_inventory.s(order_id), capture_payment.s(), send_email.s())()
```

如果 `send_email` 失败，前面的库存预留和支付捕获不会自动 rollback。

业务工作流应由状态机驱动：

```text
order.pending
→ inventory_reserved
→ payment_captured
→ notification_sent
→ completed
```

每一步 task 读取当前状态，幂等推进到下一状态。

## 6. Saga 与补偿

补偿不是数据库 rollback，而是新的业务动作：

```text
capture payment 成功，但 reserve shipment 失败
→ refund payment
→ mark order failed_compensated
```

补偿也会失败，所以要有：

- compensation task。
- compensation status。
- retry/backoff。
- 人工处理队列。

## 7. Runbook：Outbox 堵塞

```text
症状：订单状态已更新，但下游 read model/通知延迟

1. 查询 outbox
   pending count, oldest age, attempts, last_error
2. 查 publisher worker
   active/reserved/runtime/error
3. 查 broker/Kafka/Redis Stream
   publish latency/error
4. 判断
   - publisher 停了？
   - poison event schema？
   - 下游 broker 不可用？
   - stuck publishing？
5. 修复
   - 重启 publisher
   - stuck publishing → pending
   - poison → DLQ/manual
   - 修 schema/consumer
6. 验证
   oldest pending age 下降
```

## 常见误区

- `transaction.on_commit(lambda: task.delay())` 就等于 outbox。
- 认为 outbox 保证 exactly-once。
- chain/group/chord 当分布式事务。
- 没有 reconciliation job。
- 补偿流程没有人工出口。

## 作业

为订单支付工作流设计 outbox + saga：订单创建、库存预留、支付捕获、通知、失败补偿。写出状态机、outbox schema、publisher、consumer 幂等、reconciliation、人工处理字段。

## 评估标准

- 能解释双写失败窗口。
- 能设计 outbox 表和 publisher 并发抢占。
- 能说明为什么 consumer 必须幂等。
- 能设计补偿状态机和 reconciliation job。
