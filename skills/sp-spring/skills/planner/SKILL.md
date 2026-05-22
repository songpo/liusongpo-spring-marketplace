---
name: planner
description: Spring 项目规划助手，全面沟通需求并生成架构设计和实施计划，支持 Spring Boot、Spring Cloud、Spring Batch 等
---

# Spring 项目规划助手

**这是推荐首先使用的技能。** 在开始编码之前，让我们先明确你要做什么，避免走弯路。

## 🎯 为什么要先规划？

很多项目失败不是因为技术问题，而是因为：
- ❌ 需求不清晰，做到一半发现方向错了
- ❌ 架构选择不当，后期难以扩展
- ❌ 技术选型失误，团队不熟悉导致进度延误
- ❌ 没有计划，不知道先做什么后做什么

**先花 30 分钟规划，可以节省 30 天的返工时间。**

## 何时使用

- ✅ 启动任何新的 Spring Boot 项目（强烈推荐）
- ✅ 对现有项目进行重大重构或功能扩展
- ✅ 不确定该用什么架构和技术栈
- ✅ 需要向团队或领导汇报技术方案
- ✅ 需要制定详细的实施计划

## 我会帮你做什么？

### 第一步：了解你的需求

我会通过一系列问题，帮你梳理清楚：

**业务层面：**
- 你要解决什么问题？
- 目标用户是谁？
- 核心功能有哪些？
- 优先级如何？

**技术层面：**
- 团队规模和技术背景？
- 性能和并发要求？
- 是否需要高可用？
- 预算和时间限制？

**非功能需求：**
- 安全要求（认证、授权、加密）
- 测试要求（覆盖率、自动化）
- 运维要求（监控、日志、备份）

### 第二步：给出技术方案

基于你的需求，我会推荐：

**架构方案：**
- 单体 vs 微服务 vs 模块化单体
- 分层架构 vs DDD 架构
- 是否需要 CQRS

**技术选型：**
- 持久化方案（JPA / Spring Data JDBC / MyBatis）
- 缓存方案（Redis / Caffeine）
- 消息队列（RabbitMQ / Kafka）
- 其他中间件

**模块划分：**
- 如何划分模块/限界上下文
- 模块间如何通信
- 数据如何隔离

### 第三步：生成文档和计划

我会为你生成：

1. **需求文档**（`requirements.md`）
2. **架构设计文档**（`architecture.md`）
3. **技术选型文档**（`tech-stack.md`）
4. **数据模型设计**（`data-model.md`）
5. **API 设计文档**（`api-design.md`）
6. **实施计划**（`implementation-plan.md`）
7. **检查清单**（`checklists.md`）

### 第四步：指导你使用其他技能

根据规划结果，我会告诉你接下来应该使用哪些技能：

```
如果选择了模块化单体架构：
→ 使用 /springboot-modulith-cn

如果选择了 DDD 多模块架构：
→ 使用 /springboot-ddd-architecture-cn

需要设计领域模型：
→ 使用 /springboot-ddd-cn

需要读写分离：
→ 使用 /springboot-ddd-cqrs-cn

编写测试：
→ 使用 /springboot-testing-cn
```

## 🚀 开始使用

### 启动规划会话

```
用户：/springboot-project-planner-cn

助手：你好！👋 在开始编码之前，让我先了解一下你的项目。

请用一两句话描述：你想做什么？解决什么问题？

用户：我想做一个在线教育平台，学生可以购买课程、观看视频、做练习题。

助手：很好！在线教育平台。让我详细了解一下...

【问题 1/7】团队和时间
- 团队有多少人？
- 大家对 Spring Boot 熟悉吗？
- 项目计划多久完成？

用户：3 个后端开发，都熟悉 Spring Boot，计划 3 个月完成 MVP。

助手：明白了。【问题 2/7】用户规模...
```

### 快速模式（已有明确需求）

如果你已经很清楚要做什么，可以直接告诉我：

```
用户：/springboot-project-planner-cn

我要做一个电商平台，核心功能：商品管理、订单、支付、物流。
团队 3 人，都熟悉 Spring Boot 和 DDD。
预计 1 万用户，日活 1000，峰值并发 100。
需要 JWT 认证，RBAC 权限。
3 个月完成 MVP。

助手：好的，我已经了解了基本情况。让我确认几个关键点...
[快速确认后直接生成方案]
```

## 规划流程详解

### 阶段 1：需求沟通（Discovery）

我会通过以下维度与你沟通，全面了解项目需求：

#### 1.1 项目背景

**我会询问：**
- 项目的业务目标是什么？
- 解决什么问题？服务哪些用户？
- 项目规模预期（用户量、数据量、并发量）
- 项目周期和里程碑
- 团队规模和技术背景

**示例对话：**
```
助手：请描述一下这个项目的业务目标？
用户：我们要做一个电商平台，支持商品管理、订单处理、支付和物流跟踪。

助手：预期的用户规模和并发量是多少？
用户：初期预计 1 万用户，日活 1000，峰值并发 100。

助手：团队有多少人？技术栈熟悉程度如何？
用户：3 个后端开发，都熟悉 Spring Boot，对 DDD 有一定了解。
```

#### 1.2 功能需求

**我会询问：**
- 核心功能有哪些？
- 功能优先级如何？
- 是否有特殊的业务规则？
- 需要集成哪些外部系统？

**输出：功能清单**
```markdown
## 功能需求清单

### 核心功能（MVP）
1. 用户管理
   - 用户注册/登录
   - 个人信息管理
   - 权限控制

2. 商品管理
   - 商品 CRUD
   - 分类管理
   - 库存管理

3. 订单管理
   - 创建订单
   - 订单支付
   - 订单状态跟踪

### 次要功能（V2）
1. 优惠券系统
2. 评价系统
3. 推荐系统
```

#### 1.3 架构需求

**我会询问：**
- 单体还是微服务？
- 是否需要高可用？
- 性能要求（响应时间、吞吐量）
- 是否需要国际化？
- 是否需要多租户？

**输出：架构约束**
```markdown
## 架构约束

- **部署模式**：模块化单体（Spring Modulith）
- **可用性**：99.9%（允许短暂停机维护）
- **性能**：API 响应时间 < 200ms（P95）
- **扩展性**：支持水平扩展
- **国际化**：暂不需要
- **多租户**：不需要
```

#### 1.4 安全需求

**我会询问：**
- 认证方式（JWT、Session、OAuth2）
- 授权模型（RBAC、ABAC）
- 数据加密需求
- 审计日志需求
- 合规要求（GDPR、等保）

**输出：安全清单**
```markdown
## 安全需求

### 认证授权
- 认证方式：JWT
- 授权模型：RBAC（基于角色）
- 密码策略：BCrypt 加密，强密码要求

### 数据安全
- 敏感数据加密：用户手机号、身份证号
- HTTPS 强制
- SQL 注入防护
- XSS 防护

### 审计
- 操作日志：记录所有写操作
- 登录日志：记录登录尝试
- 日志保留：90 天
```

#### 1.5 数据需求

**我会询问：**
- 数据库选择（MySQL、PostgreSQL）
- 数据量预估
- 是否需要读写分离？
- 是否需要缓存？
- 数据备份策略

**输出：数据架构**
```markdown
## 数据架构

### 数据库
- 主库：MySQL 8.0
- 读写分离：暂不需要
- 分库分表：暂不需要

### 缓存
- Redis：用于会话、热点数据
- 本地缓存：Caffeine（配置、字典）

### 备份
- 全量备份：每天凌晨 2 点
- 增量备份：每小时
- 保留周期：30 天
```

#### 1.6 测试需求

**我会询问：**
- 测试覆盖率要求
- 是否需要自动化测试？
- 性能测试需求
- 是否需要 CI/CD？

**输出：测试策略**
```markdown
## 测试策略

### 单元测试
- 覆盖率目标：70%
- 工具：JUnit 5 + Mockito

### 集成测试
- 覆盖率目标：核心流程 100%
- 工具：Spring Boot Test + Testcontainers

### 性能测试
- 工具：JMeter
- 场景：下单流程、商品查询

### CI/CD
- CI：GitHub Actions
- CD：Docker + Kubernetes
```

#### 1.7 运维需求

**我会询问：**
- 部署环境（云服务商、容器化）
- 监控和告警需求
- 日志管理
- 灾难恢复计划

**输出：运维方案**
```markdown
## 运维方案

### 部署
- 环境：阿里云 ECS
- 容器化：Docker + Docker Compose
- 编排：暂不使用 K8s

### 监控
- APM：Spring Boot Actuator + Prometheus + Grafana
- 日志：ELK Stack
- 告警：钉钉机器人

### 备份恢复
- RTO：4 小时
- RPO：1 小时
```

### 阶段 2：架构设计（Design）

基于需求沟通的结果，我会生成详细的架构设计文档。

#### 2.1 技术选型

```markdown
# 技术选型文档

## 后端框架
- **Spring Boot 3.2**：主框架
- **Spring Modulith 1.1**：模块化架构
- **Spring Data JDBC**：持久化（DDD 友好）
- **Spring Security**：安全框架

## 数据存储
- **MySQL 8.0**：主数据库
- **Redis 7.0**：缓存
- **MinIO**：对象存储（图片、文件）

## 中间件
- **RabbitMQ**：消息队列
- **Elasticsearch**：搜索引擎

## 工具库
- **MapStruct**：对象映射
- **Lombok**：减少样板代码
- **Validation**：参数校验

## 测试
- **JUnit 5**：单元测试
- **Testcontainers**：集成测试
- **WireMock**：外部服务 Mock

## 构建部署
- **Maven**：构建工具
- **Docker**：容器化
- **GitHub Actions**：CI/CD

## 选型理由

### 为什么选择 Spring Modulith？
- 团队规模小（3 人），微服务过重
- 保持模块边界清晰，未来可拆分
- 事件驱动通信，降低耦合

### 为什么选择 Spring Data JDBC？
- DDD 友好，聚合根概念契合
- 简单透明，无延迟加载等魔法
- 性能好，适合中小型项目
```

#### 2.2 模块划分

```markdown
# 模块设计

## 应用模块

### 1. user（用户模块）
**职责：** 用户注册、登录、个人信息管理

**公开 API：**
- `UserManagement`：用户管理接口
- `AuthenticationService`：认证服务

**领域事件：**
- `UserRegisteredEvent`：用户注册成功
- `UserProfileUpdatedEvent`：用户信息更新

**依赖：** 无

---

### 2. product（商品模块）
**职责：** 商品管理、分类、库存

**公开 API：**
- `ProductManagement`：商品管理接口
- `InventoryService`：库存服务

**领域事件：**
- `ProductCreatedEvent`：商品创建
- `StockReservedEvent`：库存预留
- `StockReleasedEvent`：库存释放

**依赖：** 无

---

### 3. order（订单模块）
**职责：** 订单创建、支付、状态管理

**公开 API：**
- `OrderManagement`：订单管理接口

**领域事件：**
- `OrderCreatedEvent`：订单创建
- `OrderPaidEvent`：订单支付成功
- `OrderCancelledEvent`：订单取消

**依赖：** user, product

---

### 4. payment（支付模块）
**职责：** 支付处理、退款

**公开 API：**
- `PaymentService`：支付服务接口

**领域事件：**
- `PaymentCompletedEvent`：支付完成
- `RefundCompletedEvent`：退款完成

**依赖：** order

---

### 5. notification（通知模块）
**职责：** 邮件、短信通知

**公开 API：**
- `NotificationService`：通知服务接口

**领域事件：** 无

**依赖：** 无（监听其他模块事件）

---

## 模块依赖图

```
user ←─── order ←─── payment
         ↑
product ─┘

notification（监听所有事件）
```

## 目录结构

```
ecommerce-platform/
├── pom.xml
└── src/main/java/com/example/ecommerce/
    ├── EcommercePlatformApplication.java
    ├── user/
    │   ├── User.java
    │   ├── UserService.java
    │   ├── UserController.java
    │   └── internal/
    │       ├── UserRepository.java
    │       └── UserEntity.java
    ├── product/
    │   ├── Product.java
    │   ├── ProductService.java
    │   ├── ProductController.java
    │   └── internal/
    ├── order/
    │   ├── Order.java
    │   ├── OrderService.java
    │   ├── OrderController.java
    │   └── internal/
    ├── payment/
    │   ├── Payment.java
    │   ├── PaymentService.java
    │   └── internal/
    └── notification/
        ├── NotificationService.java
        └── internal/
```
```

#### 2.3 数据模型设计

```markdown
# 数据模型设计

## 用户模块

### users 表
```sql
CREATE TABLE users (
    id VARCHAR(36) PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    password VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    phone VARCHAR(20),
    status VARCHAR(20) NOT NULL,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,
    INDEX idx_username (username),
    INDEX idx_email (email)
);
```

### user_roles 表
```sql
CREATE TABLE user_roles (
    user_id VARCHAR(36) NOT NULL,
    role VARCHAR(50) NOT NULL,
    PRIMARY KEY (user_id, role),
    FOREIGN KEY (user_id) REFERENCES users(id)
);
```

## 商品模块

### products 表
```sql
CREATE TABLE products (
    id VARCHAR(36) PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    price DECIMAL(10, 2) NOT NULL,
    stock INT NOT NULL DEFAULT 0,
    category_id VARCHAR(36),
    status VARCHAR(20) NOT NULL,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,
    INDEX idx_category (category_id),
    INDEX idx_status (status)
);
```

## 订单模块

### orders 表
```sql
CREATE TABLE orders (
    id VARCHAR(36) PRIMARY KEY,
    user_id VARCHAR(36) NOT NULL,
    total_amount DECIMAL(10, 2) NOT NULL,
    status VARCHAR(20) NOT NULL,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,
    INDEX idx_user (user_id),
    INDEX idx_status (status),
    INDEX idx_created_at (created_at)
);
```

### order_items 表
```sql
CREATE TABLE order_items (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    order_id VARCHAR(36) NOT NULL,
    product_id VARCHAR(36) NOT NULL,
    product_name VARCHAR(200) NOT NULL,
    quantity INT NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    FOREIGN KEY (order_id) REFERENCES orders(id)
);
```
```

#### 2.4 API 设计

```markdown
# API 设计

## 用户 API

### 注册
```http
POST /api/users/register
Content-Type: application/json

{
  "username": "john",
  "password": "Pass123!",
  "email": "john@example.com",
  "phone": "13800138000"
}

Response 201:
{
  "id": "uuid",
  "username": "john",
  "email": "john@example.com"
}
```

### 登录
```http
POST /api/users/login
Content-Type: application/json

{
  "username": "john",
  "password": "Pass123!"
}

Response 200:
{
  "token": "jwt-token",
  "expiresIn": 3600
}
```

## 商品 API

### 创建商品
```http
POST /api/products
Authorization: Bearer {token}
Content-Type: application/json

{
  "name": "iPhone 15",
  "description": "最新款 iPhone",
  "price": 5999.00,
  "stock": 100,
  "categoryId": "uuid"
}

Response 201:
{
  "id": "uuid",
  "name": "iPhone 15",
  "price": 5999.00,
  "stock": 100
}
```

### 查询商品列表
```http
GET /api/products?page=0&size=20&category=uuid

Response 200:
{
  "content": [...],
  "page": 0,
  "size": 20,
  "totalElements": 100
}
```

## 订单 API

### 创建订单
```http
POST /api/orders
Authorization: Bearer {token}
Content-Type: application/json

{
  "items": [
    {
      "productId": "uuid",
      "quantity": 2
    }
  ]
}

Response 201:
{
  "id": "uuid",
  "totalAmount": 11998.00,
  "status": "PENDING"
}
```
```

### 阶段 3：实施计划（Planning）

```markdown
# 实施计划

## 迭代 1：基础设施搭建（1 周）

### 目标
搭建项目基础框架和开发环境

### 任务清单
- [ ] 创建 Spring Boot 项目骨架
- [ ] 配置 Spring Modulith
- [ ] 配置数据库连接（MySQL + Redis）
- [ ] 配置日志（Logback + ELK）
- [ ] 配置 CI/CD 流水线
- [ ] 编写开发规范文档

### 交付物
- 可运行的项目骨架
- 开发环境搭建文档
- CI/CD 流水线

### 验收标准
- 项目可以启动
- 数据库连接正常
- CI 流水线可以构建成功

---

## 迭代 2：用户模块（1 周）

### 目标
实现用户注册、登录、权限管理

### 任务清单
- [ ] 设计用户领域模型
- [ ] 实现用户注册功能
- [ ] 实现用户登录（JWT）
- [ ] 实现 RBAC 权限控制
- [ ] 编写单元测试（覆盖率 > 70%）
- [ ] 编写集成测试
- [ ] API 文档（Swagger）

### 交付物
- 用户模块代码
- 单元测试 + 集成测试
- API 文档

### 验收标准
- 用户可以注册、登录
- JWT 认证正常工作
- 权限控制生效
- 测试覆盖率达标

---

## 迭代 3：商品模块（1 周）

### 目标
实现商品管理和库存管理

### 任务清单
- [ ] 设计商品领域模型
- [ ] 实现商品 CRUD
- [ ] 实现分类管理
- [ ] 实现库存管理
- [ ] 实现商品搜索（Elasticsearch）
- [ ] 编写测试
- [ ] API 文档

### 交付物
- 商品模块代码
- 测试代码
- API 文档

### 验收标准
- 商品 CRUD 功能正常
- 库存扣减/释放正确
- 搜索功能可用

---

## 迭代 4：订单模块（2 周）

### 目标
实现订单创建、支付、状态管理

### 任务清单
- [ ] 设计订单领域模型
- [ ] 实现订单创建流程
- [ ] 实现库存预留机制
- [ ] 实现订单支付流程
- [ ] 实现订单取消流程
- [ ] 实现订单状态机
- [ ] 集成支付网关（Mock）
- [ ] 编写测试
- [ ] API 文档

### 交付物
- 订单模块代码
- 测试代码
- API 文档

### 验收标准
- 订单创建流程完整
- 库存预留/释放正确
- 支付流程正常
- 订单状态流转正确

---

## 迭代 5：支付和通知模块（1 周）

### 目标
实现支付处理和通知功能

### 任务清单
- [ ] 实现支付模块
- [ ] 集成真实支付网关
- [ ] 实现通知模块
- [ ] 实现邮件通知
- [ ] 实现短信通知
- [ ] 编写测试

### 交付物
- 支付模块代码
- 通知模块代码
- 测试代码

### 验收标准
- 支付功能正常
- 通知发送成功

---

## 迭代 6：性能优化和上线准备（1 周）

### 目标
性能优化、安全加固、上线准备

### 任务清单
- [ ] 性能测试和优化
- [ ] 安全扫描和加固
- [ ] 监控和告警配置
- [ ] 生产环境部署
- [ ] 编写运维文档
- [ ] 用户手册

### 交付物
- 性能测试报告
- 安全扫描报告
- 运维文档
- 用户手册

### 验收标准
- 性能指标达标
- 无高危安全漏洞
- 监控告警正常
- 生产环境稳定运行
```

### 阶段 4：进度跟踪（Tracking）

在实施过程中，我会帮助你：

#### 4.1 每日检查清单

```markdown
# 每日检查清单

## 开发进度
- [ ] 今日计划任务是否完成？
- [ ] 是否有阻塞问题？
- [ ] 代码是否已提交？
- [ ] 测试是否通过？

## 代码质量
- [ ] 代码审查是否完成？
- [ ] 测试覆盖率是否达标？
- [ ] 是否有技术债务？

## 文档
- [ ] API 文档是否更新？
- [ ] 设计文档是否同步？
```

#### 4.2 迭代回顾

```markdown
# 迭代回顾模板

## 迭代 X 回顾

### 完成情况
- 计划任务：10 个
- 完成任务：8 个
- 未完成任务：2 个
- 完成率：80%

### 做得好的地方
1. 测试覆盖率达到 75%
2. 代码审查及时
3. 文档更新同步

### 需要改进的地方
1. 任务估算不准确
2. 集成测试环境不稳定
3. 沟通不够及时

### 行动计划
1. 改进任务估算方法
2. 优化测试环境
3. 每日站会
```

#### 4.3 里程碑检查

```markdown
# 里程碑检查清单

## MVP 上线检查

### 功能完整性
- [ ] 所有核心功能已实现
- [ ] 所有 API 已测试
- [ ] 所有集成测试通过

### 性能
- [ ] 性能测试完成
- [ ] 响应时间 < 200ms（P95）
- [ ] 并发支持 100 用户

### 安全
- [ ] 安全扫描完成
- [ ] 无高危漏洞
- [ ] 认证授权正常

### 运维
- [ ] 监控配置完成
- [ ] 告警规则配置
- [ ] 备份恢复测试

### 文档
- [ ] API 文档完整
- [ ] 运维文档完整
- [ ] 用户手册完整
```

## 使用方式

### 启动规划会话

```
用户：/springboot-project-planner-cn

助手：你好！我是 Spring Boot 项目规划助手。让我们一起规划你的项目。

首先，请简单描述一下你的项目：
1. 项目的业务目标是什么？
2. 解决什么问题？
3. 预期的用户规模？

用户：我想做一个电商平台...

助手：好的，电商平台。让我详细了解一下需求...
[开始系统化沟通]
```

### 生成文档

沟通完成后，我会生成以下文档：

1. **需求文档**（`requirements.md`）
2. **架构设计文档**（`architecture.md`）
3. **技术选型文档**（`tech-stack.md`）
4. **数据模型设计**（`data-model.md`）
5. **API 设计文档**（`api-design.md`）
6. **实施计划**（`implementation-plan.md`）
7. **检查清单**（`checklists.md`）

### 跟踪进度

在实施过程中：

```
用户：检查迭代 2 的进度

助手：好的，让我检查用户模块的完成情况...
[检查任务清单，生成进度报告]
```

## 最佳实践

### 1. 充分沟通
不要急于开始编码，花时间充分沟通需求

### 2. 文档先行
先完成设计文档，再开始实施

### 3. 小步迭代
每个迭代 1-2 周，快速交付可用功能

### 4. 持续跟踪
每日检查进度，及时发现问题

### 5. 定期回顾
每个迭代结束后回顾，持续改进

## 相关技能

规划完成后，使用以下技能进行实施：

- `/springboot-ddd-architecture-cn` - 实现 DDD 架构
- `/springboot-modulith-cn` - 实现模块化架构
- `/springboot-ddd-cn` - 设计领域模型
- `/springboot-ddd-cqrs-cn` - 实现 CQRS
- `/springboot-testing-cn` - 编写测试
