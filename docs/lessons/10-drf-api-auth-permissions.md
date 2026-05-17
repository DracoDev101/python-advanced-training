# Lesson 10：DRF/API 设计、认证与权限

## 学习目标

完成本课后，你应该能够：

- 设计稳定、可演进的 REST API 响应和错误格式。
- 区分 serializer validation、domain validation、permission check 的职责。
- 正确选择 Session、Token、JWT、API Key 的使用场景。
- 为分页、幂等、限流、权限失败、审计日志建立生产证据。

## 关键问题

1. serializer 是否应该包含全部业务规则？
2. ViewSet 是否总是比 APIView 更好？
3. 认证失败、权限失败、业务校验失败应该返回什么状态码？
4. POST 接口如何避免客户端重试造成重复订单？

## 核心结论

API 层是外部契约，不是内部 ORM 的镜像。生产 API 要稳定：

```text
request schema
→ authentication
→ permission
→ serializer validation
→ command DTO
→ domain/service validation
→ response schema
→ structured audit log
```

不要把 DB model 字段原样暴露给客户端，也不要把 serializer 当成业务服务层。

## 1. API 响应格式

成功响应示例：

```json
{
  "data": {
    "id": 123,
    "status": "pending",
    "created_at": "2026-05-17T12:00:00Z"
  },
  "request_id": "req_123"
}
```

错误响应示例：

```json
{
  "error": {
    "code": "order_out_of_stock",
    "message": "Inventory is not enough",
    "details": {
      "sku": "SKU-1"
    },
    "request_id": "req_123"
  }
}
```

要求：

- 错误 code 稳定，message 可读。
- details 不暴露内部 SQL/堆栈。
- request_id 必须返回。

## 2. Serializer validation vs domain validation

Serializer 负责协议层：

```python
class CreateOrderSerializer(serializers.Serializer):
    sku = serializers.CharField(max_length=64)
    quantity = serializers.IntegerField(min_value=1, max_value=100)
```

Domain/service 负责业务不变量：

```python
def create_order(cmd: CreateOrderCommand) -> CreateOrderResult:
    if is_sku_discontinued(cmd.sku):
        raise SkuDiscontinued(cmd.sku)
    if not can_user_order(cmd.user_id):
        raise UserNotAllowed()
```

不要把所有业务规则塞进 serializer：

- serializer 依赖 HTTP/DRF 语境。
- Celery/Kafka consumer 也可能触发同样业务流程。
- service 层更容易单测。

## 3. APIView vs ViewSet

`ViewSet` 适合标准资源 CRUD：

```text
GET /orders
POST /orders
GET /orders/{id}
PATCH /orders/{id}
```

`APIView` 适合明确动作或复杂工作流：

```text
POST /orders/{id}/cancel
POST /payments/callback
POST /inventory/reservations
```

不要为了 router 方便，把复杂业务动作伪装成 CRUD。

## 4. 认证方式选择

| 方式 | 适合 | 风险 |
|---|---|---|
| Session | 浏览器 Web、同站应用 | CSRF、跨域复杂 |
| Token | 简单服务端 API | token 泄漏、吊销困难 |
| JWT | 跨服务/移动端、无状态验证 | 吊销、过期、claim 膨胀 |
| API Key | 服务到服务、低频集成 | 权限粒度、轮换 |
| OAuth/OIDC | 第三方授权、企业 SSO | 接入复杂 |

生产要求：

- token 不进日志。
- 支持轮换/吊销。
- 权限失败有 audit log。
- 高风险操作需要额外校验或短期凭证。

## 5. Permission 设计

对象级权限：

```python
class IsOrderOwner(BasePermission):
    def has_object_permission(self, request, view, obj):
        return obj.user_id == request.user.id
```

列表 API 不能只靠 object permission，因为列表查询已经发生。必须在 query 层过滤：

```python
def list_orders_for_user(user_id: int):
    return Order.objects.filter(user_id=user_id)
```

错误示例：

```python
orders = Order.objects.all()
# serializer 后再过滤，这是数据泄漏风险
```

## 6. 幂等 POST

客户端重试可能重复创建订单。

使用 `Idempotency-Key`：

```http
POST /orders
Idempotency-Key: idem_abc
```

数据库表：

```python
class IdempotencyRecord(models.Model):
    key = models.CharField(max_length=128, unique=True)
    user_id = models.BigIntegerField()
    request_hash = models.CharField(max_length=64)
    response_json = models.JSONField(null=True)
    created_at = models.DateTimeField(auto_now_add=True)
```

流程：

```text
receive request
→ hash body
→ insert idempotency record unique(key,user)
→ if conflict: compare hash and return previous response
→ run service
→ store response
```

注意：幂等 key 要绑定用户/租户，不能全局裸用。

## 7. 分页与排序

Offset pagination：

```text
GET /orders?limit=50&offset=1000
```

问题：深分页慢，数据变化时重复/遗漏。

Cursor pagination：

```text
GET /orders?cursor=eyJjcmVhdGVkX2F0Ijoi..."
```

适合时间线/订单列表。

生产要求：

- 排序字段必须稳定，例如 `(created_at, id)`。
- 有匹配索引。
- limit 有上限。
- response 包含 `next_cursor`。

## 8. 限流与审计

DRF throttle 可以做基础限流，但生产常需要网关/Redis 级限流。

审计日志字段：

```json
{
  "event": "api_permission_denied",
  "request_id": "req_1",
  "user_id": 42,
  "path": "/api/orders/123",
  "action": "order.cancel",
  "reason": "not_owner"
}
```

高风险 API：

- payment callback
- admin operation
- bulk export
- permission change
- API key creation

必须有审计日志。

## 9. 常见误区

### 误区 1：直接用 `ModelSerializer(fields='__all__')`

容易泄露内部字段，例如成本、风控状态、软删除标记。

### 误区 2：只在前端做权限控制

API 必须做认证和权限。前端隐藏按钮不是权限。

### 误区 3：POST 没有幂等设计

移动网络、代理重试、客户端重试都会造成重复请求。

### 误区 4：错误 message 当稳定契约

客户端应该依赖 error code，不应该解析 message。

## 10. 课堂讨论题

1. `serializer.validate()` 和 service validation 的边界如何划分？
2. 订单取消接口应该是 `PATCH /orders/{id}` 还是 `POST /orders/{id}/cancel`？
3. JWT 吊销有哪些方案？代价是什么？
4. Idempotency-Key 如何处理“同 key 不同 body”？

## 11. 作业

设计 `POST /orders` API：

- request schema
- response schema
- error schema
- permission 策略
- idempotency 策略
- audit log 字段
- 分别说明 serializer 与 service 的校验规则

## 12. 评估标准

- 能区分 API/schema/domain/permission 边界。
- 能设计稳定错误格式。
- 能选择认证方式并说明风险。
- 能为 POST 设计幂等机制。
- 能给出权限失败和高风险操作审计字段。
