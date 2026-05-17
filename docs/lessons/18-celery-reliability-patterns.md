# Lesson 18：任务幂等、重试、超时与限流

## 学习目标

完成本课后，你应该能够：

- 解释 Celery task 为什么必须按 at-least-once 语义设计。
- 为外部 API、邮件、库存、读模型更新等不同任务设计幂等策略。
- 正确配置 retry/backoff/jitter、soft/hard time limit、rate limit、queue routing。
- 识别 retry storm、poison message、重复副作用、worker lost 等生产故障。

## 关键问题

1. task 重试后会不会重复扣款、重复发邮件、重复写 read model？
2. `self.retry()` 和抛异常有什么区别？
3. soft time limit 与 hard time limit 分别解决什么问题？
4. 限流应该放在 Celery、应用、Redis，还是外部 API client？

## 核心结论

Celery 可靠性不是“打开 autoretry”。可靠 task 必须同时满足：

```text
stable payload
+ business idempotency key
+ bounded retry/backoff/jitter
+ timeout budget
+ queue isolation
+ poison message escape hatch
+ observable attempts and failure reasons
```

默认假设：任务可能执行 0 次、1 次或多次；worker 可能在任意点崩溃；broker 可能重投；外部 API 可能成功但响应超时。

## 1. 幂等：以业务结果为中心

技术 task id 不是业务幂等 id。业务幂等要使用稳定业务键：

```text
send email: notification_id / template + recipient + business event_id
payment capture: payment_intent_id / provider idempotency key
read model update: event_id + aggregate_version
inventory reservation: order_id + sku unique constraint
webhook delivery: endpoint_id + event_id
```

示例：邮件发送去重：

```python
class NotificationDelivery(models.Model):
    notification_id = models.CharField(max_length=128, unique=True)
    recipient = models.EmailField()
    status = models.CharField(max_length=32)
    provider_message_id = models.CharField(max_length=128, null=True)
```

Task：

```python
@app.task(bind=True, acks_late=True)
def send_notification(self, notification_id: str):
    delivery = NotificationDelivery.objects.select_for_update().get(
        notification_id=notification_id
    )
    if delivery.status == "sent":
        return
    provider_id = email_client.send(..., idempotency_key=notification_id)
    delivery.status = "sent"
    delivery.provider_message_id = provider_id
    delivery.save(update_fields=["status", "provider_message_id"])
```

即使 task 重跑，也不会重复产生业务效果。

## 2. Retry：只重试可恢复错误

不要对所有异常重试。

可重试：

- 网络超时。
- 连接失败。
- 429 / rate limited。
- 部分 5xx。
- deadlock / lock wait timeout（有限重试）。

不可重试：

- payload schema 错误。
- 权限/认证失败。
- 业务状态非法。
- 404 永久不存在。

配置：

```python
@app.task(
    bind=True,
    autoretry_for=(TransientGatewayError,),
    retry_backoff=True,
    retry_backoff_max=300,
    retry_jitter=True,
    max_retries=8,
    acks_late=True,
)
def capture_payment(self, payment_id: str):
    ...
```

手动 retry 可以记录更细证据：

```python
try:
    call_provider()
except ProviderRateLimited as exc:
    logger.warning("payment_retry", extra={"payment_id": payment_id, "reason": "rate_limited", "retries": self.request.retries})
    raise self.retry(exc=exc, countdown=min(300, 2 ** self.request.retries))
```

## 3. Retry storm

事故模式：下游 API 挂了，所有 task 同时 retry，指数放大队列。

证据：

```text
task_retry_total 激增
queue_depth 上升
external_api_error_rate 上升
同一 provider 的 attempt 分布集中
```

缓解：

- jitter。
- provider 级 circuit breaker。
- queue 限流。
- 暂停非关键队列。
- DLQ/延迟重试队列。
- 全局最大尝试次数。

## 4. Time limit

```python
task_soft_time_limit = 240
task_time_limit = 300
```

soft limit 抛 `SoftTimeLimitExceeded`，task 可清理资源：

```python
from celery.exceptions import SoftTimeLimitExceeded

@app.task(bind=True, soft_time_limit=240, time_limit=300)
def generate_report(self, report_id: str):
    try:
        render_pdf(report_id)
    except SoftTimeLimitExceeded:
        mark_report_failed(report_id, reason="soft_timeout")
        raise
```

hard limit 会杀 worker 子进程，不能依赖 finally 执行。

原则：

- 外部 API timeout 必须小于 task soft time limit。
- task soft time limit 必须小于 hard time limit。
- worker termination grace period 必须大于正常任务完成时间，或任务可恢复。

## 5. Rate limit 与队列隔离

Celery rate limit：

```python
@app.task(rate_limit="100/m")
def call_partner_api(...):
    ...
```

但 provider 级限流常需要 Redis token bucket，因为多个 worker/服务实例共享配额。

队列隔离：

```python
task_routes = {
    "payments.tasks.capture_payment": {"queue": "critical"},
    "notifications.tasks.send_email": {"queue": "email"},
    "reports.tasks.generate_pdf": {"queue": "heavy"},
}
```

避免 PDF 生成压垮支付确认。

## 6. Poison message

poison message 是永远处理不了的消息：schema 错误、业务状态不可能、第三方永久拒绝。

策略：

```text
attempts <= N: retry with backoff
attempts > N: write failed state / DLQ / human review
```

Celery 中可记录失败表：

```python
class FailedTask(models.Model):
    task_id = models.CharField(max_length=128)
    task_name = models.CharField(max_length=255)
    business_key = models.CharField(max_length=255)
    reason = models.TextField()
    payload = models.JSONField()
    created_at = models.DateTimeField(auto_now_add=True)
```

## 7. 生产日志字段

```json
{
  "event": "celery_task_retry",
  "task_id": "uuid",
  "task_name": "payments.tasks.capture_payment",
  "queue": "critical",
  "business_key": "payment:123",
  "request_id": "req_1",
  "event_id": "evt_1",
  "attempt": 3,
  "max_retries": 8,
  "next_countdown_s": 16,
  "error_type": "ProviderTimeout"
}
```

## 8. Runbook：重复副作用

```text
症状：用户收到两封邮件 / 支付重复 capture
1. 查业务键
   notification_id / payment_id / event_id
2. 查 task 证据
   task_id、retries、worker、时间线
3. 查幂等记录
   unique constraint 是否存在
   provider idempotency key 是否传递
4. 判断
   task 重跑？provider timeout 后其实成功？consumer replay？
5. 修复
   增加唯一约束/幂等表
   修正 ack/retry 策略
   对已重复副作用做补偿
6. 验证
   重放同一 event/task 不产生第二次副作用
```

## 作业

为支付 capture、邮件发送、Mongo read model 更新分别设计 Celery 幂等方案，列出业务键、唯一约束、retry 策略、time limit、日志字段和失败补偿。

## 评估标准

- 能区分技术 task id 和业务幂等 id。
- 能判断哪些错误可重试。
- 能配置 backoff/jitter/time limit/rate limit。
- 能设计 poison message 的退出路径。
