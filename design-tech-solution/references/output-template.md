# 技术方案输出模板

## 示例：订单处理功能

### 概述
- 状态：草稿
- 需求参考：需求分析/订单处理.md

### 系统流程
```
1. 用户提交订单 -> POST /api/v1/orders
2. OrderService 通过 InventoryClient 校验库存
3. OrderService 使用 PricingEngine 计算价格
4. 订单以 PENDING 状态持久化
5. PaymentService 异步处理扣款
6. 成功：状态 -> PAID，发出 OrderPaidEvent
7. 失败：状态 -> PAYMENT_FAILED，通知用户
```

### 数据模型
```sql
CREATE TABLE orders (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL REFERENCES users(id),
    status VARCHAR(20) NOT NULL, -- PENDING, PAID, FAILED, CANCELLED
    total_amount DECIMAL(12,2) NOT NULL,
    currency VARCHAR(3) NOT NULL DEFAULT 'USD',
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE order_items (
    id UUID PRIMARY KEY,
    order_id UUID NOT NULL REFERENCES orders(id),
    product_id UUID NOT NULL,
    quantity INT NOT NULL CHECK (quantity > 0),
    unit_price DECIMAL(12,2) NOT NULL
);
```

### 外部依赖
| 服务 | 用途 | 故障模式 | 降级方案 |
|------|------|----------|----------|
| 库存服务 | 库存校验 | 超时 | 拒绝订单并提示重试 |
| 支付网关 | 信用卡扣款 | 5xx | 加入重试队列，通知用户 |
| 计价引擎 | 动态定价 | 超时 | 使用缓存的基础价格 |

### 模块设计

#### 模块：OrderController
- 职责：HTTP 请求处理，输入 DTO 校验
- 接口：`POST /api/v1/orders`，`GET /api/v1/orders/:id`
- 内部函数：`createOrder(dto) -> OrderResponse`

#### 模块：OrderService
- 职责：业务逻辑，编排协调
- 接口：`createOrder(cmd) -> Order`，`getOrder(id) -> Order`
- 内部函数：`validateInventory()`，`calculatePricing()`，`persistOrder()`

### 改动清单
| 文件 | 类型 | 复杂度 | 说明 |
|------|------|--------|------|
| src/orders/controller.ts | 新建 | 中 | REST 端点 |
| src/orders/service.ts | 新建 | 高 | 核心业务逻辑 |
| src/orders/repository.ts | 新建 | 低 | 数据库访问层 |
| src/inventory/client.ts | 修改 | 低 | 添加库存校验方法 |
| migrations/003_add_orders.sql | 新建 | 低 | Schema 迁移 |

### 非功能性设计
- 性能：订单创建目标 p99 <500ms。缓存定价结果 5 分钟。
- 安全：GET 订单时校验用户所有权。限流：每用户每分钟 10 单。
- 可观测性：记录订单生命周期事件。指标：`orders_created_total`。
- 可靠性：库存校验重试 3 次，指数退避。失败支付进入死信队列。

### 风险与应对
| 风险 | 影响 | 应对措施 |
|------|------|----------|
| 重复扣款 | 高 | 支付请求使用幂等键 |
| 超卖 | 高 | 悲观锁或原子递减 |
| 价格过时 | 中 | 定价缓存设 TTL，回退基础价格 |
