# 任务拆解输出模板

## 示例：订单处理功能

### 概述
- 总任务数：8
- 预估总工时：14 小时
- 最大并行深度：3 层

### 第 1 层：基础
| 编号 | 文件 | 类型 | 复杂度 | 依赖 | 说明 |
|------|------|------|--------|------|------|
| T-01 | migrations/003_add_orders.sql | 新建 | 低 | 无 | 创建 orders 和 order_items 表及索引 |
| T-02 | src/orders/types.ts | 新建 | 低 | 无 | 定义 Order、OrderItem、OrderStatus 的 DTO 和枚举 |

### 第 2 层：核心逻辑
| 编号 | 文件 | 类型 | 复杂度 | 依赖 | 说明 |
|------|------|------|--------|------|------|
| T-03 | src/orders/repository.ts | 新建 | 中 | T-01, T-02 | 实现 OrderRepository，含 CRUD 和状态流转 |
| T-04 | src/inventory/client.ts | 修改 | 低 | 无 | 添加 `checkStock(productId, qty)` 方法 |
| T-05 | src/orders/service.ts | 新建 | 高 | T-02, T-03, T-04 | 实现 OrderService：校验、计价调用、持久化 |

### 第 3 层：接口与集成
| 编号 | 文件 | 类型 | 复杂度 | 依赖 | 说明 |
|------|------|------|--------|------|------|
| T-06 | src/orders/controller.ts | 新建 | 中 | T-02, T-05 | 实现 POST /orders 和 GET /orders/:id，含认证中间件 |
| T-07 | src/orders/events.ts | 新建 | 低 | T-02 | 定义 OrderPaidEvent 和 OrderFailedEvent 载荷 |

### 第 4 层：测试与打磨
| 编号 | 文件 | 类型 | 复杂度 | 依赖 | 说明 |
|------|------|------|--------|------|------|
| T-08 | src/orders/service.test.ts | 新建 | 中 | T-05 | OrderService 单元测试，模拟依赖 |
| T-09 | src/orders/controller.test.ts | 新建 | 中 | T-06 | Controller 端点集成测试 |

### 依赖图
```
T-01 ─┬─> T-03 ─┬─> T-05 ─┬─> T-06 ──> T-09
T-02 ─┘         └─> T-04 ─┘         └─> T-08
                              └─> T-07
```

### 风险标记
| 任务编号 | 风险 | 缓解措施 |
|----------|------|----------|
| T-05 | 计价集成复杂度高 | 为 PricingEngine 添加显式契约测试 |
| T-06 | 认证中间件交互 | 用集成测试验证中间件行为 |
