# Lesson 4：import、配置与项目启动链路

## 学习目标

完成本课后，你应该能够：

- 解释 Python import cache、module side effect、circular import 的生产影响。
- 画出 Django、Gunicorn、Daphne、Celery worker 的启动链路差异。
- 设计 `settings/base.py`、`dev.py`、`prod.py`、`test.py` 的配置分层。
- 识别 `AppConfig.ready()`、模块级初始化、monkey patch 时机带来的风险。

## 关键问题

1. 为什么“import 一个模块”可能已经连接数据库、注册 signal 或启动线程？
2. 为什么同一份 Django settings 在 web 和 Celery worker 中可能表现不同？
3. `gevent.monkey.patch_all()` 为什么必须非常早执行？
4. `gunicorn --preload` 会改变哪些初始化时机？

## 核心结论

Python 后端的启动链路是生产稳定性的第一道边界。很多线上问题不是业务代码逻辑错，而是“代码在错误的时间执行”：

```text
import time side effect
→ settings 未加载完成
→ AppConfig.ready 重复注册
→ gunicorn preload 后 fork
→ DB/Redis connection 被复制
→ Celery worker import task 触发外部调用
→ gevent monkey patch 太晚
```

配置和启动必须遵循两条原则：

1. import 阶段不做外部 I/O。
2. 不同入口的初始化路径必须显式、可测试。

## 1. Python import cache 与 side effect

当执行：

```python
import orders.services
```

Python 会：

1. 查 `sys.modules` 是否已有缓存。
2. 找到模块文件。
3. 执行模块顶层代码。
4. 把模块对象放入 `sys.modules`。

危险在第 3 步：模块顶层代码会被执行。

错误示例：

```python
# orders/clients.py
import redis

redis_client = redis.Redis.from_url("redis://localhost:6379")
redis_client.ping()  # import 时就进行外部 I/O
```

风险：

- Django 启动时 Redis 不可用，整个 web 起不来。
- Celery autodiscover tasks 时触发连接。
- 单元测试 import 变慢或依赖外部服务。
- gunicorn preload 后连接对象被 fork 复制。

更好的写法：

```python
from functools import lru_cache
import redis
from django.conf import settings

@lru_cache(maxsize=1)
def get_redis_client() -> redis.Redis:
    return redis.Redis.from_url(settings.REDIS_URL)
```

连接验证放到 health check 或 startup check，而不是 import 阶段。

## 2. Circular import 的根因通常是分层错

典型错误：

```python
# orders/models.py
from orders.services import price_order

# orders/services.py
from orders.models import Order
```

这不是简单“换个 import 位置”就算修复。通常说明：

- model 层依赖 service 层。
- service 层又依赖 model 层。
- 业务规则和持久化边界混在一起。

改法：

```text
orders/
  models.py          # ORM model
  repositories.py    # ORM query/write
  services.py        # use repository, no model import from model side
  types.py           # dataclass/TypedDict shared contracts
```

让 `types.py` 放轻量类型，减少循环依赖。

## 3. Django settings 分层

推荐结构：

```text
config/settings/
  __init__.py
  base.py
  dev.py
  test.py
  prod.py
```

`base.py`：共同配置，不含环境特有 secret。

```python
from pathlib import Path
import os

BASE_DIR = Path(__file__).resolve().parents[2]

INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "rest_framework",
    "orders",
]

TIME_ZONE = "UTC"
USE_TZ = True

DEFAULT_AUTO_FIELD = "django.db.models.BigAutoField"
```

`prod.py`：生产安全配置。

```python
from .base import *

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
        "NAME": os.environ["MYSQL_DATABASE"],
        "USER": os.environ["MYSQL_USER"],
        "PASSWORD": os.environ["MYSQL_PASSWORD"],
        "CONN_MAX_AGE": 60,
        "OPTIONS": {
            "charset": "utf8mb4",
        },
    }
}
```

`test.py`：测试隔离。

```python
from .base import *

DEBUG = False
PASSWORD_HASHERS = ["django.contrib.auth.hashers.MD5PasswordHasher"]
CELERY_TASK_ALWAYS_EAGER = True
CELERY_TASK_EAGER_PROPAGATES = True
```

注意：`CELERY_TASK_ALWAYS_EAGER` 只适合部分单测。它不能模拟真实 broker、worker ack、prefetch、retry 的全部语义。

## 4. 启动链路对比

### 4.1 Django management command

```bash
python manage.py check
```

链路：

```text
manage.py
→ set DJANGO_SETTINGS_MODULE
→ django.setup()
→ populate INSTALLED_APPS
→ AppConfig.ready()
→ command.handle()
```

### 4.2 Gunicorn WSGI

```bash
gunicorn config.wsgi:application -w 4
```

链路：

```text
gunicorn master
→ import config.wsgi
→ django.setup()
→ fork workers
→ workers accept requests
```

如果使用 `--preload`：

```text
master imports Django before fork
→ memory pages shared copy-on-write
→ any opened connection/global mutable object may be inherited by workers
```

`--preload` 可能降低内存，但要避免 import 阶段创建 DB/Redis/socket/thread。

### 4.3 Daphne ASGI

```bash
daphne config.asgi:application
```

链路：

```text
daphne process
→ import config.asgi
→ django.setup()
→ ASGI application handles http/websocket/lifespan
```

Daphne 面向 ASGI，通常用于 Channels/WebSocket。它的连接生命周期与短 HTTP 请求不同，要考虑 heartbeat、backpressure、channel layer。

### 4.4 Celery worker

```bash
celery -A config worker -l info
```

链路：

```text
celery command
→ import config.celery app
→ set DJANGO_SETTINGS_MODULE
→ django.setup()
→ autodiscover_tasks
→ fork/threads/gevent pool
→ consume broker messages
```

Celery import tasks 时也会执行模块顶层代码。所以 task module 顶层不能做外部 I/O。

## 5. `AppConfig.ready()` 的边界

常见用途：注册 signals。

```python
class OrdersConfig(AppConfig):
    name = "orders"

    def ready(self):
        import orders.signals  # noqa
```

风险：

- 在测试中多次初始化时重复注册。
- ready 阶段做 DB query 导致启动慢/失败。
- Celery worker 和 web 都执行 ready，副作用被放大。

不要这样：

```python
class OrdersConfig(AppConfig):
    def ready(self):
        Order.objects.count()  # bad: startup DB query
        start_background_thread()  # bad: 每个 worker 都可能启动
```

## 6. monkey patch 时机

如果使用 gevent：

```python
from gevent import monkey
monkey.patch_all()
```

必须在导入 socket/ssl/requests/redis/mysql client 等 I/O 库之前执行。

错误顺序：

```python
import requests
from gevent import monkey
monkey.patch_all()
```

此时 `requests` 依赖的底层模块可能已经绑定了未 patch 的对象，导致部分 I/O 仍然阻塞。

生产建议：

- 能不用 gevent/eventlet 就先不用。
- 如果用，入口文件必须清楚标注 patch 时机。
- 给 gevent worker 单独压测，不要假设 sync worker 配置可直接迁移。

## 7. 代码实验：证明 import side effect

创建 `bad_module.py`：

```python
print("importing bad_module")

STATE = []
STATE.append("created at import time")
```

创建 `probe_import.py`：

```python
import importlib
import sys

print("before import", "bad_module" in sys.modules)
import bad_module
print("after import", "bad_module" in sys.modules)

import bad_module
print("second import does not re-run top-level")

importlib.reload(bad_module)
print("reload re-runs top-level")
```

运行：

```bash
python probe_import.py
```

观察：

- 第一次 import 执行顶层代码。
- 第二次 import 使用 `sys.modules` cache。
- `reload` 会重新执行顶层代码。

这解释了为什么 Django autoreload、测试 runner、Celery worker 启动时，side effect 可能比你想象得更复杂。

## 8. 生产检查清单

### 8.1 import 阶段禁止事项

- 禁止连接数据库并执行 query。
- 禁止 `redis.ping()`。
- 禁止启动线程/定时器。
- 禁止读取大文件到内存。
- 禁止调用外部 HTTP API。
- 禁止生产者在 import 时发送 Celery/Kafka/Redis Stream 消息。

### 8.2 settings 检查

```bash
python manage.py check --deploy --settings=config.settings.prod
python manage.py diffsettings --settings=config.settings.prod
```

检查项：

- `DEBUG=False`
- `ALLOWED_HOSTS` 非空
- cookie secure
- secret 不在代码里
- DB charset/collation 明确
- cache/broker URL 来自环境变量

### 8.3 启动命令检查

```bash
# WSGI
python -m gunicorn config.wsgi:application --check-config

# ASGI import 检查
python - <<'PY'
import os
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings.prod')
import config.asgi
print(config.asgi.application)
PY

# Celery inspect
celery -A config inspect ping
```

## 9. 常见误区

### 误区 1：把所有环境差异写进一个巨大 settings.py

结果是 dev/test/prod 条件判断交织，风险难审查。

### 误区 2：在 `ready()` 里修复业务状态

`ready()` 是应用注册完成信号，不是业务启动任务系统。修复任务应使用 management command、Celery beat 或迁移脚本。

### 误区 3：Gunicorn preload 一定安全

只有 import 阶段无外部连接、无大可变全局状态时，preload 才更安全。

### 误区 4：Celery task 文件只是定义函数，不会执行

task module import 时，顶层代码一定会执行。

## 10. 课堂讨论题

1. 为什么 DB connection 不应该在 import 阶段创建？
2. `gunicorn --preload` 与 Django global cache 有什么关系？
3. Celery worker 和 web 进程共用 settings 时，哪些配置应该不同？
4. gevent monkey patch 放在 `settings.py` 里是否合适？为什么？

## 11. 作业

设计一个 Django 项目启动检查文档，包含：

- settings 分层路径。
- WSGI/ASGI/Celery 三个入口。
- import 阶段禁止事项。
- `check --deploy` 输出。
- 对 `--preload` 的使用决策。
- 如果使用 gevent，monkey patch 放在哪里。

## 12. 评估标准

- 能解释 import cache 和 module side effect。
- 能画出 Gunicorn/Daphne/Celery 启动链路。
- 能设计安全的 settings 分层。
- 能指出 `AppConfig.ready()` 的副作用风险。
- 能解释 monkey patch 的时机要求。
