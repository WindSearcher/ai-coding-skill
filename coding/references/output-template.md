# 编码输出模板

## 示例 1（简单场景）：实现头像上传 API 端点

### 实现摘要：T-04

#### 变更文件
- `src/controllers/avatar-controller.ts` - 新建，实现头像上传控制器
- `src/routes/user-routes.ts` - 修改，注册 `/users/:id/avatar` 路由
- `src/middleware/upload.ts` - 新建，配置 multer 中间件处理 multipart/form-data

#### 决策说明
- **选择 multer 而非 express-fileupload**：项目已有 multer 依赖，且团队更熟悉其 API
- **文件大小限制设为 10MB**：根据需求分析中的边界场景 BE-02，需要在中间件层面限制
- **使用内存存储而非磁盘存储**：直接上传到 OSS，无需本地临时文件，减少 I/O 开销

#### 测试情况
- 运行 `npm test tests/unit/avatar-controller.test.ts`
- 测试覆盖：
  - ✅ 成功上传头像（200 响应，返回 CDN URL）
  - ✅ 文件大小超限（400 响应，错误信息清晰）
  - ✅ 非图片文件类型（400 响应，拒绝上传）
  - ✅ 用户无权修改他人头像（403 响应）
- 测试覆盖率：92%（语句），88%（分支）

#### 待办事项
- [ ] 后续优化：实现图片压缩（sharp 库），减少存储空间
- [ ] 需要运维配置 OSS bucket 的 CORS 策略
- [ ] 监控告警：上传失败率 >5% 时触发告警

---

## 示例 2（复杂场景）：实现 OrderService 核心逻辑

### 实现摘要：T-07

#### 变更文件
- `src/orders/order-service.ts` - 新建，核心业务逻辑（245 行）
- `src/orders/types.ts` - 修改，添加 CreateOrderCommand 类型
- `src/orders/errors.ts` - 修改，添加 InsufficientStockError, PricingError
- `src/inventory/client.ts` - 修改，实现 checkStock 方法
- `src/pricing/engine.ts` - 修改，实现 calculate 方法

#### 决策说明
- **使用 Saga 模式而非分布式事务**：
  - 原方案假设使用两阶段提交（2PC），但调研后发现外部支付网关不支持
  - 改为 Saga 模式：创建订单（补偿：取消订单）-> 扣减库存（补偿：恢复库存）-> 发起支付
  - 每个步骤都有对应的补偿操作，确保最终一致性
  
- **库存校验使用悲观锁**：
  - 原方案假设使用乐观锁（版本号），但高并发场景下冲突率高
  - 改为 `SELECT ... FOR UPDATE`，虽然吞吐量略低，但避免超卖
  
- **价格计算缓存策略调整**：
  - 原方案假设缓存 5 分钟，但实际测试发现促销价格变更频繁
  - 改为缓存 1 分钟，并提供手动刷新缓存的 API

- **与技术方案偏差**：
  - 偏差 1：技术方案假设同步调用支付网关，实际改为异步（先创建订单，后台 Worker 处理支付）
  - 原因：支付网关平均响应时间 2s，同步调用会导致 API 超时
  - 影响：用户提交订单后立即返回"订单创建成功，处理中"，通过 WebSocket 推送支付结果

#### 测试情况
- 运行 `npm test tests/unit/order-service.test.ts`
- 测试覆盖：
  - ✅ 正常流程：库存充足 -> 价格计算成功 -> 订单创建成功
  - ✅ 库存不足：抛出 InsufficientStockError，回滚已分配的资源
  - ✅ 价格计算失败：使用兜底价格（基础价格），记录警告日志
  - ✅ 并发场景：10 个用户同时购买最后 1 件商品，只有 1 个成功
  - ✅ 补偿逻辑：库存扣减失败后，已创建的订单状态改为 CANCELLED
- 测试覆盖率：87%（语句），82%（分支）
- 集成测试：`tests/integration/order-flow.test.ts` 通过（Mock 外部服务）

#### 性能数据
- 单次订单创建平均耗时：180ms（p50），320ms（p95），450ms（p99）
- 满足技术方案中的性能目标：p99 <500ms
- QPS 压测结果：500 QPS 时错误率 <0.1%

#### 待办事项
- [ ] 高优先级：实现支付失败重试 Worker（T-13），当前使用简单 setTimeout
- [ ] 中优先级：添加分布式链路追踪（trace ID），便于排查跨服务问题
- [ ] 低优先级：订单创建成功后发送通知邮件（需求分析中的 FR-04，推迟到 v2）
- [ ] 技术债务：OrderService 的 createOrder 方法 245 行，建议后续拆分为多个私有方法（单个方法 <50 行）
- [ ] 需要 DBA 审查 SQL 查询计划，确认索引使用正确
