# 知识归档输出模板

## 示例 1（简单场景）：用户头像上传功能归档

### 背景
- **需求来源**：PRD-2024-03-15 用户体验优化
- **业务目标**：允许用户上传自定义头像，提升个性化体验和平台活跃度
- **利益相关者**：产品团队（张三）、前端团队（李四）、后端团队（王五）

### 设计决策
| 决策 | 理由 | 已拒绝的替代方案 |
|------|------|----------------|
| 使用阿里云 OSS 存储图片 | 团队已有 OSS 使用经验，CDN 集成简单 | 1. 本地存储：扩展性差，需自行实现 CDN<br>2. AWS S3：团队不熟悉，配置复杂 |
| 限制文件大小 10MB | 平衡用户体验和存储成本 | 1. 5MB：部分高清图片无法上传<br>2. 20MB：存储成本过高 |
| 同步上传而非异步 | 简单场景无需异步，用户体验更好 | 异步上传：需额外实现进度查询和通知机制 |

### 实现概述
- **核心模块**：
  - `AvatarController`：处理 HTTP 请求，校验文件类型和大小
  - `AvatarService`：调用 OSS SDK 上传图片，生成缩略图
  - `UploadMiddleware`：配置 multer 处理 multipart/form-data
- **入口点**：`POST /api/v1/users/:id/avatar`
- **数据流**：客户端 -> Controller（校验） -> Service（上传 OSS） -> 返回 CDN URL
- **外部依赖**：
  - 阿里云 OSS SDK v2.1.0
  - multer v1.4.5（文件解析）
  - sharp v0.32.6（图片压缩，计划 v2 实现）

### 配置
| 名称 | 默认值 | 说明 |
|------|--------|------|
| `STORAGE_OSS_BUCKET` | `user-avatars` | OSS bucket 名称 |
| `STORAGE_OSS_REGION` | `cn-hangzhou` | OSS 区域 |
| `STORAGE_MAX_FILE_SIZE` | `10485760` | 最大文件大小（10MB，字节） |
| `STORAGE_ALLOWED_TYPES` | `image/jpeg,image/png,image/gif` | 允许的文件类型 |
| `CDN_DOMAIN` | `https://cdn.example.com` | CDN 域名，用于生成图片 URL |

### 运维指南
- **健康检查**：
  - 调用 `GET /api/v1/health/storage`，检查 OSS 连接是否正常
  - 预期响应：`{ "status": "ok", "storage": "connected" }`
- **常见问题**：
  - **问题**：上传失败，返回 500 错误
    - **诊断**：检查 OSS 配置是否正确，网络是否可达
    - **解决**：验证 `STORAGE_OSS_*` 环境变量，测试 OSS SDK 连接
  - **问题**：上传成功但 CDN 返回 404
    - **诊断**：CDN 缓存未刷新或配置错误
    - **解决**：清除 CDN 缓存，检查 bucket 权限是否为公共读
- **监控**：
  - 指标：`avatar_upload_total`（上传次数），`avatar_upload_duration_seconds`（上传耗时）
  - 告警：上传失败率 >5%（5 分钟窗口），P99 延迟 >2s

### 维护注意事项
- **技术债务**：
  - 未实现图片压缩（使用 sharp 库），当前直接上传原图，存储成本较高
  - 计划：v2 迭代中实现，预估 2 小时工作量
- **未来改进**：
  - 支持头像裁剪（前端集成 cropper.js）
  - 支持多种尺寸缩略图（小/中/大）
  - 集成内容审核（检测违规图片）
- **扩展考虑**：
  - 当用户量超过 100 万时，考虑使用 OSS 生命周期策略自动清理不活跃用户的头像
  - 当 QPS 超过 1000 时，考虑引入 CDN 预 warm 机制
- **相关文档**：
  - [需求分析](../requirements/2024-03-15-用户头像上传.requirements.md)
  - [技术方案](../design/2024-03-15-用户头像上传.tech-design.md)
  - [代码审查](../review/2024-03-15-用户头像上传.review.md)

---

## 示例 2（复杂场景）：订单处理系统归档

### 背景
- **需求来源**：PRD-2024-02-01 电商平台核心交易链路
- **业务目标**：实现完整的订单创建、支付、履约流程，支撑日均 10 万单交易规模
- **利益相关者**：产品团队（赵六）、交易团队（孙七）、支付团队（周八）、仓储团队（吴九）、财务团队（郑十）

### 设计决策

#### 核心架构决策
| 决策 | 理由 | 已拒绝的替代方案 |
|------|------|----------------|
| 使用 Saga 模式保证最终一致性 | 外部支付网关不支持 2PC，且跨服务分布式事务性能差 | 1. 两阶段提交（2PC）：支付网关不支持，且阻塞时间长<br>2. TCC 模式：实现复杂度高，需每个服务提供 Try/Confirm/Cancel |
| 库存校验使用悲观锁（SELECT FOR UPDATE） | 避免超卖，高并发场景下冲突率低于乐观锁 | 1. 乐观锁（版本号）：压测发现冲突率 15%，用户体验差<br>2. Redis 预扣减：复杂度高，需处理 Redis 与 DB 不一致 |
| 支付处理改为异步（后台 Worker） | 支付网关平均响应 2s，同步调用导致 API 超时 | 1. 同步调用：P99 延迟 3.5s，超过 2s SLA<br>2. 前端轮询：增加服务器压力，用户体验差 |
| 使用 Redis 缓存价格（TTL 60s） | 价格计算频繁查询数据库，P99 延迟 800ms | 1. 无缓存：数据库 QPS 峰值 5000，接近瓶颈<br>2. 长缓存（5 分钟）：促销价格变更不及时 |

#### 数据模型决策
| 决策 | 理由 | 已拒绝的替代方案 |
|------|------|----------------|
| 订单和订单项分表存储 | 一对多关系，避免单行过大，便于查询优化 | 1. JSON 字段存储订单项：无法高效查询和统计<br>2. 宽表（订单 + 订单项合并）：数据冗余，更新复杂 |
| 订单状态使用枚举字符串 | 可读性好，便于调试和日志分析 | 1. 整数状态码：需维护映射表，调试不直观<br>2. 状态机库（xstate）：过度设计，简单场景无需 |

#### 性能与安全决策
| 决策 | 理由 | 已拒绝的替代方案 |
|------|------|----------------|
| 支付回调验证签名防伪造 | 安全合规要求，防止恶意用户伪造支付成功 | 1. 仅验证订单状态：存在安全风险<br>2. IP 白名单：支付网关 IP 动态变化，维护成本高 |
| 限流：每用户每分钟 10 单 | 防止恶意刷单，保护下游服务 | 1. 无限流：黑产可瞬间创建百万订单<br>2. 全局限流：误伤正常用户 |

### 实现概述
- **核心模块**：
  - `OrderService`：订单创建核心逻辑，编排库存校验、价格计算、订单持久化（Saga 模式）
  - `PaymentService`：支付处理，调用支付网关，处理回调，更新订单状态
  - `OrderController`：REST API 端点，请求校验，权限检查
  - `PaymentWebhook`：支付回调处理，签名验证，幂等性保证
  - `PaymentRetryWorker`：支付失败重试，指数退避，最多 3 次
  - `OrderRepository`：数据访问层，订单 CRUD，状态流转
- **入口点**：
  - `POST /api/v1/orders` - 创建订单
  - `GET /api/v1/orders/:id` - 查询订单
  - `POST /api/v1/payments/webhook` - 支付回调
- **数据流**：
  ```
  用户 -> OrderController -> OrderService（Saga）:
    1. 校验库存（悲观锁）
    2. 计算价格（Redis 缓存）
    3. 创建订单（PENDING 状态）
    4. 发起支付（异步）
  
  PaymentWorker -> 支付网关 -> PaymentWebhook:
    5. 支付网关回调
    6. 验证签名
    7. 更新订单状态（PAID/FAILED）
    8. 发出 OrderPaidEvent/OrderFailedEvent
  ```
- **外部依赖**：
  - 库存服务 v2.3.0（gRPC 接口）
  - 计价引擎 v1.8.2（REST API）
  - 支付网关 v3.1.0（SDK，支持签名验证）
  - Redis 7.0（缓存和分布式锁）
  - PostgreSQL 15（主数据库）

### 配置
| 名称 | 默认值 | 说明 |
|------|--------|------|
| `DB_HOST` | `localhost` | PostgreSQL 地址 |
| `DB_PORT` | `5432` | PostgreSQL 端口 |
| `DB_NAME` | `orders_db` | 数据库名称 |
| `REDIS_URL` | `redis://localhost:6379` | Redis 连接字符串 |
| `PRICE_CACHE_TTL` | `60` | 价格缓存 TTL（秒） |
| `PAYMENT_GATEWAY_API_KEY` | `<secret>` | 支付网关 API 密钥（Vault 管理） |
| `PAYMENT_CALLBACK_SECRET` | `<secret>` | 支付回调签名密钥（Vault 管理） |
| `ORDER_CREATE_RATE_LIMIT` | `10` | 每用户每分钟创建订单限制 |
| `PAYMENT_RETRY_MAX_ATTEMPTS` | `3` | 支付失败最大重试次数 |
| `PAYMENT_RETRY_BASE_DELAY` | `1000` | 重试基础延迟（毫秒，指数退避） |

### 运维指南

#### 健康检查
- **订单服务健康**：`GET /api/v1/health`
  - 检查项：数据库连接、Redis 连接、库存服务连通性
  - 预期响应：`{ "status": "ok", "db": "connected", "redis": "connected", "inventory": "ok" }`
- **支付 Worker 健康**：监控消息队列堆积情况
  - 告警：待处理支付任务 >1000 持续 5 分钟

#### 常见问题
- **问题 1**：订单创建失败，返回 500 错误
  - **现象**：用户提交订单后收到 "Internal Server Error"
  - **诊断**：
    1. 检查日志：`grep "order-service" /var/log/app.log | grep ERROR`
    2. 检查数据库连接：`psql -h $DB_HOST -U $DB_USER -d $DB_NAME -c "SELECT 1"`
    3. 检查库存服务：`grpc_health_probe -addr inventory-service:9090`
  - **解决**：
    - 数据库连接失败：重启连接池，检查网络
    - 库存服务不可用：触发降级策略，返回 "请稍后重试"

- **问题 2**：支付回调重复到达，订单状态异常
  - **现象**：同一订单被多次标记为已支付
  - **诊断**：检查幂等性逻辑，查看日志中的 `payment callback received` 记录
  - **解决**：
    - 确认使用了 `orderId + status` 作为幂等键
    - 检查数据库唯一索引：`CREATE UNIQUE INDEX idx_order_payment ON orders(id, status) WHERE status='PAID'`
    - 手动修复：`UPDATE orders SET status='PAID' WHERE id='<order_id>'`

- **问题 3**：库存超卖
  - **现象**：商品库存为负数
  - **诊断**：检查是否使用了悲观锁，查看数据库事务隔离级别
  - **解决**：
    - 确认 SQL 使用 `SELECT ... FOR UPDATE`
    - 检查事务是否正确提交/回滚
    - 紧急修复：手动调整库存，通知受影响用户

#### 监控
- **核心指标**：
  - `orders_created_total`（订单创建总数）
  - `orders_payment_success_rate`（支付成功率，目标 >95%）
  - `order_creation_duration_seconds`（订单创建耗时，P99 <500ms）
  - `inventory_check_failures_total`（库存校验失败数）
  - `payment_retry_queue_size`（支付重试队列大小）
- **告警规则**：
  - 订单创建失败率 >5%（5 分钟窗口） -> P1 告警
  - 支付成功率 <90%（15 分钟窗口） -> P1 告警
  - P99 延迟 >1s（10 分钟窗口） -> P2 告警
  - 支付重试队列 >1000 -> P2 告警
- **日志关键字**：
  - `ORDER_CREATED`：订单创建成功
  - `PAYMENT_SUCCESS`：支付成功
  - `PAYMENT_FAILED`：支付失败
  - `INVENTORY_INSUFFICIENT`：库存不足
  - `SAGA_COMPENSATION`：Saga 补偿触发

#### 回滚流程
1. **代码回滚**：
   ```bash
   git revert <commit-hash>
   npm run deploy
   ```
2. **数据库回滚**（如 Schema 变更）：
   ```bash
   npm run migrate:down  # 执行 down 迁移
   ```
3. **数据修复**（如超卖）：
   - 导出受影响订单：`SELECT * FROM orders WHERE created_at > '<rollback_time>'`
   - 手动调整库存：`UPDATE products SET stock = stock + <quantity> WHERE id = <product_id>`
   - 通知用户：发送短信/邮件告知订单取消

### 维护注意事项

#### 技术债务
- **高优先级**：
  - `OrderService.createOrder()` 方法 245 行，违反单一职责原则
    - 影响：可读性差，单元测试复杂
    - 计划：拆分为 4 个私有方法，预估 4 小时
  - 支付失败重试使用简单 `setTimeout`，非持久化队列
    - 影响：服务重启后重试任务丢失
    - 计划：迁移到 Redis Queue 或 RabbitMQ，预估 8 小时

- **中优先级**：
  - 缺少分布式链路追踪（trace ID）
    - 影响：跨服务排查问题困难
    - 计划：集成 OpenTelemetry，预估 6 小时
  - 订单列表查询未分页
    - 影响：用户订单多时响应慢
    - 计划：添加游标分页，预估 3 小时

#### 未来改进
- **v2 规划**：
  - 支持资源级权限（如"只能查看自己创建的订单"）
  - 支持订单取消和退款流程
  - 集成消息通知（订单状态变更短信/邮件）
  - 支持批量订单创建（B2B 场景）
- **v3 规划**：
  - 支持多币种和汇率转换
  - 支持预售和定金模式
  - 集成风控系统（异常订单检测）

#### 扩展考虑
- **性能瓶颈**：
  - 当前单实例 QPS 500，预计 6 个月后达到瓶颈
  - 解决方案：水平扩展 OrderService，使用无状态设计
  - 数据库连接池上限 100，需监控连接使用情况
- **存储增长**：
  - 订单表每月增长约 300 万行，1 年后查询性能下降
  - 解决方案：按月份分区表，归档 1 年前订单到冷存储
- **依赖风险**：
  - 支付网关 SLA 99.9%，月度停机时间约 43 分钟
  - 解决方案：实现多支付网关冗余，主网关故障时自动切换

#### 相关文档
- [需求分析](../requirements/2024-02-01-订单处理系统.requirements.md)
- [技术方案](../design/2024-02-01-订单处理系统.tech-design.md)
- [任务拆解](../tasks/2024-02-01-订单处理系统.tasks.md)
- [代码审查](../review/2024-02-01-订单处理系统.review.md)
- [API 文档](../../api/order-api.md)
- [数据库 Schema](../../migrations/003_add_orders.sql)

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
