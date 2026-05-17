# Lesson 29：Observability、部署与 Runbook

## 学习目标

- 能设计日志、指标、追踪三件套，并贯通 HTTP、DB、Celery、Kafka、Redis Stream。
- 能编写部署健康检查、ready/liveness、迁移策略和回滚策略。
- 能把事故处理写成“症状 → 证据 → 命令 → 修复 → 验证”的 runbook。
- 能定义 SLI/SLO 与告警，避免噪音告警。

## 1. 结构化日志字段

最小字段：

```json
{
  "ts": "2026-05-18T00:00:00Z",
  "level": "INFO",
  "service": "orders-api",
  "env": "prod",
  "request_id": "req_1",
  "user_id": "u_1",
  "order_id": "o_1",
  "task_id": "celery_1",
  "event_id": "evt_1",
  "duration_ms": 42,
  "error_type": null
}
```

字段要短、稳定、可索引，不要把整段 payload 打进日志。

## 2. 指标

核心 RED/USE：

```text
http_requests_total{route,status}
http_request_duration_seconds_bucket{route}
http_errors_total{route,error_type}
db_query_duration_seconds_bucket{op,table}
db_pool_wait_seconds
celery_queue_depth{queue}
celery_task_runtime_seconds_bucket{task}
kafka_consumer_lag{group,topic,partition}
redis_stream_pending{stream,group}
```

告警优先基于用户影响：错误率、延迟、业务 lag，而不是单点 CPU。

## 3. tracing

一次订单支付链路：

```text
HTTP POST /orders
→ MySQL transaction
→ Outbox write
→ publisher Celery task
→ Kafka produce
→ consumer read model update
→ Redis cache invalidation
```

trace 要携带：request_id、trace_id、event_id、aggregate_id。

## 4. 部署策略

发布前：

```bash
python manage.py check --deploy
python manage.py migrate --plan
python manage.py sqlmigrate app 0123
python -m pytest
mkdocs build --strict
```

生产迁移：优先 expand/contract：

```text
1. add nullable column / new table
2. deploy code writing both
3. backfill in batches
4. switch read path
5. remove old column later
```

## 5. 健康检查

- liveness：进程是否活着，不要依赖 DB。
- readiness：是否能接流量，可检查 DB/Redis 基础连通和迁移版本。
- startup：冷启动期间允许更长时间。

## 6. Runbook 模板

```text
标题：Celery critical queue lag 高
触发：oldest age > 60s for 5m
影响：支付确认延迟
证据命令：celery inspect active/reserved, broker queue depth, task p95
判断分支：worker down / downstream slow / poison task / prefetch
修复动作：扩容、限流、DLQ、暂停非关键任务
验证：oldest age < 10s, retry rate 回落, 用户延迟恢复
回滚：恢复上一版本 worker image
```

## 7. 事故复盘

复盘不是追责。必须产出：

- 用户影响窗口。
- 检测时间、确认时间、缓解时间、恢复时间。
- 哪个告警有效，哪个噪音。
- 哪个 runbook 缺失。
- 长期修复项与 owner。

## 作业

为综合项目写一页生产运行手册：SLI/SLO、dashboard、告警、部署 checklist、回滚步骤、五个核心 runbook。

## 评估标准

- 能贯通日志、指标、追踪。
- 能设计不误杀的健康检查。
- 能写出可执行 runbook，而不是抽象建议。
