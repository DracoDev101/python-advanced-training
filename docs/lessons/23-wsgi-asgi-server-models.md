# Lesson 23：WSGI、ASGI 与 Python Web Server 模型

## 学习目标

- 能用接口形态解释 WSGI 与 ASGI 的根本差异。
- 理解 Django sync view、async view、middleware、ORM 在 WSGI/ASGI 下的执行边界。
- 能识别 thread-sensitive、sync_to_async、async_to_sync 带来的延迟与死锁风险。
- 能基于证据选择 Gunicorn、Uvicorn、Daphne、ASGI/WSGI 部署形态。

## 1. 核心结论

WSGI 是“一次请求一次同步 callable”：

```python
def application(environ, start_response):
    start_response("200 OK", [("Content-Type", "text/plain")])
    return [b"ok"]
```

ASGI 是“连接级协议状态机”：

```python
async def app(scope, receive, send):
    event = await receive()
    await send({"type": "http.response.start", "status": 200, "headers": []})
    await send({"type": "http.response.body", "body": b"ok"})
```

差异不是“ASGI 一定更快”，而是 ASGI 能表达长连接、WebSocket、lifespan、流式事件和并发 I/O。Django 项目如果大量 ORM/同步 middleware，盲目改 ASGI 不会自动获得吞吐提升，反而可能增加 sync/async 边界成本。

## 2. WSGI 请求路径

典型路径：

```text
client → nginx → gunicorn master → sync worker process → Django WSGIHandler
→ middleware → view → ORM/Redis/API → response
```

每个 sync worker 同一时刻只处理一个请求。并发来自：

- 多 worker 进程。
- gthread worker 的多线程。
- gevent/eventlet worker 的协作式 greenlet。

证据命令：

```bash
ps -o pid,ppid,rss,cmd -C gunicorn
ss -tanp | grep :8000
curl -w 'connect=%{time_connect} ttfb=%{time_starttransfer} total=%{time_total}\n' http://127.0.0.1:8000/health
```

## 3. ASGI scope/receive/send

HTTP scope：

```json
{"type":"http","method":"GET","path":"/orders/1","headers":[...]}
```

WebSocket scope：

```json
{"type":"websocket","path":"/ws/orders/1","subprotocols":[]}
```

ASGI 允许一个 app 通过 receive/send 处理多事件：连接建立、消息、断连、body 分块、lifespan startup/shutdown。

## 4. Django sync/async 边界

Django 支持 async view，但很多组件仍是 sync 或 thread-sensitive：

```python
from asgiref.sync import sync_to_async

async def order_detail(request, order_id):
    order = await sync_to_async(load_order, thread_sensitive=True)(order_id)
    return JsonResponse({"id": order.id})
```

风险：

- 每次边界切换有调度成本。
- thread_sensitive 会串行化某些同步调用，避免连接/线程不安全，但限制并发。
- 在 async view 中直接调用同步 ORM 会触发 `SynchronousOnlyOperation` 或阻塞 event loop。

## 5. 什么时候保留 WSGI

适合 WSGI：

- 主要是短 HTTP 请求。
- ORM 与同步第三方 SDK 占主导。
- 不需要 WebSocket/长轮询/SSE。
- 团队熟悉 sync debugging 与多进程部署。

## 6. 什么时候引入 ASGI

适合 ASGI：

- WebSocket/Channels。
- Server-Sent Events、流式响应。
- 大量原生 async I/O，例如 async HTTP client、async Redis、async Kafka consumer。
- 需要 lifespan 管理异步连接池。

## 7. 生产反模式

| 反模式 | 后果 |
|---|---|
| async view 里调用 requests | 阻塞 event loop |
| ASGI 下把所有 ORM 包 sync_to_async 且高频调用 | thread-sensitive 队列堆积 |
| 同一服务混合复杂 sync/async middleware | latency 抖动，栈难排查 |
| 用 WebSocket 承载所有状态同步但无 backpressure | 单个慢客户端拖垮 worker |

## 8. 证据先行排障

```text
症状：ASGI 服务 p95 升高
1. 看 event loop lag
2. 看 sync_to_async 调用耗时和排队时间
3. 看 DB pool/connection 数
4. py-spy 看是否卡在 requests/pymysql/socket recv
5. 如果是 WebSocket，看连接数、每连接 buffer、发送失败率
6. 修复后验证 p95、event loop lag、worker RSS 回落
```

## 作业

给一个 Django 订单服务设计两种部署：纯 WSGI 与 ASGI+Channels。列出适用接口、worker 数、连接池、超时、观测指标和回滚策略。

## 评估标准

- 能写出 WSGI 和 ASGI 最小接口。
- 能解释 async Django 的同步边界。
- 能用证据判断 ASGI 是否真的改善系统。
