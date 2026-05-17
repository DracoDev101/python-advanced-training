# Lesson 21：Kafka 事件系统与 Python Consumer

## 学习目标

- 理解 topic、partition、offset、consumer group、rebalance 的真实语义。
- 设计 Python consumer 的 poll/process/commit 循环。
- 判断 ordering、manual commit、batch size、max poll interval、consumer lag 的生产影响。
- 区分 `aiokafka` 与 `confluent-kafka` 的使用边界。

## 核心结论

Kafka 是分区日志，不是普通队列。可靠消费的核心是：

```text
message key 决定分区与局部顺序
consumer group 决定并行消费
offset commit 决定重放窗口
业务幂等决定重复是否安全
lag 决定背压和延迟
```

## 1. Topic / Partition / Offset

```text
topic order-events
  partition 0: offset 0,1,2...
  partition 1: offset 0,1,2...
```

同一个 partition 内有序，跨 partition 无全局顺序。订单事件通常用：

```text
key = order_id / aggregate_id
```

这样同一订单进入同一 partition，保证单订单内顺序。

## 2. Consumer group

```text
group order-read-model
  consumer A owns partition 0,1
  consumer B owns partition 2,3
```

同一个 group 内，每个 partition 同时只给一个 consumer。扩容 consumer 超过 partition 数不会增加吞吐。

## 3. Rebalance

触发：

- consumer 加入/退出。
- 心跳超时。
- `max.poll.interval.ms` 超时。
- partition 增加。

风险：rebalance 期间消费暂停；未提交 offset 的消息会被新 owner 重放。

Python consumer 如果处理太慢，可能超过 `max.poll.interval.ms`，触发 rebalance。

## 4. Manual commit

伪代码：

```python
consumer = Consumer({...,
    "enable.auto.commit": False,
    "group.id": "order-read-model",
})

while True:
    msg = consumer.poll(1.0)
    if msg is None:
        continue
    event = decode(msg.value())
    try:
        process_idempotently(event)
    except RetryableError:
        continue  # 不提交，稍后重放；注意会阻塞该 partition 后续消息
    except PoisonError:
        write_dlq(event, msg)
        consumer.commit(message=msg)
    else:
        consumer.commit(message=msg)
```

提交 offset 前 crash：消息重放。提交 offset 后 crash：消息不会重放。因此必须先处理成功，再 commit。

## 5. Batch processing

批量提高吞吐，但扩大重放窗口：

```text
poll 500 messages
process 300 succeeded
crash before commit
→ 300 messages replay
```

策略：

- 幂等处理。
- 按 partition 顺序 commit。
- batch size 与处理耗时匹配 `max.poll.interval.ms`。
- 长处理转交 Celery，consumer 只做轻量验证和 enqueue？要小心又引入双写。

## 6. Lag

```bash
kafka-consumer-groups --bootstrap-server localhost:9092 --describe --group order-read-model
```

字段：

```text
CURRENT-OFFSET
LOG-END-OFFSET
LAG
CONSUMER-ID
HOST
CLIENT-ID
```

lag 高不一定是 Kafka 问题，可能是 MongoDB/MySQL/外部 API 慢。

## 7. Schema

事件必须有 schema_version：

```json
{
  "schema_version": 1,
  "event_id": "evt_1",
  "event_type": "order.created",
  "aggregate_id": "order:123",
  "occurred_at": "...",
  "payload": {}
}
```

兼容性：

- backward compatible：新增 optional field。
- breaking：改名/删除/类型变化，需要版本升级和 consumer 双读。

## 8. aiokafka vs confluent-kafka

| 客户端 | 优点 | 注意 |
|---|---|---|
| confluent-kafka | librdkafka，高性能，成熟 | callback/配置较底层 |
| aiokafka | asyncio 友好 | event loop 阻塞风险，需要 async 栈一致 |

Django sync 项目里，consumer 通常是独立进程，不必强行 async。

## 9. Runbook：Kafka lag 上升

```text
1. 看 consumer group lag by partition
2. 看 consumer logs：processing latency/error/rebalance
3. 看下游：DB/Mongo/Redis latency
4. 判断热点 partition：是否某个 key 特别热
5. 判断 poison：是否卡在某个 offset
6. 处理：扩 partition/consumer、DLQ poison、优化下游、限流 producer
7. 验证：lag slope 变负，oldest event age 下降
```

## 作业

为 `order-events` 设计 Kafka topic：partition 数、message key、event schema、consumer group、commit 策略、lag 告警、poison DLQ。

## 评估标准

- 能解释 partition/offset/group/rebalance。
- 能设计手动 commit 的安全顺序。
- 能用 lag 判断背压。
- 能设计 schema 兼容演进。
