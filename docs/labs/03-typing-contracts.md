# Lab 3：为异步订单事件设计类型契约

## 目标

为订单创建流程建立三类契约：API input、service command、event payload。

## 要求

创建 `contracts.py`：

```python
from dataclasses import dataclass
from typing import Literal, TypedDict


@dataclass(frozen=True)
class CreateOrderCommand:
    user_id: int
    sku: str
    quantity: int
    request_id: str


class OrderCreatedEvent(TypedDict):
    schema_version: Literal[1]
    event_id: str
    event_type: Literal["order.created"]
    aggregate_id: str
    order_id: int
    user_id: int
    occurred_at: str
```

创建 `service.py`：

```python
from contracts import CreateOrderCommand, OrderCreatedEvent


def create_order(cmd: CreateOrderCommand) -> OrderCreatedEvent:
    return {
        "schema_version": 1,
        "event_id": "evt_1",
        "event_type": "order.created",
        "aggregate_id": f"order:123",
        "order_id": 123,
        "user_id": cmd.user_id,
        "occurred_at": "2026-01-01T00:00:00Z",
    }
```

## 验证

安装并运行：

```bash
pip install mypy
python -m mypy contracts.py service.py
```

## 变体实验

故意把 `schema_version` 改成字符串：

```python
"schema_version": "1"
```

观察 mypy 是否报错。

## 生产映射

回答：

1. 为什么 Celery/Kafka payload 不应该直接传 Django model？
2. `event_id` 和 `aggregate_id` 分别解决什么问题？
3. `schema_version` 如何支持事件演进？

## 通过标准

- 能通过 mypy 校验。
- 能解释 DTO 与 ORM model 的边界。
- 能说明异步事件字段的幂等用途。
