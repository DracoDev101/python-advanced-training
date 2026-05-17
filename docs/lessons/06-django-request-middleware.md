# Lesson 6：请求生命周期、middleware 与异常链路

## 学习目标

完成本课后，你应该能够：

- 解释一个 Django HTTP 请求从 server 进入到 response 返回的关键步骤。
- 正确设计 middleware 顺序，避免认证、request id、事务、异常处理互相污染。
- 建立统一错误响应和结构化日志字段。
- 用最小 timing middleware 采集慢请求证据，而不是只看 access log。

## 关键问题

1. middleware 的顺序为什么会影响安全和可观测性？
2. Django 在什么时候打开/关闭数据库连接？
3. exception handler 应该吞掉所有异常吗？
4. request id 应该在哪里生成，如何传到 Celery/Kafka/Redis Stream？

## 核心结论

请求生命周期不是“view 函数被调用”这么简单：

```text
server worker
→ WSGI/ASGI adapter
→ request object
→ middleware before chain
→ URL resolver
→ view / DRF dispatch
→ serializer / service / ORM
→ exception handling
→ middleware after chain
→ response render
→ connection cleanup
```

生产级 Django 项目必须让每个请求都有可追踪证据：

```text
request_id + user_id + path + status + latency_ms + db_ms + cache_ms + exception_type
```

## 1. Django 请求路径

简化路径：

```text
Gunicorn/Daphne
  → Django handler
  → request = WSGIRequest/ASGIRequest
  → middleware stack
  → URL resolver
  → view
  → template/DRF response render
  → middleware response path
```

middleware 是洋葱模型：

```text
M1 before
  M2 before
    M3 before
      view
    M3 after
  M2 after
M1 after
```

顺序写错会导致：

- request id 未覆盖全部日志。
- exception 被提前吞掉。
- auth 之前访问 `request.user`。
- security middleware 失效。
- response header 被覆盖。

## 2. Request ID Middleware

最小实现：

```python
# common/middleware.py
import time
import uuid
import logging

logger = logging.getLogger(__name__)

class RequestIDMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        request_id = request.headers.get("X-Request-ID") or str(uuid.uuid4())
        request.request_id = request_id
        started = time.perf_counter()

        try:
            response = self.get_response(request)
        except Exception as exc:
            latency_ms = int((time.perf_counter() - started) * 1000)
            logger.exception(
                "http_request_failed",
                extra={
                    "request_id": request_id,
                    "method": request.method,
                    "path": request.path,
                    "latency_ms": latency_ms,
                    "exception_type": type(exc).__name__,
                },
            )
            raise

        latency_ms = int((time.perf_counter() - started) * 1000)
        response["X-Request-ID"] = request_id
        logger.info(
            "http_request_finished",
            extra={
                "request_id": request_id,
                "method": request.method,
                "path": request.path,
                "status": response.status_code,
                "latency_ms": latency_ms,
            },
        )
        return response
```

它解决：

- 请求入口生成统一 id。
- 成功/失败都有 latency。
- response header 返回 id，方便客户端报障。

还没解决：

- DB 时间拆分。
- Redis 时间拆分。
- trace context。
- Celery/Kafka 下游传递。

## 3. middleware 顺序建议

```python
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "common.middleware.RequestIDMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "common.middleware.StructuredExceptionMiddleware",
]
```

解释：

- Security 尽量靠前。
- RequestID 早生成，覆盖后续日志。
- Session 在 Auth 前。
- Exception middleware 位置要明确：它只能处理后续层抛出的异常。

DRF 项目中，也可以用 DRF exception handler 统一 API 错误格式。

## 4. 异常链路

错误响应格式建议：

```json
{
  "error": {
    "code": "order_not_found",
    "message": "Order not found",
    "request_id": "req_123"
  }
}
```

不要把内部异常直接暴露：

```json
{
  "error": "IntegrityError: Duplicate entry '...' for key ..."
}
```

### 4.1 domain exception

```python
class DomainError(Exception):
    code = "domain_error"
    status_code = 400

class OrderNotFound(DomainError):
    code = "order_not_found"
    status_code = 404
```

### 4.2 DRF exception handler

```python
from rest_framework.views import exception_handler
from rest_framework.response import Response


def api_exception_handler(exc, context):
    response = exception_handler(exc, context)
    request = context.get("request")
    request_id = getattr(request, "request_id", "")

    if response is not None:
        response.data = {
            "error": {
                "code": "api_error",
                "message": response.data,
                "request_id": request_id,
            }
        }
        return response

    if isinstance(exc, DomainError):
        return Response(
            {
                "error": {
                    "code": exc.code,
                    "message": str(exc),
                    "request_id": request_id,
                }
            },
            status=exc.status_code,
        )

    return None  # 交给 Django 500 处理并保留 exception log
```

关键：未知异常不要静默吞掉，否则告警和错误栈会消失。

## 5. 数据库连接生命周期

Django 会为请求管理 DB connection，但生产中要关注：

- `CONN_MAX_AGE`：连接复用时间。
- worker 数量 × 每 worker 最大并发 = 潜在连接数。
- Celery worker 也会打开 DB connection。
- 长连接/async path 中不要把 sync ORM 调用放到 event loop 里阻塞。

估算：

```text
Gunicorn workers: 4
Gunicorn threads: 8
Celery workers: 4 processes
Daphne threads/sync bridges: variable
Potential DB connections ≈ 4*8 + 4 + other consumers
```

如果 MySQL `max_connections=100`，多个服务副本会很快耗尽。

## 6. 慢请求 timing：不要只看总耗时

可以先做一个最小分段指标：

```python
# common/timing.py
import time
from contextlib import contextmanager

@contextmanager
def record_timing(request, name: str):
    started = time.perf_counter()
    try:
        yield
    finally:
        elapsed_ms = int((time.perf_counter() - started) * 1000)
        timings = getattr(request, "timings", {})
        timings[name] = timings.get(name, 0) + elapsed_ms
        request.timings = timings
```

在 service 中：

```python
with record_timing(request, "inventory_ms"):
    reserve_inventory(...)
```

更成熟做法是 OpenTelemetry trace span，但在课程初期先用最小 timing 建立证据意识。

## 7. request id 传递到异步边界

Celery：

```python
send_email.delay({
    "order_id": order.id,
    "request_id": request.request_id,
    "event_id": event_id,
})
```

Kafka / Redis Stream：

```python
event = {
    "event_id": event_id,
    "event_type": "order.created",
    "request_id": request.request_id,
    "aggregate_id": f"order:{order.id}",
}
```

日志字段保持一致：

```text
request_id
order_id
event_id
task_id
stream_id
kafka_topic
kafka_partition
kafka_offset
```

## 8. 常见误区

### 误区 1：在 middleware 里做复杂业务

middleware 应处理横切关注点：request id、auth context、logging、security、metrics。业务流程应在 service。

### 误区 2：catch 所有异常并返回 200

这会破坏监控、重试和客户端语义。

### 误区 3：只记录错误日志，不记录成功慢请求

很多性能问题没有 exception，只有 latency 上升。

### 误区 4：request id 只停留在 HTTP 层

一旦进入 Celery/Kafka/Redis Stream，如果不传 request id，就无法串起完整链路。

## 9. 课堂讨论题

1. RequestIDMiddleware 应该放在 AuthenticationMiddleware 前还是后？为什么？
2. 未知异常应该被 API exception handler 转成自定义 JSON 吗？如何保留堆栈？
3. DB connection 数量如何从 worker 配置推算？
4. 一个订单事件进入 Kafka 后，如何保留 HTTP request id？

## 10. 作业

实现一个最小 RequestIDMiddleware，要求：

- 接收或生成 `X-Request-ID`。
- 成功和失败都记录结构化日志。
- response 返回 `X-Request-ID`。
- 异常时记录 `exception_type`。

然后写一个测试，验证 response header 存在。

## 11. 评估标准

- 能画出 middleware 洋葱模型。
- 能设计 request id 和异常响应格式。
- 能解释 DB connection lifecycle 与 worker 数量关系。
- 能说明 request id 如何进入 Celery/Kafka/Redis Stream。
