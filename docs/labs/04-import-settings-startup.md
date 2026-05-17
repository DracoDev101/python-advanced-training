# Lab 4：import side effect 与启动链路检查

## 目标

证明 import 顶层代码会执行，并设计 Django 启动链路检查清单。

## 实验 1：import side effect

创建 `bad_module.py`：

```python
print("importing bad_module")
STATE = []
STATE.append("created at import time")
```

创建 `probe_import.py`：

```python
import importlib
import sys

print("before import", "bad_module" in sys.modules)
import bad_module
print("after import", "bad_module" in sys.modules)

import bad_module
print("second import does not re-run top-level")

importlib.reload(bad_module)
print("reload re-runs top-level")
```

运行：

```bash
python probe_import.py
```

## 实验 2：Django 配置检查

在真实项目中运行：

```bash
python manage.py check --deploy --settings=config.settings.prod
python manage.py diffsettings --settings=config.settings.prod
```

记录：

- `DEBUG`
- `ALLOWED_HOSTS`
- cookie secure
- database engine/options
- cache/broker URL 来源

## 实验 3：入口检查

```bash
python -m gunicorn config.wsgi:application --check-config

python - <<'PY'
import os
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings.prod')
import config.asgi
print(config.asgi.application)
PY

celery -A config inspect ping
```

## 生产映射

回答：

1. 哪些代码不允许在 import 阶段执行？
2. `gunicorn --preload` 会改变什么？
3. 如果使用 gevent，`monkey.patch_all()` 应该放在哪里？

## 通过标准

- 能解释 `sys.modules` cache。
- 能列出 WSGI/ASGI/Celery 三个启动入口。
- 能给出 settings 分层检查结果。
