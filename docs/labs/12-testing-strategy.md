# Lab 12：订单流程测试矩阵

## 目标

为订单创建流程设计完整测试矩阵。

## 必须包含

| 类型 | 是否 DB | 目标 | 关键断言 |
|---|---|---|---|
| domain unit | 否 | 状态机/金额 | 状态转换正确 |
| service DB | 是 | 创建订单事务 | order/outbox 同时写入 |
| API | 是 | 协议/权限 | status/schema/error code |
| idempotency | 是 | 重复请求 | 同 key 同响应/冲突 |
| query count | 是 | N+1 防护 | 固定 SQL 数 |
| Celery eager | 可选 | task 函数逻辑 | payload/schema |
| event contract | 否 | Kafka/Stream schema | jsonschema 通过 |
| integration | 是/外部 | MySQL/Redis/Celery | smoke path 通过 |

## 命令

```bash
python -m pytest -q
python -m pytest -q --durations=20
python manage.py check --deploy --settings=config.settings.prod
python manage.py migrate --plan
```

## 通过标准

- 不 mock ORM 事务/锁。
- 明确 Celery eager mode 边界。
- 有 event contract test。
- PR fast checks 与 integration checks 分离。
