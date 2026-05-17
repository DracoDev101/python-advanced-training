# Python / Django 生产排障手册

## 排障格式

每个事故都按同一结构记录，避免直接跳到“调参”：

```text
症状
→ 影响面
→ 第一证据
→ 定位命令
→ 判断分支
→ 修复动作
→ 验证方式
→ 预防措施
```

## 常用命令速查

```bash
python manage.py check --deploy
python manage.py showmigrations
python -m pytest -q --durations=10
celery -A config inspect active
celery -A config inspect reserved
celery -A config inspect scheduled
celery -A config events
redis-cli INFO stats
redis-cli SLOWLOG GET 20
redis-cli XINFO GROUPS order-events
redis-cli XPENDING order-events order-consumers
kafka-consumer-groups --bootstrap-server localhost:9092 --describe --group order-service
mysql -e "SHOW ENGINE INNODB STATUS\G"
mysql -e "SHOW FULL PROCESSLIST"
ps -o pid,ppid,cmd,%cpu,%mem,rss -C gunicorn
ss -lntp
lsof -p <pid> | head
py-spy top --pid <pid>
strace -f -p <pid> -tt -T -e trace=network,select,poll,epoll_wait
```

---

## 1. 慢请求

### 症状

- API p95/p99 升高。
- 5xx/504 增加。
- 用户反馈页面转圈。

### 第一证据

```bash
# access log 按 route/status/duration 聚合
curl -w 'connect=%{time_connect} ttfb=%{time_starttransfer} total=%{time_total}\n' https://example.com/api/orders
py-spy dump --pid <worker_pid>
```

### 判断分支

```text
所有路由慢 → worker/DB/Redis/网络/部署
单一路由慢 → SQL、外部 API、serializer、权限过滤
CPU 高 → Python 热点或 JSON/加密/压缩
CPU 低但慢 → I/O、锁、连接池、队列等待
```

### 修复动作

- 先加 timeout 和降级，限制影响面。
- 对慢 SQL 做索引或查询改写。
- 对外部 API 加超时、重试预算、circuit breaker。
- 隔离慢接口 worker 或异步化重任务。

### 验证

```text
p95/p99 回落
5xx/504 回落
下游 DB/API latency 回落
worker active/request queue 回落
```

---

## 2. MySQL lock wait / deadlock

### 症状

- 请求偶发卡住。
- `Lock wait timeout exceeded`。
- deadlock 错误增加。

### 第一证据

```bash
mysql -e "SHOW FULL PROCESSLIST"
mysql -e "SHOW ENGINE INNODB STATUS\G"
mysql -e "SELECT * FROM performance_schema.data_locks\G"
mysql -e "SELECT * FROM performance_schema.data_lock_waits\G"
```

### 判断分支

```text
同一行热点 → 业务热点/库存/余额
范围锁/gap lock → 索引不匹配或 REPEATABLE READ next-key lock
长事务 → 应用持锁期间做外部 I/O
metadata lock → DDL 与长查询/事务冲突
```

### 修复动作

- 缩短事务，禁止事务内调用外部 API。
- 补正确索引，减少扫描与锁范围。
- 固定加锁顺序。
- 对热点资源使用队列化、分片或乐观并发。

### 验证

```text
lock wait 数下降
deadlock 下降
事务耗时下降
业务不变量测试通过
```

---

## 3. MySQL metadata lock

### 症状

- migration 卡住。
- 普通 SELECT/INSERT 也开始排队。
- processlist 中大量 `Waiting for table metadata lock`。

### 第一证据

```bash
mysql -e "SHOW FULL PROCESSLIST"
mysql -e "SELECT * FROM performance_schema.metadata_locks\G"
```

### 修复动作

- 暂停或 kill 阻塞 DDL 的长事务。
- 大表 DDL 改为 online schema change。
- 使用 expand/contract，不在高峰期 destructive migration。

### 预防

```text
发布前 sqlmigrate
大表 DDL 评审
migration timeout
灰度执行
回滚预案
```

---

## 4. Redis latency / hot key

### 症状

- cache 命中率正常但 latency 高。
- Redis CPU 高。
- API 尾延迟升高。

### 第一证据

```bash
redis-cli --latency
redis-cli --latency-history
redis-cli INFO commandstats
redis-cli SLOWLOG GET 20
redis-cli MEMORY STATS
```

### 判断分支

```text
big key → 单命令处理时间长
hot key → 单 key 请求过于集中
网络慢 → redis-cli latency 同步升高
eviction → maxmemory 策略触发
fork/AOF/RDB → 持久化导致抖动
```

### 修复动作

- 拆 big key。
- hot key 本地缓存、分片 key、请求合并。
- pipeline 减少 RTT。
- Lua 保证小范围原子操作，避免大 Lua。
- 调整 eviction 与内存水位。

---

## 5. Redis Stream PEL 堆积

### 症状

- stream 消费落后。
- consumer group pending 增长。
- 部分消息长期 idle。

### 第一证据

```bash
redis-cli XINFO STREAM order-events
redis-cli XINFO GROUPS order-events
redis-cli XPENDING order-events order-consumers
redis-cli XPENDING order-events order-consumers - + 10
```

### 判断分支

```text
pending idle 高 → consumer crash 后未 ack
单 consumer pending 多 → 消费者慢或 poison message
stream length 高 → producer 大于 consumer 吞吐
```

### 修复动作

```bash
redis-cli XAUTOCLAIM order-events order-consumers consumer-recover 60000 0-0 COUNT 100
```

- 对 poison message 写 failed 表并 ack。
- 增加 consumer，但注意同 group 并发与业务顺序。
- 对 read model 更新做幂等。

---

## 6. Celery queue backlog

### 症状

- queue depth / oldest age 上升。
- 用户后台状态长时间不变。
- worker active/reserved 异常。

### 第一证据

```bash
celery -A config status
celery -A config inspect active
celery -A config inspect reserved
celery -A config inspect scheduled
celery -A config inspect stats
```

### 判断分支

```text
worker 不在线 → 部署/secret/broker URL
reserved 很多 active 少 → prefetch 囤积
active 很多 runtime 高 → 下游慢或任务慢
retry 激增 → 下游故障导致 retry storm
worker lost → OOM/time limit/SIGKILL
```

### 修复动作

- critical queue 单独 worker。
- `worker_prefetch_multiplier=1`。
- 为慢任务单独队列。
- 对下游故障启用限流/circuit breaker。
- poison task 进入 DLQ/manual review。

---

## 7. Kafka consumer lag

### 症状

- consumer group lag 上升。
- read model 延迟。
- 某些 partition lag 远高于其他。

### 第一证据

```bash
kafka-consumer-groups --bootstrap-server localhost:9092 --describe --group order-read-model
```

### 判断分支

```text
单 partition lag 高 → hot key / poison message / 下游对某聚合慢
所有 partition lag 高 → consumer 总吞吐不足或下游慢
rebalance 频繁 → max.poll.interval/session timeout/部署抖动
consumer error 高 → schema 或业务异常
```

### 修复动作

- 先定位是否下游 DB/Mongo 慢。
- 对 poison event 写 DLQ 并 commit offset。
- 扩 consumer 不超过 partition 数。
- 调整 `max.poll.records` 与 `max.poll.interval.ms`。
- hot key 需要业务分片或重新设计 key。

---

## 8. Gunicorn worker timeout / 502 / 504

### 症状

- nginx 502/504。
- Gunicorn 日志 `WORKER TIMEOUT`。
- 部署期间错误突增。

### 第一证据

```bash
ps -o pid,ppid,rss,cmd -C gunicorn
grep 'WORKER TIMEOUT' gunicorn.error.log
py-spy dump --pid <worker_pid>
```

### 判断分支

```text
worker boot failed → import/settings/secret
timeout 前 CPU 高 → CPU-bound endpoint
timeout 前 CPU 低 → DB/API/socket 阻塞
RSS 高后重启 → memory leak/OOM
部署时集中 → graceful-timeout/terminationGracePeriod 不匹配
```

### 修复动作

- 给外部 API/DB 设置更短 timeout。
- CPU-heavy 移到 Celery/进程池。
- 调整 worker class/数量，但先计算 DB 连接数。
- 修复 graceful shutdown 配置。

---

## 9. ASGI event loop blocking

### 症状

- WebSocket ping/pong 延迟。
- async endpoint p95 抖动。
- CPU 不一定高。

### 第一证据

```text
event_loop_lag_ms
sync_to_async duration
threadpool queue wait
py-spy dump
```

### 常见根因

- async view 内调用 `requests`。
- 同步 ORM 高频 `sync_to_async(thread_sensitive=True)`。
- CPU-bound JSON/加密/压缩在 event loop 内执行。
- DNS 或未异步化 SDK 阻塞。

### 修复

- 替换为 async client。
- 同步 ORM 聚合为少量 service 调用。
- CPU-heavy 移出 event loop。
- 为 WebSocket 发送队列设置上限。

---

## 10. Memory leak / RSS growth

### 症状

- worker RSS 阶梯式上升。
- OOMKilled。
- `max_requests` 重启后恢复。

### 第一证据

```bash
cat /proc/<pid>/status | egrep 'VmRSS|VmSize|Threads'
cat /proc/<pid>/smaps_rollup
py-spy top --pid <pid>
```

Python heap：

```python
import tracemalloc
tracemalloc.start(25)
# take snapshots and compare
```

### 判断分支

```text
tracemalloc 增长 → Python 对象泄漏
RSS 增长但 tracemalloc 稳定 → native/fragmentation/COW
FD 增长 → connection/file leak
只在 preload 后增长 → copy-on-write 失效
```

### 修复

- 限制全局 cache / LRU size。
- 关闭未释放连接/文件。
- 修复 signal 重复注册。
- `max_requests` 只作为止血，不是根治。

---

## 11. Bad deploy rollback

### 症状

- 新版本错误率上升。
- migration 后旧版本不可用。
- canary 指标恶化。

### 第一证据

```bash
python manage.py migrate --plan
git diff HEAD~1..HEAD -- migrations/
# dashboard filter by version
```

### 判断分支

```text
代码 bug，无 schema 破坏 → 回滚镜像
schema expand 阶段 → 可回滚代码
schema contract/destructive → 不能简单回滚，需要 forward fix/restore
数据 backfill 错误 → dry-run/修复脚本/备份恢复
```

### 预防

- expand/contract。
- feature flag。
- canary 观察窗口。
- destructive migration 延后独立发布。
- 数据修复脚本必须 dry-run。
