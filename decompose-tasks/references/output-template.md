# 任务拆解输出模板

## 示例 1（简单场景）：添加用户头像上传功能

### 概述
- 总任务数：5
- 预估总工时：6 小时
- 最大并行深度：2 层

### 第 1 层：基础
| 编号 | 文件 | 类型 | 复杂度 | 依赖 | 说明 |
|------|------|------|--------|------|------|
| T-01 | src/types/avatar.ts | 新建 | 低 | 无 | 定义头像相关的类型和接口（AvatarUploadRequest, AvatarResponse） |
| T-02 | src/config/storage.ts | 修改 | 低 | 无 | 添加对象存储配置（bucket name, region, CDN domain） |

### 第 2 层：核心逻辑
| 编号 | 文件 | 类型 | 复杂度 | 依赖 | 说明 |
|------|------|------|--------|------|------|
| T-03 | src/services/avatar-service.ts | 新建 | 中 | T-01, T-02 | 实现头像上传核心逻辑：文件校验、上传到 OSS、生成缩略图 |

### 第 3 层：接口与集成
| 编号 | 文件 | 类型 | 复杂度 | 依赖 | 说明 |
|------|------|------|--------|------|------|
| T-04 | src/controllers/avatar-controller.ts | 新建 | 中 | T-03 | 实现 POST /api/v1/users/:id/avatar 端点，处理 multipart/form-data |
| T-05 | src/routes/user-routes.ts | 修改 | 低 | T-04 | 注册头像上传路由 |

### 第 4 层：测试与打磨
| 编号 | 文件 | 类型 | 复杂度 | 依赖 | 说明 |
|------|------|------|--------|------|------|
| T-06 | tests/unit/avatar-service.test.ts | 新建 | 中 | T-03 | 单元测试：文件校验、上传成功/失败场景 |
| T-07 | tests/integration/avatar-controller.test.ts | 新建 | 中 | T-04, T-06 | 集成测试：完整上传流程、错误响应格式 |

### 依赖图
```
T-01 -> T-03 -> T-04 -> T-05
T-02 -> T-03         -> T-06
                 -> T-04 -> T-07
```

### 风险标记
| 任务编号 | 风险 | 缓解措施 |
|----------|------|----------|
| T-03 | 大文件上传可能导致超时 | 实现分片上传，限制最大 10MB |

### 并行机会
- T-01 和 T-02 可并行（无依赖）
- T-06 和 T-07 可并行（分别测试服务和控制器）

---

## 示例 2（复杂场景）：订单处理功能

### 概述
- 总任务数：18
- 预估总工时：42 小时
- 最大并行深度：4 层

### 第 1 层：基础
| 编号 | 文件 | 类型 | 复杂度 | 依赖 | 说明 |
|------|------|------|--------|------|------|
| T-01 | migrations/003_add_orders.sql | 新建 | 低 | 无 | 创建 orders 和 order_items 表，添加索引 |
| T-02 | src/orders/types.ts | 新建 | 中 | 无 | 定义订单相关类型：Order, OrderItem, OrderStatus, CreateOrderDTO |
| T-03 | src/orders/errors.ts | 新建 | 低 | 无 | 定义订单业务异常类：InsufficientStockError, InvalidOrderError |
| T-04 | src/inventory/client.ts | 修改 | 低 | 无 | 添加 InventoryClient.checkStock() 方法 |
| T-05 | src/pricing/engine.ts | 修改 | 低 | 无 | 添加 PricingEngine.calculate() 方法，支持批量计价 |

### 第 2 层：核心逻辑
| 编号 | 文件 | 类型 | 复杂度 | 依赖 | 说明 |
|------|------|------|--------|------|------|
| T-06 | src/orders/repository.ts | 新建 | 中 | T-01, T-02 | 实现订单数据访问层：create, findById, updateStatus |
| T-07 | src/orders/order-service.ts | 新建 | 高 | T-02, T-03, T-04, T-05, T-06 | 核心业务逻辑：校验库存 -> 计算价格 -> 创建订单 -> 发起支付 |
| T-08 | src/payments/payment-service.ts | 新建 | 高 | T-02, T-06 | 支付处理：调用支付网关、处理回调、更新订单状态 |
| T-09 | src/events/order-events.ts | 新建 | 低 | T-02 | 定义订单事件：OrderCreatedEvent, OrderPaidEvent, OrderFailedEvent |

### 第 3 层：接口与集成
| 编号 | 文件 | 类型 | 复杂度 | 依赖 | 说明 |
|------|------|------|--------|------|------|
| T-10 | src/orders/controller.ts | 新建 | 中 | T-07 | 实现 REST 端点：POST /api/v1/orders, GET /api/v1/orders/:id |
| T-11 | src/payments/webhook.ts | 新建 | 中 | T-08 | 实现支付回调端点：POST /api/v1/payments/webhook |
| T-12 | src/routes/order-routes.ts | 新建 | 低 | T-10, T-11 | 注册订单相关路由 |
| T-13 | src/workers/payment-retry.ts | 新建 | 中 | T-08, T-09 | 支付失败重试 Worker：指数退避，最多 3 次 |
| T-14 | src/middleware/order-validator.ts | 新建 | 中 | T-02, T-03 | 订单请求校验中间件：DTO 校验、用户权限检查 |

### 第 4 层：测试与打磨
| 编号 | 文件 | 类型 | 复杂度 | 依赖 | 说明 |
|------|------|------|--------|------|------|
| T-15 | tests/unit/order-service.test.ts | 新建 | 高 | T-07 | 单元测试：订单创建流程、库存不足、价格计算异常 |
| T-16 | tests/unit/payment-service.test.ts | 新建 | 高 | T-08 | 单元测试：支付成功/失败、幂等性、回调处理 |
| T-17 | tests/integration/order-flow.test.ts | 新建 | 高 | T-10, T-11, T-15, T-16 | 集成测试：完整订单流程（创建 -> 支付 -> 回调 -> 状态更新） |
| T-18 | docs/api/order-api.md | 新建 | 低 | T-10, T-11 | API 文档：请求/响应示例、错误码说明 |

### 依赖图
```
T-01 -> T-06 -> T-07 -> T-10 -> T-12 -> T-17
T-02 -> T-06 -> T-07 -> T-10         -> T-15 -> T-17
     -> T-07 -> T-08 -> T-11 -> T-12
     -> T-09         -> T-13
T-03 -> T-07
T-04 -> T-07
T-05 -> T-07

并行机会：
- 第 1 层：T-01, T-02, T-03, T-04, T-05 全部可并行
- 第 2 层：T-06, T-07, T-08, T-09 中，T-06 可独立并行，T-07/T-08 需等 T-06
- 第 3 层：T-10/T-14 可并行，T-11/T-13 可并行
- 第 4 层：T-15, T-16 可并行，T-17 需等所有测试完成
```

### 风险标记
| 任务编号 | 风险 | 缓解措施 |
|----------|------|----------|
| T-07 | 订单创建涉及多个外部依赖，失败场景复杂 | 使用 Saga 模式或补偿事务，确保最终一致性 |
| T-08 | 支付回调可能重复到达 | 实现幂等性：使用订单 ID + 状态机，重复回调直接返回成功 |
| T-17 | 集成测试需要 Mock 多个外部服务 | 使用测试容器（Testcontainers）或 WireMock |
| T-13 | 重试 Worker 可能导致消息堆积 | 监控死信队列，设置告警阈值 |

### 实施建议
1. **优先实现**：T-01, T-02, T-03（基础层无依赖，可立即开始）
2. **关键路径**：T-01 -> T-06 -> T-07 -> T-10 -> T-17（最长依赖链）
3. **配对编程**：T-07 和 T-08（高复杂度，建议两人协作）
4. **提前 Mock**：T-04 和 T-05 的接口定义后，T-07 可使用 Mock 先行开发
