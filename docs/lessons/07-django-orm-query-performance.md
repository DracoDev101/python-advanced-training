# Lesson 7：ORM 查询、N+1 与 SQL 证据

## 学习目标

- 理解本课主题背后的运行机制。
- 能用最小实验观察关键证据。
- 能说明该机制在生产 Django/Celery 系统中的边界与风险。

## 关键问题

- 这个机制解决什么问题？
- 它在哪些情况下会成为性能或可靠性瓶颈？
- 出问题时第一证据应该从哪里拿？

## 核心结论

本课后续会展开完整讲义。编写时必须包含：机制解释、代码实验、生产配置、故障证据、工具验证与作业。

## 最小实验

```bash
python -m pytest
python manage.py check --deploy
```

## 生产实践

待补充：结合综合项目 `Production Order Workflow` 给出可运行示例。

## 故障证据

待补充：日志、SQL、profile、queue metrics、consumer lag 或 server worker 状态。
