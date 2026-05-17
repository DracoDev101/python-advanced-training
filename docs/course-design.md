# 课程编排设计

本文档固化《Python / Django 生产级后端系统训练营》的课程设计决策，用于指导后续讲义、实验和综合项目的编写。

---

## 1. 课程定位

本课程不是 Python 或 Django 入门课，而是面向已有基础的后端学习者/工程师的进阶训练营。

课程核心目标是帮助学习者建立三种能力：

1. **解释能力**：能解释 Python runtime、Django request/ORM、Celery worker、WSGI/ASGI、greenlet/monkey patch 的机制和边界。
2. **实现能力**：能写出可测试、可观测、可维护、可部署的 Django 后端系统。
3. **判断能力**：能在同步、异步、任务队列、长连接、进程/线程/greenlet worker 之间做生产级取舍。

课程应避免成为 Django API 百科。每个主题都必须回到一个问题：

> 这个机制如何影响生产代码的性能、可靠性、部署方式和故障排查？

---

## 2. 学习者画像

### 2.1 默认已掌握

- Python 基础语法、函数、类、异常、venv/pip。
- Django 基础：model、view、url、admin、settings。
- 基础 SQL 与 HTTP 概念。
- 能运行简单 Django 项目与测试。

### 2.2 不默认掌握

- Python 对象模型、引用计数、GC、GIL 的生产影响。
- import side effect、settings 分层、启动链路污染。
- Django middleware、transaction、connection lifecycle。
- ORM N+1、锁等待、migration 发布风险。
- Celery worker 生命周期、prefetch、acks_late、visibility timeout。
- WSGI/ASGI server、Gunicorn/Daphne/Uvicorn worker 差异。
- gevent/eventlet/greenlet 的调度模型与 monkey patch 风险。
- profiling、observability、Celery events、SQL explain、阻塞排查。

---

## 3. 总体编排策略

采用 **运行时机制 → Django 框架深水区 → 异步任务 → 部署并发 → 生产排障** 的路线。

```text
Lesson 1–4：Python 后端运行时与工程基础
Lesson 5–12：Django 生产级项目、请求、ORM、API、测试
Lesson 13–16：Celery 任务系统与可靠异步架构
Lesson 17–21：WSGI/ASGI、Gunicorn、Daphne、greenlet、monkey patch
Lesson 22–24：性能诊断、observability、部署 runbook 与综合项目
```

### 3.1 内容比例

```text
Python runtime 与并发模型：25%
Django 生产工程：35%
Celery 与异步工作流：20%
部署、observability 与排障：20%
```

### 3.2 每个主题的闭环

```text
问题引入
→ 运行/框架机制
→ 最小可复现实验
→ 生产配置或代码模式
→ 故障证据
→ 工具验证
→ 作业与评估
```

---

## 4. 12 周路线图

| 周次 | 主题 | 重点产出 |
|---|---|---|
| Week 1 | Python 后端工程观、对象/内存 | runtime map、内存引用实验 |
| Week 2 | typing、import、settings | 配置分层模板、启动链路检查 |
| Week 3 | Django request/middleware、ORM | request trace、N+1 SQL evidence |
| Week 4 | transaction/migration | 锁等待实验、零停机 migration checklist |
| Week 5 | DRF/API、cache/file、testing | API schema、cache boundary、pytest suite |
| Week 6 | Celery 架构与可靠任务 | worker lifecycle lab、idempotent task |
| Week 7 | Outbox/workflow、Celery 排障 | outbox demo、queue backlog runbook |
| Week 8 | WSGI/ASGI、Gunicorn | server matrix、worker tuning evidence |
| Week 9 | Daphne/Channels、长连接 | websocket heartbeat/backpressure lab |
| Week 10 | asyncio/thread/process/greenlet、monkey patch | 并发模型对比、patch 污染实验 |
| Week 11 | profiling、observability、deployment | pprof-like Python profiling、dashboard fields |
| Week 12 | 综合项目 Review | Production Order Workflow 演练报告 |

---

## 5. Lesson 编写模板

```text
# Lesson N：标题

## 学习目标
## 关键问题
## 核心结论
## 机制解释
## 最小实验
## 生产实践
## 故障证据
## 工具验证
## 常见误区
## 作业
## 评估标准
## 延伸阅读
```

---

## 6. 实验原则

实验必须可运行、可验证、能看到证据。

推荐命令：

```bash
python -m pytest
python -m pytest -q --durations=10
python -m pip check
python -m django check --deploy
python manage.py showmigrations
python manage.py test
celery -A config inspect active
celery -A config events
python -m pyinstrument manage.py runserver
```

涉及外部组件时，优先提供 Docker Compose：PostgreSQL、Redis、RabbitMQ。无法稳定依赖外部组件时，用 in-memory fake 或 sqlite 复现实验机制，并在 README 中列真实生产命令。
