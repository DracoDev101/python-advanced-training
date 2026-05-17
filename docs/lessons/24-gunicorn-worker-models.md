# Lesson 24：Gunicorn 进程模型与 Worker 选择

## 学习目标

- 理解 Gunicorn master/worker、信号、preload、timeout 的生产含义。
- 能为 Django API、CPU 密集接口、慢外部 API、长响应选择 worker class。
- 能解释 `preload_app`、DB connection、copy-on-write 与内存证据。
- 能写出可观测、可灰度、可优雅退出的 Gunicorn 配置。

## 1. 架构模型

```text
master process
  ├─ worker 1
  ├─ worker 2
  └─ worker N
```

master 负责监听信号、fork worker、重启异常 worker。worker 负责处理请求。

常用信号：

```bash
kill -TTIN <master_pid>   # 增加一个 worker
kill -TTOU <master_pid>   # 减少一个 worker
kill -HUP  <master_pid>   # reload config/app
kill -TERM <master_pid>   # graceful shutdown
```

## 2. worker class 选择

| worker | 并发模型 | 适用 | 风险 |
|---|---|---|---|
| sync | 进程，每 worker 一请求 | 典型 Django API | 慢 I/O 占住 worker |
| gthread | 进程 + 线程 | 同步 I/O 较多 | GIL 下 CPU 不扩展，线程安全 |
| gevent/eventlet | greenlet 协作 I/O | 大量 socket I/O | monkey patch 污染，阻塞函数破坏并发 |
| uvicorn worker | ASGI event loop | async app/WebSocket | sync ORM 边界成本 |

经验起点：

```bash
gunicorn config.wsgi:application \
  --bind 0.0.0.0:8000 \
  --workers 4 \
  --worker-class sync \
  --timeout 30 \
  --graceful-timeout 30 \
  --max-requests 2000 \
  --max-requests-jitter 200
```

## 3. timeout 不是业务超时

Gunicorn `timeout` 是 worker 静默太久，master 会杀 worker。它不是 API SLA。业务应该有：

```text
nginx proxy_read_timeout
Gunicorn timeout
Django view 外部 API timeout
DB statement timeout / lock timeout
client timeout
```

层次必须递减或有明确关系，否则会出现上游先断、下游还在执行的幽灵请求。

## 4. preload_app

```bash
gunicorn config.wsgi:application --preload
```

优点：

- master 先 import app，再 fork worker。
- 利用 copy-on-write 降低 RSS。
- 启动更早暴露 import 错误。

风险：

- preload 阶段不要创建 DB/Redis/socket 连接，否则 fork 后连接被多个 worker 继承。
- AppConfig.ready 的副作用会在 master 执行。
- 随机种子、线程、定时器在 fork 前初始化会污染 worker。

证据：

```bash
ps -o pid,ppid,rss,cmd -C gunicorn
lsof -p <worker_pid> | grep TCP
```

## 5. max_requests 与内存泄漏缓解

`max_requests` 不是修复内存泄漏，而是损害控制：

```bash
--max-requests 2000 --max-requests-jitter 200
```

jitter 避免所有 worker 同时重启。

## 6. 优雅退出

部署滚动更新时必须满足：

```text
load balancer stop new traffic
→ worker finish in-flight request
→ close DB/Redis connections
→ process exit
```

配置：

```bash
--graceful-timeout 30 --timeout 35
```

Kubernetes 还需要 `terminationGracePeriodSeconds` 大于 graceful-timeout。

## 7. Runbook：502/504 突增

```text
1. nginx error log：upstream prematurely closed? timeout?
2. gunicorn error log：WORKER TIMEOUT? worker boot failed?
3. ps：worker 数是否不足、RSS 是否爆掉
4. py-spy：worker 卡 CPU 还是 I/O
5. DB/API 指标：下游是否慢
6. 调整：worker class/数量/timeout/隔离慢接口
7. 验证：502/504、p95、worker restart rate 回落
```

## 8. worker 数估算：不要只背公式

常见公式 `(2 × CPU) + 1` 只能作为起点。生产估算要看 workload：

```text
required_concurrency ≈ arrival_rate_rps × p95_service_time_seconds
```

例如：

```text
200 rps × 0.12s = 24 in-flight requests
```

如果 sync worker 每个进程一次处理一个请求，需要至少 24 个并发槽位，通常通过多 replicas × workers 提供；如果 gthread 每 worker 8 线程，要同时计算 DB 连接数：

```text
replicas × workers × threads ≤ DB max_connections 安全阈值
```

阻塞比例越高，线程/greenlet 越可能有收益；CPU 比例越高，多线程越不解决 GIL 竞争。

## 9. backlog、keep-alive 与 upstream 排队

请求可能还没进入 Django 就已经在排队：

```text
client → nginx accept queue → upstream keepalive pool → gunicorn listen backlog → worker accept
```

证据：

```bash
ss -lntp 'sport = :8000'
ss -tan state established '( sport = :8000 )' | wc -l
netstat -s | grep -i listen
```

Gunicorn 参数：

```bash
--backlog 2048
--keep-alive 2
```

如果 nginx 已经负责 client keep-alive，Gunicorn keep-alive 不应过长，否则空闲连接占用 worker/connection 资源。

## 10. graceful reload 时间线

```text
T0: load balancer 停止向旧 pod/instance 分配新流量
T1: master 收到 TERM/HUP
T2: worker 停止 accept 新请求
T3: in-flight requests 在 graceful-timeout 内完成
T4: worker 关闭 DB/Redis 连接并退出
T5: 新版本 ready 后接流量
```

Kubernetes 中：

```yaml
terminationGracePeriodSeconds: 45
readinessProbe:
  httpGet:
    path: /health/ready
preStop:
  exec:
    command: ["/bin/sh", "-c", "sleep 5"]
```

`terminationGracePeriodSeconds` 必须大于 Gunicorn `graceful-timeout`，否则 kubelet 会先 SIGKILL。

## 11. access log 字段

建议 access log 至少包含：

```text
request_id method path status duration_ms bytes user_agent upstream_addr worker_pid
```

Gunicorn 示例：

```bash
--access-logformat '%({x-request-id}i)s %(m)s %(U)s %(s)s %(D)s %(b)s %(p)s'
```

排障时用它回答：

- 是所有路由慢，还是某个路由慢？
- 慢请求集中在哪个 worker/pod？
- 499/504 是否与 deploy 或 worker timeout 对齐？

## 12. worker_tmp_dir 与容器环境

某些 worker heartbeat 或临时文件操作会触及文件系统。容器中如果 `/tmp` 或 overlayfs 抖动，会放大 latency。常见配置：

```bash
--worker-tmp-dir /dev/shm
```

验证：

```bash
df -h /tmp /dev/shm
iostat -xz 1
```

## 作业

为三个服务选择 Gunicorn worker：支付 API、报表生成 API、只读查询 API。说明 worker class、数量估算、timeout、max_requests、preload 是否开启和证据指标。

## 评估标准

- 能解释 master/worker 与信号。
- 能选择 worker class 并说明代价。
- 能识别 preload_app 副作用。
