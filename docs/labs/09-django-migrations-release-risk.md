# Lab 9：Expand/Contract Migration 设计

## 目标

为 `orders_order.external_id` 设计安全迁移，最终达到 `NOT NULL UNIQUE`。

## 任务

写出 4 步发布：

1. 添加 nullable column。
2. 新代码 dual write / read fallback。
3. backfill 历史数据。
4. 添加 unique/not null 约束并清理 fallback。

## 必须提供

```bash
python manage.py migrate --plan
python manage.py sqlmigrate orders <migration_id>
python manage.py showmigrations orders
```

MySQL 证据：

```sql
SHOW CREATE TABLE orders_order\G
SHOW INDEX FROM orders_order;
SELECT COUNT(*) FROM orders_order WHERE external_id IS NULL;
```

## 通过标准

- 不直接在大表上添加 `NOT NULL default`。
- backfill 可暂停、可恢复、可限速。
- 考虑旧 web/Celery/consumer 混跑窗口。
