# Lab 11：订单详情缓存与文件上传流程

## 目标

设计订单详情缓存和发票 PDF 上传流程。

## Part A：Redis Cache

要求：

- cache key 包含版本和 scope。
- TTL 带 jitter。
- DB 写入后用 `transaction.on_commit` 删除缓存。
- 列出穿透/击穿/雪崩/热 key 防护。

示例 key：

```text
order_summary:v1:user:42:order:123
```

## Part B：文件上传

设计状态机：

```text
created → uploading → uploaded → scanning → ready
                              ↘ rejected
```

要求：

- signed URL 短过期。
- 私有 bucket。
- Celery 异步扫描。
- 下载需要权限检查。

## 通过标准

- 明确 source of truth。
- 缓存失效在 commit 后。
- 文件不经 Django worker 大内存中转。
