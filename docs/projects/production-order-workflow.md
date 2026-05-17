# 综合项目：Production Order Workflow

## 项目目标

实现一个生产级 Django 后端系统，模拟订单创建、支付确认、库存预留、异步通知、WebSocket 状态推送和后台补偿流程。

## 覆盖能力

- Django REST API 与权限
- PostgreSQL transaction、row lock、migration
- Redis cache 与 Channels channel layer
- Celery 异步任务、幂等、重试、outbox、补偿
- Gunicorn + Daphne 分离部署
- observability：logs、metrics、traces
- 故障演练：慢 SQL、队列堆积、worker crash、WebSocket 断连

## 推荐目录

```text
config/
orders/
payments/
inventory/
notifications/
common/
```

## 核心工作流

```text
POST /orders
→ 创建订单 pending
→ transaction.on_commit 写 outbox / 投递 Celery
→ reserve inventory
→ payment callback
→ publish order status event
→ WebSocket 推送状态
→ failed tasks retry / compensation
```

## 评估维度

- 数据一致性：重复请求、重复 task、并发扣库存是否安全。
- 可靠性：worker crash 后是否能恢复。
- 可观测性：能否从 request id / order id / task id 串起链路。
- 部署性：Gunicorn/Daphne/Celery/Beat 是否能独立扩缩容。
- 排障性：是否提供症状、证据、命令、修复、验证的 runbook。
