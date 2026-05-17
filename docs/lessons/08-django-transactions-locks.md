# Lesson 8：事务、锁与一致性边界

## 学习目标

完成本课后，你应该能够：

- 解释 Django autocommit、`transaction.atomic()`、savepoint 的行为。
- 正确使用 `select_for_update()` 保护并发状态变更。
- 区分 MySQL isolation level、gap lock、deadlock、lock wait timeout 的现象。
- 设计 `transaction.on_commit()`、outbox、幂等 key，避免 DB 与 Celery/Kafka/Redis Stream 双写问题。

## 关键问题

1. Django 默认是不是每个请求一个事务？
2. `atomic()` 能否自动防止所有并发问题？
3. 为什么在事务内直接发 Celery task 有风险？
4. MySQL deadlock 是不是一定代表代码错了？如何重试？

## 核心结论

事务只保护数据库内的一组读写，不自动保护外部系统：

```text
DB transaction ✅
Redis write ❌
Celery publish ❌
Kafka publish ❌
HTTP call ❌
```

生产一致性要明确边界：

```text
transaction.atomic
→ row lock / invariant check
→ write business rows
→ write outbox rows
→ commit
→ on_commit trigger async publisher
→ consumer idempotency
```

## 1. Django autocommit

Django 默认 autocommit：每条 SQL 如果不在显式事务中，会自动提交。

```python
order.status = "paid"
order.save(update_fields=["status"])
```

这条 update 执行后即提交。

显式事务：

```python
from django.db import transaction

with transaction.atomic():
    order.status = "paid"
    order.save(update_fields=["status"])
    Payment.objects.create(order=order, amount_cents=1000)
```

如果 block 内异常抛出，事务回滚。

## 2. savepoint 与嵌套 atomic

```python
with transaction.atomic():
    create_order()
    try:
        with transaction.atomic():
            create_optional_audit()
    except AuditError:
        pass
    finalize_order()
```

内层 `atomic()` 默认创建 savepoint。内层失败可以回滚到 savepoint，外层继续。

注意：不要滥用深层嵌套事务。它会增加理解成本，也可能延长锁持有时间。

## 3. 并发扣库存：错误示例

```python
def reserve_inventory(sku: str, quantity: int):
    inv = Inventory.objects.get(sku=sku)
    if inv.available < quantity:
        raise OutOfStock()
    inv.available -= quantity
    inv.save(update_fields=["available"])
```

两个请求并发：

```text
T1 read available=1
T2 read available=1
T1 save available=0
T2 save available=0
```

结果：两个请求都认为成功，库存被超卖。

## 4. 使用 `select_for_update()`

```python
from django.db import transaction


def reserve_inventory(sku: str, quantity: int):
    with transaction.atomic():
        inv = (
            Inventory.objects
            .select_for_update()
            .get(sku=sku)
        )
        if inv.available < quantity:
            raise OutOfStock()
        inv.available -= quantity
        inv.save(update_fields=["available"])
```

`select_for_update()` 会锁住选中的行直到事务结束。

生产注意：

- 必须在事务内使用。
- 锁持有时间越短越好。
- 事务内不要调用外部 API。
- 锁顺序必须一致，降低 deadlock。

## 5. MySQL isolation 与 gap lock

InnoDB 常见隔离级别：

- READ COMMITTED
- REPEATABLE READ（MySQL 默认常见）

在 REPEATABLE READ 下，某些范围查询 + `FOR UPDATE` 可能产生 next-key lock/gap lock，锁住不存在的范围，影响插入。

示例：

```sql
SELECT * FROM inventory
WHERE sku BETWEEN 'A' AND 'Z'
FOR UPDATE;
```

可能锁住范围，导致其他事务插入符合范围的新 sku 被阻塞。

Django 代码里看不到 gap lock，必须通过 MySQL 证据确认：

```sql
SHOW PROCESSLIST;
SHOW ENGINE INNODB STATUS\G
SELECT * FROM performance_schema.data_locks;
```

## 6. deadlock 与重试

deadlock 示例：

```text
T1 locks order 1 → wants order 2
T2 locks order 2 → wants order 1
```

数据库会选择一个事务回滚，返回 deadlock error。

这不一定表示数据库坏了。高并发下 deadlock 是需要被设计和重试的失败模式。

策略：

- 统一锁顺序，例如按 id 升序锁。
- 缩短事务。
- 避免事务内外部 I/O。
- 对可重试事务做有限重试 + jitter。

伪代码：

```python
import random
import time
from django.db import OperationalError, transaction


def with_deadlock_retry(fn, attempts=3):
    for i in range(attempts):
        try:
            with transaction.atomic():
                return fn()
        except OperationalError as exc:
            if "Deadlock" not in str(exc) or i == attempts - 1:
                raise
            time.sleep(0.05 * (2 ** i) + random.random() * 0.02)
```

实际项目中要按数据库驱动错误码判断，不要只靠字符串。

## 7. `transaction.on_commit()`

错误示例：

```python
with transaction.atomic():
    order = Order.objects.create(...)
    send_order_email.delay(order.id)
```

风险：worker 可能在事务提交前执行，查不到订单或看到旧状态。

修复：

```python
with transaction.atomic():
    order = Order.objects.create(...)
    transaction.on_commit(lambda: send_order_email.delay(order.id))
```

但这仍然没有解决“commit 成功但 broker publish 失败”的双写问题。

## 8. Outbox Pattern

更可靠的做法：

```python
with transaction.atomic():
    order = Order.objects.create(...)
    OutboxEvent.objects.create(
        event_id=event_id,
        event_type="order.created",
        aggregate_id=f"order:{order.id}",
        payload=payload,
        status="pending",
    )
    transaction.on_commit(lambda: publish_outbox.delay(event_id))
```

publisher task：

```python
def publish_outbox(event_id: str):
    event = OutboxEvent.objects.get(event_id=event_id)
    if event.status == "published":
        return
    publish_to_kafka_or_stream(event.payload)
    event.status = "published"
    event.save(update_fields=["status"])
```

更成熟做法：后台 publisher 扫描 pending outbox，避免只依赖 `on_commit` 触发。

一致性语义：

- 业务 row 和 outbox row 原子提交。
- 消息发布可以重试。
- consumer 必须幂等，因为可能重复发布/消费。

## 9. 事务内禁止事项

事务内不要做：

- 外部 HTTP API。
- Redis/Kafka 长时间阻塞 publish。
- 发送邮件。
- 调用 Celery `.get()` 等待结果。
- 大量计算或文件处理。

原因：会延长锁持有时间，放大 lock wait 和 deadlock。

## 10. 证据采集：锁等待现场

MySQL：

```sql
SHOW PROCESSLIST;
SHOW ENGINE INNODB STATUS\G
SELECT * FROM performance_schema.data_locks;
SELECT * FROM performance_schema.data_lock_waits;
```

Django 日志至少包含：

```json
{
  "event": "inventory_reserve_failed",
  "request_id": "req_1",
  "sku": "SKU-1",
  "quantity": 1,
  "exception_type": "OperationalError",
  "db_error_code": 1213
}
```

Celery/Kafka/outbox 证据：

```text
outbox pending count
oldest pending age
publish retry count
consumer dedup hit count
```

## 11. 常见误区

### 误区 1：`atomic()` 等于并发安全

`atomic()` 只是事务边界。并发不变量还需要锁、唯一约束、条件 update 或幂等设计。

### 误区 2：所有读都加 `select_for_update()`

锁越多越安全是错的。锁会降低并发、制造等待和 deadlock。

### 误区 3：`on_commit()` 等于 outbox

`on_commit()` 只保证回调在 commit 后执行，不保证 broker publish 成功后可恢复。

### 误区 4：忽略唯一约束

很多幂等场景应该让数据库唯一约束兜底：

```python
class IdempotencyRecord(models.Model):
    key = models.CharField(max_length=128, unique=True)
```

## 12. 课堂讨论题

1. 库存扣减应该用 row lock、条件 update，还是乐观锁？各自适合什么场景？
2. 为什么事务内调用外部支付 API 危险？
3. `on_commit` 与 outbox 的边界是什么？
4. Kafka consumer 重放同一事件时，数据库如何保证幂等？

## 13. 作业

设计一个订单支付确认流程：

```text
payment callback
→ mark payment succeeded
→ mark order paid
→ reserve/confirm inventory
→ publish order.paid event
```

要求：

- 写出事务边界。
- 写出需要锁的表/行。
- 写出唯一约束或幂等 key。
- 写出 outbox event 字段。
- 写出 deadlock/lock wait 的第一证据命令。

## 14. 评估标准

- 能解释 autocommit、atomic、savepoint。
- 能正确使用 `select_for_update()`。
- 能说明 MySQL gap lock/deadlock 的证据来源。
- 能设计 `on_commit` + outbox + 幂等 consumer。
- 能列出事务内禁止做的操作。
