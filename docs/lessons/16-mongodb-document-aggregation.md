# Lesson 16：MongoDB 文档模型与聚合查询

## 学习目标

完成本课后，你应该能够：

- 以访问模式为中心设计 MongoDB document model，而不是把关系模型原样搬过去。
- 判断 embed vs reference、read model、materialized view 的取舍。
- 设计 compound index、partial index、TTL index，并用 `explain()` 验证查询计划。
- 在 Django 系统中把 MongoDB 作为读模型/文档存储/事件投影，而不是强行 ORM 化。

## 关键问题

1. MongoDB 文档模型什么时候比关系表更合适？
2. 为什么 “一个业务对象一个巨大文档” 也可能是错误设计？
3. Aggregation pipeline 慢时怎么定位是哪一 stage 慢？
4. Django 项目里如何同时使用 MySQL 作为 source of truth、MongoDB 作为 read model？

## 核心结论

MongoDB 的核心不是“没有 schema”，而是文档聚合边界：

```text
一起读取、一起变更、生命周期一致 → 倾向 embed
独立增长、独立查询、生命周期不同 → 倾向 reference
```

在 Django 生产系统中，MongoDB 常见定位：

- 复杂读模型：订单详情聚合视图。
- 半结构化文档：审计上下文、外部 webhook payload。
- 事件投影：Kafka/Redis Stream consumer materialize read model。

MySQL/PostgreSQL 仍可作为强一致 source of truth。

## 1. Document modeling：围绕访问模式

订单读模型示例：

```json
{
  "_id": "order:123",
  "order_id": 123,
  "user_id": 42,
  "status": "paid",
  "total_cents": 12000,
  "created_at": "2026-05-17T12:00:00Z",
  "items": [
    {"sku": "SKU-1", "name": "Keyboard", "qty": 1, "price_cents": 8000},
    {"sku": "SKU-2", "name": "Mouse", "qty": 1, "price_cents": 4000}
  ],
  "payment": {
    "status": "succeeded",
    "provider": "stripe",
    "paid_at": "2026-05-17T12:01:00Z"
  },
  "updated_at": "2026-05-17T12:01:02Z"
}
```

适合：订单详情页一次读取所有信息。

不适合：频繁单独更新 `items` 中海量元素，或需要跨订单聚合复杂财务报表。

## 2. Embed vs Reference

### 2.1 Embed

适合：

- 子对象数量有上限。
- 总是随父对象读取。
- 生命周期依附父对象。
- 更新冲突可控。

订单 item 快照适合 embed，因为订单创建后 item 快照基本稳定。

### 2.2 Reference

适合：

- 子对象数量无上限。
- 子对象独立查询和更新。
- 多父对象共享。
- 需要独立权限/生命周期。

例如用户地址簿、商品主数据，不应完全 embed 到所有订单里作为唯一来源。但订单里可以 embed 下单时快照。

## 3. 文档大小与数组增长

MongoDB 单文档大小有上限（常见 16MB）。即使没到上限，大数组也会导致：

- 更新成本高。
- 网络传输大。
- 内存占用高。
- `$push` 热点文档冲突。

危险设计：把用户所有订单都 embed 到一个 user document。

```json
{
  "user_id": 42,
  "orders": [ ... thousands/millions ... ]
}
```

更好：一个订单一个读模型文档，按 `user_id + created_at` 建索引查询列表。

## 4. Django + MongoDB 的集成边界

建议不要把 MongoDB 强行塞进 Django ORM。使用 PyMongo/Motor 封装 repository：

```python
from pymongo import MongoClient

class OrderReadModelRepository:
    def __init__(self, client: MongoClient):
        self.collection = client["read_models"]["orders"]

    def upsert_order(self, doc: dict) -> None:
        self.collection.update_one(
            {"_id": f"order:{doc['order_id']}"},
            {"$set": doc},
            upsert=True,
        )

    def get_order(self, order_id: int) -> dict | None:
        return self.collection.find_one({"_id": f"order:{order_id}"})
```

Django service 不应直接到处调用 Mongo collection。通过 repository 隔离：

- 便于测试。
- 便于重建读模型。
- 便于切换索引和集合结构。

## 5. 索引设计

订单列表：

```javascript
db.orders.createIndex({ user_id: 1, created_at: -1, _id: 1 })
```

查询：

```javascript
db.orders.find({ user_id: 42 })
  .sort({ created_at: -1, _id: 1 })
  .limit(50)
```

状态筛选：

```javascript
db.orders.createIndex({ user_id: 1, status: 1, created_at: -1 })
```

TTL index：

```javascript
db.webhook_payloads.createIndex({ received_at: 1 }, { expireAfterSeconds: 2592000 })
```

Partial index：

```javascript
db.orders.createIndex(
  { user_id: 1, updated_at: -1 },
  { partialFilterExpression: { status: "pending" } }
)
```

不要为每个字段单独建索引。MongoDB 同样需要围绕查询模式设计 compound index。

## 6. explain 证据

```javascript
db.orders.find({ user_id: 42, status: "paid" })
  .sort({ created_at: -1 })
  .limit(50)
  .explain("executionStats")
```

关注：

| 字段 | 问题信号 |
|---|---|
| winningPlan.stage | `COLLSCAN` |
| totalDocsExamined | 远大于返回数量 |
| totalKeysExamined | 索引扫描过大 |
| executionTimeMillis | 耗时高 |
| hasSortStage | 内存排序风险 |

理想：

```text
IXSCAN → FETCH
nReturned = 50
totalDocsExamined 接近 50
totalKeysExamined 接近 50
```

## 7. Aggregation pipeline

示例：按日统计订单金额：

```javascript
db.orders.aggregate([
  { $match: { status: "paid", created_at: { $gte: ISODate("2026-01-01") } } },
  { $group: {
      _id: { day: { $dateTrunc: { date: "$created_at", unit: "day" } } },
      total_cents: { $sum: "$total_cents" },
      count: { $sum: 1 }
  }},
  { $sort: { "_id.day": 1 } }
])
```

优化原则：

- `$match` 尽量靠前，并命中索引。
- `$project` 提前裁剪大字段。
- 避免对大集合无索引 `$sort`。
- 大聚合考虑离线/预聚合。

explain：

```javascript
db.orders.explain("executionStats").aggregate([...])
```

## 8. Read concern / Write concern / Read preference

写关注：

```javascript
{ writeConcern: { w: "majority", j: true } }
```

读关注：

```javascript
{ readConcern: { level: "majority" } }
```

读偏好：

```text
primary
primaryPreferred
secondary
secondaryPreferred
nearest
```

生产取舍：

- 强一致读：primary + majority。
- 可接受延迟读：secondaryPreferred，但要考虑 replica lag。
- 读模型本身可能就是 eventually consistent，不要在 API 里假装强一致。

## 9. 从事件构建 MongoDB read model

事件：

```json
{
  "event_id": "evt_1",
  "event_type": "order.paid",
  "aggregate_id": "order:123",
  "occurred_at": "2026-05-17T12:01:00Z",
  "payload": {"order_id": 123, "status": "paid"}
}
```

consumer：

```python
def apply_order_event(event: dict):
    event_id = event["event_id"]
    with mysql_transaction_for_idempotency(event_id) as first_time:
        if not first_time:
            return
        if event["event_type"] == "order.paid":
            mongo.orders.update_one(
                {"_id": f"order:{event['payload']['order_id']}"},
                {
                    "$set": {
                        "status": "paid",
                        "payment.paid_at": event["occurred_at"],
                        "updated_at": now_iso(),
                    }
                },
                upsert=True,
            )
```

幂等记录可以存在 MySQL，也可以存在 MongoDB unique index：

```javascript
db.processed_events.createIndex({ event_id: 1 }, { unique: true })
```

要考虑乱序事件：

- event version / sequence。
- occurred_at 不是严格顺序保证。
- 对同一 aggregate 使用 Kafka key 保证单 partition 顺序，Redis Stream 单 stream 基本有序但多 publisher/多 stream 仍需业务版本。

## 10. MongoDB 慢查询 Runbook

```text
症状：订单详情 API 使用 Mongo read model 后 P95 上升

1. 应用证据
   - mongo_latency_ms、collection、operation、query_shape
2. Mongo 证据
   - profiler / slow query log
   - db.currentOp()
   - explain("executionStats")
3. 判断
   - COLLSCAN？
   - totalDocsExamined 过大？
   - hasSortStage？
   - 返回文档太大？
   - secondary lag？
4. 修复
   - compound index
   - projection 裁剪字段
   - 拆文档/限制数组
   - 预聚合/read model 重建
5. 验证
   - nReturned vs docsExamined 接近
   - executionTimeMillis 降低
   - API P95 恢复
```

命令：

```javascript
db.setProfilingLevel(1, { slowms: 100 })
db.system.profile.find().sort({ ts: -1 }).limit(5).pretty()
db.currentOp({ active: true })
```

## 11. 常见误区

### 误区 1：MongoDB 不需要 schema

MongoDB 不强制表 schema，但应用仍需要 schema 契约、版本和校验。否则 read model 演进会失控。

### 误区 2：把关系 join 全部搬到应用层

如果每个 API 都要查 5 个 collection 再拼装，MongoDB 并没有简化读路径。应构建面向页面/API 的 read model。

### 误区 3：无脑 embed

无限增长数组、热点文档、高频局部更新都会让 embed 变成瓶颈。

### 误区 4：读 secondary 默认安全

secondary 可能有复制延迟。对刚写后读的路径，必须明确一致性要求。

## 12. 课堂讨论题

1. 订单 item 快照为什么适合 embed，而商品主数据不适合作为唯一 embed？
2. MongoDB read model 和 MySQL source of truth 之间如何处理延迟？
3. `COLLSCAN` 一定是事故吗？什么情况下可以接受？
4. Kafka consumer 更新 MongoDB read model 时如何处理重复和乱序？

## 13. 作业

为 `Production Order Workflow` 设计 MongoDB 订单读模型：

- document shape。
- 列表查询索引。
- 详情查询索引。
- event consumer upsert 逻辑。
- 幂等表/unique index。
- explain 验证标准。
- read model rebuild 策略。

## 14. 评估标准

- 能围绕访问模式设计 document model。
- 能解释 embed/reference 的边界。
- 能设计 compound/partial/TTL index。
- 能用 explain 定位 COLLSCAN、排序和扫描过大问题。
- 能把 MongoDB 放进 Django + Kafka/Redis Stream 的 read model 架构中。
