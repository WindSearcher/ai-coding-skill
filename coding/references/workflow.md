# 编码工作流程详解

## 增量实现模式

1. **先搭骨架**
   ```typescript
   // 在实现前先定义接口
   export interface OrderService {
     createOrder(cmd: CreateOrderCommand): Promise<Order>;
   }
   ```

2. ** stub 实现**
   ```typescript
   export class OrderServiceImpl implements OrderService {
     async createOrder(cmd: CreateOrderCommand): Promise<Order> {
       // TODO: 实现
       throw new Error("未实现");
     }
   }
   ```

3. **填充核心逻辑**
   - 先实现主流程
   - 添加校验
   - 添加错误处理

4. **边界场景与打磨**
   - 处理 null、空输入、边界值
   - 添加指标/日志
   - 如需优化则优化（先分析再优化）

## 测试驱动开发（可选）

当方案稳定时：
1. 先写失败的测试
2. 写最少代码让它通过
3. 重构
4. 重复

## 常见陷阱

- **过度工程**：在需要之前就添加抽象
- **忽视现有模式**：始终参考类似功能是如何实现的
- **跳过错误处理**：每个外部调用都可能失败
- **硬编码**：超时、限制、URL 等应使用配置
