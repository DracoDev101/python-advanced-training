# Lesson 12：测试体系、factory、fixtures 与契约测试

## 学习目标

完成本课后，你应该能够：

- 为 Django 项目建立分层测试策略：unit、service、ORM、API、contract、integration。
- 使用 pytest-django、factory_boy、freezegun、responses/respx 构建可维护测试。
- 避免 Celery eager mode、mock 过度、fixture 巨石化带来的误判。
- 为 ORM query count、事务并发、事件 schema、API 错误格式建立回归保护。

## 关键问题

1. 哪些逻辑应该不依赖 Django DB 也能单测？
2. API 测试是否需要覆盖所有 service 分支？
3. Celery eager mode 能证明真实 worker 语义吗？
4. 什么时候应该写 contract test，而不是只写 unit test？

## 核心结论

测试不是越接近端到端越好。生产级测试要按风险分层：

```text
domain unit: 快速验证规则
service test: 验证事务和状态变化
ORM/query test: 验证 SQL 数量和过滤权限
API test: 验证协议、认证、错误格式
contract test: 验证事件/API schema
integration smoke: 验证 Redis/MySQL/Celery/Kafka 等连接和关键路径
```

## 1. 测试金字塔在 Django 中的落地

| 层 | 例子 | 是否访问 DB | 速度 |
|---|---|---|---|
| domain unit | 状态机、金额计算 | 否 | 极快 |
| service | 创建订单、扣库存 | 是 | 中 |
| query | 列表权限、N+1 | 是 | 中 |
| API | POST /orders | 是 | 中慢 |
| contract | event schema | 可否 | 快 |
| integration | Celery worker + Redis | 是/外部 | 慢 |

原则：

- 业务规则尽量从 ORM/model 中抽出，便于无 DB 单测。
- DB 相关行为必须用真实 DB 测试，不要 mock ORM。
- 外部系统用 contract + 少量 integration smoke。

## 2. pytest-django 基础

安装：

```bash
pip install pytest pytest-django factory-boy freezegun responses
```

`pytest.ini`：

```ini
[pytest]
DJANGO_SETTINGS_MODULE = config.settings.test
python_files = tests.py test_*.py *_tests.py
addopts = -q --reuse-db
```

使用 DB：

```python
import pytest

@pytest.mark.django_db
def test_create_order():
    ...
```

事务测试：

```python
@pytest.mark.django_db(transaction=True)
def test_concurrent_reserve_inventory():
    ...
```

## 3. factory_boy：替代巨型 fixtures

```python
import factory
from orders.models import Order

class UserFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = "auth.User"

    username = factory.Sequence(lambda n: f"user{n}")

class OrderFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Order

    user = factory.SubFactory(UserFactory)
    status = "pending"
    total_cents = 1000
```

优点：

- 每个测试只创建需要的数据。
- 字段默认值集中。
- 比共享 JSON fixture 更易维护。

避免：

- 一个 `big_fixture.json` 装所有表。
- 测试之间依赖固定自增 ID。
- factory 默认创建过深对象图。

## 4. 时间测试：freezegun

```python
from freezegun import freeze_time

@freeze_time("2026-01-01T00:00:00Z")
def test_order_expiration():
    order = OrderFactory(created_at=timezone.now())
    assert order.expires_at.isoformat().startswith("2026-01-02")
```

时间相关代码必须可测试：

- 订单过期。
- retry backoff。
- token expiry。
- Redis TTL。

## 5. API 测试

```python
@pytest.mark.django_db
def test_create_order_api(api_client, user):
    api_client.force_authenticate(user=user)
    resp = api_client.post(
        "/api/orders",
        {"sku": "SKU-1", "quantity": 1},
        HTTP_IDEMPOTENCY_KEY="idem_1",
        HTTP_X_REQUEST_ID="req_1",
    )
    assert resp.status_code == 201
    assert resp.json()["data"]["id"]
    assert resp["X-Request-ID"] == "req_1"
```

API 测试关注：

- status code。
- response schema。
- auth/permission。
- idempotency。
- error code。

不要在 API 测试里断言过多内部实现细节。

## 6. Query count 测试

```python
@pytest.mark.django_db
def test_order_list_query_count(django_assert_num_queries, user):
    OrderFactory.create_batch(10, user=user)

    with django_assert_num_queries(2):
        rows = list_orders_for_api(user_id=user.id)
        list(rows)
```

用途：防止 serializer/模板引入 N+1。

注意：query count 不是越少越好，目标是稳定且合理。

## 7. 外部 HTTP mock

使用 `responses`：

```python
import responses

@responses.activate
def test_payment_gateway_timeout():
    responses.add(
        responses.POST,
        "https://pay.example.com/charges",
        body=responses.ConnectionError("timeout"),
    )

    with pytest.raises(PaymentGatewayUnavailable):
        create_charge(...)
```

要求：

- 覆盖 timeout。
- 覆盖 5xx retry。
- 覆盖 4xx 不重试。
- 验证 Idempotency-Key header。

## 8. Celery 测试边界

Eager mode：

```python
CELERY_TASK_ALWAYS_EAGER = True
```

能测试：

- task 函数逻辑。
- payload schema。
- retry 分支的一部分。

不能测试：

- broker 连接。
- worker ack。
- prefetch。
- visibility timeout。
- worker crash 后重投。
- gevent/prefork pool 行为。

因此需要两类测试：

```text
unit/functional: eager mode
integration smoke: real broker + worker + one happy path
```

## 9. Event contract test

Kafka/Redis Stream/Celery payload：

```python
from jsonschema import validate

ORDER_CREATED_SCHEMA = {
    "type": "object",
    "required": ["schema_version", "event_id", "event_type", "aggregate_id", "occurred_at", "payload"],
    "properties": {
        "schema_version": {"const": 1},
        "event_type": {"const": "order.created"},
        "event_id": {"type": "string"},
        "aggregate_id": {"type": "string"},
        "occurred_at": {"type": "string"},
        "payload": {"type": "object"},
    },
}


def test_order_created_event_contract():
    event = build_order_created_event(order_id=123, user_id=42)
    validate(event, ORDER_CREATED_SCHEMA)
```

这是跨服务契约的底线。改 event 字段必须改 schema 和 consumer。

## 10. 并发测试

并发事务测试要用真实数据库，并标记 transaction：

```python
@pytest.mark.django_db(transaction=True)
def test_concurrent_inventory_reservation():
    # 使用 threading.Barrier 控制两个线程同时进入扣减
    ...
```

SQLite 不适合验证 MySQL/InnoDB 锁语义。需要 MySQL/PostgreSQL 测试环境。

## 11. CI 分层

建议：

```text
PR fast checks:
  ruff/format/type/unit/service/API(query count)

Nightly or pre-release:
  MySQL integration
  Redis integration
  Celery worker smoke
  Kafka consumer smoke
  migration rehearsal
```

命令示例：

```bash
python -m pytest -q
python -m pytest -q --durations=20
python manage.py check --deploy --settings=config.settings.prod
python manage.py migrate --plan
```

## 12. 常见误区

### 误区 1：mock ORM

ORM 行为、事务、锁、query count 都应该用真实 DB 测试。

### 误区 2：只测 happy path

生产事故常来自 timeout、duplicate request、retry、permission denied、old schema。

### 误区 3：Celery eager mode 当真实 worker

它不是。需要明确写在测试策略里。

### 误区 4：snapshot API 响应但不校验语义

snapshot 容易固化噪声。关键字段和错误 code 要显式断言。

## 13. 课堂讨论题

1. 哪些测试必须使用真实 MySQL，而不能用 SQLite？
2. API 测试和 service 测试如何避免重复？
3. Kafka event schema 改字段时，contract test 应该如何失败？
4. 为什么 query count 是性能回归测试，而不是功能测试？

## 14. 作业

为订单创建流程设计测试矩阵：

- domain unit
- service DB test
- API auth/permission test
- idempotency test
- query count test
- Celery task eager test
- event contract test
- MySQL transaction integration test

每类写出测试目标、是否访问 DB、关键断言。

## 15. 评估标准

- 能设计分层测试策略。
- 能使用 factory 替代巨型 fixture。
- 能说明 Celery eager mode 的边界。
- 能为 query count、event schema、事务并发写回归测试。
- 能设计 PR fast checks 与 integration checks。
