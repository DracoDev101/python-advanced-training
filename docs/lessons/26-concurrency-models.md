# Lesson 26：asyncio、线程、进程与 greenlet 对比

## 学习目标

- 能区分抢占式线程、协作式 coroutine/greenlet、多进程并行。
- 能根据 CPU/I/O/阻塞库选择 asyncio、thread、process、gevent。
- 能处理 cancellation、timeout、contextvars、连接池与背压。
- 能用实验观察 event loop 阻塞、GIL、线程池耗尽。

## 1. 并发模型地图

| 模型 | 调度 | 适合 | 不适合 |
|---|---|---|---|
| thread | OS 抢占 | 同步 I/O 库并发 | CPU-bound 受 GIL 限制 |
| asyncio | 协作 await | 原生 async I/O | 阻塞调用会卡 loop |
| process | 多解释器 | CPU-bound | IPC 成本高 |
| greenlet | 协作 monkey patched I/O | 大量传统 socket I/O | patch 边界复杂 |

## 2. GIL 的现实含义

GIL 不等于“线程没用”。线程对 I/O-bound 有用，因为阻塞 I/O 会释放 GIL；CPU-bound Python bytecode 不会并行。

证据：

```bash
python -m py_spy top --pid <pid>
```

如果多个线程都在 Python CPU frame 争抢，增加线程不会线性提升吞吐。

## 3. asyncio cancellation

取消不是强杀线程。取消在 await 点注入 `CancelledError`：

```python
async def handler():
    try:
        async with asyncio.timeout(2):
            await call_api()
    except TimeoutError:
        ...
```

如果 coroutine 长时间不 await，取消无法及时生效。

## 4. ThreadPoolExecutor 边界

Django/ASGI 常用线程池包装同步代码。风险：

```text
async requests → sync_to_async → thread pool
thread waits DB/API
pool exhausted
event loop 仍活着但请求排队
```

观测：线程池队列长度、active thread、sync_to_async latency。

## 5. ProcessPoolExecutor

适合 CPU 密集：图片处理、压缩、报表计算。注意：

- 参数与结果需要 pickle。
- Django model object 不适合作为进程参数。
- 子进程内不要复用父进程 DB connection。

## 6. greenlet 与 gevent

greenlet 是用户态协程。gevent 通过 monkey patch 把 socket 等阻塞调用变成协作式等待。

优点：旧同步代码可获得高 I/O 并发。代价：patch 时机、第三方库兼容、debug stack 复杂。

## 7. 选择规则

```text
短同步 Django API：Gunicorn sync/gthread
原生 async I/O：ASGI + asyncio
大量传统 HTTP/Redis socket I/O：可评估 gevent，但要做 patch 审计
CPU-bound：Celery prefork / ProcessPool / 独立服务
长连接：ASGI/Daphne/Uvicorn
```

## 8. Runbook：CPU 高但吞吐低

```text
1. top/htop：CPU 核心使用率
2. py-spy top：Python frame 热点
3. 区分 CPU-bound / I/O wait / lock wait
4. 看 worker class 与 concurrency
5. CPU-bound 移出 Web worker 或用进程池/Celery
6. 验证 p95、CPU、队列等待时间
```

## 作业

给四类任务选择并发模型：调用 20 个外部 API、生成大型 PDF、WebSocket 广播、批量 MySQL 导入。写出选择理由、失败模式与观测指标。

## 评估标准

- 能解释 GIL 对线程的真实影响。
- 能处理 asyncio timeout/cancellation。
- 能识别线程池/进程池/greenlet 的边界。
