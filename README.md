# 刘松坡的 Spring Boot Skills Marketplace

这是一个专注于 Spring Boot 开发的技能市场，包含了 Spring Boot 开发中常用的最佳实践和模式。

## 包含的技能

### springboot-project-planner-cn ⭐ 推荐首先使用
Spring Boot 项目规划助手，包括：
- 全面的需求沟通（功能、架构、安全、测试、运维）
- 自动生成架构设计文档
- 生成详细的实施计划和检查清单
- 进度跟踪和迭代回顾

### springboot-ddd-architecture-cn
从零开始创建完整的 Spring Boot DDD 架构，包括：
- 完整的项目结构和模块划分
- 分层架构详解（领域层、应用层、基础设施层、接口层）
- Maven 多模块配置
- 完整的电商订单系统实战案例

### springboot-modulith-cn
使用 Spring Modulith 构建模块化单体应用，包括：
- 模块划分和边界定义
- 事件驱动的模块间通信
- 架构验证和文档生成
- 与 DDD 结合的最佳实践

### springboot-ddd-cn
在 Spring Boot 应用中实践领域驱动设计（DDD）时使用，包括：
- 设计聚合、实体、值对象
- 领域服务的实现
- 将领域逻辑与基础设施分离

### springboot-ddd-cqrs-cn
在 Spring Boot 应用中实践 CQRS（命令查询职责分离）模式，包括：
- 命令和查询的分离
- 读写模型分离
- 事件驱动同步
- 最终一致性处理

### springboot-testing-cn
为 Spring Boot 应用设计测试策略，包括：
- 单元测试编写
- 集成测试实践
- 控制器、服务和仓储的测试方法

## 安装

### 添加 marketplace

```bash
claude plugin marketplace add https://github.com/songpo/liusongpo-springboot-marketplace
```

### 安装技能

```bash
# 单独安装技能
claude plugin install springboot-project-planner-cn@liusongpo-springboot-marketplace
claude plugin install springboot-ddd-architecture-cn@liusongpo-springboot-marketplace
claude plugin install springboot-modulith-cn@liusongpo-springboot-marketplace
claude plugin install springboot-ddd-cn@liusongpo-springboot-marketplace
claude plugin install springboot-ddd-cqrs-cn@liusongpo-springboot-marketplace
claude plugin install springboot-testing-cn@liusongpo-springboot-marketplace
```

## 使用

安装后，在 Claude Code 中使用 `/skills` 命令查看可用技能，或直接调用：

```
/springboot-project-planner-cn   # 项目规划（推荐首先使用）
/springboot-ddd-architecture-cn  # 创建完整的 DDD 架构
/springboot-modulith-cn          # 模块化单体架构
/springboot-ddd-cn               # DDD 核心概念
/springboot-ddd-cqrs-cn          # CQRS 模式
/springboot-testing-cn           # 测试策略
```

### 完整的项目开发流程

#### 第 1 步：项目规划 ⭐

**使用技能：** `/springboot-project-planner-cn`

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
用户：/springboot-project-planner-cn
助手：你好！让我们一起规划你的项目。请描述一下项目的业务目标？
用户：我想做一个电商平台...
助手：[开始系统化沟通，涵盖功能、架构、安全、测试、运维等方面]
```

#### 第 2 步：架构实现

根据规划文档选择合适的架构方案：

**方案 A：DDD 多模块架构（大型项目）**

1. 使用 `/springboot-ddd-architecture-cn` 创建分层架构
2. 使用 `/springboot-ddd-cn` 设计领域模型
3. 如需读写分离，使用 `/springboot-ddd-cqrs-cn`

**方案 B：模块化单体架构（中小型项目，推荐）**

1. 使用 `/springboot-modulith-cn` 创建模块化架构
2. 在每个模块内使用 `/springboot-ddd-cn` 设计领域模型
3. 使用 Modulith 的事件驱动通信

#### 第 3 步：测试和质量保证

**使用技能：** `/springboot-testing-cn`

- 编写单元测试
- 编写集成测试
- 使用 Modulith 的模块测试

#### 第 4 步：进度跟踪

回到 `/springboot-project-planner-cn`，使用检查清单跟踪进度：
- 每日检查清单
- 迭代回顾
- 里程碑检查

### 推荐使用顺序（新项目）

```
1. /springboot-project-planner-cn     ← 开始：全面规划
   ↓
2. /springboot-modulith-cn            ← 创建模块化架构
   或 /springboot-ddd-architecture-cn  （大型项目）
   ↓
3. /springboot-ddd-cn                 ← 设计领域模型
   ↓
4. /springboot-ddd-cqrs-cn            ← 可选：读写分离
   ↓
5. /springboot-testing-cn             ← 编写测试
   ↓
6. /springboot-project-planner-cn     ← 回到：跟踪进度
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
