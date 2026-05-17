# Lab 30：综合项目生产演练

## 目标

完成架构 review、压测报告、故障注入和 go/no-go 评审。

## 前置条件

- 使用课程综合项目或等价 Django 项目。
- 在本地/预发环境运行，不直接对生产做破坏性实验。
- 记录所有命令输出、日志片段、profile 文件或指标截图。

## 实验步骤

### 1. 准备最小可复现路径

写出一个最小 endpoint/task/consumer，使本 Lab 只观察一个机制。避免把多个未知因素混在一起。

### 2. 执行命令

```bash
hey -n 10000 -c 100 http://localhost:8000/api/orders
```
```bash
redis-cli --latency
```
```bash
mysql -e "SHOW ENGINE INNODB STATUS\G"
```
```bash
python manage.py drill_failure --scenario kafka_lag
```

### 3. 观察点

- 延迟：p50/p95/p99 或 queue oldest age。
- 资源：CPU、RSS、连接数、线程数、worker 数。
- 证据：日志字段、profile 栈、slow log、consumer lag、Redis latency。
- 失败：timeout、重试、断连、worker restart、DLQ。

### 4. 生产映射问题

- 这个实验在生产中对应哪类事故？
- 第一证据应该从哪里拿？
- 修复动作是否有副作用？
- 如何验证修复真正生效？

## 通过标准

- 提交 architecture/runbook/load-test/failure-drill 文档
- 至少完成 5 个故障注入
- 能给出上线 go/no-go 判断

## 提交物

- `notes.md`：实验步骤、观察证据、结论。
- `commands.log`：关键命令与输出。
- `fix-plan.md`：如果该问题发生在生产，给出修复与回滚方案。
