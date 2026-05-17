# Lesson 7：ORM 查询、N+1 与 SQL 证据

## 学习目标

完成本课后，你应该能够：

- 解释 QuerySet lazy evaluation 的触发时机。
- 用 `select_related`、`prefetch_related` 修复典型 N+1。
- 读懂 Django 生成的 SQL，并用 `EXPLAIN` 判断索引是否生效。
- 区分 ORM 代码“看起来优雅”和 SQL 执行计划“真的可接受”。

## 关键问题

1. QuerySet 什么时候真正执行 SQL？
2. 为什么模板或 serializer 里访问 relation 会造成 N+1？
3. `select_related` 和 `prefetch_related` 的区别是什么？
4. 慢查询优化为什么必须看真实 SQL 和 EXPLAIN？

## 核心结论

Django ORM 是 SQL 生成器，不是性能保护层。生产优化要回到证据：

```text
ORM code
→ generated SQL
→ query count
→ EXPLAIN plan
→ rows examined
→ index/filter/order/group evidence
```

不要凭“用了 ORM 所以安全”或“加了索引所以快”做判断。

## 1. QuerySet lazy evaluation

```python
qs = Order.objects.filter(status="pending")
```

此时通常还没执行 SQL。触发执行的操作包括：

```python
list(qs)
len(qs)
bool(qs)
for obj in qs:
    ...
qs[0]
repr(qs)
```

危险示例：

```python
logger.info("pending orders: %s", qs)
```

某些情况下 `repr(qs)` 会触发查询，日志语句也可能影响性能。

## 2. N+1 问题

模型：

```python
class Order(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    status = models.CharField(max_length=32)

class OrderItem(models.Model):
    order = models.ForeignKey(Order, related_name="items", on_delete=models.CASCADE)
    sku = models.CharField(max_length=64)
```

错误代码：

```python
orders = Order.objects.filter(status="paid")[:100]
for order in orders:
    print(order.user.email)
```

SQL 形态：

```text
1 query: select orders
100 queries: select user where id = ?
```

修复：

```python
orders = (
    Order.objects
    .filter(status="paid")
    .select_related("user")[:100]
)
```

`select_related` 用 SQL join，适合 ForeignKey / OneToOne。

对于 reverse/many relation：

```python
orders = (
    Order.objects
    .filter(status="paid")
    .prefetch_related("items")[:100]
)
```

`prefetch_related` 通常发两条 SQL，再在 Python 侧组装，适合 many-to-many、reverse FK。

## 3. Serializer N+1

DRF serializer 常见陷阱：

```python
class OrderSerializer(serializers.ModelSerializer):
    user_email = serializers.CharField(source="user.email")

    class Meta:
        model = Order
        fields = ["id", "status", "user_email"]
```

如果 view：

```python
orders = Order.objects.filter(status="paid")
return Response(OrderSerializer(orders, many=True).data)
```

serializer 访问 `user.email` 时会触发 N+1。

修复不在 serializer，而在 query：

```python
orders = Order.objects.filter(status="paid").select_related("user")
```

原则：serializer 不应该偷偷决定查询性能。列表 API 要有专门 query/selector。

## 4. 观察 SQL

开发环境可用：

```python
from django.db import connection

before = len(connection.queries)
list(Order.objects.filter(status="paid")[:10])
after = len(connection.queries)
print("queries", after - before)
print(connection.queries[-1]["sql"])
```

测试中可用：

```python
from django.test import TestCase

class OrderQueryTest(TestCase):
    def test_order_list_query_count(self):
        with self.assertNumQueries(2):
            list_orders_for_api(user_id=1)
```

生产不要依赖 `connection.queries`，它需要 debug cursor，可能带来开销。生产用：

- 数据库 slow query log。
- APM/OpenTelemetry DB span。
- 采样 SQL 日志。
- ProxySQL / MySQL performance_schema。

## 5. EXPLAIN：以 MySQL 为例

拿到 SQL 后运行：

```sql
EXPLAIN SELECT id, user_id, status, created_at
FROM orders_order
WHERE status = 'paid'
ORDER BY created_at DESC
LIMIT 50;
```

关注字段：

| 字段 | 含义 | 风险 |
|---|---|---|
| type | join/access 类型 | `ALL` 可能全表扫 |
| key | 使用的索引 | NULL 表示未用索引 |
| rows | 估计扫描行数 | 远大于返回行数有风险 |
| Extra | 额外操作 | `Using filesort`、`Using temporary` |

为上面查询设计索引：

```python
class Order(models.Model):
    status = models.CharField(max_length=32)
    created_at = models.DateTimeField()

    class Meta:
        indexes = [
            models.Index(fields=["status", "-created_at"], name="order_status_created_idx"),
        ]
```

注意：索引字段顺序必须匹配过滤和排序路径。不是“每个字段单独加索引”就行。

## 6. annotate、aggregate 与 Subquery

统计每个订单 item 数：

```python
from django.db.models import Count

orders = Order.objects.annotate(item_count=Count("items"))
```

风险：

- join 后行数膨胀。
- group by 可能使用 temporary/filesort。
- 与 pagination 组合时要验证 SQL。

复杂查询可以用 `Subquery`：

```python
from django.db.models import OuterRef, Subquery

latest_payment = (
    Payment.objects
    .filter(order_id=OuterRef("pk"))
    .order_by("-created_at")
    .values("status")[:1]
)

orders = Order.objects.annotate(latest_payment_status=Subquery(latest_payment))
```

但不要为了“ORM 纯度”写出不可读 SQL。必要时可以：

- 写 raw SQL。
- 创建数据库 view。
- 做读模型表。
- 用 MongoDB/Elasticsearch 做特殊读路径。

## 7. 大批量遍历

危险：

```python
for order in Order.objects.filter(status="paid"):
    process(order)
```

如果结果很大，会把大量 model instances 缓存在 QuerySet 内部。

改法：

```python
for order in Order.objects.filter(status="paid").iterator(chunk_size=1000):
    process(order)
```

批量更新：

```python
Order.objects.filter(status="expired").update(status="closed")
```

注意：`update()` 不会调用 model `save()` 和 signals。

## 8. 常见误区

### 误区 1：只在本地小数据量验证

本地 100 行数据看不出索引选择、filesort、临时表、锁等待。

### 误区 2：把所有 relation 都 prefetch

过度 prefetch 会导致内存上涨和无用 SQL。只 prefetch 当前 API 真正需要的关系。

### 误区 3：分页前做复杂 Python 过滤

```python
orders = [o for o in Order.objects.all() if o.is_visible_to(user)]
```

这会把过滤放到应用层，破坏数据库分页和索引。

### 误区 4：看到 `EXPLAIN key` 非空就认为没问题

还要看 rows、filtered、Extra，以及真实执行耗时。

## 9. 课堂讨论题

1. `select_related` 为什么不能替代所有 `prefetch_related`？
2. SerializerMethodField 里查询数据库有什么风险？
3. 列表 API 的 query count 应该如何写测试？
4. 什么情况下你会放弃 ORM，写 raw SQL 或读模型？

## 10. 作业

设计一个订单列表 API 查询，要求返回：

- order id
- status
- user email
- item count
- latest payment status

要求：

- 写出 ORM 查询。
- 写出预期 SQL 数量。
- 写出 MySQL 索引设计。
- 写出需要 EXPLAIN 验证的 SQL。

## 11. 评估标准

- 能解释 QuerySet lazy evaluation。
- 能识别并修复 N+1。
- 能用 query count 测试保护列表 API。
- 能读 EXPLAIN 的核心字段。
- 能设计符合过滤/排序路径的复合索引。
