# Lesson 27：gevent/eventlet 与 monkey patch 边界

## 学习目标

- 理解 monkey patch 修改了哪些标准库对象。
- 能解释 patch 时机为什么必须早于导入 I/O 库。
- 能审计 Django/Celery/Gunicorn gevent/eventlet 模式的兼容性。
- 能检测阻塞调用、patch 污染和隐藏的生产风险。

## 1. monkey patch 做了什么

```python
from gevent import monkey
monkey.patch_all()
```

常见被替换模块：

```text
socket, ssl, select, time, threading, subprocess, os, signal
```

目标是让传统阻塞 I/O 在等待时 yield 给 event loop。

## 2. patch 时机

必须在导入 requests、redis、pymysql、boto3 等 I/O 库之前 patch：

```python
# gunicorn_conf.py or very early entrypoint
from gevent import monkey
monkey.patch_all()

from myapp import create_app
```

如果先导入库再 patch，库内部可能已经缓存原始 socket/select，导致部分调用仍阻塞整个进程。

## 3. Gunicorn gevent worker

```bash
gunicorn config.wsgi:application \
  --worker-class gevent \
  --workers 4 \
  --worker-connections 1000
```

适合大量 I/O 等待。风险：

- CPU-bound task 阻塞整个 worker 里的 greenlet。
- 未 patch 或不可 patch 的 C 扩展阻塞。
- Django ORM/DB driver 连接池与 greenlet 局部状态需验证。

## 4. Celery gevent/eventlet pool

```bash
celery -A config worker -P gevent -c 100
```

仅适合 I/O-bound task。不要用于 CPU-bound 或大量本地计算。Celery 的 soft time limit 在非 prefork 池中支持差异明显，必须验证版本与行为。

## 5. 检测 patch 状态

```python
from gevent import monkey
print(monkey.is_module_patched("socket"))
print(monkey.is_object_patched("socket", "socket"))
```

检测阻塞：

```bash
python -m py_spy top --pid <pid>
strace -f -p <pid> -e trace=network,select,poll,epoll_wait
```

如果 py-spy 显示某 greenlet 长时间卡在 CPU frame，gevent 无法帮忙。

## 6. context/local 问题

线程本地变量、request context、连接复用在 greenlet 下可能语义变化。要验证：

- request_id 是否串线。
- DB connection 是否被错误共享。
- logging contextvars 是否跨请求污染。

## 7. 反模式

| 反模式 | 后果 |
|---|---|
| 在 Django settings 中间才 patch | 部分库未 patch，延迟随机卡死 |
| gevent worker 里跑 CPU-heavy endpoint | 同 worker 所有 greenlet 饥饿 |
| Celery gevent pool 处理大 PDF | 任务互相阻塞，time limit 行为异常 |
| patch 后依赖 thread local 语义 | request context 串线 |

## 8. Runbook：gevent 服务偶发全进程卡住

```text
1. py-spy dump：看是否单个 CPU frame 占住
2. strace：是否卡未 patch 的 blocking syscall
3. gevent hub latency 指标
4. 检查 patch 时机和导入顺序
5. 隔离 CPU-heavy 路由或改 sync/prefork
6. 验证 hub latency、p95、worker timeout
```

## 作业

审计一个 Django 项目是否适合 gevent：列出 I/O 库、导入顺序、CPU-heavy 路由、DB driver、request context、测试方案与回滚方案。

## 评估标准

- 能说清 patch 的影响面和时机。
- 能给出 gevent/eventlet 的适用/禁用条件。
- 能用工具找出阻塞 greenlet 的证据。
