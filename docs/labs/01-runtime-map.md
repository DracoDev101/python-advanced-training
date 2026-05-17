# Lab 1：Python 后端运行时地图与慢请求拆解

## 目标

用一个最小脚本区分 I/O-bound 与 CPU-bound，并把结果映射到 Django/Gunicorn/Celery 的部署决策。

## 文件

创建：

```text
labs-runtime/runtime_probe.py
```

内容：

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

## 运行

```bash
python labs-runtime/runtime_probe.py
```

## 观察点

记录表格：

| function | workers=1 | workers=4 | workers=16 | 解释 |
|---|---:|---:|---:|---|
| io_bound | | | | |
| cpu_bound | | | | |

## 生产映射

回答：

1. 如果 Django 请求主要等待外部 HTTP API，Gunicorn `gthread` 是否有意义？
2. 如果 Celery task 主要做 CPU-heavy 加密/图片处理，为什么 prefork 更合理？
3. 如果 HTTP P95 上升，哪些证据能证明是 worker 排队，而不是 DB 慢？

## 通过标准

- 能解释 I/O-bound 在线程下变快的原因。
- 能解释 CPU-bound 不会随线程线性扩展的原因。
- 能把实验结论映射到 Gunicorn/Celery worker 类型选择。
