# 刘松坡的 Spring Boot Skills Marketplace

这是一个专注于 Spring Boot 开发的技能市场，包含了 Spring Boot 开发中常用的最佳实践和模式。

## 包含的技能

### springboot-ddd-cn
在 Spring Boot 应用中实践领域驱动设计（DDD）时使用，包括：
- 设计聚合、实体、值对象
- 领域服务的实现
- 将领域逻辑与基础设施分离

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
claude plugin install springboot-ddd-cn@liusongpo-springboot-marketplace
claude plugin install springboot-testing-cn@liusongpo-springboot-marketplace
```

## 使用

安装后，在 Claude Code 中使用 `/skills` 命令查看可用技能，或直接调用：

```
/springboot-ddd-cn
/springboot-testing-cn
```

## 贡献

欢迎提交新的 Spring Boot 相关技能！

## 许可证

MIT License
