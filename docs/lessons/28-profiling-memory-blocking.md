# Lesson 28：性能剖析、内存泄漏与阻塞定位

## 学习目标

- 能区分 CPU 慢、I/O 慢、锁等待、队列等待、内存泄漏。
- 能使用 py-spy、pyinstrument、cProfile、tracemalloc、memray、strace、lsof。
- 能设计低风险生产采样策略。
- 能把性能问题转化为可验证的修复假设。

## 1. 证据分类

```text
CPU: py-spy/scalene/cProfile
Python heap: tracemalloc/objgraph/memray
Native/RSS: smaps/memray/container metrics
I/O syscall: strace/lsof/ss
DB: slow query log/EXPLAIN/lock wait
Queue: Celery/Kafka/Redis Stream lag
```

先分类，再优化。

## 2. py-spy 生产采样

无需改代码：

```bash
py-spy top --pid <pid>
py-spy record -o profile.svg --pid <pid> --duration 30
py-spy dump --pid <pid>
```

适合定位 CPU 热点、阻塞栈、线程状态。注意权限与容器 ptrace 限制。

## 3. pyinstrument 请求级剖析

适合开发/预发：

```python
from pyinstrument import Profiler
profiler = Profiler()
profiler.start()
# call view/service
profiler.stop()
print(profiler.output_text(unicode=True, color=True))
```

能看到时间花在哪些调用层级。

## 4. tracemalloc

```python
import tracemalloc
tracemalloc.start(25)
snap1 = tracemalloc.take_snapshot()
# run workload
snap2 = tracemalloc.take_snapshot()
for stat in snap2.compare_to(snap1, 'lineno')[:20]:
    print(stat)
```

适合 Python heap 增长；不解释 native extension 或 allocator RSS 不回收。

## 5. memray

适合深度内存分析：

```bash
python -m memray run -o out.bin manage.py reproduce_leak
python -m memray flamegraph out.bin
```

可以看到 native allocation。

## 6. strace/lsof/ss

```bash
strace -f -p <pid> -tt -T -e trace=network,select,poll,epoll_wait
lsof -p <pid> | grep TCP
ss -tanp | grep python
```

用于判断进程是否卡在 socket、DNS、文件、pipe。

## 7. 内存泄漏 runbook

```text
1. 确认 RSS 曲线：持续增长还是平台阶梯
2. 对比 Python heap：tracemalloc snapshot
3. 对比对象数：objgraph / gc.get_objects
4. 看 native：memray / smaps
5. 查缓存、全局 list、LRU 无上限、闭包引用、signals 重复注册
6. 加 max_requests 只做止血
7. 修复后用同一压测 workload 对比增长斜率
```

## 8. 阻塞定位 runbook

```text
症状：请求 p99 高但 CPU 不高
1. py-spy dump 看线程卡在哪
2. strace 看 syscall duration
3. DB slow query / lock waits
4. Redis slowlog / network latency
5. HTTP client timeout/retry
6. 修复后验证 p99、下游 latency、连接池等待时间
```

## 作业

构造一个有内存泄漏和慢外部调用的最小 Django service，用 tracemalloc 与 py-spy 分别给出证据截图/文本，并提出修复。

## 评估标准

- 能按证据分类性能问题。
- 能选择合适 profiling 工具。
- 能避免无证据调参。
