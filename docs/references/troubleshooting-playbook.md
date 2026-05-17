# Python / Django 生产排障手册

## 排障格式

```text
症状
→ 影响面
→ 第一证据
→ 定位命令
→ 常见根因
→ 修复动作
→ 验证方式
```

## 常用命令

```bash
python manage.py check --deploy
python manage.py showmigrations
python -m pytest -q --durations=10
celery -A config inspect active
celery -A config inspect reserved
celery -A config inspect scheduled
celery -A config events
redis-cli XINFO GROUPS order-events
redis-cli XPENDING order-events order-consumers
kafka-consumer-groups --bootstrap-server localhost:9092 --describe --group order-service
mysql -e "SHOW ENGINE INNODB STATUS\G"
ps -o pid,ppid,cmd,%cpu,%mem -C gunicorn
ss -lntp
```

## 典型场景

- 慢请求：先分解 HTTP handler、DB SQL、外部 API、模板渲染和锁等待。
- 慢 SQL：保留 SQL、参数、EXPLAIN、索引、行数估计与实际耗时。
- 队列堆积：区分 arrival rate、worker concurrency、task runtime、broker 问题。
- 内存上涨：区分 Python heap、C extension、copy-on-write 失效、worker 泄漏。
- monkey patch 异常：记录 patch 时机、导入顺序、阻塞调用、第三方库兼容性。

- Redis Stream 堆积：检查 `XINFO GROUPS`、`XPENDING`、consumer idle time、是否需要 `XAUTOCLAIM`。
- Kafka lag：检查 consumer group lag、rebalance、单 partition 热点、poison message 与 DLQ。
- MySQL 锁等待：保留 SQL、transaction id、InnoDB status、processlist 与 EXPLAIN。
- MongoDB 慢查询：保留 query shape、winningPlan、COLLSCAN、index bounds、aggregation stage 耗时。
