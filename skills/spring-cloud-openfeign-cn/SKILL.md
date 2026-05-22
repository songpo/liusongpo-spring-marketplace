---
name: spring-cloud-openfeign-cn
description: 在 Spring Boot 应用中使用 OpenFeign 实现声明式 HTTP 客户端、服务调用、负载均衡、超时重试时使用
---

# Spring Cloud OpenFeign 声明式客户端指南

当你需要在微服务架构中实现服务间 HTTP 调用时使用此技能,包括声明式客户端定义、请求配置、负载均衡、超时重试、拦截器等功能。

## 何时使用

- 微服务间 HTTP 调用
- 声明式 REST 客户端
- 与服务注册中心集成(Eureka/Consul)
- 客户端负载均衡
- 请求/响应拦截
- 统一错误处理
- 日志和监控

## Maven 依赖

**Spring Initializr 依赖选择:**
- OpenFeign: `cloud-feign`
- Spring Cloud LoadBalancer: `cloud-loadbalancer`
- Eureka Discovery Client: `cloud-eureka` (可选)
- Resilience4J: `cloud-resilience4j` (可选)

```xml
<dependencies>
    <!-- OpenFeign -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
    
    <!-- 负载均衡(Spring Cloud LoadBalancer) -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-loadbalancer</artifactId>
    </dependency>
    
    <!-- 服务发现(可选) -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    
    <!-- 断路器(可选) -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
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

## 1. 基础使用

### 1.1 启用 Feign 客户端

```java
package com.example.orderservice;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.openfeign.EnableFeignClients;

@SpringBootApplication
@EnableFeignClients  // 启用 Feign 客户端扫描
public class OrderServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}
```

### 1.2 定义 Feign 客户端

```java
package com.example.orderservice.client;

import com.example.orderservice.dto.User;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.*;

import java.util.List;

// 使用服务名(需要服务发现)
@FeignClient(name = "user-service")
public interface UserClient {
    
    @GetMapping("/api/users/{id}")
    User getUserById(@PathVariable("id") Long id);
    
    @GetMapping("/api/users")
    List<User> getAllUsers();
    
    @PostMapping("/api/users")
    User createUser(@RequestBody User user);
    
    @PutMapping("/api/users/{id}")
    User updateUser(@PathVariable("id") Long id, @RequestBody User user);
    
    @DeleteMapping("/api/users/{id}")
    void deleteUser(@PathVariable("id") Long id);
}

// 使用固定 URL(不需要服务发现)
@FeignClient(name = "user-service", url = "http://localhost:8081")
public interface UserClientWithUrl {
    
    @GetMapping("/api/users/{id}")
    User getUserById(@PathVariable("id") Long id);
}
```

### 1.3 使用 Feign 客户端

```java
package com.example.orderservice.service;

import com.example.orderservice.client.UserClient;
import com.example.orderservice.dto.User;
import org.springframework.stereotype.Service;

@Service
public class OrderService {
    
    private final UserClient userClient;
    
    public OrderService(UserClient userClient) {
        this.userClient = userClient;
    }
    
    public void createOrder(Long userId, String product) {
        // 调用用户服务
        User user = userClient.getUserById(userId);
        
        if (user != null) {
            // 创建订单逻辑
            System.out.println("Creating order for user: " + user.getName());
        }
    }
}
```

## 2. 请求参数配置

### 2.1 路径参数和查询参数

```java
@FeignClient(name = "product-service")
public interface ProductClient {
    
    // 路径参数
    @GetMapping("/api/products/{id}")
    Product getProduct(@PathVariable("id") Long id);
    
    // 查询参数
    @GetMapping("/api/products")
    List<Product> searchProducts(
        @RequestParam("keyword") String keyword,
        @RequestParam("category") String category,
        @RequestParam(value = "page", defaultValue = "0") int page,
        @RequestParam(value = "size", defaultValue = "10") int size
    );
    
    // 多个路径参数
    @GetMapping("/api/categories/{categoryId}/products/{productId}")
    Product getProductInCategory(
        @PathVariable("categoryId") Long categoryId,
        @PathVariable("productId") Long productId
    );
}
```

### 2.2 请求头

```java
@FeignClient(name = "auth-service")
public interface AuthClient {
    
    // 单个请求头
    @GetMapping("/api/user/profile")
    UserProfile getProfile(@RequestHeader("Authorization") String token);
    
    // 多个请求头
    @PostMapping("/api/auth/login")
    LoginResponse login(
        @RequestHeader("X-Client-Id") String clientId,
        @RequestHeader("X-Client-Secret") String clientSecret,
        @RequestBody LoginRequest request
    );
    
    // 使用 Map 传递多个请求头
    @GetMapping("/api/data")
    Data getData(@RequestHeader Map<String, String> headers);
}
```

### 2.3 请求体

```java
@FeignClient(name = "order-service")
public interface OrderClient {
    
    // JSON 请求体
    @PostMapping("/api/orders")
    Order createOrder(@RequestBody OrderRequest request);
    
    // 表单提交
    @PostMapping(value = "/api/orders/form", 
                 consumes = "application/x-www-form-urlencoded")
    Order createOrderByForm(@RequestBody Map<String, ?> formData);
}
```

## 3. 全局配置

### 3.1 application.yml 配置

```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          # 默认配置(应用于所有客户端)
          default:
            connect-timeout: 5000      # 连接超时(毫秒)
            read-timeout: 10000        # 读取超时(毫秒)
            logger-level: full         # 日志级别: none/basic/headers/full
            
          # 特定客户端配置
          user-service:
            connect-timeout: 3000
            read-timeout: 5000
            logger-level: basic
            
          product-service:
            connect-timeout: 2000
            read-timeout: 8000

# 启用压缩
feign:
  compression:
    request:
      enabled: true
      mime-types: text/xml,application/xml,application/json
      min-request-size: 2048
    response:
      enabled: true
      
# 启用断路器
  circuitbreaker:
    enabled: true
```

### 3.2 Java 配置类

```java
package com.example.config;

import feign.Logger;
import feign.Request;
import feign.Retryer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.concurrent.TimeUnit;

@Configuration
public class FeignConfig {
    
    // 日志级别
    @Bean
    public Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL;
    }
    
    // 超时配置
    @Bean
    public Request.Options requestOptions() {
        return new Request.Options(
            5000, TimeUnit.MILLISECONDS,  // 连接超时
            10000, TimeUnit.MILLISECONDS  // 读取超时
        );
    }
    
    // 重试策略
    @Bean
    public Retryer feignRetryer() {
        // 最大重试3次,初始间隔100ms,最大间隔1s
        return new Retryer.Default(100, 1000, 3);
    }
}
```

### 3.3 为特定客户端应用配置

```java
// 方式1: 在 @FeignClient 注解中指定
@FeignClient(name = "user-service", configuration = FeignConfig.class)
public interface UserClient {
    // ...
}

// 方式2: 全局配置
@EnableFeignClients(defaultConfiguration = FeignConfig.class)
public class Application {
    // ...
}
```

## 4. 拦截器

### 4.1 请求拦截器

```java
package com.example.config;

import feign.RequestInterceptor;
import feign.RequestTemplate;
import org.springframework.stereotype.Component;

@Component
public class FeignRequestInterceptor implements RequestInterceptor {
    
    @Override
    public void apply(RequestTemplate template) {
        // 添加通用请求头
        template.header("X-Request-Id", generateRequestId());
        template.header("X-Client-Version", "1.0.0");
        
        // 添加认证令牌
        String token = getAuthToken();
        if (token != null) {
            template.header("Authorization", "Bearer " + token);
        }
        
        // 添加查询参数
        template.query("timestamp", String.valueOf(System.currentTimeMillis()));
    }
    
    private String generateRequestId() {
        return java.util.UUID.randomUUID().toString();
    }
    
    private String getAuthToken() {
        // 从上下文或缓存获取令牌
        return "your-auth-token";
    }
}
```

### 4.2 认证拦截器

```java
package com.example.config;

import feign.RequestInterceptor;
import feign.RequestTemplate;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Component;

@Component
public class OAuth2FeignRequestInterceptor implements RequestInterceptor {
    
    @Override
    public void apply(RequestTemplate template) {
        Authentication authentication = SecurityContextHolder
            .getContext()
            .getAuthentication();
            
        if (authentication != null && authentication.getDetails() instanceof OAuth2AuthenticationDetails) {
            OAuth2AuthenticationDetails details = 
                (OAuth2AuthenticationDetails) authentication.getDetails();
            template.header("Authorization", "Bearer " + details.getTokenValue());
        }
    }
}
```

## 5. 错误处理

### 5.1 自定义错误解码器

```java
package com.example.config;

import feign.Response;
import feign.codec.ErrorDecoder;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;

@Component
public class CustomErrorDecoder implements ErrorDecoder {
    
    private final ErrorDecoder defaultErrorDecoder = new Default();
    
    @Override
    public Exception decode(String methodKey, Response response) {
        HttpStatus status = HttpStatus.valueOf(response.status());
        
        switch (status) {
            case NOT_FOUND:
                return new ResourceNotFoundException(
                    "Resource not found: " + methodKey);
                    
            case BAD_REQUEST:
                return new BadRequestException(
                    "Bad request: " + methodKey);
                    
            case UNAUTHORIZED:
                return new UnauthorizedException(
                    "Unauthorized: " + methodKey);
                    
            case INTERNAL_SERVER_ERROR:
                return new ServiceUnavailableException(
                    "Service unavailable: " + methodKey);
                    
            default:
                return defaultErrorDecoder.decode(methodKey, response);
        }
    }
}
```

### 5.2 降级处理(Fallback)

```java
package com.example.client;

import com.example.dto.User;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.stereotype.Component;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;

import java.util.Collections;
import java.util.List;

// 指定降级类
@FeignClient(name = "user-service", fallback = UserClientFallback.class)
public interface UserClient {
    
    @GetMapping("/api/users/{id}")
    User getUserById(@PathVariable("id") Long id);
    
    @GetMapping("/api/users")
    List<User> getAllUsers();
}

// 降级实现
@Component
class UserClientFallback implements UserClient {
    
    @Override
    public User getUserById(Long id) {
        // 返回默认用户
        User defaultUser = new User();
        defaultUser.setId(id);
        defaultUser.setName("Unknown User");
        return defaultUser;
    }
    
    @Override
    public List<User> getAllUsers() {
        // 返回空列表
        return Collections.emptyList();
    }
}
```

### 5.3 带异常信息的降级

```java
package com.example.client;

import feign.hystrix.FallbackFactory;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.stereotype.Component;

@FeignClient(name = "user-service", fallbackFactory = UserClientFallbackFactory.class)
public interface UserClient {
    @GetMapping("/api/users/{id}")
    User getUserById(@PathVariable("id") Long id);
}

@Component
class UserClientFallbackFactory implements FallbackFactory<UserClient> {
    
    @Override
    public UserClient create(Throwable cause) {
        return new UserClient() {
            @Override
            public User getUserById(Long id) {
                // 记录异常
                System.err.println("Error calling user-service: " + cause.getMessage());
                
                // 返回降级数据
                User defaultUser = new User();
                defaultUser.setId(id);
                defaultUser.setName("Fallback User");
                return defaultUser;
            }
        };
    }
}
```

## 6. 负载均衡

### 6.1 与 Eureka 集成

```yaml
# application.yml
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
    fetch-registry: true
    register-with-eureka: true

spring:
  application:
    name: order-service
```

```java
// 使用服务名自动负载均衡
@FeignClient(name = "user-service")  // user-service 从 Eureka 获取实例列表
public interface UserClient {
    @GetMapping("/api/users/{id}")
    User getUserById(@PathVariable("id") Long id);
}
```

### 6.2 自定义负载均衡策略

```java
package com.example.config;

import org.springframework.cloud.client.ServiceInstance;
import org.springframework.cloud.loadbalancer.core.RandomLoadBalancer;
import org.springframework.cloud.loadbalancer.core.ReactorLoadBalancer;
import org.springframework.cloud.loadbalancer.core.ServiceInstanceListSupplier;
import org.springframework.cloud.loadbalancer.support.LoadBalancerClientFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.env.Environment;

@Configuration
public class LoadBalancerConfig {
    
    // 使用随机策略
    @Bean
    public ReactorLoadBalancer<ServiceInstance> randomLoadBalancer(
            Environment environment,
            LoadBalancerClientFactory loadBalancerClientFactory) {
        String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new RandomLoadBalancer(
            loadBalancerClientFactory.getLazyProvider(name, ServiceInstanceListSupplier.class),
            name
        );
    }
}
```

## 7. 日志配置

### 7.1 启用 Feign 日志

```yaml
# application.yml
logging:
  level:
    com.example.client: DEBUG  # 设置 Feign 客户端包的日志级别
```

```java
package com.example.config;

import feign.Logger;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class FeignLogConfig {
    
    @Bean
    public Logger.Level feignLoggerLevel() {
        // NONE: 不记录(默认)
        // BASIC: 仅记录请求方法、URL、响应状态码和执行时间
        // HEADERS: 记录 BASIC 级别的信息以及请求和响应头
        // FULL: 记录请求和响应的所有信息
        return Logger.Level.FULL;
    }
}
```

### 7.2 自定义日志记录器

```java
package com.example.config;

import feign.Logger;
import feign.Request;
import feign.Response;
import org.slf4j.LoggerFactory;

import java.io.IOException;

public class CustomFeignLogger extends Logger {
    
    private static final org.slf4j.Logger log = 
        LoggerFactory.getLogger(CustomFeignLogger.class);
    
    @Override
    protected void log(String configKey, String format, Object... args) {
        log.info(String.format(methodTag(configKey) + format, args));
    }
    
    @Override
    protected void logRequest(String configKey, Level logLevel, Request request) {
        log.info("Request: {} {}", request.httpMethod(), request.url());
        log.info("Headers: {}", request.headers());
    }
    
    @Override
    protected Response logAndRebufferResponse(
            String configKey, Level logLevel, Response response, long elapsedTime) 
            throws IOException {
        log.info("Response: {} in {}ms", response.status(), elapsedTime);
        return super.logAndRebufferResponse(configKey, logLevel, response, elapsedTime);
    }
}
```

## 8. 文件上传下载

### 8.1 文件上传

```java
package com.example.client;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestPart;
import org.springframework.web.multipart.MultipartFile;

@FeignClient(name = "file-service")
public interface FileClient {
    
    @PostMapping(value = "/api/files/upload", 
                 consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    FileUploadResponse uploadFile(@RequestPart("file") MultipartFile file);
    
    @PostMapping(value = "/api/files/upload-multiple",
                 consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    FileUploadResponse uploadFiles(
        @RequestPart("files") MultipartFile[] files,
        @RequestPart("metadata") String metadata
    );
}
```

**配置文件上传编码器:**

```xml
<dependency>
    <groupId>io.github.openfeign.form</groupId>
    <artifactId>feign-form</artifactId>
    <version>3.8.0</version>
</dependency>
```

```java
package com.example.config;

import feign.codec.Encoder;
import feign.form.spring.SpringFormEncoder;
import org.springframework.beans.factory.ObjectFactory;
import org.springframework.boot.autoconfigure.http.HttpMessageConverters;
import org.springframework.cloud.openfeign.support.SpringEncoder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class FeignMultipartConfig {
    
    @Bean
    public Encoder feignFormEncoder(ObjectFactory<HttpMessageConverters> converters) {
        return new SpringFormEncoder(new SpringEncoder(converters));
    }
}
```

### 8.2 文件下载

```java
package com.example.client;

import feign.Response;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;

@FeignClient(name = "file-service")
public interface FileDownloadClient {
    
    @GetMapping("/api/files/{id}/download")
    Response downloadFile(@PathVariable("id") Long id);
}
```

**使用示例:**

```java
package com.example.service;

import com.example.client.FileDownloadClient;
import feign.Response;
import org.springframework.stereotype.Service;

import java.io.IOException;
import java.io.InputStream;
import java.nio.file.Files;
import java.nio.file.Paths;

@Service
public class FileService {
    
    private final FileDownloadClient fileDownloadClient;
    
    public FileService(FileDownloadClient fileDownloadClient) {
        this.fileDownloadClient = fileDownloadClient;
    }
    
    public void downloadAndSaveFile(Long fileId, String savePath) throws IOException {
        Response response = fileDownloadClient.downloadFile(fileId);
        
        try (InputStream inputStream = response.body().asInputStream()) {
            Files.copy(inputStream, Paths.get(savePath));
        }
    }
}
```

## 9. 最佳实践

### 9.1 超时配置建议

| 场景 | 连接超时 | 读取超时 |
|-----|---------|---------|
| 快速查询 | 2-3秒 | 3-5秒 |
| 普通业务 | 3-5秒 | 5-10秒 |
| 复杂计算 | 5秒 | 30-60秒 |
| 文件操作 | 5秒 | 60-120秒 |

### 9.2 重试策略

```java
// 自定义重试器
@Bean
public Retryer feignRetryer() {
    // 不重试(生产环境推荐,配合断路器使用)
    return Retryer.NEVER_RETRY;
    
    // 或者有限重试
    // return new Retryer.Default(
    //     100,   // 初始间隔
    //     1000,  // 最大间隔
    //     3      // 最大重试次数
    // );
}
```

### 9.3 连接池配置

```yaml
feign:
  httpclient:
    enabled: true
    max-connections: 200          # 最大连接数
    max-connections-per-route: 50 # 每个路由最大连接数
    connection-timeout: 5000      # 连接超时
    time-to-live: 900             # 连接存活时间(秒)
```

### 9.4 监控和指标

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  metrics:
    tags:
      application: ${spring.application.name}
```

## 10. 常见问题

### 10.1 超时问题

**症状:** 请求经常超时

**解决方案:**
```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          default:
            connect-timeout: 10000
            read-timeout: 30000
```

### 10.2 负载均衡不生效

**原因:** 未引入负载均衡依赖

**解决方案:**
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```

### 10.3 日志不输出

**检查清单:**
- 确认日志级别配置正确
- 确认 Logger.Level 配置为 BASIC 或以上
- 检查包路径是否正确

### 10.4 请求头丢失

**原因:** 未配置请求拦截器传递请求头

**解决方案:**
```java
@Component
public class HeaderPropagationInterceptor implements RequestInterceptor {
    @Override
    public void apply(RequestTemplate template) {
        ServletRequestAttributes attributes = 
            (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        if (attributes != null) {
            HttpServletRequest request = attributes.getRequest();
            // 传递特定请求头
            String authorization = request.getHeader("Authorization");
            if (authorization != null) {
                template.header("Authorization", authorization);
            }
        }
    }
}
```

## 参考资源

- [Spring Cloud OpenFeign 官方文档](https://docs.spring.io/spring-cloud-openfeign/docs/current/reference/html/)
- [OpenFeign GitHub](https://github.com/OpenFeign/feign)
- [Spring Cloud LoadBalancer](https://spring.io/guides/gs/spring-cloud-loadbalancer/)
