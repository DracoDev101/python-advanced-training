# Lesson 14：Redis 基础数据结构与缓存边界

## 学习目标

完成本课后，你应该能够：

- 根据访问模式选择 Redis string/hash/list/set/zset，而不是把 Redis 当万能 dict。
- 设计 cache aside、negative cache、singleflight、TTL jitter、热 key 缓解方案。
- 解释 Redis 单线程执行模型、pipeline、Lua、慢命令、内存淘汰对 Django 服务的影响。
- 判断 Redis lock、rate limit、session、排行榜、去重集合等场景的正确性边界。

## 关键问题

1. Redis 很快，为什么仍然会成为系统瓶颈？
2. 缓存击穿时，为什么所有请求可能同时打到 MySQL？
3. Redis distributed lock 能不能保护强一致库存？
4. `KEYS *`、大 value、大 hash、大 zset 为什么危险？

## 核心结论

Redis 是远程内存数据结构服务器，不是“免费的全局变量”。它的生产边界是：

```text
single-thread command execution
+ network round trip
+ memory limit/eviction
+ persistence/replication tradeoff
+ key design / TTL / hot key
+ consistency boundary with DB
```

任何 Redis 设计都要回答：

- 这份数据的 source of truth 是谁？
- Redis 丢失/过期/延迟时业务如何退化？
- key 数量、value 大小、TTL 分布是否可控？
- 是否有命令级证据：latency、slowlog、hit ratio、evicted keys？

## 1. 数据结构选择

| 结构 | 常见用途 | 风险 |
|---|---|---|
| string | cache value、counter、lock token | 大 value、序列化成本 |
| hash | 对象字段、局部更新 | big hash rehash/阻塞 |
| list | 简单队列 | ack/replay 能力弱 |
| set | 去重、集合关系 | 大集合操作阻塞 |
| zset | 排行榜、延迟队列 | 大范围查询/删除成本 |
| stream | 事件流/consumer group | PEL 管理、trim 策略 |

### string：订单摘要缓存

```text
SET order_summary:v1:123 <json> EX 300
GET order_summary:v1:123
```

### hash：用户权限局部字段

```text
HSET user_perm:v3:42 can_order 1 can_refund 0
HGETALL user_perm:v3:42
EXPIRE user_perm:v3:42 300
```

### zset：延迟任务/排行榜

```text
ZADD retry_schedule 1710000000 event_id
ZRANGEBYSCORE retry_schedule -inf 1710000000 LIMIT 0 100
```

注意：Redis zset 延迟队列没有 Celery/Kafka 的完整投递语义，需要自己处理并发 claim、失败重试和幂等。

## 2. 单线程模型与慢命令

Redis 命令通常在单线程事件循环里执行。一个慢命令会阻塞后续命令。

危险命令/模式：

```text
KEYS *
SMEMBERS huge_set
HGETALL huge_hash
LRANGE huge_list 0 -1
DEL huge_key_with_millions_members
large Lua script
```

替代：

```text
SCAN / HSCAN / SSCAN / ZSCAN
UNLINK 替代 DEL 大 key
分页读取
限制 value 大小
```

证据：

```bash
redis-cli SLOWLOG GET 20
redis-cli LATENCY DOCTOR
redis-cli INFO commandstats
redis-cli INFO memory
```

## 3. Cache aside 深水区

基本模式：

```python
value = cache.get(key)
if value is None:
    value = read_db()
    cache.set(key, value, timeout=ttl)
return value
```

问题：并发 miss 时，所有请求都回源 DB。

### 3.1 Singleflight / mutex 防击穿

```python
import uuid
from django.core.cache import cache


def get_with_mutex(key: str, ttl: int, loader):
    value = cache.get(key)
    if value is not None:
        return value

    lock_key = f"lock:{key}"
    token = str(uuid.uuid4())
    got_lock = cache.add(lock_key, token, timeout=10)
    if got_lock:
        try:
            value = loader()
            cache.set(key, value, timeout=ttl)
            return value
        finally:
            # Django cache 没有原子 compare-delete；生产用 Redis Lua
            cache.delete(lock_key)

    # 没拿到锁：短暂等待后重读，或返回 stale
    for _ in range(5):
        time.sleep(0.05)
        value = cache.get(key)
        if value is not None:
            return value
    return loader()  # 最后兜底，但要限流
```

生产要用 Redis token + Lua 原子释放，避免误删别人锁。

### 3.2 Stale-while-revalidate

对读多、允许短暂陈旧的数据：

```text
cache value = {data, soft_expire_at, hard_expire_at}
soft expired: 返回旧值，同时异步刷新
hard expired: 阻塞回源或降级
```

这比热门 key 到期瞬间所有请求等待 DB 更平滑。

## 4. Negative cache 防穿透

不存在的订单 ID 被反复请求：

```python
def get_order(order_id: int):
    key = f"order:v1:{order_id}"
    marker = cache.get(key)
    if marker == "__NULL__":
        raise OrderNotFound()
    if marker is not None:
        return marker

    order = Order.objects.filter(id=order_id).first()
    if order is None:
        cache.set(key, "__NULL__", timeout=30)
        raise OrderNotFound()
    cache.set(key, serialize(order), timeout=300)
    return order
```

注意：negative cache TTL 要短。否则刚创建的数据可能短时间仍被认为不存在。

## 5. 热 key

现象：

- Redis CPU 高。
- 单 key QPS 极高。
- 应用侧 cache latency 上升。
- MySQL 可能没压力，因为全打 Redis。

证据：

```bash
redis-cli --hotkeys       # 采样命令，生产谨慎
redis-cli INFO commandstats
redis-cli LATENCY LATEST
```

应用侧可以采样 key：

```json
{
  "event": "cache_get",
  "key_family": "order_summary",
  "hit": true,
  "latency_ms": 3
}
```

不要记录完整 key 中的敏感信息。

缓解：

- 本地进程 LRU 短 TTL。
- key 分片：`hot_key:{shard}`，适合可合并数据。
- stale-while-revalidate。
- 降低单 key value 大小。
- CDN/edge cache（若是公开数据）。

## 6. Pipeline 与 Lua

### 6.1 pipeline 降低 RTT

错误：

```python
for order_id in ids:
    cache.get(f"order:v1:{order_id}")
```

每个 get 一次网络往返。

使用底层 Redis client pipeline：

```python
pipe = redis.pipeline()
for order_id in ids:
    pipe.get(f"order:v1:{order_id}")
values = pipe.execute()
```

Pipeline 不保证事务语义，只减少 round trip。

### 6.2 Lua 做原子组合操作

释放锁：

```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
  return redis.call("DEL", KEYS[1])
else
  return 0
end
```

Python：

```python
release_lock = redis.register_script("""
if redis.call('GET', KEYS[1]) == ARGV[1] then
  return redis.call('DEL', KEYS[1])
else
  return 0
end
""")
release_lock(keys=[lock_key], args=[token])
```

Lua 必须短小，长脚本会阻塞 Redis。

## 7. Rate limit

固定窗口：

```text
INCR rate:user:42:202605171530
EXPIRE rate:user:42:202605171530 120
```

风险：窗口边界突刺。

滑动窗口可用 zset：

```text
ZADD rate:user:42 now request_id
ZREMRANGEBYSCORE rate:user:42 -inf now-60
ZCARD rate:user:42
EXPIRE rate:user:42 120
```

生产建议：

- 高风险 API 在网关层限流。
- 应用层做业务维度二次限流。
- 限流日志必须包含 key family，而不是敏感 token。

## 8. Eviction 与内存

查看：

```bash
redis-cli INFO memory
redis-cli CONFIG GET maxmemory-policy
redis-cli INFO stats | grep evicted_keys
```

常见 policy：

```text
noeviction
allkeys-lru
volatile-lru
allkeys-lfu
volatile-ttl
```

如果 Redis 同时承担 session、cache、Celery broker、Redis Stream，必须隔离实例或至少隔离 DB/namespace，并明确 eviction 风险。

强烈建议：

```text
cache Redis      可淘汰
session Redis    谨慎淘汰
broker Redis     不应和 cache 混用
stream Redis     需要 trim/保留策略
```

## 9. Redis lock 的正确性边界

适合：

- 防止缓存击穿时重复回源。
- 控制后台 job 单实例执行。
- 非强一致、短时间互斥。

不适合单独保护：

- 金额扣减。
- 库存强一致。
- 权限变更强一致。

这些要由数据库唯一约束、行锁、事务状态机兜底。

Redis lock 最小要求：

```text
SET key token NX EX ttl
业务执行时间 < ttl
释放时 compare token + DEL via Lua
失败要可重试/可补偿
```

## 10. Runbook：Redis 延迟上升

```text
症状：API P95 上升，DB 正常，Redis latency 高

1. 确认应用证据
   - cache_latency_ms、cache_hit_ratio、key_family
2. Redis 运行证据
   - INFO stats/memory/commandstats
   - SLOWLOG GET 20
   - LATENCY DOCTOR
3. 检查危险命令
   - KEYS、HGETALL huge hash、SMEMBERS huge set、Lua
4. 检查内存
   - used_memory、evicted_keys、fragmentation_ratio
5. 检查热 key
   - hotkeys 采样/应用侧 key family 采样
6. 修复
   - 拆 key、分页、pipeline、UNLINK、TTL jitter、隔离实例
7. 验证
   - Redis latency 降低、slowlog 清空、API P95 恢复
```

## 11. 课堂讨论题

1. 为什么 Redis list 不适合作为需要 ack/replay 的可靠队列？
2. `cache.delete` 应该在事务内还是 `on_commit` 后？
3. Redis lock 释放为什么必须校验 token？
4. 如果 Redis cache 和 Celery broker 共用实例，可能出现什么级联故障？

## 12. 作业

为订单详情和用户权限设计 Redis 方案：

- key 结构。
- 数据结构选择。
- TTL + jitter。
- 失效策略。
- 防穿透/击穿方案。
- 监控指标和 runbook。

## 13. 评估标准

- 能按访问模式选择 Redis 数据结构。
- 能解释单线程模型和慢命令风险。
- 能设计 cache aside 的一致性与击穿保护。
- 能说明 Redis lock 的正确释放和强一致边界。
- 能写出 Redis 延迟上升的证据链。
