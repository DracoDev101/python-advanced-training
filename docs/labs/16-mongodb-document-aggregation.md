# Lab 16：MongoDB 订单读模型与 explain

## 目标

为订单详情页设计 MongoDB read model，并用 explain 验证查询计划。

## 任务

1. 设计订单读模型 document。
2. 设计列表索引：`user_id + status + created_at`。
3. 设计 TTL index 用于 webhook payload 保留。
4. 写 event consumer upsert 逻辑。
5. 写 explain 验证标准。

## Mongo 命令

```javascript
db.orders.createIndex({ user_id: 1, status: 1, created_at: -1, _id: 1 })
db.webhook_payloads.createIndex({ received_at: 1 }, { expireAfterSeconds: 2592000 })

db.orders.find({ user_id: 42, status: "paid" })
  .sort({ created_at: -1, _id: 1 })
  .limit(50)
  .explain("executionStats")
```

## 通过标准

- `winningPlan` 不应是 `COLLSCAN`。
- `totalDocsExamined` 应接近 `nReturned`。
- 文档大小和数组增长有上限策略。
- read model 有重建策略和幂等处理。
