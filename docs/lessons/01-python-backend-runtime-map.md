# Lesson 1：Python 后端工程观与运行时地图

## 学习目标

完成本课后，你应该能够：

- 画出一个生产级 Python/Django 后端的运行路径：HTTP 请求、数据库访问、缓存、任务队列、事件系统、长连接。
- 区分 Python 后端中的几种执行单元：process、thread、coroutine、greenlet、Celery task、Kafka consumer。
- 解释为什么“Python 慢”不是一个可操作的诊断结论，必须拆成 CPU、I/O、锁、DB、队列、网络和部署模型。
- 为后续课程建立统一的证据语言：日志、SQL、profile、metrics、trace、queue depth、consumer lag。

## 关键问题

1. 一个 Django 请求从 socket 进入，到返回 JSON，中间经过哪些层？
2. Gunicorn、Daphne、Celery worker、Kafka consumer 是同一种进程吗？它们的生命周期有什么差异？
3. 为什么同样是“异步”，`asyncio`、Celery、Kafka、Redis Stream、greenlet 解决的是不同问题？
4. 线上慢请求出现时，第一批证据应该采集什么？

## 核心结论

Python 生产后端不是“Django + 一堆库”，而是多个运行时边界的组合：

```text
client
  → load balancer / reverse proxy
  → WSGI server: Gunicorn sync/gthread/gevent
      → Django sync request path
      → ORM / cache / external API
  → ASGI server: Daphne/Uvicorn
      → async view / Channels / WebSocket
  → broker / stream / log
      → Celery worker
      → Redis Stream consumer
      → Kafka consumer group
  → MySQL / PostgreSQL / MongoDB / Redis
```

后续所有章节都围绕这条原则展开：

> 不先定位运行边界，就无法正确解释性能、可靠性和故障证据。

## 1. 问题引入：一个订单接口为什么会慢？

假设线上接口：

```http
POST /api/orders
```

表面现象：P95 从 120ms 上升到 3s。

不要马上猜“Python 慢”或“数据库慢”。先拆路径：

```text
HTTP accept
→ Gunicorn worker 排队
→ Django middleware
→ auth/session/cache
→ serializer validation
→ ORM transaction
→ MySQL row lock / slow query
→ Redis cache set
→ transaction.on_commit
→ Celery task publish
→ response render
```

任何一段都可能导致 3s：

| 位置 | 典型症状 | 第一证据 |
|---|---|---|
| Gunicorn worker | 请求排队、worker timeout | access log、worker count、CPU、`ps` |
| Django middleware | 全局慢、每个接口都慢 | request timing middleware |
| ORM | 某些接口慢 | SQL log、EXPLAIN、lock wait |
| MySQL transaction | 偶发慢、并发时慢 | InnoDB status、processlist |
| Redis | 热 key、网络抖动 | command latency、slowlog |
| Celery publish | 响应尾部变慢 | broker latency、publish error |
| 外部 API | 依赖超时 | timeout log、trace span |

## 2. 运行边界地图

### 2.1 WSGI：同步请求/响应协议

WSGI 的核心是一个同步 callable：

```python
# 简化版 WSGI callable

def application(environ, start_response):
    status = "200 OK"
    headers = [("Content-Type", "text/plain")]
    start_response(status, headers)
    return [b"hello"]
```

Django 的传统部署路径：

```text
Gunicorn master
  ├─ sync worker process 1 → one request at a time per worker
  ├─ sync worker process 2
  └─ sync worker process N
```

特点：

- worker 是进程，隔离性好。
- sync worker 单进程内通常一次处理一个请求。
- 并发主要靠多进程，而不是单进程里跑很多协程。
- 适合典型 CRUD，但如果外部 I/O 很慢，需要合理 timeout，否则 worker 会被占住。

### 2.2 ASGI：异步事件协议

ASGI 把连接抽象为 `scope + receive + send`：

```python
async def app(scope, receive, send):
    assert scope["type"] == "http"
    await send({
        "type": "http.response.start",
        "status": 200,
        "headers": [(b"content-type", b"text/plain")],
    })
    await send({
        "type": "http.response.body",
        "body": b"hello",
    })
```

Django 支持 ASGI，但要注意：

- async view 不等于整个项目都异步。
- ORM 大量路径仍然是同步边界，需要 `sync_to_async` 或线程池包装。
- WebSocket/长连接通常由 Daphne/Uvicorn + Channels 处理。

### 2.3 Celery：后台任务执行系统

Celery 不是 HTTP server。它是：

```text
producer: Django process
  → broker: Redis/RabbitMQ
  → worker: Celery worker process pool
  → result backend: optional
```

生产语义重点：

- task 是否幂等。
- ack 在执行前还是执行后。
- worker crash 后任务是否会重投。
- retry/backoff 会不会放大事故。
- 队列堆积时是否有隔离队列和限流。

### 2.4 Redis Stream：轻量事件流

Redis Stream 适合轻量 event log：

```text
XADD order-events * type order.created order_id 123
XREADGROUP GROUP order-consumers c1 COUNT 10 STREAMS order-events >
XACK order-events order-consumers 1710000000000-0
```

它比 list queue 多了：

- stream id
- consumer group
- pending entries list
- replay / claim 能力

但它不是 Kafka：

- 分区扩展模型不同。
- 长期存储和大规模 replay 能力弱于 Kafka。
- 运维和语义适合中小规模事件流、Django 内部工作流、可靠 consumer group。

### 2.5 Kafka：分区日志与消费者组

Kafka 的核心不是“队列”，而是 partitioned log：

```text
topic: order-events
  partition 0: offset 0, 1, 2, ...
  partition 1: offset 0, 1, 2, ...
consumer group: order-read-model
  member A owns partition 0
  member B owns partition 1
```

关键边界：

- 单 partition 内有序；跨 partition 不保证全局有序。
- offset commit 决定失败重放窗口。
- consumer lag 是背压的第一信号。
- exactly-once 对业务系统通常仍需业务幂等。

## 3. Python 并发模型的最小地图

| 模型 | 适合 | 不适合 | 典型组件 |
|---|---|---|---|
| process | CPU 隔离、稳定部署 | 高频共享状态 | Gunicorn sync worker、Celery prefork |
| thread | 阻塞 I/O 并发 | CPU-bound | Gunicorn gthread、ThreadPoolExecutor |
| asyncio coroutine | 高并发 async I/O | 调同步 ORM/阻塞库 | ASGI、aiokafka、async HTTP client |
| greenlet | monkey-patched blocking I/O | patch 不完整/阻塞 C 扩展 | gevent/eventlet worker |
| external worker | 耗时任务、重试 | 同步强一致请求路径 | Celery、Kafka consumer |

### GIL 的正确位置

GIL 不代表 Python 不能并发处理 I/O。它意味着：

- 同一解释器进程中，同一时刻通常只有一个线程执行 Python bytecode。
- I/O 阻塞时会释放 GIL，线程可以切换。
- CPU-bound 任务用线程不能线性扩展，应使用多进程、C 扩展或外部计算服务。
- Django CRUD 的瓶颈常常不是 GIL，而是 DB、锁、外部 API、worker 数量、连接池。

## 4. 代码实验：用最小程序区分 CPU 与 I/O

创建 `runtime_probe.py`：

```python
import concurrent.futures
import hashlib
import time


def io_bound(i: int) -> int:
    time.sleep(0.1)
    return i


def cpu_bound(i: int) -> str:
    data = str(i).encode() * 100_000
    h = b""
    for _ in range(800):
        h = hashlib.sha256(h + data).digest()
    return h.hex()


def run_threads(fn, n: int, workers: int):
    started = time.perf_counter()
    with concurrent.futures.ThreadPoolExecutor(max_workers=workers) as ex:
        list(ex.map(fn, range(n)))
    return time.perf_counter() - started


if __name__ == "__main__":
    for fn in [io_bound, cpu_bound]:
        for workers in [1, 4, 16]:
            elapsed = run_threads(fn, n=32, workers=workers)
            print(fn.__name__, "workers=", workers, "elapsed=", round(elapsed, 3))
```

运行：

```bash
python runtime_probe.py
```

预期观察：

- `io_bound` 会随线程数增加显著变快。
- `cpu_bound` 不会按线程数线性变快，甚至可能变慢。

这个实验对应生产判断：

- 外部 API/Redis/MySQL 网络等待可以通过并发隐藏部分延迟。
- CPU-heavy JSON 序列化、加密、图片处理、复杂聚合不能靠线程解决。
- Celery CPU-heavy task 用 prefork 比 thread/gevent 更合理。

## 5. 生产实践：先定义服务角色，不要混跑所有东西

一个可维护的 Django 生产部署至少拆成：

```text
web-sync:
  command: gunicorn config.wsgi:application -k sync -w 4
  responsibility: 普通 HTTP CRUD

web-asgi:
  command: daphne config.asgi:application
  responsibility: WebSocket / async endpoint

worker-default:
  command: celery -A config worker -Q default -c 4
  responsibility: 常规后台任务

worker-critical:
  command: celery -A config worker -Q critical -c 2
  responsibility: 关键短任务，避免被慢任务饿死

consumer-events:
  command: python manage.py consume_order_events
  responsibility: Redis Stream / Kafka consumer
```

不要把所有职责塞进一个进程：

- Web 进程不应该执行长耗时任务。
- Celery worker 不应该顺手启动 HTTP server。
- Kafka consumer 不应该共享 Django request-local 假设。
- gevent worker 不应该混用未验证的阻塞库。

## 6. 故障证据：慢请求第一现场

### 6.1 必须打出的结构化日志字段

```json
{
  "ts": "2026-05-17T12:00:00Z",
  "level": "info",
  "event": "http_request_finished",
  "request_id": "req_abc",
  "method": "POST",
  "path": "/api/orders",
  "status": 201,
  "latency_ms": 842,
  "db_ms": 610,
  "redis_ms": 8,
  "external_ms": 0,
  "user_id": 42
}
```

如果只有一条 access log：

```text
POST /api/orders 201 842ms
```

你只能知道慢，不能知道慢在哪里。

### 6.2 第一批命令

```bash
# Web 进程状态
ps -o pid,ppid,cmd,%cpu,%mem -C gunicorn

# 端口监听
ss -lntp

# 最近错误日志
journalctl -u my-django-web -n 200 --no-pager

# 数据库连接/锁，MySQL 示例
mysql -e "SHOW PROCESSLIST"
mysql -e "SHOW ENGINE INNODB STATUS\G"

# Celery 队列现场
celery -A config inspect active
celery -A config inspect reserved
celery -A config inspect scheduled

# Redis Stream 堆积
redis-cli XINFO GROUPS order-events
redis-cli XPENDING order-events order-consumers

# Kafka lag
kafka-consumer-groups --bootstrap-server localhost:9092 --describe --group order-service
```

## 7. 常见误区

### 误区 1：把 async 当成性能银弹

如果 async view 内部调用同步 ORM 或阻塞 HTTP client，事件循环仍会被占住。你只是把同步阻塞藏在 async 函数里。

### 误区 2：把 Celery 当成一致性保证

Celery 只能提供任务执行机制，不自动解决：

- DB commit 后消息一定发出。
- task 重试后不会重复扣库存。
- worker crash 后业务状态一定正确。

这些要靠 `transaction.on_commit`、outbox、幂等 key、状态机和补偿流程。

### 误区 3：混淆 Redis Stream 和 Kafka

Redis Stream 可以做轻量事件流，但 Kafka 更适合：

- 多服务共享事件总线。
- 长期保留与大规模 replay。
- 高吞吐 partition 扩展。
- 独立 consumer group 多读模型。

### 误区 4：过早使用 gevent/eventlet

greenlet worker 能提高阻塞 I/O 并发，但前提是：

- monkey patch 足够早。
- 依赖库兼容。
- 没有长时间 CPU-bound 代码。
- 有阻塞检测和回退方案。

## 8. 课堂讨论题

1. 一个订单创建接口里，哪些步骤必须在 HTTP 请求内完成，哪些可以异步？为什么？
2. 如果 Celery 队列堆积但 HTTP 仍然很快，用户体验和数据一致性分别会出什么问题？
3. Redis Stream 与 Kafka 都能消费事件，什么情况下你会选择其中一个？
4. Gunicorn sync worker timeout 是根因还是症状？如何证明？

## 9. 作业

画出你熟悉的一个 Django 服务的运行路径，至少包含：

- HTTP server 类型：Gunicorn/Daphne/Uvicorn。
- 数据库：MySQL/PostgreSQL/MongoDB。
- 缓存：Redis。
- 异步任务：Celery/Redis Stream/Kafka。
- 关键 metrics：HTTP latency、DB latency、queue depth、consumer lag。

要求：为每个节点写出一个“出问题时的第一证据”。

## 10. 评估标准

- 能把“慢”拆成具体运行边界，而不是泛泛猜测。
- 能说明 WSGI、ASGI、Celery、Kafka consumer 的生命周期差异。
- 能列出至少 8 个生产排障命令或指标。
- 能指出 async、Celery、Kafka、Redis Stream 各自解决的问题边界。

## 11. 延伸阅读

- Python 官方文档：Execution model、`asyncio`、`concurrent.futures`。
- Django 官方文档：Deployment、ASGI、Database transactions。
- Celery 官方文档：Workers、Monitoring and Management。
- Gunicorn design 文档。
- ASGI specification。
