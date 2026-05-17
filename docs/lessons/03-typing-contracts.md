# Lesson 3：类型系统、契约与工程边界

## 学习目标

完成本课后，你应该能够：

- 理解 Python gradual typing 的价值边界：它是工程契约工具，不是运行时安全网。
- 在 Django 项目中区分 DTO、domain object、ORM model、serializer 的职责。
- 使用 `Protocol`、`TypedDict`、`dataclass`、Pydantic/DRF serializer 建立清晰边界。
- 解释为什么“全项目 any 化”会让类型系统失去价值。

## 关键问题

1. Python 类型标注在运行时是否自动校验？
2. Django ORM model 是否应该直接作为所有层的输入输出类型？
3. `Protocol` 解决什么问题？为什么比继承抽象类更轻？
4. 类型检查如何帮助 Celery/Kafka/Redis Stream 这类异步边界？

## 核心结论

Python 类型系统的生产价值不在“变成 Java”，而在于明确边界：

```text
HTTP JSON
  → serializer validation
  → command DTO
  → service/domain
  → repository protocol
  → ORM / external storage
  → event schema
```

类型标注要优先用在边界处：

- API 输入输出。
- service function 参数和返回值。
- repository interface。
- Celery task payload。
- Kafka/Redis Stream event schema。

## 1. Gradual typing 的定位

Python 类型标注默认不影响运行时：

```python
def add(a: int, b: int) -> int:
    return a + b

print(add("1", "2"))  # 运行时输出 '12'
```

需要静态工具：

```bash
python -m mypy src
python -m pyright
```

类型系统能发现：

- 明显参数类型错误。
- Optional 未处理。
- 返回值不符合契约。
- DTO 字段缺失。
- repository 实现不满足 interface。

不能发现：

- 数据库里真实数据违反业务规则。
- JSON runtime payload 未校验。
- 并发竞争。
- SQL 性能问题。
- Celery task 被重复投递。

## 2. Django 项目的四种对象

### 2.1 ORM Model

```python
class Order(models.Model):
    status = models.CharField(max_length=32)
    total_cents = models.IntegerField()
```

职责：

- 映射数据库表。
- 表达持久化字段和关系。
- 可承载少量与持久化强相关的方法。

不建议：

- 直接作为外部 API schema。
- 直接塞进 Celery task payload。
- 在 model method 里调用复杂外部 API。

### 2.2 Serializer / API Schema

```python
class CreateOrderSerializer(serializers.Serializer):
    sku = serializers.CharField()
    quantity = serializers.IntegerField(min_value=1)
```

职责：

- 校验 HTTP 输入。
- 输出 API 表示。
- 屏蔽内部字段。

### 2.3 Command DTO

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class CreateOrderCommand:
    user_id: int
    sku: str
    quantity: int
    request_id: str
```

职责：

- 把通过校验的输入传入 service。
- 不依赖 DRF，也不依赖 ORM。
- 适合测试和异步边界。

### 2.4 Event Schema

```python
from typing import Literal, TypedDict

class OrderCreatedEvent(TypedDict):
    event_id: str
    type: Literal["order.created"]
    order_id: int
    user_id: int
    occurred_at: str
```

职责：

- 作为 Redis Stream/Kafka/Celery payload 的稳定契约。
- 支持幂等消费和 replay。

## 3. `Protocol`：面向能力而不是继承树

Repository 协议：

```python
from typing import Protocol

class OrderRepository(Protocol):
    def get_for_update(self, order_id: int) -> "Order": ...
    def save(self, order: "Order") -> None: ...
```

Service 不依赖具体 ORM 实现：

```python
class OrderService:
    def __init__(self, repo: OrderRepository):
        self.repo = repo

    def cancel(self, order_id: int) -> None:
        order = self.repo.get_for_update(order_id)
        order.cancel()
        self.repo.save(order)
```

测试可用 fake：

```python
class FakeOrderRepo:
    def __init__(self, order):
        self.order = order

    def get_for_update(self, order_id: int):
        return self.order

    def save(self, order) -> None:
        self.order = order
```

`Protocol` 的价值：

- 不强迫实现类继承某个 base class。
- 更符合 Python duck typing。
- 让 service 层更容易测试。

## 4. 类型边界与 Celery task

错误示例：

```python
@app.task
def send_order_email(order: Order):
    ...
```

问题：

- ORM model 不能稳定序列化。
- task 执行时数据库状态可能已变化。
- 重试时对象快照不可靠。

更好的 payload：

```python
class SendOrderEmailPayload(TypedDict):
    order_id: int
    request_id: str
    event_id: str

@app.task(bind=True)
def send_order_email(self, payload: SendOrderEmailPayload) -> None:
    order_id = payload["order_id"]
    ...
```

异步边界只传：

- ID。
- 幂等 key。
- trace/request id。
- 必要快照字段。

## 5. 类型边界与 Kafka/Redis Stream

事件 schema 应该版本化：

```python
class OrderEventV1(TypedDict):
    schema_version: Literal[1]
    event_id: str
    event_type: Literal["order.created", "order.cancelled"]
    aggregate_id: str
    occurred_at: str
    payload: dict[str, object]
```

生产要求：

- `event_id` 用于 consumer dedup。
- `schema_version` 支持演进。
- `aggregate_id` 支持按业务键分区，例如 Kafka message key。
- `occurred_at` 与 broker timestamp 分开。

## 6. Django dynamic model 与类型工具的冲突

Django 的动态能力会让类型工具不容易推断：

```python
order.user.email
Order.objects.filter(status="pending")
```

建议：

- 核心 service 层使用明确 DTO 和 Protocol。
- ORM 查询集中在 repository/query module。
- 不追求 100% 类型覆盖；优先覆盖业务边界和异步边界。
- 对复杂 QuerySet 返回值写小的 projection DTO。

示例：

```python
@dataclass(frozen=True)
class PendingOrderRow:
    order_id: int
    user_id: int
    total_cents: int
```

## 7. 工具验证

建议开发依赖：

```bash
pip install mypy django-stubs djangorestframework-stubs
```

`mypy.ini` 起点：

```ini
[mypy]
python_version = 3.12
plugins = mypy_django_plugin.main
warn_return_any = true
warn_unused_ignores = true
disallow_untyped_defs = true
no_implicit_optional = true

[mypy.plugins.django-stubs]
django_settings_module = config.settings.test
```

运行：

```bash
python -m mypy .
```

## 8. 生产实践：类型策略分层

| 层 | 类型要求 | 原因 |
|---|---|---|
| views/serializers | 输入输出 schema 明确 | 外部边界 |
| service | `disallow_untyped_defs` | 业务核心 |
| repository | Protocol + DTO | 隔离 ORM |
| Celery tasks | TypedDict payload | 异步契约 |
| Kafka/Stream events | versioned schema | 跨服务契约 |
| migrations/admin | 可适度放宽 | Django 动态代码多 |

## 9. 常见误区

### 误区 1：类型标注会自动校验请求 JSON

不会。HTTP JSON 必须通过 serializer/Pydantic/手写 validation 校验。

### 误区 2：所有函数都写 `dict` 就算有类型

```python
def handle(payload: dict) -> dict:
    ...
```

这几乎没有契约价值。至少使用 `TypedDict` 或 dataclass。

### 误区 3：把 ORM model 当 DTO

ORM model 带数据库行为、lazy relation、manager、state，不适合作为跨层/跨进程 payload。

### 误区 4：为了 mypy 牺牲框架自然写法

类型系统服务于工程边界，不是目标本身。Django 动态层可以适度放宽，但核心业务层要严格。

## 10. 课堂讨论题

1. 一个 `CreateOrderSerializer` 校验后的数据，为什么不直接传 `serializer.validated_data` 到所有下游？
2. Celery task payload 里传 `order_id` 与传完整 order snapshot 各有什么取舍？
3. Kafka event 的 schema_version 应该放在哪里？为什么？
4. `Protocol` 与 ABC 在 Python 项目中的取舍是什么？

## 11. 作业

为订单创建流程设计三类类型：

1. API input serializer。
2. service command dataclass。
3. `order.created` event `TypedDict`。

要求：

- 包含 `request_id`、`event_id`、`schema_version`。
- 说明哪些字段来自 HTTP，哪些字段由服务端生成。
- 说明哪些字段可用于幂等。

## 12. 评估标准

- 能解释 gradual typing 的边界。
- 能区分 ORM model、serializer、DTO、event schema。
- 能使用 `Protocol` 定义 repository 契约。
- 能为 Celery/Kafka/Redis Stream 设计类型安全 payload。
