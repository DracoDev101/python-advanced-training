# Lab 24：Gunicorn worker、timeout 与 preload 证据

## 目标

用一个 sleep endpoint 与 CPU endpoint 对比 sync/gthread/gevent 的行为，观察 worker timeout 与 RSS。

## 前置条件

- 使用课程综合项目或等价 Django 项目。
- 在本地/预发环境运行，不直接对生产做破坏性实验。
- 记录所有命令输出、日志片段、profile 文件或指标截图。

## 实验步骤

### 1. 准备最小可复现路径

写出一个最小 endpoint/task/consumer，使本 Lab 只观察一个机制。避免把多个未知因素混在一起。

### 2. 执行命令

```bash
gunicorn config.wsgi:application --workers 2 --timeout 5
```
```bash
ps -o pid,ppid,rss,cmd -C gunicorn
```
```bash
py-spy top --pid <worker_pid>
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

- 能解释 worker timeout 日志
- 能说明 preload_app 的连接副作用
- 能给出 worker class 选择理由

## 提交物

- `notes.md`：实验步骤、观察证据、结论。
- `commands.log`：关键命令与输出。
- `fix-plan.md`：如果该问题发生在生产，给出修复与回滚方案。
