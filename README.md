# 刘松坡的 Spring Boot Skills Marketplace

这是一个专注于 Spring Boot 开发的技能市场，包含了 Spring Boot 开发中常用的最佳实践和模式。

## 包含的技能

### springboot-ddd-architecture-cn
从零开始创建完整的 Spring Boot DDD 架构，包括：
- 完整的项目结构和模块划分
- 分层架构详解（领域层、应用层、基础设施层、接口层）
- Maven 多模块配置
- 完整的电商订单系统实战案例

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
claude plugin install springboot-ddd-architecture-cn@liusongpo-springboot-marketplace
claude plugin install springboot-ddd-cn@liusongpo-springboot-marketplace
claude plugin install springboot-ddd-cqrs-cn@liusongpo-springboot-marketplace
claude plugin install springboot-testing-cn@liusongpo-springboot-marketplace
```

## 使用

安装后，在 Claude Code 中使用 `/skills` 命令查看可用技能，或直接调用：

```
/springboot-ddd-architecture-cn  # 创建完整的 DDD 架构
/springboot-ddd-cn               # DDD 核心概念
/springboot-ddd-cqrs-cn          # CQRS 模式
/springboot-testing-cn           # 测试策略
```

### 推荐使用顺序

1. **新项目开始**：先使用 `/springboot-ddd-architecture-cn` 了解完整架构
2. **设计领域模型**：使用 `/springboot-ddd-cn` 设计聚合和实体
3. **实现 CQRS**：如需读写分离，使用 `/springboot-ddd-cqrs-cn`
4. **编写测试**：使用 `/springboot-testing-cn` 完善测试

## 贡献

欢迎提交新的 Spring Boot 相关技能！

## 许可证

MIT License
