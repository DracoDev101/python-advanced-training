# Lesson 22：Kafka 与业务一致性：Outbox、CDC、幂等消费

## 学习目标

- 解释 DB 与 Kafka 双写失败窗口。
- 比较 application outbox publisher 与 Debezium CDC outbox。
- 设计 idempotent consumer、retry topic、DLQ、replay。
- 处理乱序、重复、schema 演进和 consumer lag/backpressure。

## 核心结论

Kafka 不能自动让数据库事务和消息发布成为一个事务。生产上常见两条路线：

```text
应用层 Outbox Publisher：Django 写 outbox，Celery/worker 发布 Kafka
CDC Outbox：Django 写 outbox，Debezium 读取 binlog 发布 Kafka
```

两者都默认 at-least-once；consumer 必须幂等。

## 1. Application outbox

优点：实现简单，应用可控。

缺点：publisher 自己维护、可能重复发布、要处理 stuck 状态。

```text
DB transaction writes outbox
publisher scans pending
producer sends Kafka
mark published
```

## 2. Debezium CDC outbox

流程：

```text
Django transaction writes outbox table
MySQL binlog records insert
Debezium reads binlog
Outbox SMT transforms row to Kafka event
Kafka consumers process
```

优点：publish 与 DB commit 顺序天然一致，不需要应用轮询 publisher。

风险/成本：

- Debezium/Kafka Connect 运维复杂。
- schema/SMT 配置需要治理。
- binlog retention、connector lag、snapshot 恢复。
- outbox 表清理策略。

## 3. Idempotent consumer

```python
class ProcessedKafkaEvent(models.Model):
    event_id = models.CharField(max_length=128, unique=True)
    topic = models.CharField(max_length=128)
    partition = models.IntegerField()
    offset = models.BigIntegerField()
    processed_at = models.DateTimeField(auto_now_add=True)
```

处理顺序：

```text
begin DB transaction
insert processed_event(event_id unique)
apply business change
commit DB
commit Kafka offset
```

如果 offset commit 前 crash，事件重放，unique(event_id) 让重复安全。

## 4. Retry topic / DLQ

不要让 poison message 永远卡住 partition。

策略：

```text
main topic: order-events
retry topic: order-events-retry-1m / 10m / 1h
dlq topic: order-events-dlq
```

DLQ 字段：

```json
{
  "original_topic": "order-events",
  "original_partition": 3,
  "original_offset": 9012,
  "event_id": "evt_1",
  "error_type": "SchemaValidationError",
  "error_message": "missing field payload.order_id",
  "failed_at": "...",
  "payload": {}
}
```

## 5. Replay

Replay 不是简单把 topic 从头读一遍。要回答：

- consumer 幂等是否覆盖全部 side effect？
- 是否要重建 read model 到新 collection/table？
- 是否会触发外部邮件/支付重复？
- 是否按时间窗口、event type、aggregate id 过滤？

读模型重建建议：

```text
create new read_model_v2
replay events into v2
compare counts/checksum
switch read path
keep old for rollback
```

## 6. 乱序与版本

同一 aggregate 用同一 Kafka key，可保证单 partition 顺序。但仍要处理：

- producer bug 发送旧版本。
- replay 与 live 同时写。
- 多 topic 合并。

事件中加入：

```json
{
  "aggregate_id": "order:123",
  "aggregate_version": 7
}
```

consumer 只应用版本递增事件：

```text
if event.version <= current.version: ignore duplicate/old
else apply and set version
```

## 7. Backpressure

consumer lag 上升时，不要盲目扩容：

- partition 数限制并行度。
- 下游 DB/Mongo 可能已经瓶颈。
- 扩容可能放大下游压力。

处理顺序：

```text
identify hot partition/key
measure process latency
measure downstream latency
optimize/DLQ poison
then scale consumers/partitions
```

## 8. Runbook：CDC connector lag

```text
症状：DB 已提交，Kafka 事件延迟

1. Debezium connector status
2. Kafka Connect task errors
3. binlog retention / position
4. outbox table insert rate
5. Kafka broker produce latency
6. consumer lag separate check
7. 修复 connector / broker / schema error
8. 验证 connector lag 和 topic produce 恢复
```

## 常见误区

- 认为 Kafka exactly-once 自动解决业务幂等。
- offset 当业务处理状态。
- DLQ 不包含原始 topic/partition/offset，无法 replay。
- replay 触发外部 side effect。
- CDC 后忘记 outbox 清理。

## 作业

设计订单事件 Kafka 一致性方案：application outbox 与 Debezium CDC 二选一，写出 outbox 表、Kafka key、event schema、consumer 幂等表、retry/DLQ、replay、lag runbook。

## 评估标准

- 能比较 application outbox 与 CDC outbox。
- 能设计 idempotent consumer。
- 能设计 retry topic/DLQ/replay。
- 能处理乱序、重复和 backpressure。
