---
name: spring-boot-cn
description: Spring Boot 核心开发实践，包括自动配置、Starter、配置管理、Actuator 和部署
---

# Spring Boot 核心开发实践指南

当你需要深入了解和使用 Spring Boot 核心功能时使用此技能，包括自动配置原理、Starter 开发、配置管理、监控和部署等。

## 何时使用

- 理解 Spring Boot 自动配置原理
- 开发自定义 Starter
- 管理应用配置（application.yml/properties）
- 使用 Profile 管理多环境配置
- 集成 Actuator 监控端点
- 应用打包和部署
- 容器化部署（Docker）
- 性能优化和调优

## Maven 依赖

**Spring Initializr 依赖选择:**
- Spring Boot DevTools: `devtools` (开发工具)
- Spring Configuration Processor: `configuration-processor` (配置处理器)
- Spring Boot Actuator: `actuator` (监控端点)

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.0</version>
</parent>

<dependencies>
    <!-- Spring Boot Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    
    <!-- Spring Boot Actuator -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    
    <!-- Spring Boot DevTools -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-devtools</artifactId>
        <scope>runtime</scope>
        <optional>true</optional>
    </dependency>
    
    <!-- Configuration Processor -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-configuration-processor</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```

## 1. Spring Boot 核心概念

### 1.1 自动配置原理

Spring Boot 通过 `@EnableAutoConfiguration` 实现自动配置。

**工作原理：**

```java
@SpringBootApplication
// 等价于以下三个注解
// @SpringBootConfiguration
// @EnableAutoConfiguration
// @ComponentScan
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

**自动配置加载流程：**

1. `@EnableAutoConfiguration` 触发自动配置
2. 读取 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
3. 根据条件注解（`@ConditionalOnClass`、`@ConditionalOnMissingBean` 等）决定是否加载配置类
4. 加载符合条件的配置类，创建 Bean

**查看自动配置报告：**

```yaml
# application.yml
debug: true
```

或启动时添加参数：
```bash
java -jar app.jar --debug
```

### 1.2 条件注解

Spring Boot 使用条件注解控制配置类的加载。

**常用条件注解：**

```java
// 当类路径存在指定类时
@ConditionalOnClass(DataSource.class)

// 当类路径不存在指定类时
@ConditionalOnMissingClass("com.example.SomeClass")

// 当容器中存在指定 Bean 时
@ConditionalOnBean(DataSource.class)

// 当容器中不存在指定 Bean 时
@ConditionalOnMissingBean(DataSource.class)

// 当指定属性存在时
@ConditionalOnProperty(name = "app.feature.enabled", havingValue = "true")

// 当 Web 应用类型匹配时
@ConditionalOnWebApplication(type = Type.SERVLET)

// 当资源存在时
@ConditionalOnResource(resources = "classpath:config.properties")
```

**示例：自定义条件配置**

```java
@Configuration
@ConditionalOnClass(RedisTemplate.class)
@ConditionalOnProperty(name = "app.redis.enabled", havingValue = "true", matchIfMissing = true)
public class RedisAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }
}
```

## 2. 配置管理

### 2.1 配置文件

Spring Boot 支持多种配置文件格式。

**application.yml（推荐）：**

```yaml
# 服务器配置
server:
  port: 8080
  servlet:
    context-path: /api
  compression:
    enabled: true
    mime-types: application/json,application/xml,text/html,text/xml,text/plain

# 应用配置
spring:
  application:
    name: my-app
  
  # 数据源配置
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: ${DB_USERNAME:postgres}
    password: ${DB_PASSWORD:postgres}
    hikari:
      maximum-pool-size: 10
      minimum-idle: 5
      connection-timeout: 30000
  
  # JPA 配置
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
    properties:
      hibernate:
        format_sql: true
        dialect: org.hibernate.dialect.PostgreSQLDialect

# 自定义配置
app:
  name: My Application
  version: 1.0.0
  features:
    cache-enabled: true
    async-enabled: true
```

**application.properties：**

```properties
server.port=8080
spring.application.name=my-app
spring.datasource.url=jdbc:postgresql://localhost:5432/mydb
```

### 2.2 配置属性绑定

使用 `@ConfigurationProperties` 绑定配置属性。

**定义配置类：**

```java
@ConfigurationProperties(prefix = "app")
@Validated
public class AppProperties {
    
    @NotBlank
    private String name;
    
    @NotBlank
    private String version;
    
    private Features features = new Features();
    
    public static class Features {
        private boolean cacheEnabled = true;
        private boolean asyncEnabled = true;
        
        // Getters and Setters
    }
    
    // Getters and Setters
}
```

**启用配置属性：**

```java
@Configuration
@EnableConfigurationProperties(AppProperties.class)
public class AppConfig {
    
    @Bean
    public AppService appService(AppProperties properties) {
        return new AppService(properties);
    }
}
```

**或使用 @ConfigurationPropertiesScan：**

```java
@SpringBootApplication
@ConfigurationPropertiesScan("com.example.config")
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

### 2.3 Profile 环境配置

使用 Profile 管理不同环境的配置。

**配置文件命名：**
- `application.yml` - 通用配置
- `application-dev.yml` - 开发环境
- `application-test.yml` - 测试环境
- `application-prod.yml` - 生产环境

**application.yml：**

```yaml
spring:
  application:
    name: my-app
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:dev}

# 通用配置
app:
  name: My Application
```

**application-dev.yml：**

```yaml
server:
  port: 8080

spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb_dev
    username: dev
    password: dev

logging:
  level:
    root: INFO
    com.example: DEBUG
```

**application-prod.yml：**

```yaml
server:
  port: 80

spring:
  datasource:
    url: jdbc:postgresql://prod-db:5432/mydb
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}

logging:
  level:
    root: WARN
    com.example: INFO
```

**激活 Profile：**

```bash
# 方式 1：命令行参数
java -jar app.jar --spring.profiles.active=prod

# 方式 2：环境变量
export SPRING_PROFILES_ACTIVE=prod
java -jar app.jar

# 方式 3：JVM 参数
java -Dspring.profiles.active=prod -jar app.jar
```

**代码中使用 Profile：**

```java
@Configuration
@Profile("dev")
public class DevConfig {
    
    @Bean
    public DataSource devDataSource() {
        // 开发环境数据源
    }
}

@Configuration
@Profile("prod")
public class ProdConfig {
    
    @Bean
    public DataSource prodDataSource() {
        // 生产环境数据源
    }
}
```

### 2.4 外部化配置

Spring Boot 配置加载优先级（从高到低）：

1. 命令行参数
2. SPRING_APPLICATION_JSON 环境变量
3. ServletConfig 初始化参数
4. ServletContext 初始化参数
5. JNDI 属性
6. Java 系统属性（System.getProperties()）
7. 操作系统环境变量
8. RandomValuePropertySource（random.*）
9. jar 包外的 application-{profile}.yml
10. jar 包内的 application-{profile}.yml
11. jar 包外的 application.yml
12. jar 包内的 application.yml
13. @PropertySource 注解
14. 默认属性（SpringApplication.setDefaultProperties）

**示例：使用环境变量**

```yaml
spring:
  datasource:
    url: ${DATABASE_URL:jdbc:postgresql://localhost:5432/mydb}
    username: ${DATABASE_USERNAME:postgres}
    password: ${DATABASE_PASSWORD:postgres}
```

## 3. 开发自定义 Starter

### 3.1 Starter 命名规范

- 官方 Starter：`spring-boot-starter-{name}`
- 第三方 Starter：`{name}-spring-boot-starter`

### 3.2 创建自定义 Starter

**项目结构：**

```
my-spring-boot-starter/
├── pom.xml
└── src/main/java/com/example/starter/
    ├── MyAutoConfiguration.java
    ├── MyProperties.java
    └── MyService.java
└── src/main/resources/
    └── META-INF/spring/
        └── org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

**pom.xml：**

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>
    
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-configuration-processor</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```

**配置属性类：**

```java
@ConfigurationProperties(prefix = "my.service")
public class MyProperties {
    
    private boolean enabled = true;
    private String prefix = "default";
    private int timeout = 30;
    
    // Getters and Setters
}
```

**服务类：**

```java
public class MyService {
    
    private final MyProperties properties;
    
    public MyService(MyProperties properties) {
        this.properties = properties;
    }
    
    public String process(String input) {
        if (!properties.isEnabled()) {
            return input;
        }
        return properties.getPrefix() + ": " + input;
    }
}
```

**自动配置类：**

```java
@Configuration
@ConditionalOnClass(MyService.class)
@EnableConfigurationProperties(MyProperties.class)
@ConditionalOnProperty(name = "my.service.enabled", havingValue = "true", matchIfMissing = true)
public class MyAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    public MyService myService(MyProperties properties) {
        return new MyService(properties);
    }
}
```

**注册自动配置：**

创建文件 `src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`：

```
com.example.starter.MyAutoConfiguration
```

### 3.3 使用自定义 Starter

**添加依赖：**

```xml
<dependency>
    <groupId>com.example</groupId>
    <artifactId>my-spring-boot-starter</artifactId>
    <version>1.0.0</version>
</dependency>
```

**配置：**

```yaml
my:
  service:
    enabled: true
    prefix: "Custom"
    timeout: 60
```

**使用：**

```java
@RestController
public class MyController {
    
    private final MyService myService;
    
    public MyController(MyService myService) {
        this.myService = myService;
    }
    
    @GetMapping("/process")
    public String process(@RequestParam String input) {
        return myService.process(input);
    }
}
```

## 4. Spring Boot Actuator

### 4.1 启用 Actuator

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

### 4.2 配置端点

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
      base-path: /actuator
  
  endpoint:
    health:
      show-details: when-authorized
      probes:
        enabled: true
    
    info:
      enabled: true
  
  metrics:
    export:
      prometheus:
        enabled: true
  
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
```

### 4.3 常用端点

| 端点 | 描述 |
|------|------|
| `/actuator/health` | 健康检查 |
| `/actuator/info` | 应用信息 |
| `/actuator/metrics` | 指标信息 |
| `/actuator/env` | 环境属性 |
| `/actuator/loggers` | 日志配置 |
| `/actuator/threaddump` | 线程转储 |
| `/actuator/heapdump` | 堆转储 |
| `/actuator/prometheus` | Prometheus 指标 |

### 4.4 自定义健康检查

```java
@Component
public class CustomHealthIndicator implements HealthIndicator {
    
    @Override
    public Health health() {
        // 检查外部服务
        boolean serviceAvailable = checkExternalService();
        
        if (serviceAvailable) {
            return Health.up()
                    .withDetail("service", "available")
                    .withDetail("timestamp", System.currentTimeMillis())
                    .build();
        } else {
            return Health.down()
                    .withDetail("service", "unavailable")
                    .withDetail("error", "Connection timeout")
                    .build();
        }
    }
    
    private boolean checkExternalService() {
        // 实现检查逻辑
        return true;
    }
}
```

### 4.5 自定义 Info 端点

```yaml
info:
  app:
    name: ${spring.application.name}
    version: @project.version@
    description: My Spring Boot Application
  build:
    artifact: @project.artifactId@
    group: @project.groupId@
```

或通过代码：

```java
@Component
public class CustomInfoContributor implements InfoContributor {
    
    @Override
    public void contribute(Info.Builder builder) {
        builder.withDetail("custom", Map.of(
                "key1", "value1",
                "key2", "value2"
        ));
    }
}
```

## 5. 应用打包和部署

### 5.1 打包为可执行 JAR

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
                <excludes>
                    <exclude>
                        <groupId>org.springframework.boot</groupId>
                        <artifactId>spring-boot-devtools</artifactId>
                    </exclude>
                </excludes>
            </configuration>
        </plugin>
    </plugins>
</build>
```

**打包：**

```bash
mvn clean package
```

**运行：**

```bash
java -jar target/my-app-1.0.0.jar
```

### 5.2 Docker 容器化

**Dockerfile（多阶段构建）：**

```dockerfile
# 构建阶段
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests

# 运行阶段
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

# 创建非 root 用户
RUN addgroup -S spring && adduser -S spring -G spring
USER spring:spring

# 复制 JAR 文件
COPY --from=build /app/target/*.jar app.jar

# 健康检查
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/actuator/health || exit 1

# 暴露端口
EXPOSE 8080

# 启动应用
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**构建镜像：**

```bash
docker build -t my-app:1.0.0 .
```

**运行容器：**

```bash
docker run -d \
  --name my-app \
  -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=prod \
  -e DATABASE_URL=jdbc:postgresql://db:5432/mydb \
  my-app:1.0.0
```

### 5.3 Docker Compose

**docker-compose.yml：**

```yaml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      SPRING_PROFILES_ACTIVE: prod
      DATABASE_URL: jdbc:postgresql://db:5432/mydb
      DATABASE_USERNAME: postgres
      DATABASE_PASSWORD: postgres
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 3s
      retries: 3
      start_period: 40s
  
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres-data:
```

**启动：**

```bash
docker-compose up -d
```

## 6. 性能优化

### 6.1 启动优化

**延迟初始化：**

```yaml
spring:
  main:
    lazy-initialization: true
```

**排除不需要的自动配置：**

```java
@SpringBootApplication(exclude = {
    DataSourceAutoConfiguration.class,
    HibernateJpaAutoConfiguration.class
})
public class Application {
    // ...
}
```

### 6.2 JVM 参数优化

```bash
java -Xms512m -Xmx2g \
     -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=200 \
     -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/var/log/heapdump.hprof \
     -jar app.jar
```

### 6.3 连接池优化

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
      leak-detection-threshold: 60000
```

## 7. 开发工具

### 7.1 Spring Boot DevTools

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <scope>runtime</scope>
    <optional>true</optional>
</dependency>
```

**功能：**
- 自动重启
- LiveReload
- 全局配置
- 远程调试

### 7.2 配置处理器

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
</dependency>
```

**功能：**
- 生成配置元数据
- IDE 自动补全
- 配置验证

## 最佳实践

1. **使用 YAML 配置文件**：更易读，支持层级结构
2. **使用 Profile 管理环境**：分离不同环境的配置
3. **使用 @ConfigurationProperties**：类型安全的配置绑定
4. **启用 Actuator**：监控应用健康状态
5. **使用 DevTools**：提高开发效率
6. **容器化部署**：使用 Docker 实现一致的部署环境
7. **优化启动时间**：使用延迟初始化和排除不需要的自动配置
8. **配置外部化**：使用环境变量管理敏感信息
9. **健康检查**：实现自定义健康检查
10. **日志管理**：使用结构化日志和日志级别

## 参考资源

- [Spring Boot 官方文档](https://docs.spring.io/spring-boot/docs/current/reference/html/)
- [Spring Boot Actuator](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html)
- [Spring Boot DevTools](https://docs.spring.io/spring-boot/docs/current/reference/html/using.html#using.devtools)
- [创建自定义 Starter](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.developing-auto-configuration)
