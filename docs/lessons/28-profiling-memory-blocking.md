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

## 9. 如何阅读 flamegraph

Flamegraph 不是“最高的函数一定有问题”。阅读顺序：

```text
宽度 = 累计采样时间
顶部 = 调用栈叶子
同一横向位置 = 调用父子关系
self time 高 = 函数自身耗时
total time 高 = 函数及子调用耗时
```

判断例子：

```text
json.dumps 很宽：序列化成本高，可能 response 太大
pymysql read_packet 很宽：DB 等待或网络等待，不是 Python CPU
ssl recv 很宽：外部 HTTPS 慢或无 timeout
list comprehension 很宽：Python CPU 热点，可考虑算法/批量/缓存
```

不要只凭 profile 改代码。要把 profile 与 request route、SQL、队列、下游 latency 对齐。

## 10. RSS 增长但 tracemalloc 不增长

这通常说明问题不在 Python heap，可能是：

- native extension 分配，例如 compression/image/crypto。
- pymalloc arena 未归还 OS。
- 内存碎片。
- fork 后 copy-on-write 失效。
- C 库缓存或连接 buffer。

证据路径：

```bash
cat /proc/<pid>/status | egrep 'VmRSS|VmSize|Threads'
cat /proc/<pid>/smaps_rollup
pmap -x <pid> | tail -n 20
```

如果 `tracemalloc` 稳定但 RSS 持续上涨，用 memray 或 smaps，而不是继续查 Python list。

## 11. preload_app 与 copy-on-write 失效

Gunicorn `preload_app` 依赖 fork 后共享只读内存。如果 worker 启动后修改大量全局对象，共享页会变成私有页，RSS 上升。

典型触发：

```text
全局 LRU cache 预热后继续写
import 阶段加载大模型/大字典，worker 内再修改
metrics registry 动态创建大量 label
日志/trace 对象持有全局缓冲
```

证据：

```bash
smem -r -k -p $(pgrep -d, gunicorn)
cat /proc/<pid>/smaps_rollup | egrep 'Shared|Private|Pss'
```

看 PSS 比单纯 RSS 更能反映真实内存占用。

## 12. FD / connection leak

连接泄漏常表现为 p99 升高、`Too many open files`、DB/Redis 连接数持续上涨。

命令：

```bash
ls /proc/<pid>/fd | wc -l
lsof -p <pid> | awk '{print $5,$8,$9}' | sort | uniq -c | sort -nr | head
cat /proc/<pid>/limits | grep 'open files'
ss -tanp | grep <pid> | awk '{print $1,$4,$5}' | sort | uniq -c
```

常见根因：

- requests/httpx response 未关闭。
- streaming response 中途异常未 close。
- 每请求创建 Redis/Kafka client。
- WebSocket disconnect 未清理 group/connection。
- file upload 临时文件未关闭。

## 13. profiling 的生产安全边界

生产采样要控制风险：

```text
只采样单个 pod/worker
限制 duration，例如 30s
记录审批/事故号
避免 dump 请求 body、token、PII
确认容器具备 SYS_PTRACE 或相应权限
采样后清理 profile 文件
```

Kubernetes 常见限制：

```yaml
securityContext:
  capabilities:
    add: ["SYS_PTRACE"]
```

如果不能 ptrace，退而求其次：应用内 pyinstrument、OpenTelemetry span、endpoint 级 timing middleware。

## 作业

构造一个有内存泄漏和慢外部调用的最小 Django service，用 tracemalloc 与 py-spy 分别给出证据截图/文本，并提出修复。

## 评估标准

- 能按证据分类性能问题。
- 能选择合适 profiling 工具。
- 能避免无证据调参。
