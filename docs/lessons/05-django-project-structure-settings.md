# Lesson 5：Django 项目结构与 settings 分层

## 学习目标

完成本课后，你应该能够：

- 设计一个适合生产演进的 Django 项目目录，而不是把业务全部塞进 `views.py` 和 `models.py`。
- 区分 Django app、domain module、service、repository、query、task、event producer 的职责。
- 设计 `base/dev/test/prod` settings 分层，并解释哪些配置必须按入口区分：web、Celery、Daphne、consumer。
- 建立项目启动时的配置验证清单，避免 secret、DEBUG、连接池、broker URL 等问题进入生产。

## 关键问题

1. Django app 应该按“技术层”拆，还是按“业务能力”拆？
2. 为什么 `models.py` 里不应该承载复杂跨系统业务流程？
3. settings 分层只是为了好看，还是为了降低发布风险？
4. 同一个项目的 Gunicorn、Daphne、Celery worker、Kafka consumer 是否应该完全共享配置？

## 核心结论

生产级 Django 项目结构的目标不是“看起来 Clean Architecture”，而是让变化边界清楚：

```text
HTTP/API 变化 → serializers/views
业务流程变化 → services/workflows
持久化变化 → repositories/queries
异步执行变化 → tasks/consumers/events
部署环境变化 → settings/* + env
```

推荐按业务能力拆 app，再在 app 内部按职责拆模块：

```text
orders/
  models.py
  serializers.py
  views.py
  urls.py
  services.py
  repositories.py
  queries.py
  tasks.py
  events.py
  selectors.py
  tests/
```

## 1. 反例：所有逻辑都塞进 view

```python
class OrderView(APIView):
    def post(self, request):
        serializer = CreateOrderSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        order = Order.objects.create(
            user=request.user,
            sku=serializer.validated_data["sku"],
            quantity=serializer.validated_data["quantity"],
        )
        redis_client.delete(f"orders:{request.user.id}")
        send_email.delay(order.id)
        return Response({"id": order.id}, status=201)
```

问题：

- HTTP、业务、ORM、Redis、Celery 混在一起。
- 无法单独测试业务规则。
- `send_email.delay` 如果在事务提交前执行，worker 可能查不到订单。
- 缓存失效和任务投递失败没有一致性边界。
- 后续加 Kafka/Redis Stream outbox 时，view 会继续膨胀。

## 2. 推荐分层：薄 view，厚 service/workflow

### 2.1 API 层只做协议转换

```python
# orders/views.py
class OrderView(APIView):
    def post(self, request):
        serializer = CreateOrderSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        cmd = CreateOrderCommand(
            user_id=request.user.id,
            sku=serializer.validated_data["sku"],
            quantity=serializer.validated_data["quantity"],
            request_id=request.headers.get("X-Request-ID", ""),
        )
        result = create_order(cmd)
        return Response({"id": result.order_id}, status=201)
```

### 2.2 service 层承载业务流程

```python
# orders/services.py
from django.db import transaction


def create_order(cmd: CreateOrderCommand) -> CreateOrderResult:
    with transaction.atomic():
        order = Order.objects.create(
            user_id=cmd.user_id,
            sku=cmd.sku,
            quantity=cmd.quantity,
            status=Order.Status.PENDING,
        )
        event = build_order_created_event(order, request_id=cmd.request_id)
        OutboxEvent.objects.create(
            event_id=event["event_id"],
            event_type=event["event_type"],
            payload=event,
        )

        transaction.on_commit(lambda: publish_outbox_event.delay(event["event_id"]))

    return CreateOrderResult(order_id=order.id)
```

要点：

- DB 写入和 outbox 写入在同一事务。
- Celery 只在 commit 后投递。
- Celery payload 只传 `event_id`，worker 自己查 outbox。
- 未来切 Kafka/Redis Stream，不需要改 API 层。

## 3. app 拆分：按业务能力优先

推荐：

```text
orders/
payments/
inventory/
notifications/
accounts/
common/
```

不推荐一上来按技术拆：

```text
models/
views/
serializers/
services/
```

原因：

- 订单相关代码会分散在多个大目录里。
- 改一个业务流程需要跨技术目录查找。
- app 边界无法对应 migration、测试、权限和事件。

但也不要过度拆：

```text
order_create/
order_cancel/
order_pay/
```

过细 app 会导致 migration、权限、admin、import 关系变复杂。

## 4. 推荐目录模板

```text
config/
  settings/
    base.py
    dev.py
    test.py
    prod.py
  urls.py
  asgi.py
  wsgi.py
  celery.py

orders/
  __init__.py
  apps.py
  models.py
  admin.py
  urls.py
  serializers.py
  views.py
  services.py
  repositories.py
  queries.py
  events.py
  tasks.py
  consumers.py
  permissions.py
  tests/
    test_create_order_api.py
    test_create_order_service.py
    test_order_events.py

common/
  logging.py
  middleware.py
  observability.py
  idempotency.py
  time.py
```

### `repositories.py` 与 `queries.py` 的区别

- `repositories.py`：写路径、事务、锁、状态变更。
- `queries.py` / `selectors.py`：读路径、列表、搜索、projection。

示例：

```python
# orders/repositories.py
class OrderRepository:
    def get_for_update(self, order_id: int) -> Order:
        return Order.objects.select_for_update().get(id=order_id)

    def save(self, order: Order) -> None:
        order.save(update_fields=["status", "updated_at"])
```

```python
# orders/queries.py
def list_recent_orders(user_id: int) -> QuerySet[Order]:
    return (
        Order.objects
        .filter(user_id=user_id)
        .select_related("user")
        .order_by("-created_at")[:50]
    )
```

## 5. settings 分层

### 5.1 `base.py`：稳定公共配置

```python
# config/settings/base.py
INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "rest_framework",
    "orders",
]

MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "common.middleware.RequestIDMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
]

ROOT_URLCONF = "config.urls"
DEFAULT_AUTO_FIELD = "django.db.models.BigAutoField"
USE_TZ = True
TIME_ZONE = "UTC"
```

### 5.2 `prod.py`：生产安全配置

```python
# config/settings/prod.py
from .base import *
import os

DEBUG = False
SECRET_KEY = os.environ["DJANGO_SECRET_KEY"]
ALLOWED_HOSTS = os.environ["DJANGO_ALLOWED_HOSTS"].split(",")

SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True

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
            "init_command": "SET sql_mode='STRICT_TRANS_TABLES'",
        },
    }
}

CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.redis.RedisCache",
        "LOCATION": os.environ["REDIS_CACHE_URL"],
        "TIMEOUT": 300,
    }
}

CELERY_BROKER_URL = os.environ["CELERY_BROKER_URL"]
CELERY_RESULT_BACKEND = os.environ.get("CELERY_RESULT_BACKEND")
KAFKA_BOOTSTRAP_SERVERS = os.environ.get("KAFKA_BOOTSTRAP_SERVERS", "")
```

### 5.3 `test.py`：测试速度与隔离

```python
# config/settings/test.py
from .base import *

DEBUG = False
SECRET_KEY = "test-secret"
PASSWORD_HASHERS = ["django.contrib.auth.hashers.MD5PasswordHasher"]
EMAIL_BACKEND = "django.core.mail.backends.locmem.EmailBackend"

CELERY_TASK_ALWAYS_EAGER = True
CELERY_TASK_EAGER_PROPAGATES = True
```

注意：eager mode 只能测试 task 函数逻辑，不能覆盖 broker、ack、prefetch、worker crash。

## 6. 不同入口的配置差异

| 入口 | 命令 | 重点配置 |
|---|---|---|
| Web WSGI | `gunicorn config.wsgi:application` | DB connection age、worker count、timeout |
| Web ASGI | `daphne config.asgi:application` | channel layer、WebSocket timeout、Redis |
| Celery worker | `celery -A config worker` | queue、concurrency、prefetch、time limit |
| Celery beat | `celery -A config beat` | schedule storage、timezone |
| Kafka consumer | `python manage.py consume_events` | group id、offset、batch size |

不要假设一个 `.env` 适合所有入口。比如：

- Web 需要低 timeout，防止 worker 被占满。
- Celery 慢任务可以有更长 time limit，但必须隔离队列。
- Kafka consumer 需要独立 group id、commit 策略、batch size。
- Daphne 需要考虑长连接和 Redis channel layer。

## 7. 配置验证

### 7.1 Django deploy check

```bash
python manage.py check --deploy --settings=config.settings.prod
```

### 7.2 diffsettings

```bash
python manage.py diffsettings --settings=config.settings.prod
```

### 7.3 环境变量检查

```python
# common/config_checks.py
import os

REQUIRED_ENV = [
    "DJANGO_SECRET_KEY",
    "DJANGO_ALLOWED_HOSTS",
    "MYSQL_HOST",
    "MYSQL_DATABASE",
    "MYSQL_USER",
    "MYSQL_PASSWORD",
    "REDIS_CACHE_URL",
    "CELERY_BROKER_URL",
]


def check_required_env() -> list[str]:
    return [name for name in REQUIRED_ENV if not os.environ.get(name)]
```

在 CI 或容器启动前运行，避免启动到一半才失败。

## 8. 常见误区

### 误区 1：把 service 写成“换个名字的 view”

如果 service 仍然接收 `request`、返回 `Response`，它没有隔离协议层。

### 误区 2：repository 包装所有 ORM 方法

不需要为每个 `.filter()` 写无意义 wrapper。repository 应聚焦事务、锁、写路径和复杂查询边界。

### 误区 3：settings 里写复杂业务逻辑

settings 应是配置，不应 import model、连接 Redis、读取数据库。

### 误区 4：Celery task 直接调用 view/service 而不考虑事务

task 是异步入口，它没有 HTTP request 生命周期，也不应该依赖 request-local 状态。

## 9. 课堂讨论题

1. 订单取消逻辑应该放在 model method、service，还是 view？为什么？
2. 什么情况下你会新建一个 Django app？什么情况下只在现有 app 里加 module？
3. 为什么 Kafka consumer 不应该复用 HTTP view 的 serializer 作为唯一契约？
4. `prod.py` 中哪些配置变化需要发布前审批？

## 10. 作业

为 `Production Order Workflow` 设计项目目录，并说明：

- 哪些 app 是业务边界。
- 每个 app 中 `services.py`、`repositories.py`、`events.py` 的职责。
- Web、Celery、Daphne、Kafka consumer 的入口命令。
- 生产 settings 必需环境变量清单。

## 11. 评估标准

- 能设计按业务能力组织的 Django app 结构。
- 能解释 view/service/repository/event/task 的边界。
- 能写出安全的 settings 分层。
- 能列出不同运行入口的配置差异。
- 能用 `check --deploy` 和 `diffsettings` 验证配置。
