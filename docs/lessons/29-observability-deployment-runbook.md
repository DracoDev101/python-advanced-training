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

## 8. Metrics cardinality：最常见的自毁点

不要把高基数字段放进 metric label：

| 禁止 label | 原因 |
|---|---|
| user_id | 用户数无限增长 |
| order_id | 每个订单一个 time series |
| request_id | 每个请求一个 time series |
| exception_message | 文本无限变化 |
| raw path `/orders/123` | path 参数爆炸 |

正确做法：

```text
route=/orders/{id}
status=2xx/4xx/5xx 或具体 status
error_type=ProviderTimeout
queue=critical
```

日志可以承载高基数字段，指标只承载有限维度。

## 9. Histogram bucket 按 SLA 设计

如果 API SLO 是 p95 < 200ms，bucket 应围绕阈值加密：

```python
HTTP_BUCKETS = [0.005, 0.01, 0.025, 0.05, 0.1, 0.2, 0.3, 0.5, 1, 2, 5]
```

队列 age bucket：

```python
QUEUE_AGE_BUCKETS = [1, 5, 10, 30, 60, 120, 300, 600, 1800]
```

不要只用平均值。平均值会掩盖尾延迟和少数老消息。

## 10. OpenTelemetry 贯通异步链路

HTTP trace 自动传播到 Celery/Kafka 需要显式注入上下文：

```python
from opentelemetry.propagate import inject, extract

headers = {}
inject(headers)
produce_event(payload, headers=headers)
```

consumer：

```python
ctx = extract(event.headers)
with tracer.start_as_current_span("process_order_event", context=ctx):
    process(event)
```

Celery task headers 也应携带 trace/request/event：

```python
capture_payment.apply_async(args=[payment_id], headers={"traceparent": traceparent, "request_id": request_id})
```

否则 trace 会在异步边界断开，事故中无法解释“HTTP 成功但后台为什么慢”。

## 11. Canary 发布观察窗口

Canary 不只是“先发 5% 流量”。必须定义观察指标与退出条件：

```text
窗口：10-30 分钟，覆盖一个业务高峰周期更好
核心：5xx、p95/p99、DB slow query、Celery retry、Kafka lag
对照：新旧版本同 route 同 percentile
退出：任一核心 SLO 破坏立即 rollback
```

发布 checklist：

```text
migration 已完成 expand 阶段
新旧版本可同时读写
feature flag 可关闭
rollback 不依赖 reverse destructive migration
dashboard 已过滤 version label
```

## 12. Error budget 与告警分级

告警分级：

```text
P0：用户核心路径不可用，立即唤醒
P1：SLO 快速消耗，工作时间外也通知
P2：容量/趋势风险，工作时间处理
P3：优化建议，不分页
```

避免每个组件独立大喊。优先告用户影响：

```text
order_create_error_rate > 2% for 5m
payment_confirm_lag_p95 > 60s for 10m
critical_queue_oldest_age > 120s for 5m
```

组件指标用于定位，不一定都要分页。

## 作业

为综合项目写一页生产运行手册：SLI/SLO、dashboard、告警、部署 checklist、回滚步骤、五个核心 runbook。

## 评估标准

- 能贯通日志、指标、追踪。
- 能设计不误杀的健康检查。
- 能写出可执行 runbook，而不是抽象建议。
