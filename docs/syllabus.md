# Python / Django 生产级后端系统训练营：课程大纲

## 模块一：Python 后端运行时与工程基础

### Lesson 1：Python 后端工程观与运行时地图

- Python 在 Web 后端中的优势与约束
- 解释器、进程、线程、协程、greenlet 的全局地图
- 同步请求、异步请求、后台任务、长连接的不同运行路径
- 生产系统中“慢”的常见来源：CPU、I/O、锁、DB、队列、网络

### Lesson 2：对象模型、引用与内存证据

- object header、引用、可变/不可变对象
- 引用计数、循环引用、gc module
- copy/deepcopy、默认参数陷阱
- 内存泄漏证据：tracemalloc、objgraph、RSS vs Python heap

### Lesson 3：类型系统、契约与工程边界

- gradual typing 的定位
- mypy/pyright 能防什么，不能防什么
- Protocol、TypedDict、dataclass、pydantic 的边界
- 类型契约与 Django dynamic model 的冲突处理

### Lesson 4：import、配置与项目启动链路

- import cache、module side effect、circular import
- Django settings 分层与环境变量
- AppConfig.ready 的边界
- gunicorn/daphne/celery 启动链路差异

## 模块二：Django 生产级工程深水区

### Lesson 5：Django 项目结构与 settings 分层

- config/apps 分层
- domain/service/repository 在 Django 项目中的取舍
- settings base/dev/prod/test
- secrets、12-factor、django-environ

### Lesson 6：请求生命周期、middleware 与异常链路

- URL resolver、view、template/response
- middleware 顺序与 short-circuit
- DB connection lifecycle
- exception handler、structured error、request id

### Lesson 7：ORM 查询、N+1 与 SQL 证据

- QuerySet lazy evaluation
- select_related/prefetch_related
- annotate/aggregate/subquery
- django-debug-toolbar、connection.queries、EXPLAIN

### Lesson 8：事务、锁与一致性边界

- autocommit、atomic、savepoint
- select_for_update、deadlock、lock timeout
- on_commit 与异步任务投递
- isolation level 与业务不变量

### Lesson 9：Migration、数据演进与发布风险

- schema migration vs data migration
- backward-compatible migration
- large table alter 风险
- expand/contract 发布策略

### Lesson 10：DRF/API 设计、认证与权限

- serializer validation 与 domain validation
- ViewSet/APIView 取舍
- JWT/session/token auth
- permission、throttle、pagination、error schema

### Lesson 11：缓存、Session、文件与外部资源

- cache aside、key design、TTL jitter
- session backend 选择
- file upload、storage、signed URL
- 外部 API timeout/retry/circuit breaker

### Lesson 12：测试体系、factory、fixtures 与契约测试

- pytest-django、transactional_db
- factory_boy、freezegun、responses
- service 层单测 vs API 集成测试
- Celery eager mode 的误区

## 模块三：Celery 任务系统与可靠异步架构

### Lesson 13：Celery 架构、Broker 与 Worker 生命周期

- broker/backend/worker/beat
- Redis vs RabbitMQ 取舍
- worker pool：prefork、threads、gevent、eventlet、solo
- prefetch、acks、visibility timeout

### Lesson 14：任务幂等、重试、超时与限流

- task idempotency key
- retry/backoff/jitter
- soft/hard time limit
- rate limit、routing、priority queue
- poison message 与人工补偿

### Lesson 15：事务 Outbox、任务编排与补偿

- Django transaction.on_commit 的边界
- outbox pattern
- chain/group/chord 的失败语义
- saga/compensation

### Lesson 16：Celery 监控、队列堆积与排障

- inspect active/reserved/scheduled
- events、Flower、Prometheus exporter
- queue latency、task runtime、failure rate
- worker lost、OOM、broker reconnect

## 模块四：WSGI/ASGI、Server 与并发模型

### Lesson 17：WSGI、ASGI 与 Python Web Server 模型

- WSGI sync callable
- ASGI scope/receive/send
- Django sync/async view 边界
- sync_to_async / async_to_sync 成本

### Lesson 18：Gunicorn 进程模型与 Worker 选择

- master/worker 信号模型
- sync/gthread/gevent/eventlet/uvicorn worker
- timeout、graceful-timeout、max-requests
- preload_app 与内存/连接副作用

### Lesson 19：Daphne、Channels 与长连接

- Daphne 在 ASGI 栈中的定位
- Channels、channel layer、Redis
- WebSocket heartbeat、backpressure
- 长连接部署与水平扩展

### Lesson 20：asyncio、线程、进程与 greenlet 对比

- cooperative vs preemptive scheduling
- CPU-bound vs I/O-bound
- ThreadPoolExecutor/ProcessPoolExecutor
- asyncio cancellation 与 timeout
- greenlet context switch 模型

### Lesson 21：gevent/eventlet 与 monkey patch 边界

- monkey.patch_all 改了什么
- socket/ssl/thread/time/select/subprocess 影响面
- patch 时机：必须早于 import I/O 库
- Django/Celery/Gunicorn gevent worker 风险
- 如何检测阻塞和 patch 污染

## 模块五：诊断、部署与综合项目

### Lesson 22：性能剖析、内存泄漏与阻塞定位

- pyinstrument、cProfile、scalene
- tracemalloc、memray
- py-spy、strace、lsof
- slow SQL、lock wait、connection pool exhaustion

### Lesson 23：Observability、部署与 Runbook

- structured logging、request id、task id
- metrics：latency、error、queue depth、DB pool
- tracing：HTTP → DB → Celery
- docker compose、systemd、Kubernetes 基础部署策略
- runbook：症状 → 证据 → 命令 → 修复 → 验证

### Lesson 24：综合项目 Review 与生产演练

- 架构 review
- 可靠性 review
- 性能压测
- 故障注入：慢 SQL、队列堆积、worker OOM、WebSocket 断连
- 发布与回滚演练
