---
name: spring-cloud-config-cn
description: 在 Spring Boot 应用中使用 Spring Cloud Config 实现集中式配置管理、配置加密、动态刷新时使用
---

# Spring Cloud Config 配置中心指南

当你需要在微服务架构中实现集中式配置管理时使用此技能,包括配置服务器搭建、客户端集成、配置加密解密、动态刷新等功能。

## 何时使用

- 多环境配置管理(dev/test/prod)
- 微服务配置集中管理
- 配置动态刷新(无需重启)
- 敏感信息加密存储
- 配置版本控制(Git/SVN)
- 配置审计和追踪

## Maven 依赖

### 配置服务器

```xml
<dependencies>
    <!-- Config Server -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-config-server</artifactId>
    </dependency>
    
    <!-- Spring Boot Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    
    <!-- 安全认证(可选) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>2023.0.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### 配置客户端

```xml
<dependencies>
    <!-- Config Client -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-config</artifactId>
    </dependency>
    
    <!-- Bootstrap 支持 -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-bootstrap</artifactId>
    </dependency>
    
    <!-- Actuator(用于刷新) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

## 1. 配置服务器搭建

### 1.1 基于 Git 的配置服务器

```java
package com.example.configserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.config.server.EnableConfigServer;

@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

**application.yml 配置:**

```yaml
server:
  port: 8888

spring:
  application:
    name: config-server
  cloud:
    config:
      server:
        git:
          uri: https://github.com/your-org/config-repo
          # 本地 Git 仓库
          # uri: file://${user.home}/config-repo
          default-label: main
          search-paths:
            - '{application}'
            - 'common'
          # 私有仓库认证
          username: ${GIT_USERNAME}
          password: ${GIT_PASSWORD}
          # 克隆超时
          timeout: 10
          # 强制拉取
          force-pull: true
          # 跳过 SSL 验证(不推荐生产环境)
          skip-ssl-validation: false

# 安全认证
security:
  user:
    name: config-user
    password: ${CONFIG_SERVER_PASSWORD:secret}

# 监控端点
management:
  endpoints:
    web:
      exposure:
        include: health,info,refresh
```

### 1.2 本地文件系统配置

```yaml
spring:
  cloud:
    config:
      server:
        native:
          search-locations:
            - classpath:/configs/
            - file:///opt/configs/
  profiles:
    active: native
```

### 1.3 多仓库配置

```yaml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/default/config-repo
          repos:
            # 特定应用使用独立仓库
            user-service:
              pattern: user-service*
              uri: https://github.com/team-a/user-config
            order-service:
              pattern: order-service*
              uri: https://github.com/team-b/order-config
            # 特定环境使用独立仓库
            production:
              pattern: '*/prod'
              uri: https://github.com/ops/prod-config
```

## 2. 配置客户端集成

### 2.1 Bootstrap 配置

**bootstrap.yml:**

```yaml
spring:
  application:
    name: user-service
  cloud:
    config:
      uri: http://localhost:8888
      # 配置文件名(默认使用 application.name)
      name: user-service
      # 环境
      profile: dev
      # 分支
      label: main
      # 快速失败
      fail-fast: true
      # 重试配置
      retry:
        initial-interval: 1000
        max-attempts: 6
        max-interval: 2000
        multiplier: 1.1
      # 认证
      username: config-user
      password: secret
```

### 2.2 使用配置属性

```java
package com.example.userservice.config;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.stereotype.Component;

@Component
@ConfigurationProperties(prefix = "app")
@RefreshScope  // 支持动态刷新
public class AppProperties {
    
    private String name;
    private String version;
    private Database database;
    private Features features;
    
    // Getters and Setters
    
    public static class Database {
        private int maxConnections;
        private int timeout;
        // Getters and Setters
    }
    
    public static class Features {
        private boolean enableCache;
        private boolean enableMetrics;
        // Getters and Setters
    }
}
```

**使用配置:**

```java
package com.example.userservice.controller;

import com.example.userservice.config.AppProperties;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class ConfigController {
    
    private final AppProperties appProperties;
    
    public ConfigController(AppProperties appProperties) {
        this.appProperties = appProperties;
    }
    
    @GetMapping("/config")
    public AppProperties getConfig() {
        return appProperties;
    }
}
```

## 3. 配置文件组织

### 3.1 Git 仓库结构

```
config-repo/
├── application.yml              # 所有应用共享
├── application-dev.yml          # 开发环境共享
├── application-test.yml         # 测试环境共享
├── application-prod.yml         # 生产环境共享
├── user-service.yml             # user-service 默认配置
├── user-service-dev.yml         # user-service 开发环境
├── user-service-test.yml        # user-service 测试环境
├── user-service-prod.yml        # user-service 生产环境
├── order-service.yml
├── order-service-dev.yml
└── common/
    ├── database.yml
    └── redis.yml
```

### 3.2 配置优先级

优先级从高到低:
1. `{application}-{profile}.yml`
2. `{application}.yml`
3. `application-{profile}.yml`
4. `application.yml`

### 3.3 配置示例

**application.yml (共享配置):**

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics

logging:
  level:
    root: INFO
```

**user-service-prod.yml:**

```yaml
app:
  name: User Service
  version: 1.0.0
  database:
    max-connections: 50
    timeout: 30
  features:
    enable-cache: true
    enable-metrics: true

spring:
  datasource:
    url: jdbc:postgresql://prod-db:5432/userdb
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
```

## 4. 配置加密

### 4.1 配置加密密钥

**对称加密 (application.yml):**

```yaml
encrypt:
  key: my-secret-key-for-encryption
```

**非对称加密 (使用 JKS):**

```yaml
encrypt:
  key-store:
    location: classpath:/config-server.jks
    password: ${KEYSTORE_PASSWORD}
    alias: config-server-key
    secret: ${KEY_SECRET}
```

### 4.2 生成密钥库

```bash
# 生成 RSA 密钥对
keytool -genkeypair -alias config-server-key \
  -keyalg RSA -keysize 2048 \
  -keystore config-server.jks \
  -storepass changeit \
  -keypass changeit \
  -dname "CN=Config Server,OU=IT,O=Company,L=City,ST=State,C=CN"
```

### 4.3 加密配置值

**使用 API 加密:**

```bash
# 加密
curl http://localhost:8888/encrypt -d "mysecretpassword"
# 返回: AQA7xKjF...encrypted-value...

# 解密
curl http://localhost:8888/decrypt -d "AQA7xKjF...encrypted-value..."
# 返回: mysecretpassword
```

**在配置文件中使用加密值:**

```yaml
spring:
  datasource:
    password: '{cipher}AQA7xKjF...encrypted-value...'
    
app:
  api-key: '{cipher}AQBxyz123...encrypted-api-key...'
```

### 4.4 客户端解密

**默认服务端解密,客户端接收明文。如需客户端解密:**

```yaml
# bootstrap.yml
spring:
  cloud:
    config:
      server:
        encrypt:
          enabled: false  # 禁用服务端解密
```

## 5. 配置动态刷新

### 5.1 启用刷新端点

**application.yml:**

```yaml
management:
  endpoints:
    web:
      exposure:
        include: refresh,health,info
```

### 5.2 使用 @RefreshScope

```java
package com.example.service;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.stereotype.Service;

@Service
@RefreshScope  // 标记为可刷新
public class MessageService {
    
    @Value("${app.message:Default Message}")
    private String message;
    
    public String getMessage() {
        return message;
    }
}
```

### 5.3 手动刷新配置

```bash
# 刷新单个应用
curl -X POST http://localhost:8080/actuator/refresh

# 返回刷新的配置键
["app.message", "app.features.enable-cache"]
```

### 5.4 使用 Spring Cloud Bus 广播刷新

**添加依赖:**

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
```

**配置 RabbitMQ:**

```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest

management:
  endpoints:
    web:
      exposure:
        include: busrefresh
```

**广播刷新所有实例:**

```bash
curl -X POST http://localhost:8888/actuator/busrefresh
```

## 6. 配置监控

### 6.1 查看配置

```bash
# 查看默认配置
curl http://localhost:8888/user-service/default

# 查看特定环境配置
curl http://localhost:8888/user-service/dev

# 查看特定分支配置
curl http://localhost:8888/user-service/dev/feature-branch

# 查看原始配置文件
curl http://localhost:8888/user-service-dev.yml
```

### 6.2 健康检查

```yaml
management:
  health:
    config:
      enabled: true
```

```bash
curl http://localhost:8888/actuator/health
```

## 7. 最佳实践

### 7.1 配置组织

| 配置类型 | 存储位置 | 示例 |
|---------|---------|------|
| 公共配置 | application.yml | 日志级别、监控端点 |
| 环境配置 | application-{profile}.yml | 数据库连接、缓存配置 |
| 应用配置 | {application}.yml | 应用特定业务配置 |
| 敏感信息 | 加密后存储 | 密码、API密钥 |

### 7.2 安全建议

```yaml
# 1. 启用认证
spring:
  security:
    user:
      name: ${CONFIG_USER}
      password: ${CONFIG_PASSWORD}

# 2. 使用 HTTPS
server:
  ssl:
    enabled: true
    key-store: classpath:keystore.p12
    key-store-password: ${SSL_PASSWORD}

# 3. 限制端点访问
management:
  endpoints:
    web:
      exposure:
        include: health,info
```

### 7.3 高可用配置

```yaml
# 客户端配置多个服务器
spring:
  cloud:
    config:
      uri: http://config-server-1:8888,http://config-server-2:8888
      fail-fast: true
      retry:
        max-attempts: 6
```

### 7.4 配置版本管理

```bash
# Git 提交规范
git commit -m "feat(user-service): add cache configuration"
git commit -m "fix(prod): update database connection pool size"

# 使用标签管理版本
git tag -a v1.0.0 -m "Release version 1.0.0"
git push origin v1.0.0

# 客户端使用特定版本
spring:
  cloud:
    config:
      label: v1.0.0
```

## 8. 常见问题

### 8.1 配置未生效

**检查清单:**
- 确认 bootstrap.yml 配置正确
- 检查应用名称和 profile 是否匹配
- 验证配置服务器可访问
- 查看启动日志中的配置加载信息

### 8.2 刷新不生效

**原因:**
- 未添加 @RefreshScope 注解
- 配置类使用了 @Component 而非 @ConfigurationProperties
- 静态变量无法刷新

**解决方案:**
```java
// ❌ 错误:静态变量
@Value("${app.message}")
private static String message;

// ✅ 正确:实例变量 + @RefreshScope
@RefreshScope
@Component
public class Config {
    @Value("${app.message}")
    private String message;
}
```

### 8.3 加密失败

**检查:**
- 密钥配置是否正确
- 加密端点是否可访问
- 密文格式是否正确 `{cipher}...`

### 8.4 Git 仓库连接失败

**解决方案:**
```yaml
spring:
  cloud:
    config:
      server:
        git:
          uri: ${GIT_URI}
          username: ${GIT_USERNAME}
          password: ${GIT_PASSWORD}
          timeout: 10
          clone-on-start: true  # 启动时克隆
```

## 9. 与其他组件集成

### 9.1 与 Eureka 集成

```yaml
# Config Server 注册到 Eureka
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/

# 客户端从 Eureka 发现 Config Server
spring:
  cloud:
    config:
      discovery:
        enabled: true
        service-id: config-server
```

### 9.2 与 Vault 集成

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-vault-config</artifactId>
</dependency>
```

```yaml
spring:
  cloud:
    vault:
      uri: http://localhost:8200
      token: ${VAULT_TOKEN}
      kv:
        enabled: true
        backend: secret
```

## 参考资源

- [Spring Cloud Config 官方文档](https://docs.spring.io/spring-cloud-config/docs/current/reference/html/)
- [配置加密指南](https://cloud.spring.io/spring-cloud-config/reference/html/#_encryption_and_decryption)
- [Spring Cloud Bus](https://spring.io/projects/spring-cloud-bus)
