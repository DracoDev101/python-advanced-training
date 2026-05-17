# Lab 5：Django 项目结构与 settings 分层设计

## 目标

为 `Production Order Workflow` 设计一个可生产演进的 Django 项目骨架。

## 任务

画出目录结构，至少包含：

```text
config/settings/base.py
config/settings/dev.py
config/settings/test.py
config/settings/prod.py
orders/services.py
orders/repositories.py
orders/events.py
orders/tasks.py
orders/queries.py
```

## 输出要求

1. 标注每个 module 的职责。
2. 写出 WSGI、ASGI、Celery worker、Kafka consumer 入口命令。
3. 写出生产必需环境变量清单。
4. 写出 `python manage.py check --deploy` 在 CI 中的位置。

## 验证命令

```bash
python manage.py check --deploy --settings=config.settings.prod
python manage.py diffsettings --settings=config.settings.prod
```

## 通过标准

- view 不包含业务流程。
- service 不依赖 HTTP request/response。
- settings 不做外部 I/O。
- 不同入口的配置差异被显式列出。
