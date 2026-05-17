# Lab 6：Request ID Middleware 与异常链路

## 目标

实现最小 RequestIDMiddleware，采集成功和失败请求证据。

## 代码骨架

```python
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
            logger.exception(
                "http_request_failed",
                extra={"request_id": request_id, "exception_type": type(exc).__name__},
            )
            raise
        response["X-Request-ID"] = request_id
        logger.info(
            "http_request_finished",
            extra={"request_id": request_id, "latency_ms": int((time.perf_counter() - started) * 1000)},
        )
        return response
```

## 测试要求

- 未传 `X-Request-ID` 时自动生成。
- 已传 `X-Request-ID` 时沿用。
- response header 包含 `X-Request-ID`。
- view 抛异常时日志包含 `exception_type`。

## 生产映射

回答：

1. request id 如何传入 Celery task payload？
2. Kafka/Redis Stream event 中应包含哪些 trace 字段？
3. 为什么不能只记录失败请求？

## 通过标准

- middleware 顺序合理。
- 成功/失败都有结构化日志。
- request id 能跨 HTTP 与异步边界传递。
