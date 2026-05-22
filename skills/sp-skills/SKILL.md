---
name: sp-skills
description: 安装所有 Spring 技能的元插件包
---

# Spring 技能全家桶

这是一个元插件，安装它会自动安装所有 Spring 相关技能。

## 包含的技能

安装 `sp-skills` 后，以下所有技能将自动安装：

### 📋 项目工具
- **spring-project-planner-cn** - 项目规划
- **spring-initializr-cn** - 项目生成器

### 🏗️ Spring 核心框架
- **spring-framework-cn** - Spring Framework 核心
- **spring-boot-cn** - Spring Boot 开发
- **spring-data-cn** - Spring Data 数据访问
- **spring-security-cn** - Spring Security 安全
- **spring-ai-cn** - Spring AI 集成

### 🎨 架构设计
- **spring-architecture-cn** - 架构设计（DDD、CQRS、模块化单体）

### ☁️ 微服务
- **spring-cloud-cn** - Spring Cloud 微服务架构

### 🔄 企业集成
- **spring-integration-cn** - Spring Integration 企业集成模式
- **spring-batch-cn** - Spring Batch 批处理

## 使用方式

```bash
# 安装所有技能
claude plugin install sp-skills@liusongpo-spring-marketplace

# 使用技能（带命名空间）
/sp-spring:spring-project-planner-cn
/sp-spring:spring-boot-cn
/sp-spring:spring-cloud-cn
```

## 注意

这是一个元插件，本身不提供任何功能，只是用于批量安装其他技能。
