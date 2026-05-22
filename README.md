# 刘松坡的 Spring Skills Marketplace

这是一个专注于 Spring 生态系统开发的技能市场，包含了 Spring Boot、Spring Cloud、Spring Batch、Spring Integration 等开发中常用的最佳实践和模式。

## ⚠️ 重要提示

**在开始使用其他技能之前，强烈建议先使用 `/spring-project-planner-cn` 进行项目规划。**

很多项目失败不是因为技术问题，而是因为需求不清晰、架构选择不当、技术选型失误。

**先花 30 分钟规划，可以节省 30 天的返工时间。**

## 包含的技能

### 🚀 项目初始化

#### spring-initializr-cn
使用 Spring Initializr API 快速生成项目脚手架，包括：
- 自动调用 start.spring.io API 生成项目
- 根据需求选择依赖和配置
- 生成标准的项目结构
- 支持 Maven/Gradle、多种 Java 版本

### 📋 项目规划

#### 🎯 spring-project-planner-cn（强烈推荐首先使用）

**为什么要先规划？**
- ❌ 需求不清晰 → 做到一半发现方向错了
- ❌ 架构选择不当 → 后期难以扩展
- ❌ 技术选型失误 → 团队不熟悉导致进度延误
- ✅ 先规划再编码 → 避免返工，提高效率

**这个技能会帮你：**
- 通过系统化问题梳理需求（功能、架构、安全、测试、运维）
- 推荐合适的架构方案和技术选型
- 自动生成架构设计文档和实施计划
- 指导你使用其他技能完成实施

### 🏗️ Spring 核心框架

#### spring-framework-cn
Spring Framework 核心框架实践，包括：
- IoC 容器和依赖注入
- 面向切面编程（AOP）
- 事件机制
- 资源管理
- 数据绑定和验证
- 任务调度和异步执行

#### spring-boot-cn
Spring Boot 核心开发实践，包括：
- 自动配置原理和机制
- 开发自定义 Starter
- 配置管理（application.yml/properties）
- Profile 环境配置
- Actuator 监控端点
- 应用打包和部署
- Docker 容器化
- 性能优化

#### spring-data-cn
Spring Data 数据访问实践，包括：
- Spring Data JPA（对象关系映射）
- Spring Data JDBC（轻量级数据访问）
- Spring Data Redis（缓存和数据存储）
- Spring Data MongoDB（文档数据库）
- 查询方法和分页
- Specification 动态查询
- 审计和版本控制

#### spring-security-cn
Spring Security 安全框架实践，包括：
- 用户认证和授权
- JWT 令牌认证实现
- OAuth2/OIDC 第三方登录
- 基于角色的权限控制（RBAC）
- 方法级权限控制
- 安全最佳实践（密码策略、防暴力破解、审计日志）

#### spring-ai-cn
Spring AI 集成实践，包括：
- 对接各种 LLM（OpenAI、Azure OpenAI、Ollama）
- 实现智能对话和流式对话
- RAG（检索增强生成）实现
- 向量数据库集成（Chroma、Pinecone）
- Prompt 工程和模板
- 函数调用（Function Calling）
- 图像生成

### 🎨 架构设计

#### spring-architecture-cn
Spring 应用架构设计实践，包括：
- 领域驱动设计（DDD）：实体、值对象、聚合、领域服务、仓储
- CQRS 模式：命令查询职责分离、读写模型分离、事件驱动同步
- 模块化单体架构：Spring Modulith、模块边界、事件驱动通信
- 分层架构：接口层、应用层、领域层、基础设施层
- 架构模式选择和最佳实践

### ☁️ Spring Cloud 微服务

#### spring-cloud-cn
Spring Cloud 微服务架构总览，包括：
- 微服务架构概念和原则
- Spring Cloud 生态系统（Gateway、Eureka、Config、OpenFeign、CircuitBreaker）
- 微服务架构模式
- 服务拆分和设计
- 数据管理和一致性
- 部署和运维

### 🔄 企业集成

#### spring-integration-cn
Spring Integration 企业集成模式实践，包括：
- 消息通道和端点
- 消息转换和路由
- 消息聚合和分割
- 文件处理和监控
- 与外部系统集成（FTP/SFTP/HTTP/JMS）
- 事件驱动架构

#### spring-batch-cn
Spring Batch 批处理实践，包括：
- Job 和 Step 配置
- ItemReader/Writer/Processor
- 分块处理
- 任务调度
- 大批量数据处理
- ETL 流程
- 数据迁移和同步

## 安装

### 添加 marketplace

```bash
# 从 GitHub 安装（推荐）
claude plugin marketplace add https://github.com/songpo/liusongpo-spring-marketplace

# 或从本地路径安装（开发测试）
claude plugin marketplace add /path/to/liusongpo-spring-marketplace
```

### 安装技能

**一键安装所有技能：**

```bash
# 安装 sp-spring 插件（包含所有 11 个子技能）
claude plugin install sp-spring@liusongpo-spring-marketplace
```

安装后，所有子技能将自动可用。

## 使用

### 🚀 快速开始（推荐流程）

```bash
# 第 1 步：先规划（必须！）
/sp-spring:planner
# 或
/planner

# 助手会通过系统化问题了解你的需求，然后推荐合适的技能
# 例如：推荐使用模块化单体架构

# 第 2 步：根据推荐使用相应技能
/sp-spring:architecture      # 创建架构
/sp-spring:framework         # Spring 核心功能
/sp-spring:boot              # Spring Boot 开发
```

### 📚 所有可用技能

安装后，在 Claude Code 中使用 `/skills` 命令查看可用技能，或直接调用：

**方式 1：带命名空间（推荐）**
```
/sp-spring:planner          # 🎯 项目规划（强烈推荐首先使用）
/sp-spring:initializr       # 快速生成项目脚手架
/sp-spring:framework        # Spring Framework 核心
/sp-spring:boot             # Spring Boot 开发
/sp-spring:data             # Spring Data 数据访问
/sp-spring:security         # Spring Security 安全
/sp-spring:ai               # Spring AI 集成
/sp-spring:architecture     # 架构设计（DDD、CQRS、模块化单体）
/sp-spring:cloud            # Spring Cloud 微服务架构
/sp-spring:integration      # 企业集成模式
/sp-spring:batch            # 批处理
```

**方式 2：不带命名空间**
```
/planner                    # 项目规划
/boot                       # Spring Boot 开发
/cloud                      # Spring Cloud 微服务
/architecture               # 架构设计
```

### 完整的项目开发流程

#### 第 1 步：项目规划 ⭐

**使用技能：** `/sp-spring:planner` 或 `/planner`

**目标：** 通过系统化沟通，全面了解需求并生成设计文档

**输出：**
- 需求文档
- 架构设计文档
- 技术选型文档
- 数据模型设计
- API 设计文档
- 实施计划
- 检查清单

**示例：**
```
用户：/sp-spring:planner
助手：你好！让我们一起规划你的项目。请描述一下项目的业务目标？
用户：我想做一个电商平台...
助手：[开始系统化沟通，涵盖功能、架构、安全、测试、运维等方面]
```

#### 第 2 步：架构实现

根据规划文档选择合适的架构方案：

**方案 A：DDD 多模块架构（大型项目）**

1. 使用 `/sp-spring:architecture` 创建分层架构
2. 使用 `/sp-spring:architecture` 设计领域模型
3. 如需读写分离，使用 CQRS 模式

**方案 B：模块化单体架构（中小型项目，推荐）**

1. 使用 `/sp-spring:architecture` 创建模块化架构
2. 在每个模块内设计领域模型
3. 使用 Modulith 的事件驱动通信

#### 第 3 步：测试和质量保证

**使用技能：** `/sp-spring:framework` 和 `/sp-spring:boot`

- 编写单元测试
- 编写集成测试
- 使用 Modulith 的模块测试

#### 第 4 步：进度跟踪

回到 `/sp-spring:planner`，使用检查清单跟踪进度：
- 每日检查清单
- 迭代回顾
- 里程碑检查

### 推荐使用顺序（新项目）

```
1. /sp-spring:planner              ← 开始：全面规划
   ↓
2. /sp-spring:architecture         ← 创建架构
   ↓
3. /sp-spring:framework            ← Spring 核心功能
   ↓
4. /sp-spring:boot                 ← Spring Boot 开发
   ↓
5. /sp-spring:data                 ← 数据访问
   ↓
6. /sp-spring:planner              ← 回到：跟踪进度
```

### Modulith + DDD 结合的优势

**Modulith 提供：**
- 模块边界和依赖管理（物理边界）
- 事件驱动的模块间通信
- 架构验证和文档生成

**DDD 提供：**
- 领域模型设计（逻辑边界）
- 聚合、实体、值对象
- 业务逻辑的组织方式

**结合使用：**
```
每个 Modulith 模块 = 一个限界上下文（Bounded Context）
模块内部使用 DDD 战术设计（聚合、实体、值对象）
模块之间通过领域事件通信
```

## 贡献

欢迎提交新的 Spring Boot 相关技能！

## 许可证

MIT License
