# 技术方案输出模板

## 示例 1（简单场景）：为用户资料添加头像上传功能

### 概述
- 状态：草稿
- 需求参考：需求分析/用户头像上传.md

### 系统流程
```
1. 用户选择图片文件 -> POST /api/v1/users/:id/avatar
2. Controller 校验文件类型（仅允许 image/jpeg, image/png, image/gif）
3. Controller 校验文件大小（<10MB）
4. Service 生成唯一文件名（UUID + 扩展名）
5. Service 上传图片到阿里云 OSS
6. OSS 返回文件 URL
7. Service 更新用户记录中的 avatar_url 字段
8. 返回 CDN URL 给客户端
```

### 数据模型
```sql
-- 修改现有 users 表
ALTER TABLE users ADD COLUMN avatar_url VARCHAR(500);
ALTER TABLE users ADD COLUMN avatar_updated_at TIMESTAMP;
CREATE INDEX idx_users_avatar_updated ON users(avatar_updated_at);
```

### 外部依赖
| 服务 | 用途 | 故障模式 | 降级方案 |
|------|------|----------|----------|
| 阿里云 OSS | 图片存储 | 网络超时、5xx 错误 | 返回错误提示“上传服务暂时不可用，请稍后重试” |
| CDN | 图片加速 | 缓存未命中 | 回源到 OSS，性能略降 |

### 模块设计

#### 模块：AvatarController
- 职责：HTTP 请求处理，文件校验（类型、大小）
- 接口：`POST /api/v1/users/:id/avatar`（multipart/form-data）
- 内部函数：`uploadAvatar(req, res) -> { cdnUrl: string }`

#### 模块：AvatarService
- 职责：调用 OSS SDK 上传图片，更新数据库
- 接口：`uploadAvatar(userId, file) -> string`（返回 CDN URL）
- 内部函数：`generateFileName()`, `uploadToOSS()`, `updateUserAvatar()`

### 改动清单
| 文件 | 类型 | 复杂度 | 说明 |
|------|------|--------|------|
| src/users/avatar-controller.ts | 新建 | 低 | REST 端点，文件校验 |
| src/users/avatar-service.ts | 新建 | 低 | OSS 上传逻辑 |
| src/middleware/upload.ts | 新建 | 低 | multer 配置 |
| src/users/user-model.ts | 修改 | 低 | 添加 avatar_url 字段 |
| migrations/004_add_avatar.sql | 新建 | 低 | Schema 迁移 |

### 非功能性设计
- 性能：上传接口 P99 <500ms。OSS 直传，无中间存储。
- 安全：校验文件类型（MIME + 扩展名双重校验）。限流：每用户每分钟 5 次上传。
- 可观测性：记录上传成功/失败日志。指标：`avatar_upload_total`，`avatar_upload_duration`。
- 可靠性：OSS 上传失败自动重试 2 次。失败后返回明确错误信息。

### 风险与应对
| 风险 | 影响 | 应对措施 |
|------|------|----------|
| 用户上传恶意文件（.exe, .sh） | 高 | 双重校验：MIME 类型 + 文件扩展名白名单 |
| 大文件占用带宽 | 中 | 限制 10MB，前端也做校验 |
| OSS 服务不可用 | 中 | 降级策略，返回友好错误提示 |

---

## 示例 2（中等场景）：订单处理功能

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

---

## 示例 3（复杂场景）：实时协作文档编辑系统

### 概述
- 状态：草稿
- 需求参考：需求分析/实时协作文档.md

### 系统流程
```
1. 用户打开文档 -> WebSocket 连接到 CollaborationServer
2. 服务器加载文档当前状态和版本号
3. 用户编辑内容 -> 客户端发送 Operation（插入、删除、格式化）
4. CollaborationServer 使用 OT 算法转换操作
5. 应用转换后的操作到文档状态
6. 广播操作给其他在线协作者
7. 定期（每 5s）持久化文档到数据库
8. 用户关闭文档 -> WebSocket 断开，保存最终状态
```

### 数据模型
```sql
CREATE TABLE documents (
    id UUID PRIMARY KEY,
    title VARCHAR(500) NOT NULL,
    content JSONB NOT NULL,  -- 文档内容（Prosemirror 格式）
    version INT NOT NULL DEFAULT 1,  -- 乐观锁版本号
    owner_id UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE document_permissions (
    document_id UUID NOT NULL REFERENCES documents(id),
    user_id UUID NOT NULL REFERENCES users(id),
    permission VARCHAR(20) NOT NULL,  -- OWNER, EDITOR, VIEWER
    PRIMARY KEY (document_id, user_id)
);

CREATE TABLE document_operations (
    id UUID PRIMARY KEY,
    document_id UUID NOT NULL REFERENCES documents(id),
    user_id UUID NOT NULL REFERENCES users(id),
    operation JSONB NOT NULL,  -- OT 操作
    version INT NOT NULL,  -- 操作应用的版本号
    created_at TIMESTAMP DEFAULT NOW()
);

-- 索引
CREATE INDEX idx_documents_owner ON documents(owner_id);
CREATE INDEX idx_document_permissions_user ON document_permissions(user_id);
CREATE INDEX idx_document_ops_document ON document_operations(document_id, version);
```

### 外部依赖
| 服务 | 用途 | 故障模式 | 降级方案 |
|------|------|----------|----------|
| Redis Pub/Sub | WebSocket 服务器间消息广播 | Redis 宕机 | 降级为单服务器模式，跨服务器用户无法协作 |
| PostgreSQL | 文档持久化 | 数据库连接失败 | 内存缓存保留，恢复后批量写入 |
| WebSocket 负载均衡 | 连接路由 | 负载均衡器故障 | 使用 DNS 轮询，部分用户可能连接到不同服务器 |

### 模块设计

#### 模块：CollaborationServer
- 职责：WebSocket 连接管理，操作转换和广播
- 接口：WebSocket `/ws/collaboration/:documentId`
- 内部函数：`handleConnection()`, `transformOperation()`, `broadcastOperation()`
- 关键算法：Operational Transformation (OT)

#### 模块：OTEngine
- 职责：操作转换算法实现
- 接口：`transform(op1, op2) -> { op1', op2' }`
- 内部函数：`transformInsert()`, `transformDelete()`, `transformFormat()`
- 参考论文："Operational Transformation in Real-time Group Editors"

#### 模块：DocumentService
- 职责：文档 CRUD，权限校验，持久化
- 接口：`getDocument()`, `saveDocument()`, `checkPermission()`
- 内部函数：`loadDocument()`, `persistDocument()`, `validatePermission()`

#### 模块：PresenceService
- 职责：在线用户管理，光标位置同步
- 接口：`getUserPresence()`, `updateCursor()`
- 内部函数：`trackUser()`, `broadcastPresence()`, `cleanupDisconnected()`

### 改动清单
| 文件 | 类型 | 复杂度 | 说明 |
|------|------|--------|------|
| src/collaboration/server.ts | 新建 | 高 | WebSocket 服务器，连接管理 |
| src/collaboration/ot-engine.ts | 新建 | 高 | OT 算法实现 |
| src/collaboration/document-service.ts | 新建 | 中 | 文档持久化和权限 |
| src/collaboration/presence-service.ts | 新建 | 中 | 在线用户和光标 |
| src/collaboration/types.ts | 新建 | 中 | 类型定义（Operation, Document, Presence） |
| migrations/010_add_collaboration.sql | 新建 | 低 | Schema 迁移 |
| src/documents/controller.ts | 修改 | 低 | 添加协作相关端点 |

### 非功能性设计

#### 性能
- **延迟目标**：操作广播 P95 <100ms（同服务器），P95 <300ms（跨服务器）
- **并发目标**：单文档支持 50 人同时编辑，单服务器支持 10,000 个 WebSocket 连接
- **水平扩展**：
  - 使用 Redis Pub/Sub 实现多服务器间操作广播
  - 文档按 ID 哈希分配到主服务器，避免并发冲突
  - 无状态设计，可随时扩容服务器实例
- **缓存策略**：
  - 文档内容缓存在内存（LRU Cache，最多 1000 个文档）
  - 操作历史保留最近 100 个版本，用于冲突解决
  - 每 5 秒批量持久化到数据库，减少写压力

#### 安全
- **鉴权**：WebSocket 连接时验证 JWT Token
- **授权**：检查用户是否有文档编辑权限（EDITOR 或 OWNER）
- **输入校验**：
  - 操作大小限制（单个操作 <10KB）
  - 操作频率限制（每用户每秒最多 20 个操作）
  - 防止恶意操作（如无限循环插入）
- **数据保护**：文档内容加密存储（AES-256），传输层使用 WSS（WebSocket Secure）

#### 可观测性
- **日志**：
  - 记录用户连接/断开事件
  - 记录操作应用和广播（采样率 10%，避免日志风暴）
  - 记录 OT 转换失败和冲突
- **指标**：
  - `websocket_connections_total`（当前连接数）
  - `operations_processed_total`（处理的操作数）
  - `operation_latency_seconds`（操作处理延迟）
  - `ot_transform_failures_total`（OT 转换失败数）
  - `document_persist_duration_seconds`（持久化耗时）
- **链路追踪**：每个操作分配 trace ID，贯穿客户端 -> 服务器 -> 广播 -> 持久化

#### 可靠性
- **断线重连**：
  - 客户端检测到断开后自动重连（指数退避，最多 5 次）
  - 重连后同步缺失的操作（基于版本号）
- **冲突解决**：
  - OT 算法保证最终一致性
  - 极端情况下（OT 转换失败）回退到 Operational Conflict Resolution (OCR)：锁定文档，手动合并
- **数据丢失防护**：
  - 操作先写入内存队列，再广播和持久化
  - 服务器宕机时，未持久化的操作从 Redis 队列恢复
  - 定期备份文档快照（每 100 个版本）

### 风险与应对
| 风险 | 影响 | 应对措施 |
|------|------|----------|
| OT 算法复杂度高，转换失败 | 高 | 充分单元测试，覆盖所有操作组合；实现 OCR 降级方案 |
| WebSocket 连接数过多导致内存溢出 | 高 | 监控连接数，设置上限；使用流控和背压机制 |
| 跨服务器操作延迟高 | 中 | Redis Pub/Sub 延迟监控；延迟 >500ms 时告警；考虑同地域部署 |
| 文档内容过大（>10MB）导致性能下降 | 中 | 限制文档大小；实现分页加载；大文档使用 CRDT 替代 OT |
| 恶意用户频繁发送操作导致 DoS | 中 | 限流（每用户每秒 20 操作）；异常检测（操作模式异常时封禁） |
