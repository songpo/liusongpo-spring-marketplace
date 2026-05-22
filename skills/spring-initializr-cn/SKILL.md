---
name: spring-initializr-cn
description: 使用 Spring Initializr API 自动生成 Spring Boot 项目脚手架,与用户沟通需求后下载并解压项目模板
---

# Spring Initializr 项目生成器

当你需要快速创建 Spring Boot 项目脚手架时使用此技能,通过与用户沟通项目需求,自动调用 start.spring.io API 生成并下载项目模板。

## 何时使用

- 创建新的 Spring Boot 项目
- 快速搭建项目脚手架
- 根据需求选择依赖和配置
- 生成标准的项目结构

## 工作流程

### 1. 收集项目需求

与用户沟通以下信息:

**基础配置:**
- 项目类型: Maven 或 Gradle (推荐 Maven)
- Spring Boot 版本 (推荐最新稳定版)
- Java 版本 (推荐 17 或 21)
- 编程语言: Java, Kotlin 或 Groovy (推荐 Java)
- 打包方式: Jar 或 War (推荐 Jar)

**项目元数据:**
- Group ID (如: com.example)
- Artifact ID (如: demo)
- 项目名称
- 项目描述
- 包名 (如: com.example.demo)

**依赖选择:**
根据项目类型询问需要的依赖:
- Web 应用: `web`, `webflux`
- 数据访问: `data-jpa`, `data-mongodb`, `data-redis`
- 安全: `security`, `oauth2-client`
- 消息队列: `kafka`, `amqp`
- 云服务: `cloud-config-client`, `cloud-eureka`, `cloud-feign`
- 其他: `actuator`, `lombok`, `validation`

### 2. 构建 API 请求

使用 start.spring.io API 生成项目:

**API 端点:**
```
https://start.spring.io/starter.zip
```

**请求参数:**
- `type`: 项目类型 (maven-project, gradle-project)
- `language`: 编程语言 (java, kotlin, groovy)
- `bootVersion`: Spring Boot 版本
- `baseDir`: 项目根目录名
- `groupId`: Group ID
- `artifactId`: Artifact ID
- `name`: 项目名称
- `description`: 项目描述
- `packageName`: 包名
- `packaging`: 打包方式 (jar, war)
- `javaVersion`: Java 版本
- `dependencies`: 依赖列表 (逗号分隔)

### 3. 下载并解压项目

```bash
# 下载项目（使用查询参数）
curl 'https://start.spring.io/starter.zip?type=maven-project&language=java&bootVersion=4.0.6&baseDir=demo&groupId=com.example&artifactId=demo&name=demo&description=Demo%20project%20for%20Spring%20Boot&packageName=com.example.demo&packaging=jar&javaVersion=17&dependencies=web,data-jpa,mysql,lombok,actuator' \
  -o demo.zip

# 解压项目
unzip demo.zip -d ./

# 清理 zip 文件
rm demo.zip
```

### 4. 验证项目结构

检查生成的项目结构:
```
demo/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/example/demo/
│   │   │       └── DemoApplication.java
│   │   └── resources/
│   │       └── application.properties
│   └── test/
│       └── java/
│           └── com/example/demo/
│               └── DemoApplicationTests.java
├── pom.xml (或 build.gradle)
├── .gitignore
└── README.md
```

## 常用依赖组合

### Web 应用
```
dependencies=web,lombok,devtools,actuator
```

### RESTful API
```
dependencies=web,data-jpa,mysql,validation,lombok,actuator
```

### 微服务
```
dependencies=web,cloud-eureka,cloud-config-client,cloud-feign,actuator
```

### 响应式应用
```
dependencies=webflux,data-r2dbc,postgresql,lombok,actuator
```

### 消息驱动应用
```
dependencies=web,kafka,data-jpa,mysql,lombok,actuator
```

## 完整示例

### 示例 1: 创建标准 Web 应用

```bash
curl 'https://start.spring.io/starter.zip?type=maven-project&language=java&bootVersion=4.0.6&baseDir=my-web-app&groupId=com.mycompany&artifactId=my-web-app&name=My%20Web%20Application&description=A%20web%20application%20built%20with%20Spring%20Boot&packageName=com.mycompany.webapp&packaging=jar&javaVersion=17&dependencies=web,thymeleaf,data-jpa,mysql,validation,lombok,devtools,actuator' \
  -o my-web-app.zip && \
  unzip my-web-app.zip && \
  rm my-web-app.zip
```

### 示例 2: 创建微服务项目

```bash
curl 'https://start.spring.io/starter.zip?type=maven-project&language=java&bootVersion=4.0.6&baseDir=user-service&groupId=com.mycompany.microservices&artifactId=user-service&name=User%20Service&description=User%20management%20microservice&packageName=com.mycompany.microservices.user&packaging=jar&javaVersion=17&dependencies=web,data-jpa,postgresql,cloud-eureka,cloud-config-client,cloud-feign,cloud-resilience4j,actuator,lombok' \
  -o user-service.zip && \
  unzip user-service.zip && \
  rm user-service.zip
```

### 示例 3: 创建响应式应用

```bash
curl 'https://start.spring.io/starter.zip?type=maven-project&language=java&bootVersion=4.0.6&baseDir=reactive-app&groupId=com.mycompany&artifactId=reactive-app&name=Reactive%20Application&description=Reactive%20application%20with%20WebFlux&packageName=com.mycompany.reactive&packaging=jar&javaVersion=17&dependencies=webflux,data-r2dbc,postgresql,data-redis-reactive,lombok,actuator' \
  -o reactive-app.zip && \
  unzip reactive-app.zip && \
  rm reactive-app.zip
```

### 示例 4: 创建 Kotlin + Gradle 项目

```bash
curl 'https://start.spring.io/starter.zip?type=gradle-project-kotlin&language=kotlin&bootVersion=4.0.6&baseDir=kotlin-app&groupId=com.example&artifactId=kotlin-app&name=Kotlin%20Application&description=Spring%20Boot%20application%20with%20Kotlin&packageName=com.example.kotlin&packaging=jar&javaVersion=21&configurationFileFormat=yaml&dependencies=web,data-jpa,postgresql,lombok' \
  -o kotlin-app.zip && \
  unzip kotlin-app.zip && \
  rm kotlin-app.zip
```

## 可用依赖 ID 参考

### Web
- `web` - Spring Web (REST API)
- `webflux` - Spring Reactive Web
- `websocket` - WebSocket
- `spring-restclient` - HTTP Client
- `spring-webclient` - Reactive HTTP Client

### 数据访问
- `data-jpa` - Spring Data JPA
- `data-jdbc` - Spring Data JDBC
- `data-r2dbc` - Spring Data R2DBC (响应式)
- `data-mongodb` - Spring Data MongoDB
- `data-redis` - Spring Data Redis
- `data-elasticsearch` - Spring Data Elasticsearch

### 数据库驱动
- `mysql` - MySQL Driver
- `postgresql` - PostgreSQL Driver
- `h2` - H2 Database
- `mariadb` - MariaDB Driver
- `oracle` - Oracle Driver
- `sqlserver` - MS SQL Server Driver

### 安全
- `security` - Spring Security
- `oauth2-client` - OAuth2 Client
- `oauth2-resource-server` - OAuth2 Resource Server
- `oauth2-authorization-server` - OAuth2 Authorization Server

### 消息队列
- `kafka` - Spring for Apache Kafka
- `amqp` - Spring for RabbitMQ
- `activemq` - Spring for Apache ActiveMQ
- `artemis` - Spring for Apache ActiveMQ Artemis

### Spring Cloud
- `cloud-config-client` - Config Client
- `cloud-config-server` - Config Server
- `cloud-eureka` - Eureka Discovery Client
- `cloud-eureka-server` - Eureka Server
- `cloud-feign` - OpenFeign
- `cloud-gateway` - Spring Cloud Gateway
- `cloud-loadbalancer` - Cloud LoadBalancer
- `cloud-resilience4j` - Resilience4J

### 开发工具
- `devtools` - Spring Boot DevTools
- `lombok` - Lombok
- `configuration-processor` - Configuration Processor
- `docker-compose` - Docker Compose Support

### 监控和运维
- `actuator` - Spring Boot Actuator
- `prometheus` - Prometheus
- `distributed-tracing` - Distributed Tracing
- `zipkin` - Zipkin

### 测试
- `testcontainers` - Testcontainers

### 其他
- `validation` - Validation
- `cache` - Spring Cache Abstraction
- `batch` - Spring Batch
- `integration` - Spring Integration
- `mail` - Java Mail Sender
- `quartz` - Quartz Scheduler

## 使用技巧

### 1. 查看可用依赖

```bash
# 获取所有可用依赖
curl https://start.spring.io/dependencies | jq '.dependencies.values[].values[] | {id, name, description}'
```

### 2. 查看可用的 Spring Boot 版本

```bash
# 获取可用版本
curl https://start.spring.io/metadata/client | jq '.bootVersion.values[] | {id, name}'
```

### 3. 生成 pom.xml 而不是完整项目

```bash
curl 'https://start.spring.io/pom.xml?type=maven-build&bootVersion=4.0.6&dependencies=web,data-jpa,mysql' \
  -o pom.xml
```

### 4. 生成 build.gradle

```bash
curl 'https://start.spring.io/build.gradle?type=gradle-build&bootVersion=4.0.6&dependencies=web,data-jpa,mysql' \
  -o build.gradle
```

## 最佳实践

### 1. 项目命名规范

- Group ID: 使用反向域名 (如: com.company.project)
- Artifact ID: 使用小写字母和连字符 (如: user-service)
- Package Name: 遵循 Java 包命名规范 (如: com.company.project.user)

### 2. 依赖选择原则

- **最小化原则**: 只添加必需的依赖
- **稳定版本**: 使用稳定的 Spring Boot 版本
- **兼容性**: 确保依赖之间相互兼容

### 3. 项目结构建议

生成项目后,建议添加以下目录结构:

```
src/main/java/com/example/demo/
├── config/          # 配置类
├── controller/      # 控制器
├── service/         # 服务层
├── repository/      # 数据访问层
├── model/           # 实体类
├── dto/             # 数据传输对象
└── exception/       # 异常处理
```

### 4. 后续配置

生成项目后需要配置:

**application.properties / application.yml:**
```properties
# 数据库配置
spring.datasource.url=jdbc:mysql://localhost:3306/mydb
spring.datasource.username=root
spring.datasource.password=password

# JPA 配置
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true

# 服务器配置
server.port=8080
```

## 常见问题

### 1. 下载失败

**原因:** 网络问题或参数错误

**解决方案:**
```bash
# 添加详细输出
curl -v 'https://start.spring.io/starter.zip?type=maven-project&dependencies=web' \
  -o project.zip
```

### 2. 依赖冲突

**原因:** 选择了不兼容的依赖

**解决方案:** 检查依赖兼容性,使用 Spring Boot 推荐的依赖组合

### 3. 解压失败

**原因:** zip 文件损坏或路径问题

**解决方案:**
```bash
# 检查 zip 文件
file project.zip

# 使用绝对路径解压
unzip project.zip -d /path/to/destination
```

## 参考资源

- [Spring Initializr 官网](https://start.spring.io/)
- [Spring Initializr API 文档](https://github.com/spring-io/initializr)
- [Spring Boot 官方文档](https://docs.spring.io/spring-boot/docs/current/reference/html/)
