---
name: analyze-project-structure
description: 分析并总结 Java 项目的工程结构、目录布局、技术栈、架构模式和代码风格。当用户提出以下需求时使用：(1) 分析、探索或理解 Java 项目的结构或架构，(2) 识别项目使用的技术栈、框架或构建工具，(3) 获取代码库的组织方式和关键模块概览，(4) 判断项目采用的架构模式（MVC/DDD/微服务等），(5) 了解项目的代码风格和规范。触发关键词包括：'分析项目结构'、'project architecture'、'技术栈分析'、'项目结构'、'codebase overview'、'了解这个项目'、'目录结构'、'工程结构'、'架构模式'。
---

# 工程结构分析

系统化分析 Java 项目的工程结构，提供清晰、结构化的概览。适用于独立使用场景，帮助用户快速理解陌生项目的整体架构。

## 分析流程

按以下步骤顺序执行，聚焦关键信息，避免过度分析。

### 1. 工程结构树

以递归方式遍历目录结构，生成带中文注释的树形结构：

**执行规则：**
1. 排除 `target/`、`.git/`、`.idea/`、`.vscode/`、`node_modules/` 等构建和 IDE 目录
2. 展示到叶节点文件夹（最后一级目录），但不包含具体文件
3. 每个关键目录后添加 `# 注释` 说明用途
4. 对于 Monorepo 或多模块项目，展示所有模块的顶层结构

**推荐命令：**
```bash
# 使用 tree 命令（优先）
tree -L 4 -d -I 'target|.git|.idea|.vscode|node_modules' 2>/dev/null

# 或使用 find 命令
find . -maxdepth 4 -type d -not -path '*/target/*' -not -path '*/.git/*' -not -path '*/.idea/*' | sort | head -100
```

**分析要点：**
- 识别项目类型：单模块 vs 多模块（Maven）vs 多模块（Gradle）
- 关注源码目录结构：`src/main/java/` 下的包组织方式
- 标记关键目录：controller、service、repository、config、dto 等
- 识别架构分层：是否按技术分层（MVC）或按业务分层（DDD）

### 2. 技术栈识别

通过特征文件和构建配置检测技术栈：

**构建工具识别：**
- **Maven**: `pom.xml`（读取 `<dependencies>` 和 `<plugins>` 节）
- **Gradle**: `build.gradle` 或 `build.gradle.kts`（读取 `dependencies` 块）
- **Gradle Kotlin DSL**: `build.gradle.kts`、`settings.gradle.kts`

**框架识别（通过依赖）：**
- **Spring Boot**: `spring-boot-starter-*`、`@SpringBootApplication`
- **Spring MVC**: `spring-webmvc`、`@Controller`、`@RestController`
- **Spring Cloud**: `spring-cloud-*`、`@EnableEurekaServer` 等
- **MyBatis**: `mybatis-spring-boot-starter`、`@Mapper`
- **JPA/Hibernate**: `spring-boot-starter-data-jpa`、`@Entity`
- **Dubbo**: `dubbo-spring-boot-starter`、`@DubboService`

**数据库识别：**
- **MySQL**: `mysql-connector-java`
- **PostgreSQL**: `postgresql`
- **Oracle**: `ojdbc8`
- **Redis**: `spring-boot-starter-data-redis`、`jedis`、`lettuce`

**中间件识别：**
- **消息队列**: `spring-kafka`、`rabbitmq`、`activemq`
- **注册中心**: `spring-cloud-starter-netflix-eureka`、`nacos-client`
- **配置中心**: `spring-cloud-starter-alibaba-nacos-config`

**执行步骤：**
1. 读取 `pom.xml` 或 `build.gradle` 文件
2. 提取 `<dependencies>` 或 `dependencies {}` 块
3. 识别核心依赖（Spring Boot、数据库、中间件等）
4. 记录版本信息（如 `spring-boot-starter-web:3.2.0`）

### 3. 架构模式分析

基于目录结构和代码特征判断架构模式：

**MVC 架构特征：**
```
src/main/java/
├── controller/     # 控制器层（处理 HTTP 请求）
├── service/        # 服务层（业务逻辑）
├── repository/     # 数据访问层（DAO）
├── model/          # 数据模型
└── config/         # 配置类
```
- **判断依据**：按技术职责分层，每层包含所有业务模块
- **适用场景**：简单 CRUD 应用、中小型项目

**DDD 架构特征：**
```
src/main/java/
├── order/                  # 订单限界上下文
│   ├── application/        # 应用服务（用例编排）
│   ├── domain/             # 领域层（核心业务逻辑）
│   │   ├── model/          # 聚合根、实体、值对象
│   │   ├── service/        # 领域服务
│   │   └── repository/     # 仓储接口
│   ├── infrastructure/     # 基础设施层（实现）
│   │   ├── repository/     # 仓储实现
│   │   └── mapper/         # MyBatis Mapper
│   └── interfaces/         # 接口层（Controller、MQ Listener）
├── user/                   # 用户限界上下文
└── shared/                 # 共享内核
```
- **判断依据**：按业务领域分包，每个包内有完整的分层结构
- **适用场景**：复杂业务系统、大型项目

**微服务架构特征：**
```
project-root/
├── user-service/           # 用户服务（独立 Spring Boot 应用）
│   ├── pom.xml
│   └── src/main/java/
├── order-service/          # 订单服务
├── payment-service/        # 支付服务
├── gateway/                # API 网关
└── docker-compose.yml
```
- **判断依据**：多个独立的可部署服务，各有自己的 `pom.xml` 或 `build.gradle`
- **识别标志**：`spring-cloud-starter-gateway`、`@EnableDiscoveryClient`、Docker Compose

**分层架构特征：**
```
src/main/java/
├── web/            # 展示层（Controller、DTO）
├── business/       # 业务层（Service、BO）
├── persistence/    # 持久层（DAO、Entity）
└── common/         # 公共层（工具类、常量）
```
- **判断依据**：明确的四层分离，层间单向依赖

**分析步骤：**
1. 观察 `src/main/java/` 下的顶级包结构
2. 判断是按技术分层还是按业务分层
3. 检查是否有明确的架构模式标志（如 DDD 的 aggregate、entity、valueobject）
4. 结合依赖判断是否使用了特定架构框架（如 Axon、Eventuate）

### 4. 代码风格分析

通过抽样代码文件分析项目的编码规范和风格：

**命名规范检查：**
- **类名**：PascalCase（如 `OrderService`）
- **方法名**：camelCase（如 `createOrder()`）
- **常量**：UPPER_SNAKE_CASE（如 `MAX_RETRY_COUNT`）
- **包名**：全小写（如 `com.example.order`）

**代码组织检查：**
- 是否使用 Lombok（`@Data`、`@Builder`、`@Slf4j`）
- 是否使用记录类（Java 14+ `record`）
- 异常处理方式（全局异常处理器 `@ControllerAdvice`）
- 日志框架（SLF4J + Logback/Log4j2）

**设计模式识别：**
- **工厂模式**：`*Factory` 类
- **策略模式**：`*Strategy` 接口和实现
- **模板方法**：`Abstract*` 类
- **建造者模式**：`*Builder` 类
- **观察者模式**：`ApplicationEvent`、`@EventListener`

**配置风格：**
- 注解配置 vs XML 配置
- 外部化配置（`application.yml`、`application.properties`）
- 多环境配置（`application-dev.yml`、`application-prod.yml`）

**抽样策略：**
1. 读取 2-3 个 Controller 文件，检查 REST API 风格
2. 读取 2-3 个 Service 文件，检查业务逻辑组织方式
3. 读取 1 个配置类，检查配置风格
4. 读取 1 个实体类，检查 ORM 使用方式

## 输出格式

以结构化报告呈现分析结果：

```markdown
## 项目描述

- **项目名称**: （取自 pom.xml 的 `<artifactId>` 或目录名）
- **项目类型**: （单模块应用 / 多模块 Maven / 多模块 Gradle / Monorepo / 微服务）
- **主语言**: Java （版本号，如 Java 17）
- **构建工具**: Maven 3.9 / Gradle 8.5
- **核心框架**: Spring Boot 3.2 / Spring Cloud 2023

## 工程结构树

```
（带注释的完整树形目录结构）
```

## 关键模块

| 模块 | 路径 | 职责 |
|------|------|------|
| ... | ... | ... |

## 技术栈

- **运行环境**: Java 17（LTS）
- **核心框架**: Spring Boot 3.2, Spring MVC
- **ORM 框架**: MyBatis 3.5 / Spring Data JPA
- **数据库**: MySQL 8.0 / PostgreSQL 15
- **缓存**: Redis 7（Lettuce 客户端）
- **消息队列**: Kafka 3.6 / RabbitMQ 3.12
- **注册中心**: Nacos 2.3 / Eureka
- **核心依赖**:
  - lombok（简化代码）
  - mapstruct（对象映射）
  - hutool / guava（工具库）
  - springfox / springdoc（API 文档）

## 架构模式

- **架构类型**: （MVC / DDD / 微服务 / 分层架构）
- **分层方式**: （按技术分层 / 按业务领域分层）
- **模块划分**: （单体多模块 / 微服务 / Monorepo）
- **通信方式**: （REST / gRPC / Dubbo / 消息队列）

## 代码风格

- **命名规范**: （符合 Java 编码规范 / 有自定义规范）
- **代码简化**: （使用 Lombok / 使用 Record / 传统方式）
- **异常处理**: （全局异常处理器 / 分散处理）
- **日志框架**: （SLF4J + Logback / Log4j2）
- **设计模式**: （列举识别到的主要模式）
- **配置风格**: （注解配置 + YAML / XML 配置 + Properties）

## 架构特点

- （总结项目的关键架构决策和设计亮点）
```

## 输出示例

### 示例 1：Spring Boot 多模块项目（MVC 架构）

```markdown
## 项目描述

- **项目名称**: ecommerce-platform
- **项目类型**: 多模块 Maven 项目
- **主语言**: Java 17
- **构建工具**: Maven 3.9
- **核心框架**: Spring Boot 3.2

## 工程结构树

```
ecommerce-platform/
├── ecommerce-common/           # 公共模块（工具类、常量、异常）
│   ├── src/main/java/
│   │   └── com/ecommerce/common/
│   │       ├── exception/      # 全局异常定义
│   │       ├── response/       # 统一响应封装
│   │       ├── utils/          # 工具类
│   │       └── constants/      # 常量定义
│   └── pom.xml
│
├── ecommerce-dao/              # 数据访问模块
│   ├── src/main/java/
│   │   └── com/ecommerce/dao/
│   │       ├── entity/         # JPA 实体类
│   │       ├── repository/     # Spring Data JPA Repository
│   │       └── mapper/         # MyBatis Mapper
│   └── pom.xml
│
├── ecommerce-service/          # 业务逻辑模块
│   ├── src/main/java/
│   │   └── com/ecommerce/service/
│   │       ├── order/          # 订单服务
│   │       │   ├── OrderService.java
│   │       │   └── impl/
│   │       ├── user/           # 用户服务
│   │       └── product/        # 商品服务
│   └── pom.xml
│
├── ecommerce-web/              # Web 层（Spring Boot 主应用）
│   ├── src/main/java/
│   │   └── com/ecommerce/
│   │       ├── EcommerceApplication.java  # 主启动类
│   │       └── web/
│   │           ├── controller/ # REST Controller
│   │           │   ├── OrderController.java
│   │           │   ├── UserController.java
│   │           │   └── ProductController.java
│   │           ├── dto/        # 数据传输对象
│   │           │   ├── request/
│   │           │   └── response/
│   │           ├── config/     # 配置类
│   │           └── filter/     # 过滤器
│   ├── src/main/resources/
│   │   ├── application.yml     # 主配置文件
│   │   ├── application-dev.yml
│   │   └── application-prod.yml
│   └── pom.xml
│
├── pom.xml                     # 父 POM（依赖管理）
└── README.md
```

## 关键模块

| 模块 | 路径 | 职责 |
|------|------|------|
| 公共模块 | ecommerce-common/ | 异常定义、统一响应、工具类 |
| 数据访问 | ecommerce-dao/ | JPA 实体、Repository、MyBatis Mapper |
| 业务逻辑 | ecommerce-service/ | 订单、用户、商品服务实现 |
| Web 层 | ecommerce-web/ | REST API、Controller、DTO |

## 技术栈

- **运行环境**: Java 17（LTS）
- **核心框架**: Spring Boot 3.2, Spring MVC
- **ORM 框架**: Spring Data JPA + MyBatis（混合使用）
- **数据库**: MySQL 8.0
- **缓存**: Redis 7（Lettuce 客户端）
- **核心依赖**:
  - lombok 1.18.30（简化代码）
  - mapstruct 1.5.5（DTO 与 Entity 映射）
  - springdoc-openapi 2.3（API 文档）
  - hutool 5.8（工具库）
  - validation-api（参数校验）

## 架构模式

- **架构类型**: MVC 架构（按技术分层）
- **分层方式**: Controller -> Service -> Repository -> Entity（四层分离）
- **模块划分**: 多模块 Maven 项目（common、dao、service、web）
- **通信方式**: RESTful API

## 代码风格

- **命名规范**: 符合 Java 编码规范（PascalCase 类名、camelCase 方法名）
- **代码简化**: 广泛使用 Lombok（@Data、@Builder、@Slf4j）
- **异常处理**: 全局异常处理器（@ControllerAdvice + @ExceptionHandler）
- **日志框架**: SLF4J + Logback
- **设计模式**: 
  - 工厂模式（OrderFactory 创建不同类型订单）
  - 策略模式（PaymentStrategy 支持多种支付方式）
  - 建造者模式（DTO 使用 @Builder）
- **配置风格**: 注解配置 + YAML 多环境配置

## 架构特点

- **多模块分离**: 按技术职责划分模块，依赖关系清晰（web -> service -> dao -> common）
- **混合 ORM**: JPA 用于简单 CRUD，MyBatis 用于复杂查询
- **统一响应封装**: 所有 API 返回 `Result<T>` 格式
- **参数校验**: 使用 JSR-303 注解（@Valid、@NotNull）
- **API 文档**: 集成 SpringDoc（Swagger UI）
```

### 示例 2：DDD 架构项目

```markdown
## 项目描述

- **项目名称**: order-system
- **项目类型**: 单模块 Maven 项目
- **主语言**: Java 17
- **构建工具**: Maven 3.9
- **核心框架**: Spring Boot 3.2

## 工程结构树

```
order-system/
├── src/main/java/
│   └── com/example/order/
│       ├── OrderApplication.java         # Spring Boot 主启动类
│       │
│       ├── order/                        # 订单限界上下文
│       │   ├── application/              # 应用层
│       │   │   ├── OrderApplicationService.java
│       │   │   ├── dto/
│       │   │   │   ├── CreateOrderCommand.java
│       │   │   │   └── OrderDTO.java
│       │   │   └── assembler/            # DTO 装配器
│       │   │       └── OrderAssembler.java
│       │   │
│       │   ├── domain/                   # 领域层（核心）
│       │   │   ├── model/
│       │   │   │   ├── Order.java        # 聚合根
│       │   │   │   ├── OrderItem.java    # 实体
│       │   │   │   ├── OrderStatus.java  # 值对象
│       │   │   │   └── Address.java      # 值对象
│       │   │   ├── service/              # 领域服务
│       │   │   │   └── PricingDomainService.java
│       │   │   ├── repository/           # 仓储接口
│       │   │   │   └── OrderRepository.java
│       │   │   ├── event/                # 领域事件
│       │   │   │   └── OrderCreatedEvent.java
│       │   │   └── exception/            # 领域异常
│       │   │       └── OrderNotFoundException.java
│       │   │
│       │   ├── infrastructure/           # 基础设施层
│       │   │   ├── repository/           # 仓储实现
│       │   │   │   └── OrderRepositoryImpl.java
│       │   │   ├── mapper/               # MyBatis Mapper
│       │   │   │   └── OrderMapper.java
│       │   │   ├── entity/               # 数据库实体
│       │   │   │   └── OrderPO.java
│       │   │   └── config/               # 基础设施配置
│       │   │       └── MyBatisConfig.java
│       │   │
│       │   └── interfaces/               # 接口层
│       │       ├── controller/           # REST API
│       │       │   └── OrderController.java
│       │       ├── listener/             # 消息监听
│       │       │   └── PaymentSuccessListener.java
│       │       └── scheduler/            # 定时任务
│       │           └── OrderTimeoutScheduler.java
│       │
│       └── shared/                       # 共享内核
│           ├── domain/
│           │   ├── BaseEntity.java
│           │   └── Money.java            # 通用值对象
│           └── infrastructure/
│               └── RedisUtil.java
│
├── src/main/resources/
│   ├── application.yml
│   └── mapper/                           # MyBatis XML
│       └── OrderMapper.xml
│
└── pom.xml
```

## 关键模块

| 模块 | 路径 | 职责 |
|------|------|------|
| 应用层 | order/application/ | 用例编排、DTO 转换、事务管理 |
| 领域层 | order/domain/ | 聚合根、实体、值对象、领域服务 |
| 基础设施层 | order/infrastructure/ | 数据库访问、外部服务调用 |
| 接口层 | order/interfaces/ | REST API、MQ Listener、定时任务 |
| 共享内核 | shared/ | 通用领域模型和工具 |

## 技术栈

- **运行环境**: Java 17（LTS）
- **核心框架**: Spring Boot 3.2
- **ORM 框架**: MyBatis 3.5
- **数据库**: PostgreSQL 15
- **缓存**: Redis 7
- **消息队列**: Kafka 3.6（领域事件发布）
- **核心依赖**:
  - lombok 1.18.30
  - mapstruct 1.5.5
  - spring-kafka 3.1
  - mybatis-spring-boot-starter 3.0

## 架构模式

- **架构类型**: DDD（领域驱动设计）
- **分层方式**: 按业务领域分层（Application -> Domain -> Infrastructure -> Interfaces）
- **模块划分**: 单模块，按限界上下文分包
- **通信方式**: REST API + Kafka 事件驱动

## 代码风格

- **命名规范**: DDD 标准命名（Aggregate、Entity、ValueObject、Repository）
- **代码简化**: 使用 Lombok（@Data、@Value 用于值对象）
- **异常处理**: 领域异常 + 全局异常处理器
- **日志框架**: SLF4J + Logback
- **设计模式**:
  - 聚合模式（Order 聚合根管理 OrderItem）
  - 仓储模式（OrderRepository 接口 + OrderRepositoryImpl 实现）
  - 工厂模式（OrderFactory 创建聚合根）
  - 观察者模式（Domain Event + @EventListener）
- **配置风格**: 注解配置 + YAML

## 架构特点

- **DDD 四层架构**: Application、Domain、Infrastructure、Interfaces 职责清晰
- **聚合根保护**: 订单状态变更必须通过 Order 聚合根的方法
- **领域事件驱动**: 订单创建后发布 OrderCreatedEvent，异步触发后续流程
- **仓储抽象**: Domain 层定义接口，Infrastructure 层实现，解耦 ORM 框架
- **值对象不可变**: 使用 `@Value` 注解保证值对象不可变性
- **共享内核**: 提取通用领域模型（Money、BaseEntity）避免重复
```

## 执行准则

- **聚焦关键信息**: 跳过生成文件（`target/`、`.class`）、IDE 配置（`.idea/`、`.vscode/`）
- **先广度后深度**: 先展示整体结构，再根据用户兴趣深入特定模块
- **不臆测**: 若某项技术不确定，以可能性形式给出并附证据（如"可能是 DDD，因为发现了 domain/ 包"）
- **尊重 `.gitignore`**: 除非用户明确要求，否则不枚举被忽略的目录
- **抽样分析**: 代码风格分析时，每个类型读取 2-3 个代表性文件即可，避免过度分析
- **中文注释**: 工程结构树的每个关键目录都必须有中文注释说明用途