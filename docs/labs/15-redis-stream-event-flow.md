# Lab 15：Redis Stream Consumer Group 与 Pending 恢复

## 目标

设计 `order.created` 的 Redis Stream 消费链路。

## 必须包含

- stream name：`order-events`
- group name：`order-read-model`
- consumer name 规则。
- event fields：`event_id/event_type/aggregate_id/payload/request_id`。
- 幂等表：`ProcessedEvent(event_id unique)`。
- DLQ stream：`order-events-dlq`。
- `XAUTOCLAIM` reclaim 策略。
- trim 策略。

## 命令

```bash
redis-cli XGROUP CREATE order-events order-read-model '$' MKSTREAM
redis-cli XADD order-events '*' event_id evt_1 event_type order.created order_id 123
redis-cli XREADGROUP GROUP order-read-model c1 COUNT 10 STREAMS order-events '>'
redis-cli XPENDING order-events order-read-model
redis-cli XAUTOCLAIM order-events order-read-model c2 60000 0-0 COUNT 10
redis-cli XACK order-events order-read-model <stream_id>
```

## 通过标准

- ack 在业务处理成功或 DLQ 后。
- 重复消息由 event_id 幂等处理。
- pending 堆积有 runbook。
- 能说明 Redis Stream 与 Kafka/Celery 边界。
