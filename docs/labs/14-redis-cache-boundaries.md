# Lab 14：Redis Cache、热 key 与击穿保护

## 目标

为订单详情缓存设计生产方案，并写出 Redis 延迟上升 runbook。

## 任务

设计：

- key：包含 domain/version/scope/id。
- value：JSON 字段和大小上限。
- TTL：基础 TTL + jitter。
- miss：negative cache。
- hot key：singleflight/stale-while-revalidate。
- invalidation：`transaction.on_commit(cache.delete)`。
- metrics：hit/miss/latency/key_family。

## Redis 证据命令

```bash
redis-cli INFO stats
redis-cli INFO memory
redis-cli INFO commandstats
redis-cli SLOWLOG GET 20
redis-cli LATENCY DOCTOR
redis-cli --hotkeys
```

## 通过标准

- 明确 MySQL 是 source of truth。
- 缓存失效不在事务提交前执行。
- Redis lock 使用 token + Lua 释放。
- 说明 Redis cache、session、broker、stream 为什么应隔离。
