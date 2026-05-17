# 阅读清单

本清单按课程模块组织。优先读官方文档和规格，再读生产实践文章。目标不是背 API，而是理解机制、边界和证据。

## Python Runtime

- Python 官方文档：Data Model、Execution Model、import system、gc、asyncio。
- `tracemalloc`、`cProfile`、`concurrent.futures`、`contextvars` 官方文档。
- CPython Developer Guide：memory management、GIL、garbage collection。
- PEP 567：Context Variables。
- PEP 654：Exception Groups and `asyncio.TaskGroup`。
- PEP 703：Making the Global Interpreter Lock Optional，了解 free-threaded Python 背景。

## Django

- Django 官方文档：settings、middleware、ORM、transactions、migrations、testing、deployment checklist。
- Django async support 文档：async views、async safety、`SynchronousOnlyOperation`。
- Django database transactions：`atomic`、`on_commit`、savepoint。
- Django migrations：data migration、squashing、operation reference。
- Django security checklist：CSRF、CORS、SECURE headers、proxy SSL header。

## Django REST Framework / API

- DRF 官方文档：serializers、viewsets、permissions、throttling、pagination、exceptions。
- JSON:API / RFC 7807 Problem Details，作为错误格式参考。
- Idempotency-Key Header draft / Stripe idempotency docs。
- OAuth 2.0 / OIDC 基础文档，用于理解 session/token/JWT/API key 的边界。

## Testing

- pytest / pytest-django 官方文档。
- factory_boy、freezegun、responses、respx。
- Hypothesis property-based testing。
- Schemathesis：OpenAPI contract/fuzz testing。
- Testcontainers Python：MySQL/Redis/Kafka integration smoke。

## MySQL / InnoDB

- MySQL Reference Manual：InnoDB architecture、transaction isolation、locking、deadlocks。
- MySQL `EXPLAIN` / `EXPLAIN ANALYZE` 文档。
- Performance Schema：statement、lock、metadata lock 相关表。
- MySQL slow query log 与 `pt-query-digest`。
- gh-ost / pt-online-schema-change 文档，用于大表 online DDL。
- InnoDB clustered index、secondary index、buffer pool、redo log 机制资料。

## Redis / Redis Stream

- Redis 官方文档：data types、pipelining、transactions、Lua scripting、eviction、persistence。
- Redis latency troubleshooting guide。
- Redis Streams introduction：`XADD`、`XREADGROUP`、`XACK`、`XPENDING`、`XAUTOCLAIM`。
- Redis keyspace notifications 与 stream trimming 文档。
- Distributed locks with Redis 与 Redlock 争议文章；重点理解 fencing token。

## MongoDB

- MongoDB Manual：schema design、indexes、compound indexes、TTL indexes。
- MongoDB Aggregation Pipeline optimization。
- `explain("executionStats")`、query planner、COLLSCAN、IXSCAN。
- read concern / write concern / read preference。
- MongoDB schema validation 与 shard key 设计。

## Celery

- Celery 官方文档：calling tasks、routing、workers、monitoring、optimizing、task retry。
- Celery configuration：`acks_late`、prefetch、visibility timeout、time limits、rate limits。
- Celery canvas：chain/group/chord 的失败语义。
- Flower / Celery events / Prometheus exporter。
- RabbitMQ 与 Redis broker 的语义差异。

## Kafka

- Kafka 官方文档：topics、partitions、consumer groups、offsets、transactions。
- Confluent Kafka Python / librdkafka configuration。
- Kafka consumer configs：`enable.auto.commit`、`max.poll.records`、`max.poll.interval.ms`、`session.timeout.ms`。
- Kafka producer configs：`acks=all`、`enable.idempotence=true`、`linger.ms`、`batch.size`、compression。
- Schema Registry compatibility modes：backward/forward/full。
- Debezium Outbox Event Router。
- Kafka retry topic / DLQ / replay 生产实践。

## Server / Concurrency

- WSGI specification：PEP 3333。
- ASGI specification：HTTP、WebSocket、lifespan。
- Gunicorn design 文档：master/worker、signals、worker classes。
- Uvicorn deployment docs。
- Daphne / Channels 文档：routing、consumer、channel layer、Redis backend。
- gevent/eventlet monkey patch 文档。
- `asgiref.sync` 文档：`sync_to_async`、`async_to_sync`、thread-sensitive。

## Profiling / Debugging

- py-spy 文档：top、record、dump、生产采样限制。
- pyinstrument 文档：request-level profiling。
- Scalene：CPU/GPU/memory profiler。
- Memray：native/Python memory allocation profiler。
- Python `tracemalloc` 文档。
- Linux `strace`、`lsof`、`ss`、`perf` 基础。
- Brendan Gregg flamegraph / USE method / latency analysis 文章。

## Observability / SRE

- OpenTelemetry Python instrumentation、propagation、sampling。
- Prometheus best practices：metric naming、labels、histograms、cardinality。
- Google SRE Book：SLI/SLO、error budget、incident response。
- RED / USE 方法。
- Grafana dashboard design best practices。
- Incident review / postmortem 模板。

## Deployment / Operations

- Django deployment checklist。
- Gunicorn + nginx deployment notes。
- Kubernetes probes：liveness、readiness、startup。
- Kubernetes graceful termination 与 preStop hook。
- Docker Compose for local integration environments。
- 12-factor app：config、logs、processes。
- Expand/contract database migration pattern。

## 推荐阅读顺序

```text
1. Django deployment + transaction + migration
2. MySQL InnoDB locks + EXPLAIN
3. Redis cache + Stream consumer group
4. Celery worker/retry/prefetch/time limit
5. Kafka consumer group + manual commit + DLQ
6. WSGI/ASGI + Gunicorn/Daphne
7. Profiling + OpenTelemetry + SRE runbook
```
