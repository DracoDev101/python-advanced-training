# Lab 8：库存扣减、行锁与 Outbox

## 目标

设计并验证一个并发安全的库存扣减流程。

## 错误实现

```python
def reserve_inventory(sku: str, quantity: int):
    inv = Inventory.objects.get(sku=sku)
    if inv.available < quantity:
        raise OutOfStock()
    inv.available -= quantity
    inv.save(update_fields=["available"])
```

## 正确方向

```python
from django.db import transaction


def reserve_inventory(sku: str, quantity: int):
    with transaction.atomic():
        inv = Inventory.objects.select_for_update().get(sku=sku)
        if inv.available < quantity:
            raise OutOfStock()
        inv.available -= quantity
        inv.save(update_fields=["available"])
```

## Outbox 要求

在同一个事务内写入：

```text
order row
inventory reservation row
outbox_event row
```

commit 后再触发 publisher。

## 并发测试思路

- 准备库存 `available=1`。
- 启动两个线程同时扣减 1。
- 期望只有一个成功。
- 最终库存不能小于 0。

## MySQL 证据命令

```sql
SHOW PROCESSLIST;
SHOW ENGINE INNODB STATUS\G
SELECT * FROM performance_schema.data_locks;
SELECT * FROM performance_schema.data_lock_waits;
```

## 通过标准

- 能解释为什么错误实现会超卖。
- 能说明 `select_for_update` 必须在事务内使用。
- 能设计 outbox 字段和 consumer 幂等 key。
- 能列出 deadlock/lock wait 第一证据。
