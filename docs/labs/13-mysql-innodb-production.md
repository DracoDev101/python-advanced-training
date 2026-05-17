# Lab 13：MySQL/InnoDB 查询计划与锁证据

## 目标

为订单列表查询设计索引，并写出慢查询/锁等待排障证据链。

## 查询场景

```text
GET /orders?user_id=42&status=paid&cursor=...
```

要求按 `created_at DESC, id DESC` 做 cursor pagination。

## 任务

1. 写出 Django ORM 查询。
2. 写出生成 SQL。
3. 设计复合索引。
4. 写出 `EXPLAIN FORMAT=JSON` 命令。
5. 写出 lock wait / deadlock / metadata lock 证据命令。

## 参考命令

```sql
EXPLAIN FORMAT=JSON SELECT ...;
EXPLAIN ANALYZE SELECT ...;
SHOW PROCESSLIST;
SHOW ENGINE INNODB STATUS\G
SELECT * FROM performance_schema.metadata_locks;
SELECT * FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC LIMIT 10;
```

## 通过标准

- 不使用大 offset 分页。
- 索引顺序能解释 where/order。
- 能指出 `Using filesort`、`ALL`、rows 过大的含义。
- 能区分慢查询、锁等待、metadata lock。
