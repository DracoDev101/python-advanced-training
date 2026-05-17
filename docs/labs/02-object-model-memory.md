# Lab 2：引用、默认参数与 tracemalloc 证据

## 目标

复现两类内存问题：

1. 可变默认参数跨调用保留状态。
2. 全局 list 持有对象导致 Python heap 增长。

## 脚本

```python
import os
import resource
import tracemalloc


def rss_mb() -> float:
    return resource.getrusage(resource.RUSAGE_SELF).ru_maxrss / 1024


def bad_append(item, buffer=[]):
    buffer.append(item)
    return buffer


LEAK = []


def leak_round(n: int):
    for i in range(n):
        LEAK.append({"i": i, "payload": "x" * 1000})


if __name__ == "__main__":
    print("default arg:", bad_append(1))
    print("default arg:", bad_append(2))

    tracemalloc.start(25)
    s1 = tracemalloc.take_snapshot()
    for round_id in range(5):
        leak_round(10_000)
        print("round", round_id, "rss_mb", round(rss_mb(), 1), "objects", len(LEAK))
    s2 = tracemalloc.take_snapshot()

    for stat in s2.compare_to(s1, "lineno")[:5]:
        print(stat)
```

## 运行

```bash
python memory_probe.py
```

## 观察点

- 第二次 `bad_append` 是否看到第一次的数据？
- RSS 是否上升？
- `tracemalloc` top line 是否指向 `LEAK.append`？

## 生产映射

回答：

1. Django 中哪些全局变量会类似 `LEAK`？
2. 如果 `tracemalloc` 看不到增长但 RSS 增长，下一步查什么？
3. Gunicorn `max_requests` 在这里是止血还是根因修复？

## 通过标准

- 能用 snapshot diff 定位分配行。
- 能解释默认参数的创建时机。
- 能说明 Python heap 与 RSS 的差异。
