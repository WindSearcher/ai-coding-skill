# 知识归档输出模板

## 示例：订单处理功能

### 背景
- **需求来源**：PRD-2024-011，工单 PROJ-482
- **业务目标**：让客户无需人工干预即可下单和付款
- **利益相关者**：产品（Alice）、工程（Bob）、财务（Carol）

### 设计决策
| 决策 | 理由 | 已拒绝的替代方案 |
|------|------|----------------|
| 异步支付处理 | 防止慢网关导致 HTTP 超时 | 同步扣款（拒绝：太慢） |
| 订单 ID 使用 UUID | 避免顺序 ID 被枚举 | 自增 bigint（拒绝：可预测性） |
| 持久化前先校验库存 | 防止超卖 | 乐观库存（拒绝：复杂度高） |

### 实现概述
- **核心模块**：
  - `OrderController` - HTTP 层，输入校验
  - `OrderService` - 业务逻辑，编排协调
  - `OrderRepository` - 数据库访问
  - `InventoryClient` - 外部库存服务包装器
- **入口点**：
  - `POST /api/v1/orders` - 创建订单
  - `GET /api/v1/orders/:id` - 查询订单
  - `OrderPaidEvent` - 供下游处理的内部事件
- **数据流**：Controller -> Service ->（库存校验 + 计价）-> Repository -> 事件发送
- **外部依赖**：
  - 库存服务 v2.3（REST）
  - 计价引擎 v1.1（gRPC）
  - 支付网关 v3（异步 webhook）

### 配置
| 名称 | 默认值 | 说明 |
|------|--------|------|
| `ORDER_TIMEOUT_MS` | 30000 | 等待库存校验的最大时间 |
| `PAYMENT_RETRY_MAX` | 3 | 失败支付的最大重试次数 |
| `PRICING_CACHE_TTL_S` | 300 | 缓存计价结果的时长 |

### 运维指南
- **健康检查**：`GET /health/orders` 返回 200 及数据库连接状态
- **常见问题**：
  - "库存超时" -> 检查库存服务健康，验证 `ORDER_TIMEOUT_MS`
  - "支付卡在 PENDING" -> 检查死信队列，通过管理端点重试
  - "计价不匹配" -> 清空计价缓存，验证计价引擎版本
- **监控**：
  - 指标：`orders_created_total`（计数器）
  - 指标：`order_processing_duration_seconds`（直方图）
  - 告警：`order_payment_failure_rate > 0.05` 持续 5 分钟

### 维护注意事项
- **技术债务**：库存客户端缺少熔断器；服务降级可能级联传播
- **未来改进**：
  - 多商品订单批量库存校验（性能）
  - 支持部分退款（业务）
- **扩展考虑**：
  - orders 表在 1000 万行后需要按 `created_at` 分区
  - 事件发送目前是同步的；高并发时考虑发件箱模式
- **相关文档**：
  - [技术方案](designs/order-processing-tech-design.md)
  - [API 规范](api/v1/orders.yaml)
  - [运维手册：支付故障](runbooks/payment-incidents.md)
