# Lesson 30：综合项目 Review 与生产演练

## 学习目标

- 用生产评审方式整合全课程：Django、MySQL、Redis、MongoDB、Redis Stream、Celery、Kafka、WSGI/ASGI、观测与部署。
- 能从架构、可靠性、性能、安全、运维五个维度 review 一个后端系统。
- 能设计故障注入和恢复演练。
- 能给出上线前 go/no-go 判断。

## 1. 综合项目目标

项目：`Production Order Workflow`

核心链路：

```text
HTTP API 创建订单
→ MySQL 写订单/库存/Outbox
→ Celery publisher 发布事件
→ Kafka/Redis Stream 事件流
→ Consumer 更新 MongoDB read model
→ Redis cache/session
→ WebSocket 推送订单状态
→ Observability + Runbook
```

## 2. 架构 review 清单

```text
边界：API / domain service / repository / event publisher 是否清晰
一致性：DB transaction 与消息发布是否有 outbox
幂等：API idempotency key、task/event consumer dedup 是否完整
数据：MySQL 索引/锁、Mongo read model、Redis key/TTL
异步：Celery retry/time limit、Kafka commit/DLQ、Stream PEL
Server：Gunicorn/Daphne worker 与连接池是否匹配
```

## 3. 可靠性 review

必须回答：

- API 请求超时后，订单是否可能已创建？客户端如何查询？
- Celery task 成功但 ack 丢失会不会重复副作用？
- Kafka consumer 处理成功但 commit 失败会怎样？
- Mongo read model 落后时 API 是否降级？
- WebSocket 丢消息后如何恢复状态？

## 4. 性能 review

压测前先定义 SLO：

```text
create order p95 < 200ms
order detail p95 < 100ms
critical celery oldest age < 10s
consumer lag < 1000 messages for 5m
websocket reconnect success > 99%
```

压测证据：

```bash
hey -n 10000 -c 100 http://localhost:8000/api/orders
py-spy record -o api.svg --pid <pid> --duration 30
redis-cli --latency
mysql -e 'SHOW ENGINE INNODB STATUS\G'
```

## 5. 故障注入

| 故障 | 期望证据 | 期望恢复 |
|---|---|---|
| MySQL 慢查询 | p95 上升、slow log | 索引/查询修复 |
| Redis 热 key | Redis CPU/latency 上升 | key 拆分/本地缓存/限流 |
| Celery worker OOM | worker lost、queue age | max_requests/拆任务/内存修复 |
| Kafka lag | consumer_lag 上升 | 扩容/修 poison/DLQ |
| WebSocket 慢客户端 | send queue 上升 | backpressure/断开 |
| 外部 API 429 | retry storm | jitter/circuit breaker |

## 6. 上线 go/no-go

Go 条件：

- 所有 migration 可 expand/contract 或已评审锁风险。
- 所有核心副作用有幂等键。
- Dashboard 与告警覆盖用户影响。
- 回滚路径已演练。
- 数据修复脚本有 dry-run。

No-go 条件：

- Outbox pending 无监控。
- Consumer 无 DLQ/replay。
- 长连接无 heartbeat/backpressure。
- 大表 migration 需要长时间锁表。
- 只有成功路径测试，无故障注入。

## 7. 最终交付物

学生需要提交：

```text
architecture.md
api-contract.md
schema-and-migrations.md
event-contract.md
runbook.md
load-test-report.md
failure-drill-report.md
```

## 8. 评审问题

1. 如果支付 provider 成功但响应超时，你的系统如何不重复扣款？
2. 如果 Kafka 重放 24 小时事件，read model 是否保持正确？
3. 如果 Redis 全部丢失，哪些能力降级，哪些必须恢复？
4. 如果 Web worker 全部滚动重启，in-flight 请求如何处理？
5. 如果某个 migration 发布后发现慢查询，如何回滚？

## 9. Final rubric：100 分

```text
架构边界与代码组织：15
API 契约、权限与幂等：15
数据一致性与事务/Outbox：20
异步系统可靠性：15
性能与容量证据：10
Observability 与 Runbook：15
故障演练与恢复：10
```

扣分项：

```text
-20：核心副作用无幂等
-15：无 outbox 或双写失败窗口未解释
-10：无 migration 回滚/expand-contract 策略
-10：无 DLQ/replay 方案
-10：压测报告只有平均值，无 p95/p99
-10：runbook 无具体命令
```

## 10. architecture.md 模板

```markdown
# Architecture

## Context
## Components
## Request flow
## Data model
## State machine
## Event flow
## Consistency boundaries
## Failure modes
## Scaling plan
## Security and permissions
## Observability
```

必须包含一张文本图或 Mermaid 图，展示 HTTP、MySQL、Outbox、Celery、Kafka/Redis Stream、Mongo read model、Redis cache、WebSocket。

## 11. load-test-report.md 模板

```markdown
# Load Test Report

## Goal
## Environment
## Dataset
## Command
## Result Summary
- rps
- p50/p95/p99
- error rate
- CPU/RSS
- DB query p95
- queue lag

## Bottleneck Evidence
## Fix Attempt
## Before/After
## Remaining Risk
```

禁止只提交 `hey` 输出。必须解释瓶颈在哪里，以及是否会影响生产 SLO。

## 12. failure-drill-report.md 模板

```markdown
# Failure Drill Report

## Scenario
## Hypothesis
## Injection Method
## Expected Signals
## Actual Signals
## User Impact
## Mitigation
## Recovery Verification
## Follow-up Fixes
```

至少覆盖：

```text
MySQL lock wait
Redis latency/hot key
Celery worker crash
Kafka consumer lag/poison event
WebSocket slow client/disconnect storm
```

## 13. Go/No-Go meeting agenda

```text
1. 本次发布范围
2. schema/migration 风险
3. 新旧版本兼容性
4. 幂等与补偿路径
5. dashboard/alert 是否就绪
6. rollback 条件与负责人
7. 数据修复脚本 dry-run 结果
8. go/no-go 决策
```

No-go 的价值在于提前阻止不可恢复风险，不是证明团队失败。

## 评估标准

- 能跨组件解释端到端一致性。
- 能用证据驱动性能与可靠性判断。
- 能给出可执行上线 checklist 与故障演练报告。
