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

## 9. Django ASGIHandler / WSGIHandler 对照

Django 并不是因为入口换成 ASGI 就变成“全异步框架”。两个入口的差异更像：协议适配层不同，内部仍要根据 view/middleware 能力决定 sync/async 路径。

```text
WSGIHandler
  __call__(environ, start_response)
  → request = WSGIRequest(environ)
  → get_response(request)
  → response iterable

ASGIHandler
  async __call__(scope, receive, send)
  → read body events
  → request = ASGIRequest(scope, body_file)
  → await get_response_async(request)
  → send response start/body events
```

在 ASGI 模式下，如果 middleware 是同步的，Django 需要做适配：

```text
async server
→ sync middleware bridge
→ sync view / ORM
→ async bridge back to server
```

所以“有一个 async view”不等于整条链路无阻塞。评估 ASGI 收益时要统计：

```text
sync middleware count
sync_to_async call count
sync_to_async wait time
DB query time
event loop lag
threadpool queue wait
```

## 10. thread_sensitive 为什么会串行化

`sync_to_async(fn, thread_sensitive=True)` 的目标是保护依赖线程局部状态或线程不安全连接的同步代码。代价是同一上下文中的这类调用可能被排到专用线程串行执行。

典型风险：

```python
async def dashboard(request):
    # 每个函数内部都走同步 ORM
    a = await sync_to_async(load_orders, thread_sensitive=True)()
    b = await sync_to_async(load_payments, thread_sensitive=True)()
    c = await sync_to_async(load_notifications, thread_sensitive=True)()
```

如果高并发请求都排到少量 thread-sensitive 执行器，症状会像“event loop 没阻塞，但 p95 变高”。证据不是 CPU，而是 bridge wait time 与 DB connection wait。

最小探针：

```python
import time
from asgiref.sync import sync_to_async

async def measured_sync_call(name, fn, *args):
    start = time.perf_counter()
    result = await sync_to_async(fn, thread_sensitive=True)(*args)
    duration_ms = (time.perf_counter() - start) * 1000
    logger.info("sync_bridge_call", extra={"name": name, "duration_ms": duration_ms})
    return result
```

## 11. event loop lag 探针

ASGI 服务要监控 event loop 是否被阻塞：

```python
import asyncio, time, logging

async def loop_lag_probe(interval=0.5):
    expected = time.perf_counter() + interval
    while True:
        await asyncio.sleep(interval)
        now = time.perf_counter()
        lag_ms = max(0, now - expected) * 1000
        logging.info("event_loop_lag", extra={"lag_ms": lag_ms})
        expected = now + interval
```

判断：

```text
lag 低、p95 高：多半是下游 I/O、threadpool/DB pool 排队
lag 高、CPU 高：event loop 内有 CPU-bound 或阻塞同步调用
lag 高、CPU 低：可能是 blocking syscall、DNS、未 await 的同步 SDK
```

## 12. ASGI lifespan 与连接池初始化

ASGI lifespan 适合初始化异步资源：

```text
startup: create async HTTP/Redis/Kafka clients
shutdown: flush metrics, close clients, drain in-flight tasks
```

不要在 import 阶段创建连接池。原因：

- Gunicorn/Uvicorn 多 worker fork 时连接可能被继承。
- 测试环境 import 会产生外部连接副作用。
- reload 时旧连接没有明确关闭。

生产验证：

```bash
lsof -p <worker_pid> | grep TCP
ss -tanp | grep python
```

## 作业

给一个 Django 订单服务设计两种部署：纯 WSGI 与 ASGI+Channels。列出适用接口、worker 数、连接池、超时、观测指标和回滚策略。

## 评估标准

- 能写出 WSGI 和 ASGI 最小接口。
- 能解释 async Django 的同步边界。
- 能用证据判断 ASGI 是否真的改善系统。
