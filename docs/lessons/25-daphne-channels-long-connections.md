# Lesson 25：Daphne、Channels 与长连接

## 学习目标

- 理解 Daphne/ASGI/Channels/channel layer 的职责分工。
- 能设计 WebSocket 鉴权、心跳、分组广播、断线恢复和 backpressure。
- 能排查连接数高、Redis channel layer 堆积、慢客户端、消息丢失。
- 能判断哪些业务适合 WebSocket，哪些应使用轮询/SSE/推送服务。

## 1. 栈结构

```text
client WebSocket
→ nginx / LB
→ Daphne ASGI server
→ Django Channels routing
→ Consumer
→ Redis channel layer
→ Celery/Kafka/DB events
```

Daphne 是 ASGI server；Channels 提供 routing、consumer、group、channel layer；Redis 常作为跨进程消息层。

## 2. Consumer 模型

```python
class OrderConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        self.order_id = self.scope["url_route"]["kwargs"]["order_id"]
        await self.channel_layer.group_add(f"order:{self.order_id}", self.channel_name)
        await self.accept()

    async def disconnect(self, code):
        await self.channel_layer.group_discard(f"order:{self.order_id}", self.channel_name)

    async def order_event(self, event):
        await self.send_json(event["payload"])
```

## 3. 鉴权与授权

WebSocket 不是绕过权限的“实时通道”。必须在 connect 阶段校验：

- session/cookie/JWT。
- 用户是否有权订阅该 order/account/tenant。
- token 过期后的断开策略。

不要只靠前端隐藏连接地址。

## 4. 心跳、断线与恢复

长连接默认会遇到 NAT、移动网络、LB idle timeout。

策略：

```text
client ping interval: 20s
server heartbeat timeout: 60s
LB idle timeout: > heartbeat timeout
client reconnect: exponential backoff + jitter
resume: client carries last_seen_event_id
```

如果业务不能丢事件，WebSocket 只负责通知“有变化”，真实状态仍从 HTTP/数据库读模型拉取。

## 5. Backpressure

慢客户端会导致发送 buffer 增长。生产策略：

- 每连接输出队列上限。
- 超过阈值丢弃低价值消息或断开连接。
- 聚合事件，发送最新状态而非全部中间态。
- 对 group fanout 做限流。

指标：

```text
ws_connections_total
ws_active_connections
ws_send_queue_size
ws_send_failures_total
channel_layer_group_send_latency_ms
redis_channel_layer_errors_total
```

## 6. Redis channel layer 边界

Redis channel layer 不是 Kafka：

- 不适合长期保留事件。
- 不适合高价值事件唯一来源。
- worker 重启/连接断开期间消息可能丢。
- group membership 是运行态状态。

高价值事件应该走 Outbox/Kafka/Redis Stream，WebSocket 用于实时投递和提示。

## 7. 部署注意

```bash
daphne -b 0.0.0.0 -p 8001 config.asgi:application
```

nginx 需要 upgrade header：

```nginx
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
proxy_read_timeout 75s;
```

水平扩展时多个 Daphne 实例共享 Redis channel layer。

## 8. Runbook：WebSocket 断连率高

```text
1. LB/nginx 日志：499/504/upgrade 失败
2. Daphne 日志：disconnect code 分布
3. heartbeat 指标：ping/pong 超时
4. Redis channel layer latency/error
5. 客户端网络与版本分布
6. 修复：调整 idle timeout、heartbeat、消息大小、backpressure
7. 验证：断连率、重连成功率、send queue 回落
```

## 作业

为订单状态页设计 WebSocket：鉴权、group 命名、事件 schema、last_seen_event_id、心跳、慢客户端策略、Redis 故障降级。

## 评估标准

- 能说明 Daphne 与 Channels 分工。
- 能设计不丢关键业务状态的长连接方案。
- 能排查慢客户端和 channel layer 问题。
