# Lesson 11：缓存、Session、文件与外部资源

## 学习目标

完成本课后，你应该能够：

- 设计 Redis cache key、TTL、失效策略，并解释缓存一致性边界。
- 区分缓存穿透、击穿、雪崩、热 key 的症状和证据。
- 选择合适的 Django session backend，并理解安全/容量风险。
- 设计文件上传、对象存储、signed URL、病毒扫描/异步处理流程。
- 为外部 API 调用设置 timeout、retry、circuit breaker 和观测字段。

## 关键问题

1. Redis cache 是否能替代数据库一致性设计？
2. 缓存失效应该 delete、更新，还是等 TTL？
3. Session 放 DB、cache、cookie 各有什么风险？
4. 文件上传为什么不应该直接走 Django worker 大内存处理？

## 核心结论

缓存和外部资源都是一致性边界。生产设计要明确：

```text
source of truth 是谁？
cache key 如何版本化？
TTL 多长？
写入后如何失效？
缓存未命中会不会打爆 DB？
外部资源失败如何降级？
```

## 1. Redis cache aside

读取路径：

```python
from django.core.cache import cache


def get_order_summary(order_id: int) -> dict:
    key = f"order_summary:v1:{order_id}"
    cached = cache.get(key)
    if cached is not None:
        return cached

    summary = build_order_summary_from_db(order_id)
    cache.set(key, summary, timeout=300)
    return summary
```

写入路径：

```python
def update_order_status(order_id: int, status: str):
    with transaction.atomic():
        Order.objects.filter(id=order_id).update(status=status)
        transaction.on_commit(lambda: cache.delete(f"order_summary:v1:{order_id}"))
```

为什么用 `on_commit`：避免事务回滚但缓存已删/已写，引入不一致窗口。

## 2. Cache key 设计

推荐：

```text
<domain>:<schema_version>:<tenant/user/scope>:<id/hash>
```

示例：

```text
order_summary:v1:order:123
user_permissions:v3:user:42
api_rate_limit:v1:user:42:/orders
```

要求：

- 包含版本号，方便 schema 变化整体失效。
- 包含租户/用户 scope，避免串数据。
- key 长度可控。
- 不把敏感 token 裸放 key。

## 3. TTL jitter

错误：所有 key 都 300 秒过期。

```python
cache.set(key, value, timeout=300)
```

风险：同一时间大量 key 过期，打爆 DB。

改法：

```python
import random

timeout = 300 + random.randint(-30, 30)
cache.set(key, value, timeout=timeout)
```

## 4. 缓存问题分类

| 问题 | 现象 | 常见修复 |
|---|---|---|
| 穿透 | 不存在的 key 反复打 DB | cache null、Bloom filter、参数校验 |
| 击穿 | 热 key 过期瞬间大量请求打 DB | mutex/singleflight、提前刷新 |
| 雪崩 | 大量 key 同时过期/Redis 故障 | TTL jitter、降级、限流 |
| 热 key | 单 key QPS 极高 | 本地缓存、分片 key、异步刷新 |

证据：

```bash
redis-cli INFO stats
redis-cli INFO commandstats
redis-cli SLOWLOG GET 20
redis-cli --hotkeys  # 需要采样，生产谨慎
```

应用指标：

```text
cache_hit_total
cache_miss_total
cache_latency_ms
db_fallback_total
hot_key_name_sampled
```

## 5. Distributed lock 边界

Redis lock 常见写法：

```python
ok = redis.set("lock:sku:1", token, nx=True, ex=10)
```

必须：

- 设置过期时间。
- value 用随机 token。
- 释放时校验 token，用 Lua 原子释放。

Lua：

```lua
if redis.call("get", KEYS[1]) == ARGV[1] then
  return redis.call("del", KEYS[1])
else
  return 0
end
```

边界：

- Redis lock 不能替代数据库事务约束。
- 网络分区、GC pause、业务执行超过 TTL 都会破坏直觉。
- 对强一致库存扣减，优先用 DB row lock/唯一约束。

## 6. Session backend 选择

| Backend | 优点 | 风险 |
|---|---|---|
| DB session | 简单、可查询 | DB 压力、清理 |
| cache session | 快 | Redis 故障影响登录态 |
| signed cookie | 无服务端存储 | 大小限制、无法服务端吊销 |
| cached_db | 折中 | 复杂度更高 |

生产注意：

- session cookie secure/httponly/samesite。
- session 数据不要放大量业务对象。
- 权限变化后是否需要强制 session 失效。

## 7. 文件上传与对象存储

不要把大文件完整读进 Django worker 内存：

```python
content = request.FILES["file"].read()  # 大文件危险
```

更好的流程：

```text
client requests upload URL
→ backend creates object key and signed URL
→ client uploads directly to S3/MinIO/GCS
→ object storage callback or client confirm
→ Celery task scans/processes file
→ DB status updated
```

文件状态机：

```text
created → uploading → uploaded → scanning → ready
                              ↘ rejected
```

安全要求：

- 限制 content-type 和大小。
- 不相信客户端文件名。
- 异步病毒扫描/格式校验。
- signed URL 短过期。
- 私有 bucket，下载也走授权。

## 8. 外部 API timeout/retry

错误：

```python
requests.post(url, json=payload)  # 无 timeout
```

正确起点：

```python
requests.post(url, json=payload, timeout=(2, 5))
```

retry 规则：

- 只重试可重试错误：连接失败、超时、部分 5xx、429。
- POST 必须有幂等 key。
- 使用 exponential backoff + jitter。
- 设置总超时预算。

日志字段：

```json
{
  "event": "external_api_call_finished",
  "request_id": "req_1",
  "provider": "payment_gateway",
  "operation": "create_charge",
  "status_code": 502,
  "latency_ms": 1800,
  "attempt": 2,
  "timeout_ms": 5000
}
```

## 9. 常见误区

### 误区 1：缓存里有就认为业务状态真实

缓存是派生数据。强一致判断要回到 DB/事务边界。

### 误区 2：删除缓存和 DB 写入不考虑事务

事务未提交时删缓存，其他请求可能读 DB 旧值再写回缓存。

### 误区 3：把 session 当用户资料存储

session 应小而短，长期用户资料应该在 DB/cache 中明确建模。

### 误区 4：外部 API 没有 timeout

无 timeout 会耗尽 Gunicorn worker/Celery worker，形成级联故障。

## 10. 课堂讨论题

1. 订单详情缓存应该在事务内删除还是 `on_commit` 后删除？为什么？
2. Redis lock 适合保护库存扣减吗？什么时候不适合？
3. signed URL 上传相比 Django 直传有什么优势？
4. 外部 API retry 如何避免重复扣款？

## 11. 作业

为订单详情 API 设计缓存方案：

- cache key
- TTL + jitter
- 失效时机
- 穿透/击穿防护
- metrics
- 回源 DB 的限流策略

再为发票 PDF 上传设计文件状态机和 Celery 处理流程。

## 12. 评估标准

- 能设计带版本和 scope 的 cache key。
- 能解释缓存穿透/击穿/雪崩/热 key。
- 能说明 Redis lock 的正确释放和边界。
- 能设计直接上传对象存储流程。
- 能为外部 API 配置 timeout/retry/日志字段。
