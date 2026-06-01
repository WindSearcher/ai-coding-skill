# 代码审查输出模板

## 示例 1（简单场景）：审查头像上传功能的 PR

### 概述
- 审查文件数：3
- 发现问题：1 个严重、1 个警告、2 个建议
- 结论：有条件通过（修复严重问题后可合并）

### 严重问题（必须修复）
| 编号 | 位置 | 问题 | 建议修复 |
|------|------|------|----------|
| C1 | `src/controllers/avatar-controller.ts:45` | 未校验文件类型，用户可上传可执行文件（.exe, .sh） | 在 multer 配置中添加 `fileFilter`，仅允许 `image/jpeg`, `image/png`, `image/gif` |

### 警告（应该修复）
| 编号 | 位置 | 问题 | 建议修复 |
|------|------|------|----------|
| W1 | `src/middleware/upload.ts:12` | 文件大小限制 10MB 硬编码，应使用配置项 | 从 `config.storage.maxFileSize` 读取，便于环境差异配置 |

### 建议（锦上添花）
| 编号 | 位置 | 问题 | 建议修复 |
|------|------|------|----------|
| S1 | `src/controllers/avatar-controller.ts:23` | 函数 `uploadAvatar` 超过 60 行，建议拆分 | 将文件校验逻辑提取为独立函数 `validateImageFile()` |
| S2 | `tests/unit/avatar-controller.test.ts` | 缺少对 CDN 上传失败的测试场景 | 添加 Mock OSS 客户端抛出异常的测试用例 |

### 亮点
- ✅ 错误响应格式统一，遵循项目的 `ApiError` 规范
- ✅ 测试覆盖率高（92%），边界场景考虑全面
- ✅ 使用内存存储避免磁盘 I/O，性能优化得当

---

## 示例 2（复杂场景）：审查订单处理系统的 PR

### 概述
- 审查文件数：12
- 发现问题：3 个严重、4 个警告、5 个建议
- 结论：需要修改（修复所有严重问题后再审查）

### 严重问题（必须修复）

#### 安全漏洞
| 编号 | 位置 | 问题 | 建议修复 |
|------|------|------|----------|
| C1 | `src/orders/controller.ts:67` | 未校验用户只能访问自己的订单，存在越权风险 | 在 `getOrder` 方法中添加 `if (order.userId !== req.user.id) throw new ForbiddenError()` |
| C2 | `src/payments/webhook.ts:34` | 支付回调未验证签名，恶意用户可伪造支付成功通知 | 使用支付网关提供的 SDK 验证签名：`paymentGateway.verifySignature(req.body, req.headers['x-signature'])` |

#### 业务逻辑错误
| 编号 | 位置 | 问题 | 建议修复 |
|------|------|------|----------|
| C3 | `src/orders/order-service.ts:128` | 库存扣减和订单创建不在同一事务中，可能导致超卖 | 使用数据库事务包裹：`await db.transaction(async (tx) => { await tx.inventory.deduct(...); await tx.orders.create(...); })` |

### 警告（应该修复）

#### 性能隐患
| 编号 | 位置 | 问题 | 建议修复 |
|------|------|------|----------|
| W1 | `src/orders/repository.ts:45` | N+1 查询问题：循环中查询每个订单的订单项 | 使用 JOIN 或批量查询：`SELECT * FROM orders o LEFT JOIN order_items oi ON o.id = oi.order_id WHERE o.id IN (...)` |
| W2 | `src/pricing/engine.ts:89` | 价格计算未使用缓存，每次调用都查询数据库 | 添加 Redis 缓存，TTL 60s：`const cached = await redis.get(`price:${productId}`); if (cached) return JSON.parse(cached);` |

#### 可维护性问题
| 编号 | 位置 | 问题 | 建议修复 |
|------|------|------|----------|
| W3 | `src/orders/order-service.ts` | `createOrder` 方法 245 行，违反单一职责原则 | 拆分为：`validateOrder()`, `checkInventory()`, `calculatePrice()`, `persistOrder()` 四个私有方法 |
| W4 | `src/events/order-events.ts:12` | 事件载荷包含敏感信息（用户邮箱、手机号） | 移除敏感字段，仅保留 `userId`，消费者需要时再查询 |

### 建议（锦上添花）
| 编号 | 位置 | 问题 | 建议修复 |
|------|------|------|----------|
| S1 | `src/orders/controller.ts` | 错误码使用魔法数字（400, 403, 500） | 使用枚举：`HttpStatusCode.BAD_REQUEST` |
| S2 | `tests/unit/order-service.test.ts:156` | Mock 对象过于复杂，测试可读性差 | 使用工厂函数 `createMockInventoryClient({ stock: 10 })` 提高可读性 |
| S3 | `src/workers/payment-retry.ts` | 重试间隔固定 5s，建议改为指数退避 | 第 1 次 1s，第 2 次 2s，第 3 次 4s：`delay = Math.pow(2, attempt) * 1000` |
| S4 | `src/orders/types.ts` | 缺少 JSDoc 注释，IDE 提示不友好 | 为 `Order`, `OrderItem`, `CreateOrderDTO` 添加 JSDoc |
| S5 | `docs/api/order-api.md` | API 文档缺少请求示例 | 添加 cURL 示例和响应示例（成功/失败） |

### 亮点
- ✅ Saga 模式实现优雅，补偿逻辑完整（库存恢复、订单取消）
- ✅ 幂等性设计到位：支付回调使用 `orderId + status` 作为幂等键
- ✅ 错误处理全面，所有外部调用都有降级策略
- ✅ 集成测试覆盖完整订单流程，包含 Mock 外部服务

### 审查总结

#### 符合设计
- ✅ 实现了技术方案中的所有核心模块
- ✅ 数据模型与设计方案一致
- ✅ 非功能性需求（性能、安全、可观测性）有所体现

#### 偏离设计
- ⚠️ 技术方案假设同步调用支付网关，实际改为异步（记录在实现摘要中，合理）
- ⚠️ 库存校验使用悲观锁而非乐观锁（实现摘要中说明原因，可接受）

#### 需求覆盖
- ✅ 需求分析中的所有用户故事都有对应实现
- ✅ 边界场景 BE-01 到 BE-09 均已处理
- ⚠️ 待澄清问题 OQ-01（资源级权限）推迟到 v2，符合预期

### 下一步行动
1. **立即修复**：C1, C2, C3（严重问题，阻塞合并）
2. **本轮修复**：W1, W2, W3, W4（警告，建议本轮修复）
3. **后续优化**：S1-S5（建议，可放入技术债务 backlog）
4. **重新审查**：修复 C1-C3 后重新触发审查，重点关注 W1-W4
