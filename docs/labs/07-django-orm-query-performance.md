# Lab 7：N+1、query count 与 EXPLAIN

## 目标

识别订单列表 API 的 N+1，并用 query count 和 EXPLAIN 保护修复。

## 场景

订单列表需要返回：

- order id
- status
- user email
- item count

## 任务

1. 写出一个会触发 N+1 的 ORM 查询。
2. 用 `select_related("user")` 修复 user 查询。
3. 用 `prefetch_related("items")` 或 `annotate(Count("items"))` 处理 item count。
4. 写测试限制 query count。
5. 写 MySQL EXPLAIN 命令。

## 测试骨架

```python
from django.test import TestCase

class OrderListQueryTest(TestCase):
    def test_query_count(self):
        with self.assertNumQueries(2):
            list_orders_for_api(user_id=1)
```

## EXPLAIN

```sql
EXPLAIN SELECT id, user_id, status, created_at
FROM orders_order
WHERE status = 'paid'
ORDER BY created_at DESC
LIMIT 50;
```

## 通过标准

- 能说明每条 SQL 从哪里来。
- 能解释 `select_related` 与 `prefetch_related` 区别。
- 能设计符合 `WHERE + ORDER BY` 的复合索引。
