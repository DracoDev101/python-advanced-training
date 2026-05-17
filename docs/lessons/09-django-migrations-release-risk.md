# Lesson 9：Migration、数据演进与发布风险

## 学习目标

完成本课后，你应该能够：

- 区分 schema migration、data migration、backfill、cleanup migration 的职责。
- 解释为什么大表 DDL 是发布风险，而不是普通代码变更。
- 设计 expand/contract 两阶段迁移，支持零停机发布。
- 针对 MySQL/InnoDB 判断哪些 migration 可能锁表、复制延迟或导致回滚困难。

## 关键问题

1. Django migration 文件能不能直接在生产大表上执行？
2. 添加一个 `NOT NULL` 字段为什么可能是事故？
3. 代码发布和 migration 执行的顺序如何设计？
4. data migration 应该放在 Django migration 里，还是独立 backfill job？

## 核心结论

Migration 是数据系统变更，不是纯代码变更。生产发布必须按兼容性设计：

```text
expand: 先添加兼容结构
→ deploy code: 新旧结构都能读写
→ backfill: 分批补数据
→ verify: 指标和抽样验证
→ contract: 删除旧结构/旧代码路径
```

对于大表，任何 DDL 都要先问：

- 会不会复制全表？
- 会不会长时间 metadata lock？
- 是否 online DDL？
- 是否能回滚？
- 应用代码是否兼容迁移前后状态？

## 1. Migration 类型

### 1.1 Schema migration

改变表结构：

```python
class Migration(migrations.Migration):
    operations = [
        migrations.AddField(
            model_name="order",
            name="external_id",
            field=models.CharField(max_length=64, null=True, db_index=True),
        )
    ]
```

### 1.2 Data migration

改变数据：

```python
from django.db import migrations


def fill_external_id(apps, schema_editor):
    Order = apps.get_model("orders", "Order")
    for order in Order.objects.filter(external_id__isnull=True).iterator(chunk_size=1000):
        order.external_id = f"ord_{order.id}"
        order.save(update_fields=["external_id"])

class Migration(migrations.Migration):
    operations = [migrations.RunPython(fill_external_id)]
```

风险：如果表有千万行，这个 migration 会跑很久，阻塞部署窗口，并且难以断点续跑。

### 1.3 Backfill job

更适合大数据量：

```bash
python manage.py backfill_order_external_id --batch-size 1000 --sleep-ms 50
```

优点：

- 可暂停、可恢复。
- 可限速。
- 可观测进度。
- 失败不会卡死 migration 状态。

## 2. 危险 migration 示例

### 2.1 大表直接添加 NOT NULL + default

```python
migrations.AddField(
    model_name="order",
    name="source",
    field=models.CharField(max_length=32, default="web"),
)
```

风险：

- 某些 MySQL 版本/场景可能重写整表。
- 长时间 metadata lock 阻塞读写。
- replica lag 上升。

更安全的 expand/contract：

```text
1. Add nullable column: source NULL
2. Deploy code: new writes fill source, reads tolerate NULL
3. Backfill old rows in batches
4. Add constraint / make NOT NULL if database supports safe operation
5. Remove fallback code
```

### 2.2 重命名字段

```python
migrations.RenameField("Order", "status", "state")
```

代码兼容性风险：旧代码读 `status`，新 schema 已变 `state`。

更安全：

```text
1. Add state nullable
2. Dual write status + state
3. Backfill state
4. Read from state with fallback status
5. Stop writing status
6. Drop status
```

## 3. MySQL/InnoDB DDL 风险

常见风险：

| 操作 | 风险 |
|---|---|
| Add column with default | 可能重写表，视版本和类型而定 |
| Add index | 大表耗时、IO 增加、复制延迟 |
| Change column type | 常见表重建风险 |
| Rename/drop column | 代码兼容性风险 |
| Add unique constraint | 可能因历史脏数据失败 |

上线前要确认：

```sql
SHOW CREATE TABLE orders_order\G
SELECT COUNT(*) FROM orders_order;
SHOW INDEX FROM orders_order;
```

执行中观察：

```sql
SHOW PROCESSLIST;
SHOW ENGINE INNODB STATUS\G
SHOW SLAVE STATUS\G; -- 如果仍使用旧术语的 MySQL 版本
SHOW REPLICA STATUS\G;
```

## 4. Django migration 操作清单

### 4.1 查看计划

```bash
python manage.py showmigrations
python manage.py migrate --plan
```

### 4.2 生成 SQL

```bash
python manage.py sqlmigrate orders 0009
```

`sqlmigrate` 是必须看的证据。不要只看 Python migration 文件。

### 4.3 分环境验证

```bash
python manage.py migrate --settings=config.settings.test
python manage.py migrate --settings=config.settings.staging
```

### 4.4 迁移后检查

```bash
python manage.py showmigrations orders
python manage.py check --deploy --settings=config.settings.prod
```

## 5. Backfill 命令设计

```python
# orders/management/commands/backfill_order_external_id.py
from django.core.management.base import BaseCommand
from django.db import transaction
import time

class Command(BaseCommand):
    def add_arguments(self, parser):
        parser.add_argument("--batch-size", type=int, default=1000)
        parser.add_argument("--sleep-ms", type=int, default=50)
        parser.add_argument("--dry-run", action="store_true")

    def handle(self, *args, **opts):
        batch_size = opts["batch_size"]
        while True:
            ids = list(
                Order.objects
                .filter(external_id__isnull=True)
                .order_by("id")
                .values_list("id", flat=True)[:batch_size]
            )
            if not ids:
                break
            if not opts["dry_run"]:
                with transaction.atomic():
                    for order in Order.objects.select_for_update().filter(id__in=ids):
                        order.external_id = f"ord_{order.id}"
                        order.save(update_fields=["external_id"])
            self.stdout.write(f"processed={len(ids)} last_id={ids[-1]}")
            time.sleep(opts["sleep_ms"] / 1000)
```

生产要求：

- 支持 dry-run。
- 批量处理，避免长事务。
- 输出进度。
- 可重复执行。
- 有 metrics：processed、remaining、error。

## 6. Migration 与 Celery/Kafka/Redis Stream 的关系

事件 schema 也会迁移。

错误做法：直接改 Kafka event 字段名：

```json
{"order_state": "paid"}
```

旧 consumer 还在读：

```json
{"status": "paid"}
```

兼容做法：

```text
1. Producer 同时发 status + order_state
2. Consumer 支持两个字段
3. 所有 consumer 升级后停止读旧字段
4. Producer 停止发送 status
```

Redis Stream/Kafka 事件有 replay 能力，旧消息可能长期存在。schema migration 不能只考虑“新代码”。

## 7. 常见误区

### 误区 1：migration 失败再回滚就好

生产 DDL 回滚可能比执行更危险。要先设计 forward-only 修复方案。

### 误区 2：data migration 放 Django migration 最省事

小表可以，大表不行。大表 backfill 要独立、可暂停、可恢复。

### 误区 3：只在空库跑 migration

空库无法暴露大表 DDL、历史脏数据、索引构建时间。

### 误区 4：忘记旧 worker

发布期间可能同时存在旧 web worker、旧 Celery worker、旧 Kafka consumer。schema 必须兼容混跑窗口。

## 8. 课堂讨论题

1. 添加 `NOT NULL` 字段的安全发布步骤是什么？
2. 为什么 rename column 通常要用 add + dual write + drop，而不是直接 rename？
3. Kafka event schema 如何做 expand/contract？
4. backfill job 如何避免打爆 MySQL？

## 9. 作业

为 `orders_order` 添加 `external_id`，要求最终 `NOT NULL UNIQUE`。

写出：

- migration 分几步。
- 每步对应代码发布要求。
- backfill 命令参数。
- 验证 SQL。
- 回滚/修复策略。

## 10. 评估标准

- 能区分 schema migration、data migration、backfill。
- 能设计 expand/contract 发布。
- 能用 `sqlmigrate` 和 MySQL 命令做证据检查。
- 能考虑旧 web/Celery/consumer 混跑窗口。
