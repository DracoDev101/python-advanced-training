# Python / Django 生产级后端系统训练营

> 面向已有 Python 基础、希望系统掌握 Django 生产级后端、异步任务、ASGI/WSGI 部署、并发模型与线上排障能力的学习者。重点不是语法入门，而是 Python 运行时机制、Django 工程化、Celery 任务系统、Daphne/Gunicorn 部署模型、greenlet/gevent/eventlet 与 monkey patch 边界，以及生产环境可靠性设计。

## 课程定位

这套资料围绕四个核心问题展开：

1. **Python 在后端生产环境里的真实边界是什么？**  
   理解对象模型、import 系统、GIL、内存管理、异常、typing、测试、性能剖析与并发模型的工程影响。

2. **Django 如何从单体 Web 框架演进为生产级业务系统骨架？**  
   覆盖 settings 分层、ORM/transaction、middleware、auth、admin、REST API、测试、缓存、文件、管理命令、迁移与多环境部署。

3. **Celery / ASGI / WSGI / greenlet 这些组件如何组合，边界在哪里？**  
   重点理解 worker 生命周期、broker/backend、幂等、重试、事务 outbox、Daphne、Gunicorn、Uvicorn worker、gevent/eventlet worker、monkey patch 的收益与风险。

4. **线上故障如何用证据定位？**  
   覆盖慢请求、慢 SQL、连接池耗尽、任务堆积、内存泄漏、死锁、阻塞 I/O、greenlet patch 污染、部署 reload 风险、observability 与 runbook。

## 学习目标

完成本课程后，学习者应该能够：

- 解释 Python 后端运行时、GIL、asyncio、thread/process/greenlet 的核心差异。
- 设计可测试、可观测、可维护的 Django 项目结构与业务分层。
- 正确使用 Django ORM、transaction、select_for_update、prefetch/select_related、migration。
- 构建可靠 Celery 任务：幂等、重试、超时、限流、路由、监控、死信/补偿。
- 选择并配置 Gunicorn、Daphne、Uvicorn、gevent/eventlet worker，并理解 monkey patch 边界。
- 用 profiling、日志、metrics、tracing、SQL explain、Celery events 定位生产问题。
- 完成一个生产级 Django + Celery + ASGI 综合项目。

## 推荐节奏

建议周期：**10–12 周**，共 24 lessons。每周 2 lessons + 1 次实验或项目增量。

每节课建议结构：

```text
1. 问题引入
2. 运行/框架机制
3. 代码实验
4. 生产实践
5. 故障证据
6. 工具验证
7. 作业与评估
```

## 目录结构

```text
python-advanced-training/
  README.md
  mkdocs.yml
  requirements.txt
  docs/
    index.md
    course-design.md
    syllabus.md
    lessons/
    labs/
    projects/
    references/
```

## 主线模块

1. Python 运行时与工程基础
2. Django 项目结构、请求链路与 ORM 深水区
3. API、认证、缓存、文件与后台管理
4. Celery 任务系统与可靠异步架构
5. WSGI/ASGI、Daphne、Gunicorn 与并发模型
6. greenlet/gevent/eventlet 与 monkey patch 边界
7. Observability、性能诊断与生产排障
8. 综合项目：Production Order Workflow

## Web 预览

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
mkdocs serve
```

构建静态站点：

```bash
mkdocs build --strict
```
