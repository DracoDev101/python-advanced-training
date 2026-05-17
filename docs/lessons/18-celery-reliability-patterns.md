# Lesson 18：任务幂等、重试、超时与限流

## 学习目标

完成本课后，你应该能够：

- 为 Celery task 设计业务幂等键，而不是依赖 Celery task id。
- 区分可重试错误、不可重试错误、poison message、人工补偿。
- 正确使用 retry/backoff/jitter、soft/hard time limit、rate limit、queue routing。
- 解释 retry storm、重复副作用、外部 API 超时、worker kill 后重投等生产事故链路。

## 关键问题

1. task 执行到一半 worker OOM，任务会不会重跑？重跑是否安全？
2. `self.retry()` 什么时候是修复，什么时候是事故放大器？
3. soft time limit 和 hard time limit 分别保护什么？
4. 发送邮件、扣款、发 Kafka event、更新 Mongo read model，幂等策略一样吗？

## 核心结论

Celery 可靠性不是 `autoretry_for=(Exception,)`。可靠 task 至少需要：

```text
业务幂等键
+ 状态机/唯一约束
+ 可分类异常
+ 有限 retry + backoff + jitter
+ time limit
+ 队列隔离
+ DLQ/补偿
+ 可观测字段
```

Celery 的投递语义通常应按 at-least-once 设计：任务可能执行 0 次、1 次或多次；你的业务必须能识别和处理重复。

## 1. 技术 ID 与业务幂等 ID

Celery task id：

```text
4b8d... task message 的技术 ID
```

业务幂等 ID：

```text
event_id=evt_123
payment_id=pay_123
idempotency_key=user42:create_order:abc
```

不要用 task id 作为业务幂等键。原因：

- 同一业务事件可能因为 outbox 重发而产生不同 task id。
- 手工补偿/重放时 task id 会变。
- Kafka/Redis Stream/Celery 三套系统需要同一个业务 event_id 串联。

## 2. 幂等表模式

```python
class ProcessedTask(models.Model):
    idempotency_key = models.CharField(max_length=160, unique=True)
    status = models.CharField(max_length=32)
    result = models.JSONField(null=True)
    created_at = models.DateTimeField(auto_now_add=True)
```

Task：

```python
@app.task(bind=True, acks_late=True, reject_on_worker_lost=True)
def send_order_email(self, payload: dict):
    key = f"send_email:{payload['event_id']}"
    try:
        with transaction.atomic():
            ProcessedTask.objects.create(idempotency_key=key, status="started")
    except IntegrityError:
        logger.info("task_duplicate_ignored", extra={"idempotency_key": key})
        return

    provider_message_id = email_client.send(
        to=payload["email"],
        template="order_created",
        idempotency_key=key,
    )

    ProcessedTask.objects.filter(idempotency_key=key).update(
        status="succeeded",
        result={"provider_message_id": provider_message_id},
    )
```

边界：如果 task 在 `ProcessedTask.create` 后、外部邮件发送前 crash，再次执行会被当重复跳过，导致邮件没发。这说明单表 started 幂等还不够。更好的状态机：

```text
pending → sending → sent
             ↘ failed_retryable
             ↘ failed_terminal
```

重复执行时根据状态决定是否继续，而不是简单忽略。

## 3. 外部 API 幂等

支付/邮件/第三方 API 必须携带 provider 支持的 idempotency key：

```python
payment_client.capture(
    payment_id=payment_id,
    amount_cents=amount,
    headers={"Idempotency-Key": f"capture:{payment_id}"},
    timeout=(2, 10),
)
```

如果 provider 不支持幂等：

- 尽量避免自动 retry。
- 先查询 provider 状态再决定补偿。
- 保留 reconciliation job。
- 需要人工审核队列。

## 4. 异常分类

```python
class RetryableDependencyError(Exception): ...
class TerminalBusinessError(Exception): ...
class PoisonMessageError(Exception): ...
```

策略：

| 错误 | 例子 | 动作 |
|---|---|---|
| retryable | HTTP 502、连接超时、DB deadlock | retry + backoff |
| terminal | 邮箱格式非法、订单不存在且确认不可恢复 | 标记失败，ack |
| poison | payload schema 错、代码 bug 反复失败 | DLQ/人工处理 |
| unknown | 未分类异常 | 少量 retry 后 DLQ |

不要 `autoretry_for=(Exception,)` 无脑重试所有异常。

## 5. Retry/backoff/jitter

```python
@app.task(
    bind=True,
    autoretry_for=(RetryableDependencyError,),
    retry_backoff=True,
    retry_backoff_max=300,
    retry_jitter=True,
    max_retries=5,
)
def publish_event(self, event_id: str):
    ...
```

为什么需要 jitter：避免大量任务同一秒重试，造成 retry storm。

重试日志：

```json
{
  "event": "celery_task_retry",
  "task_name": "orders.publish_event",
  "task_id": "...",
  "event_id": "evt_1",
  "retry": 3,
  "countdown_s": 42,
  "exception_type": "BrokerTimeout"
}
```

## 6. Soft/Hard time limit

```python
task_soft_time_limit = 240
task_time_limit = 300
```

- soft：抛 `SoftTimeLimitExceeded`，给 task 清理/标记状态机会。
- hard：强杀 worker 子进程，可能没有 finally。

处理：

```python
from celery.exceptions import SoftTimeLimitExceeded

@app.task(bind=True)
def generate_report(self, report_id: int):
    try:
        do_work(report_id)
    except SoftTimeLimitExceeded:
        mark_report_failed(report_id, reason="soft_timeout")
        raise
```

不要在 hard kill 后指望 finally 执行。

## 7. Rate limit 与队列隔离

```python
@app.task(rate_limit="10/m")
def call_payment_gateway(...): ...
```

Celery rate limit 是 worker 侧限制，不是全局强一致限流。多 worker/多队列下需要额外网关或 Redis 限流。

队列隔离：

```text
payment: 低并发、严格超时
email: 中并发、可延迟
read_model: 可扩容、幂等
heavy: 长任务，独立 worker
```

## 8. Poison message 与 DLQ

Celery 没有像 Kafka 那样天然 DLQ。可用业务表记录：

```python
class FailedTask(models.Model):
    task_name = models.CharField(max_length=200)
    idempotency_key = models.CharField(max_length=200)
    payload = models.JSONField()
    error_type = models.CharField(max_length=100)
    error_message = models.TextField()
    attempts = models.IntegerField()
    status = models.CharField(max_length=32, default="open")
```

超过 retry 上限：

```text
write failed_tasks
ack original task
alert human/channel
provide replay command
```

## 9. Runbook：Retry storm

```text
症状：Celery retry_total 暴涨，下游 API 5xx，队列 oldest age 上升

1. 证据
   - task_retry_total by task_name/exception_type
   - queue_depth / oldest_message_age
   - downstream status_code/latency
2. 判断
   - 是否所有任务同一 countdown？
   - 是否无 jitter？
   - 是否 retry 不可重试错误？
3. 止血
   - 暂停对应队列 worker
   - 降低 rate limit / 熔断外部调用
   - 把 poison payload 标记 DLQ
4. 修复
   - 异常分类
   - backoff+jitter
   - max_retries
   - provider idempotency key
5. 验证
   - retry rate 下降
   - downstream error 下降
   - oldest age 下降
```

## 10. 常见误区

- 用 task id 做业务幂等。
- 在 task 里持有完整 ORM model payload。
- 事务内 `.delay()`，worker 早于 commit 执行。
- 无限制 retry 外部 API。
- 所有任务共用 default 队列。

## 作业

为支付确认后的 `capture_payment`、`send_email`、`update_read_model` 三类 task 分别设计：幂等键、异常分类、retry、time limit、queue、DLQ、日志字段。

## 评估标准

- 能设计业务幂等键和状态机。
- 能区分 retryable/terminal/poison。
- 能说明 soft/hard time limit 的不同。
- 能写出 retry storm runbook。
