# Lesson 13：MySQL/InnoDB 生产实践

## 学习目标

完成本课后，你应该能够：

- 解释 InnoDB clustered index、secondary index、covering index 对 Django 查询的影响。
- 用 `EXPLAIN` / `EXPLAIN ANALYZE` / slow query log / Performance Schema 拿到慢 SQL 证据。
- 识别 MySQL 在事务隔离、gap lock、next-key lock、metadata lock、online DDL 上的生产风险。
- 为 Django 项目设计 MySQL 连接池、字符集、时区、索引、迁移和排障基线。

## 关键问题

1. 为什么 InnoDB 主键设计会影响所有 secondary index 的体积？
2. 为什么 “where 字段都建了索引” 不等于查询会快？
3. Django `select_for_update()` 在 MySQL REPEATABLE READ 下可能锁住什么？
4. MySQL 慢查询、锁等待、连接耗尽分别应该看哪些证据？

## 核心结论

MySQL 是生产系统的状态核心。Django ORM 最终会变成 SQL，而 SQL 最终会落到 InnoDB 的索引、锁和 Buffer Pool 上。

排查 MySQL 问题时必须沿着这条链路：

```text
Django ORM
→ generated SQL
→ EXPLAIN plan
→ index/cardinality/rows/Extra
→ runtime evidence: slow log / performance_schema / processlist
→ InnoDB evidence: lock wait / deadlock / buffer pool / redo / replica lag
```

不要只说“数据库慢”。必须说清楚是：

- 扫描行数过大。
- 索引无法用于排序/过滤。
- 锁等待。
- metadata lock。
- 连接池耗尽。
- IO/Buffer Pool 压力。
- replica lag。

## 1. Django 使用 MySQL 的基础配置

生产配置示例：

```python
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.mysql",
        "HOST": os.environ["MYSQL_HOST"],
        "PORT": os.environ.get("MYSQL_PORT", "3306"),
        "NAME": os.environ["MYSQL_DATABASE"],
        "USER": os.environ["MYSQL_USER"],
        "PASSWORD": os.environ["MYSQL_PASSWORD"],
        "CONN_MAX_AGE": 60,
        "OPTIONS": {
            "charset": "utf8mb4",
            "init_command": "SET sql_mode='STRICT_TRANS_TABLES', time_zone='+00:00'",
        },
    }
}
```

关键点：

- `utf8mb4`：不要用 MySQL 旧 `utf8`，它不是完整 UTF-8。
- `STRICT_TRANS_TABLES`：避免截断/隐式转换悄悄吞数据。
- UTC：应用层和 DB 层统一时间语义。
- `CONN_MAX_AGE`：连接复用能降低建立连接成本，但会增加常驻连接数。

连接数估算：

```text
web replicas × gunicorn workers × threads
+ celery replicas × worker processes
+ daphne/asgi sync bridge connections
+ kafka/stream consumers
+ management jobs
≤ MySQL max_connections 的安全阈值
```

## 2. InnoDB clustered index

InnoDB 表数据按主键组织。主键就是 clustered index。

如果表：

```sql
CREATE TABLE orders_order (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  user_id BIGINT NOT NULL,
  status VARCHAR(32) NOT NULL,
  created_at DATETIME(6) NOT NULL,
  total_cents INT NOT NULL,
  KEY idx_user_created (user_id, created_at)
) ENGINE=InnoDB;
```

数据行存储在主键 B+Tree 的叶子节点。secondary index `idx_user_created` 的叶子节点存的是：

```text
(user_id, created_at, primary_key id)
```

这意味着：

- 主键越大，所有 secondary index 越大。
- secondary index 命中后，如果需要的列不在索引里，还要回表按主键查完整行。
- UUID v4 随机主键会导致 B+Tree 分裂和写放大；如果必须用 UUID，考虑 UUIDv7/ULID 或单独 surrogate id。

## 3. Covering index：避免回表

查询：

```sql
SELECT id, status, created_at
FROM orders_order
WHERE user_id = 42
ORDER BY created_at DESC
LIMIT 20;
```

索引：

```python
models.Index(fields=["user_id", "-created_at", "status"], name="order_user_created_status_idx")
```

如果查询需要的列都在索引里，可能出现：

```text
Extra: Using index
```

这叫 covering index，避免回表。

但不要滥用“把所有列都塞索引”：

- 索引越大，写入越慢。
- Buffer Pool 命中率下降。
- DDL 构建索引更慢。
- 每次 update 相关列都要维护索引。

索引设计要围绕高频查询路径，而不是字段清单。

## 4. 最左前缀与排序

索引：

```sql
KEY idx_status_created (status, created_at)
```

适合：

```sql
WHERE status = 'paid' ORDER BY created_at DESC
```

不一定适合：

```sql
WHERE created_at > '2026-01-01' ORDER BY status
```

复合索引设计原则：

```text
等值过滤字段 → 范围字段/排序字段 → 覆盖字段
```

但要结合 cardinality。低选择性字段如 `status` 单独索引价值有限，但和 `created_at`、`user_id` 组合可能有效。

## 5. EXPLAIN 证据

Django 中拿 SQL：

```python
qs = Order.objects.filter(status="paid").order_by("-created_at")[:50]
print(str(qs.query))
```

MySQL：

```sql
EXPLAIN FORMAT=JSON
SELECT id, user_id, status, created_at
FROM orders_order
WHERE status = 'paid'
ORDER BY created_at DESC
LIMIT 50;
```

MySQL 8 可用：

```sql
EXPLAIN ANALYZE
SELECT ...;
```

关注：

| 字段 | 问题信号 |
|---|---|
| type | `ALL` 全表扫、`index` 全索引扫 |
| possible_keys | 候选索引是否存在 |
| key | 实际使用索引 |
| rows | 估计扫描行数是否过大 |
| filtered | 过滤后比例是否很低 |
| Extra | `Using temporary`、`Using filesort` |

典型问题：

```text
key: NULL
rows: 32000000
Extra: Using where; Using filesort
```

这不是“SQL 慢”四个字能解决的。要回到 where/order/group 的索引路径。

## 6. 隐式类型转换：索引失效杀手

字段是 varchar：

```sql
phone VARCHAR(32)
```

错误查询：

```sql
SELECT * FROM users_user WHERE phone = 13800138000;
```

MySQL 可能把列做隐式转换，导致索引失效。应传字符串：

```sql
SELECT * FROM users_user WHERE phone = '13800138000';
```

Django 中要注意 serializer 类型，不要把业务字符串 ID 转成 int。

## 7. 锁：row lock、gap lock、next-key lock

### 7.1 行锁

```python
with transaction.atomic():
    order = Order.objects.select_for_update().get(id=order_id)
```

如果按主键等值命中，通常锁住目标行。

### 7.2 范围锁风险

```python
with transaction.atomic():
    rows = (
        Order.objects
        .select_for_update()
        .filter(status="pending", created_at__lt=cutoff)
        [:100]
    )
```

在 MySQL REPEATABLE READ 下，范围查询可能产生 next-key lock，锁住索引记录和间隙。风险：其他事务插入/更新落在范围内的记录被阻塞。

降低风险：

- 用高选择性索引缩小范围。
- 分批按主键处理。
- 使用 `skip_locked=True` 做 worker 抢任务时避免互相等待：

```python
Job.objects.select_for_update(skip_locked=True).filter(status="pending")[:100]
```

注意：`skip_locked` 会跳过被锁行，适合任务队列，不适合必须完整扫描的业务。

## 8. Deadlock 证据

应用看到：

```text
OperationalError: (1213, 'Deadlock found when trying to get lock; try restarting transaction')
```

第一证据：

```sql
SHOW ENGINE INNODB STATUS\G
```

查找：

```text
LATEST DETECTED DEADLOCK
```

里面会有：

- 两个事务持有什么锁。
- 等待什么锁。
- SQL 是什么。
- InnoDB 回滚了哪个事务。

工程处理：

- 统一锁顺序。
- 缩短事务。
- 对可重试操作做有限 retry + jitter。
- 把外部 I/O 移出事务。
- 用唯一约束/状态机减少手工锁。

## 9. Metadata lock：DDL 发布事故常客

场景：

```text
T1: 一个长事务 select/update 表 orders_order 未提交
T2: ALTER TABLE orders_order ADD INDEX ... 等 metadata lock
T3..Tn: 后续普通查询被 ALTER 阻塞
```

现象：大量请求突然卡住，`SHOW PROCESSLIST` 看到：

```text
Waiting for table metadata lock
```

证据：

```sql
SHOW PROCESSLIST;
SELECT * FROM performance_schema.metadata_locks;
```

发布策略：

- DDL 前确认长事务。
- 使用 online schema change 工具，如 gh-ost / pt-online-schema-change（根据团队规范）。
- 低峰执行，设置超时和 kill 策略。
- 大表索引先在 staging 真实量级演练。

## 10. Slow query log 与 Performance Schema

启用 slow log 后关注：

```sql
SHOW VARIABLES LIKE 'slow_query_log';
SHOW VARIABLES LIKE 'long_query_time';
```

查询 digest：

```sql
SELECT DIGEST_TEXT, COUNT_STAR, AVG_TIMER_WAIT, SUM_ROWS_EXAMINED, SUM_ROWS_SENT
FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;
```

这比单条慢 SQL 更能发现系统性问题：

- 某类 SQL 调用次数巨大。
- 平均不慢但总耗时高。
- rows_examined 远大于 rows_sent。

## 11. Django 与 MySQL 的常见坑

### 11.1 `icontains` 导致索引不可用

```python
Order.objects.filter(note__icontains="abc")
```

通常无法用普通 BTree 索引高效支持。可选：

- 限制功能。
- 前缀搜索用 `startswith`。
- 单独搜索系统。
- MySQL FULLTEXT（有边界）。

### 11.2 `order_by('?')`

```python
Order.objects.order_by('?')[:10]
```

大表灾难。它可能让数据库对大量行随机排序。

### 11.3 大 offset 分页

```sql
LIMIT 50 OFFSET 1000000
```

数据库仍要跳过大量行。改用 keyset/cursor pagination：

```sql
WHERE (created_at, id) < (?, ?)
ORDER BY created_at DESC, id DESC
LIMIT 50
```

### 11.4 字符集/排序规则不一致

跨表 join 字符集/collation 不一致可能导致隐式转换和索引问题。

## 12. 生产 Runbook：慢查询

```text
症状：订单列表 P95 从 120ms 到 2s

1. 确认范围
   - 哪个 endpoint / request_id / user_id / time window
2. 拿 SQL
   - APM DB span / slow log / sampled query log
3. 看计划
   - EXPLAIN FORMAT=JSON
   - EXPLAIN ANALYZE（可控环境）
4. 看运行证据
   - rows_examined vs rows_sent
   - performance_schema digest
   - processlist 是否锁等待
5. 修复
   - 改 query / 加复合索引 / 改分页 / 拆读模型
6. 验证
   - staging EXPLAIN
   - query count test
   - P95 / slow log / rows_examined 下降
```

## 13. 课堂讨论题

1. 为什么 BIGINT 自增主键会影响 secondary index？
2. `status` 字段低选择性，什么时候仍然适合作为复合索引第一列？
3. `select_for_update(skip_locked=True)` 适合哪些场景，不适合哪些场景？
4. metadata lock 事故中，为什么普通 SELECT 也会被 DDL 阻塞？

## 14. 作业

为订单列表设计 MySQL 查询优化方案：

```text
GET /orders?status=paid&cursor=...
```

要求：

- Django ORM 查询。
- 生成 SQL。
- 复合索引设计。
- EXPLAIN 关注字段。
- 大 offset 替代方案。
- 慢查询 runbook。

## 15. 评估标准

- 能解释 InnoDB clustered/secondary/covering index。
- 能读 EXPLAIN 核心字段并给出下一步动作。
- 能识别 gap lock、deadlock、metadata lock 证据。
- 能把 Django ORM 查询优化落到 MySQL 索引和 SQL 计划。
