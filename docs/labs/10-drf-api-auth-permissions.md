# Lab 10：订单创建 API 契约与幂等

## 目标

设计 `POST /orders` 的 request/response/error schema 和 Idempotency-Key 行为。

## 输出

1. Request JSON schema。
2. Success response。
3. Error response。
4. Permission 规则。
5. `Idempotency-Key` 处理流程。
6. Audit log 字段。

## 场景

同一个用户用同一个 `Idempotency-Key` 请求两次：

- body 相同：返回第一次响应。
- body 不同：返回 `409 idempotency_key_conflict`。

## 通过标准

- 不暴露内部 ORM 字段。
- serializer validation 和 service validation 分离。
- 错误 code 稳定。
- 审计日志包含 request_id/user_id/action/reason。
