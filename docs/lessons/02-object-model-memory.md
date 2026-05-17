# Lesson 2：对象模型、引用与内存证据

## 学习目标

完成本课后，你应该能够：

- 解释 Python 中变量、对象、引用之间的关系，而不是把变量理解成“盒子”。
- 判断可变对象、默认参数、闭包、全局缓存如何造成生产内存问题。
- 区分 Python heap 增长、RSS 增长、C extension/native memory 增长。
- 使用 `sys.getrefcount`、`gc`、`tracemalloc` 获得内存证据。

## 关键问题

1. 为什么 `list.append` 会影响多个引用看到的同一个对象？
2. 为什么函数默认参数可能在生产中积累状态？
3. 为什么容器 RSS 上升，不一定能从 `tracemalloc` 看到全部来源？
4. Django 项目中最常见的内存泄漏入口是什么？

## 核心结论

Python 内存问题不能只看“对象有没有被 del”。更重要的是引用图：

```text
module global
  → cache dict
      → request/user/order objects
          → related objects
              → large payload
```

只要从 GC roots 仍然可达，对象就不会释放。

生产排查的核心不是猜，而是采证：

```text
RSS 是否增长？
→ Python heap 是否增长？
→ top allocation traceback 是什么？
→ 是否有全局容器/缓存/闭包/信号 handler 持有对象？
→ 是否是 C extension/native memory？
```

## 1. Python 变量不是对象本身

```python
orders = []
alias = orders
alias.append("order-1")
print(orders)  # ['order-1']
```

`orders` 和 `alias` 是两个名字，指向同一个 list 对象。

```text
orders ─┐
        ├──> list object: ['order-1']
alias  ─┘
```

这影响 Django 生产代码：

```python
class OrderCollector:
    def __init__(self):
        self.items = []

collector = OrderCollector()  # module global
```

如果 `collector` 是模块级单例，worker 进程生命周期内它不会随着请求结束释放。

## 2. 可变默认参数：小 bug，大事故

错误示例：

```python
def add_event(event: dict, buffer: list[dict] = []) -> list[dict]:
    buffer.append(event)
    return buffer
```

实验：

```python
print(add_event({"id": 1}))
print(add_event({"id": 2}))
```

输出：

```text
[{'id': 1}]
[{'id': 1}, {'id': 2}]
```

默认参数在函数定义时创建一次，不是每次调用创建。

正确写法：

```python
def add_event(event: dict, buffer: list[dict] | None = None) -> list[dict]:
    if buffer is None:
        buffer = []
    buffer.append(event)
    return buffer
```

生产影响：

- 请求之间串数据。
- 单个 worker 内存持续增长。
- 测试单独跑不暴露，长时间压测才暴露。

## 3. 引用计数与循环引用

CPython 主要使用引用计数，辅以 cyclic GC。

```python
import sys

x = []
print(sys.getrefcount(x))  # 注意：调用 getrefcount 本身会临时加一个引用
```

循环引用示例：

```python
class Node:
    def __init__(self):
        self.other = None


a = Node()
b = Node()
a.other = b
b.other = a

del a, b
```

如果没有外部引用，cyclic GC 可以回收。但带 `__del__`、外部资源、复杂引用链时，回收时机和行为需要验证。

## 4. `tracemalloc`：看 Python 分配栈

最小实验：

```python
import tracemalloc

leak = []


def allocate():
    for i in range(10_000):
        leak.append({"i": i, "payload": "x" * 100})


tracemalloc.start(25)
s1 = tracemalloc.take_snapshot()
allocate()
s2 = tracemalloc.take_snapshot()

for stat in s2.compare_to(s1, "lineno")[:10]:
    print(stat)
```

运行后你会看到哪一行分配最多。

生产使用建议：

- 在 staging 或可控压测环境启用。
- 不要在高流量生产长期开启深 traceback。
- 用两次 snapshot 比较，而不是只看一次 snapshot。

## 5. RSS vs Python heap

容器里常看 RSS：

```bash
ps -o pid,ppid,cmd,rss,%mem -C gunicorn
```

但 RSS 包括：

- Python heap。
- C extension 分配。
- mmap。
- allocator arenas 未归还 OS 的内存。
- copy-on-write 破坏后的私有页。

`tracemalloc` 只追踪 Python allocator 管理的内存，不覆盖所有 native memory。

如果 RSS 涨，但 `tracemalloc` 看不到明显增长，要考虑：

- `numpy`/Pillow/cryptography 等 C extension。
- 大文件读入内存。
- gunicorn `preload_app` 后 worker 写全局对象导致 copy-on-write 失效。
- gevent/eventlet monkey patch 后第三方库行为变化。

## 6. Django 常见内存泄漏入口

### 6.1 模块级 cache

```python
# bad
_user_permissions_cache: dict[int, set[str]] = {}
```

风险：

- 无 TTL。
- 无大小上限。
- 多 worker 各自一份。
- 用户越多内存越大。

改法：

- 用 Redis cache，设置 TTL 和 key 设计。
- 本地 cache 必须有 size limit，例如 LRU。

### 6.2 request 对象被后台任务/闭包持有

```python
def view(request):
    def later():
        logger.info("user=%s", request.user.id)
    background_callbacks.append(later)
```

`later` 闭包持有 `request`，而 request 又可能持有 user、session、body、resolver_match 等对象。

改法：只传最小值：

```python
def view(request):
    user_id = request.user.id
    def later():
        logger.info("user=%s", user_id)
```

### 6.3 QuerySet 被缓存而不是结果被分页

```python
# bad
cached_queryset = Order.objects.filter(status="pending")
```

QuerySet lazy evaluation 容易让人误判。更糟的是被 evaluate 后持有大量 model instances。

生产建议：

- 缓存 ID 列表或聚合结果，不缓存大 QuerySet。
- 大批量遍历用 `.iterator(chunk_size=...)`。
- 管理命令处理大表时避免一次性 `list(queryset)`。

## 7. 工具验证

### 7.1 Python heap 证据

```bash
python -X tracemalloc=25 manage.py test
```

或在代码里：

```python
import tracemalloc
tracemalloc.start(25)
```

### 7.2 GC 统计

```python
import gc
print(gc.get_count())
print(gc.get_stats())
```

### 7.3 进程 RSS

```bash
ps -o pid,ppid,cmd,rss,%mem -C gunicorn
```

### 7.4 容器视角

```bash
docker stats
cat /sys/fs/cgroup/memory.current 2>/dev/null || true
```

## 8. 常见误区

### 误区 1：`del x` 等于释放内存给 OS

`del x` 只是删除一个引用。对象是否释放取决于是否还有其他引用。即使对象释放，底层 allocator 也未必立刻把内存还给 OS。

### 误区 2：只看 `tracemalloc` 就能解释所有内存上涨

如果是 native memory 或 allocator 行为，`tracemalloc` 可能看不到。

### 误区 3：本地开发没问题，生产就不会泄漏

开发环境请求少、进程生命周期短、数据量小。生产 worker 可能运行几天，泄漏会累积。

### 误区 4：用 Gunicorn `max_requests` 替代修复

`max_requests` 可以作为止血手段，让 worker 周期性重启，但它不是根因修复。必须保留证据并定位引用来源。

## 9. 生产实践：内存保护配置

Gunicorn 示例：

```bash
gunicorn config.wsgi:application \
  --workers 4 \
  --max-requests 2000 \
  --max-requests-jitter 200 \
  --timeout 30 \
  --graceful-timeout 30
```

解释：

- `max-requests`：周期性回收 worker，缓解泄漏扩大。
- `jitter`：避免所有 worker 同时重启。
- `timeout`：避免 worker 被长时间卡死。

注意：如果 worker 每 100 个请求就 OOM，不能只调小 `max_requests`，要先抓内存证据。

## 10. 课堂讨论题

1. 为什么 `preload_app=True` 可能省内存，也可能因为 copy-on-write 失效导致问题？
2. Django 里哪些对象绝对不应该放进全局 list/dict？
3. `tracemalloc` 无明显增长但 RSS 涨，下一步查什么？
4. 大批量导出订单时，为什么 `list(Order.objects.all())` 危险？

## 11. 作业

写一个脚本，模拟两类内存增长：

1. Python list 全局持有对象，`tracemalloc` 能看到。
2. 大 `bytes` 或文件读取导致 RSS 上升。

要求输出：

- 每轮 RSS。
- `tracemalloc` top 5。
- 解释两者差异。

## 12. 评估标准

- 能解释变量/引用/对象的关系。
- 能识别可变默认参数和全局 cache 风险。
- 能用 `tracemalloc` 做 snapshot diff。
- 能区分 Python heap 与 RSS。
- 能给出 Django 内存泄漏的第一批证据。
