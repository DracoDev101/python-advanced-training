# Lab 22：Kafka Outbox/CDC 与幂等

## 目标

把本课主题落到 `Production Order Workflow`，要求写出可验证的生产设计，而不是只列 API。

## 必须输出

- 消息/task/event 字段设计。
- 幂等键与唯一约束。
- 失败重试策略。
- DLQ 或人工补偿路径。
- 关键日志字段。
- 关键 metrics。
- 第一批排障命令。

## 参考命令

```bash
celery -A config inspect active
celery -A config inspect reserved
celery -A config inspect scheduled
celery -A config inspect stats
celery -A config events
kafka-consumer-groups --bootstrap-server localhost:9092 --describe --group order-service
```

## 通过标准

- 说明 at-least-once 下如何避免重复副作用。
- 区分技术 ID 和业务幂等 ID。
- 有 oldest age / lag / runtime / failure rate 指标。
- 有 poison message 处理方案。
