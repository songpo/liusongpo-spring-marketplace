---
name: spring-cloud-circuitbreaker-cn
description: 在 Spring Boot 应用中使用 Spring Cloud Circuit Breaker 实现断路器、限流、重试、舱壁隔离等弹性模式时使用
---

# Spring Cloud Circuit Breaker 弹性模式指南

当你需要在微服务架构中实现容错和弹性时使用此技能,包括断路器模式、限流、重试、超时、舱壁隔离等功能,主要基于 Resilience4j 实现。

## 何时使用

- 防止级联故障
- 服务降级和熔断
- 限流保护
- 请求重试
- 超时控制
- 资源隔离(舱壁模式)
- 监控和指标收集

## Maven 依赖

**Spring Initializr 依赖选择:**
- Resilience4J: `cloud-resilience4j`
- Spring Boot Actuator: `actuator`
- Micrometer Prometheus: `prometheus` (可选,用于监控)

```xml
<dependencies>
    <!-- Spring Cloud Circuit Breaker -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
    </dependency>
    
    <!-- Resilience4j 核心模块 -->
    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-spring-boot3</artifactId>
    </dependency>
    
    <!-- Reactor 支持(响应式) -->
    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-reactor</artifactId>
    </dependency>
    
    <!-- Micrometer 指标(可选) -->
    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-micrometer</artifactId>
    </dependency>
    
    <!-- Actuator 监控 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
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

## 1. 断路器(Circuit Breaker)

### 1.1 基础使用

```java
package com.example.service;

import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

@Service
public class UserService {
    
    private final RestTemplate restTemplate;
    
    public UserService(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }
    
    // 使用断路器
    @CircuitBreaker(name = "userService", fallbackMethod = "getUserFallback")
    public User getUser(Long id) {
        String url = "http://user-service/api/users/" + id;
        return restTemplate.getForObject(url, User.class);
    }
    
    // 降级方法(参数和返回值必须与原方法一致)
    private User getUserFallback(Long id, Exception ex) {
        System.err.println("Circuit breaker fallback: " + ex.getMessage());
        
        User fallbackUser = new User();
        fallbackUser.setId(id);
        fallbackUser.setName("Fallback User");
        return fallbackUser;
    }
}
```

### 1.2 配置断路器

```yaml
resilience4j:
  circuitbreaker:
    configs:
      # 默认配置
      default:
        # 滑动窗口类型: COUNT_BASED(基于计数) 或 TIME_BASED(基于时间)
        sliding-window-type: COUNT_BASED
        # 滑动窗口大小
        sliding-window-size: 10
        # 最小调用次数(达到此值才开始计算失败率)
        minimum-number-of-calls: 5
        # 失败率阈值(百分比)
        failure-rate-threshold: 50
        # 慢调用率阈值(百分比)
        slow-call-rate-threshold: 100
        # 慢调用时间阈值(毫秒)
        slow-call-duration-threshold: 60000
        # 半开状态允许的调用数
        permitted-number-of-calls-in-half-open-state: 3
        # 自动从开启状态转换到半开状态的等待时间(秒)
        wait-duration-in-open-state: 60s
        # 是否自动从开启转换到半开
        automatic-transition-from-open-to-half-open-enabled: true
        # 记录的异常
        record-exceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
        # 忽略的异常
        ignore-exceptions:
          - com.example.exception.BusinessException
          
    instances:
      # 特定实例配置
      userService:
        base-config: default
        sliding-window-size: 20
        failure-rate-threshold: 60
        wait-duration-in-open-state: 30s
        
      paymentService:
        sliding-window-type: TIME_BASED
        sliding-window-size: 60  # 60秒
        minimum-number-of-calls: 10
        failure-rate-threshold: 40
```

### 1.3 断路器状态

```java
package com.example.service;

import io.github.resilience4j.circuitbreaker.CircuitBreaker;
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry;
import org.springframework.stereotype.Service;

@Service
public class CircuitBreakerMonitor {
    
    private final CircuitBreakerRegistry circuitBreakerRegistry;
    
    public CircuitBreakerMonitor(CircuitBreakerRegistry circuitBreakerRegistry) {
        this.circuitBreakerRegistry = circuitBreakerRegistry;
    }
    
    public void monitorCircuitBreaker(String name) {
        CircuitBreaker circuitBreaker = circuitBreakerRegistry.circuitBreaker(name);
        
        // 获取状态
        CircuitBreaker.State state = circuitBreaker.getState();
        System.out.println("Circuit Breaker State: " + state);
        // CLOSED: 正常状态
        // OPEN: 断路器打开,拒绝请求
        // HALF_OPEN: 半开状态,允许部分请求测试
        
        // 获取指标
        CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
        System.out.println("Failure Rate: " + metrics.getFailureRate() + "%");
        System.out.println("Number of Failed Calls: " + metrics.getNumberOfFailedCalls());
        System.out.println("Number of Successful Calls: " + metrics.getNumberOfSuccessfulCalls());
        
        // 监听状态变化
        circuitBreaker.getEventPublisher()
            .onStateTransition(event -> 
                System.out.println("State transition: " + event));
    }
}
```

## 2. 限流(Rate Limiter)

### 2.1 基础使用

```java
package com.example.service;

import io.github.resilience4j.ratelimiter.annotation.RateLimiter;
import org.springframework.stereotype.Service;

@Service
public class ApiService {
    
    // 限流保护
    @RateLimiter(name = "apiService", fallbackMethod = "rateLimitFallback")
    public String callApi(String request) {
        // 调用外部 API
        return "API Response: " + request;
    }
    
    private String rateLimitFallback(String request, Exception ex) {
        return "Rate limit exceeded. Please try again later.";
    }
}
```

### 2.2 配置限流器

```yaml
resilience4j:
  ratelimiter:
    configs:
      default:
        # 限流周期(刷新周期)
        limit-refresh-period: 1s
        # 每个周期允许的请求数
        limit-for-period: 10
        # 线程等待许可的超时时间
        timeout-duration: 0s
        
    instances:
      apiService:
        base-config: default
        limit-for-period: 5
        
      premiumApiService:
        limit-refresh-period: 1s
        limit-for-period: 100
        timeout-duration: 500ms
```

### 2.3 编程式使用

```java
package com.example.service;

import io.github.resilience4j.ratelimiter.RateLimiter;
import io.github.resilience4j.ratelimiter.RateLimiterRegistry;
import org.springframework.stereotype.Service;

import java.util.function.Supplier;

@Service
public class RateLimiterService {
    
    private final RateLimiterRegistry rateLimiterRegistry;
    
    public RateLimiterService(RateLimiterRegistry rateLimiterRegistry) {
        this.rateLimiterRegistry = rateLimiterRegistry;
    }
    
    public String executeWithRateLimit(String name, Supplier<String> supplier) {
        RateLimiter rateLimiter = rateLimiterRegistry.rateLimiter(name);
        
        // 装饰 Supplier
        Supplier<String> decoratedSupplier = RateLimiter
            .decorateSupplier(rateLimiter, supplier);
            
        try {
            return decoratedSupplier.get();
        } catch (Exception ex) {
            return "Rate limit exceeded";
        }
    }
}
```

## 3. 重试(Retry)

### 3.1 基础使用

```java
package com.example.service;

import io.github.resilience4j.retry.annotation.Retry;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

@Service
public class OrderService {
    
    private final RestTemplate restTemplate;
    
    public OrderService(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }
    
    // 自动重试
    @Retry(name = "orderService", fallbackMethod = "createOrderFallback")
    public Order createOrder(OrderRequest request) {
        String url = "http://order-service/api/orders";
        return restTemplate.postForObject(url, request, Order.class);
    }
    
    private Order createOrderFallback(OrderRequest request, Exception ex) {
        System.err.println("All retries failed: " + ex.getMessage());
        throw new OrderCreationException("Failed to create order", ex);
    }
}
```

### 3.2 配置重试

```yaml
resilience4j:
  retry:
    configs:
      default:
        # 最大重试次数(不包括首次调用)
        max-attempts: 3
        # 重试间隔
        wait-duration: 1s
        # 启用指数退避
        enable-exponential-backoff: true
        # 指数退避倍数
        exponential-backoff-multiplier: 2
        # 最大重试间隔
        exponential-max-wait-duration: 10s
        # 重试的异常
        retry-exceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
        # 忽略的异常
        ignore-exceptions:
          - com.example.exception.BusinessException
          
    instances:
      orderService:
        base-config: default
        max-attempts: 5
        wait-duration: 500ms
        
      paymentService:
        max-attempts: 3
        wait-duration: 2s
        enable-exponential-backoff: false
```

### 3.3 监听重试事件

```java
package com.example.config;

import io.github.resilience4j.retry.Retry;
import io.github.resilience4j.retry.RetryRegistry;
import org.springframework.context.annotation.Configuration;

import jakarta.annotation.PostConstruct;

@Configuration
public class RetryEventConfig {
    
    private final RetryRegistry retryRegistry;
    
    public RetryEventConfig(RetryRegistry retryRegistry) {
        this.retryRegistry = retryRegistry;
    }
    
    @PostConstruct
    public void init() {
        retryRegistry.retry("orderService")
            .getEventPublisher()
            .onRetry(event -> 
                System.out.println("Retry attempt: " + event.getNumberOfRetryAttempts()))
            .onSuccess(event -> 
                System.out.println("Retry succeeded after: " + event.getNumberOfRetryAttempts()))
            .onError(event -> 
                System.err.println("All retries failed: " + event.getLastThrowable()));
    }
}
```

## 4. 舱壁隔离(Bulkhead)

### 4.1 信号量舱壁

```java
package com.example.service;

import io.github.resilience4j.bulkhead.annotation.Bulkhead;
import org.springframework.stereotype.Service;

@Service
public class ResourceService {
    
    // 使用信号量限制并发
    @Bulkhead(name = "resourceService", type = Bulkhead.Type.SEMAPHORE, 
              fallbackMethod = "bulkheadFallback")
    public String accessResource(String resourceId) {
        // 访问受限资源
        return "Resource: " + resourceId;
    }
    
    private String bulkheadFallback(String resourceId, Exception ex) {
        return "Resource access limit reached. Please try again later.";
    }
}
```

### 4.2 线程池舱壁

```java
package com.example.service;

import io.github.resilience4j.bulkhead.annotation.Bulkhead;
import org.springframework.stereotype.Service;

import java.util.concurrent.CompletableFuture;

@Service
public class AsyncService {
    
    // 使用线程池隔离
    @Bulkhead(name = "asyncService", type = Bulkhead.Type.THREADPOOL)
    public CompletableFuture<String> asyncOperation(String input) {
        return CompletableFuture.supplyAsync(() -> {
            // 异步操作
            return "Result: " + input;
        });
    }
}
```

### 4.3 配置舱壁

```yaml
resilience4j:
  bulkhead:
    configs:
      default:
        # 最大并发调用数
        max-concurrent-calls: 10
        # 最大等待时间
        max-wait-duration: 0ms
        
    instances:
      resourceService:
        base-config: default
        max-concurrent-calls: 5
        
  thread-pool-bulkhead:
    configs:
      default:
        # 核心线程数
        core-thread-pool-size: 10
        # 最大线程数
        max-thread-pool-size: 20
        # 队列容量
        queue-capacity: 100
        # 线程存活时间
        keep-alive-duration: 20ms
        
    instances:
      asyncService:
        base-config: default
        core-thread-pool-size: 5
        max-thread-pool-size: 10
```

## 5. 超时控制(Time Limiter)

### 5.1 基础使用

```java
package com.example.service;

import io.github.resilience4j.timelimiter.annotation.TimeLimiter;
import org.springframework.stereotype.Service;

import java.util.concurrent.CompletableFuture;

@Service
public class SlowService {
    
    // 超时控制(仅支持异步方法)
    @TimeLimiter(name = "slowService", fallbackMethod = "timeoutFallback")
    public CompletableFuture<String> slowOperation(String input) {
        return CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(5000);  // 模拟慢操作
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            return "Result: " + input;
        });
    }
    
    private CompletableFuture<String> timeoutFallback(String input, Exception ex) {
        return CompletableFuture.completedFuture("Operation timed out");
    }
}
```

### 5.2 配置超时

```yaml
resilience4j:
  timelimiter:
    configs:
      default:
        # 超时时间
        timeout-duration: 3s
        # 是否取消运行中的 Future
        cancel-running-future: true
        
    instances:
      slowService:
        timeout-duration: 2s
```

## 6. 组合使用

### 6.1 多个弹性模式组合

```java
package com.example.service;

import io.github.resilience4j.bulkhead.annotation.Bulkhead;
import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import io.github.resilience4j.ratelimiter.annotation.RateLimiter;
import io.github.resilience4j.retry.annotation.Retry;
import org.springframework.stereotype.Service;

@Service
public class CompositeService {
    
    // 组合使用:限流 -> 断路器 -> 舱壁 -> 重试
    @RateLimiter(name = "composite")
    @CircuitBreaker(name = "composite")
    @Bulkhead(name = "composite")
    @Retry(name = "composite")
    public String compositeOperation(String input) {
        // 业务逻辑
        return "Result: " + input;
    }
}
```

**执行顺序(从外到内):**
1. RateLimiter - 限流检查
2. CircuitBreaker - 断路器检查
3. Bulkhead - 并发控制
4. Retry - 重试逻辑
5. 实际方法调用

### 6.2 配置组合

```yaml
resilience4j:
  ratelimiter:
    instances:
      composite:
        limit-for-period: 10
        limit-refresh-period: 1s
        
  circuitbreaker:
    instances:
      composite:
        sliding-window-size: 10
        failure-rate-threshold: 50
        wait-duration-in-open-state: 30s
        
  bulkhead:
    instances:
      composite:
        max-concurrent-calls: 5
        
  retry:
    instances:
      composite:
        max-attempts: 3
        wait-duration: 1s
```

## 7. 监控和指标

### 7.1 启用 Actuator 端点

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,circuitbreakers,ratelimiters
  endpoint:
    health:
      show-details: always
  health:
    circuitbreakers:
      enabled: true
    ratelimiters:
      enabled: true
```

### 7.2 查看指标

```bash
# 健康检查
curl http://localhost:8080/actuator/health

# 断路器状态
curl http://localhost:8080/actuator/circuitbreakers

# 断路器事件
curl http://localhost:8080/actuator/circuitbreakerevents

# 限流器状态
curl http://localhost:8080/actuator/ratelimiters

# Prometheus 指标
curl http://localhost:8080/actuator/prometheus
```

### 7.3 自定义指标

```java
package com.example.config;

import io.github.resilience4j.circuitbreaker.CircuitBreaker;
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry;
import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.context.annotation.Configuration;

import jakarta.annotation.PostConstruct;

@Configuration
public class MetricsConfig {
    
    private final CircuitBreakerRegistry circuitBreakerRegistry;
    private final MeterRegistry meterRegistry;
    
    public MetricsConfig(CircuitBreakerRegistry circuitBreakerRegistry,
                        MeterRegistry meterRegistry) {
        this.circuitBreakerRegistry = circuitBreakerRegistry;
        this.meterRegistry = meterRegistry;
    }
    
    @PostConstruct
    public void init() {
        circuitBreakerRegistry.circuitBreaker("userService")
            .getEventPublisher()
            .onStateTransition(event -> {
                meterRegistry.counter(
                    "circuit_breaker_state_transition",
                    "name", "userService",
                    "from", event.getStateTransition().getFromState().name(),
                    "to", event.getStateTransition().getToState().name()
                ).increment();
            });
    }
}
```

## 8. 与 OpenFeign 集成

### 8.1 配置 Feign 使用断路器

```yaml
spring:
  cloud:
    openfeign:
      circuitbreaker:
        enabled: true
```

```java
package com.example.client;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;

@FeignClient(name = "user-service", fallback = UserClientFallback.class)
public interface UserClient {
    
    @GetMapping("/api/users/{id}")
    User getUser(@PathVariable("id") Long id);
}

@Component
class UserClientFallback implements UserClient {
    
    @Override
    public User getUser(Long id) {
        User fallbackUser = new User();
        fallbackUser.setId(id);
        fallbackUser.setName("Fallback User");
        return fallbackUser;
    }
}
```

## 9. 最佳实践

### 9.1 断路器配置建议

| 场景 | 滑动窗口 | 失败率阈值 | 等待时间 |
|-----|---------|-----------|---------|
| 快速失败 | 10次 | 50% | 10-30秒 |
| 容错性高 | 20次 | 70% | 60秒 |
| 关键服务 | 50次 | 30% | 120秒 |

### 9.2 重试策略建议

```yaml
# 幂等操作:可以重试
resilience4j:
  retry:
    instances:
      idempotent-service:
        max-attempts: 5
        wait-duration: 500ms
        enable-exponential-backoff: true

# 非幂等操作:谨慎重试
      non-idempotent-service:
        max-attempts: 1  # 不重试
```

### 9.3 降级策略

```java
// 1. 返回默认值
private User getUserFallback(Long id, Exception ex) {
    return User.defaultUser();
}

// 2. 返回缓存数据
private User getUserFallback(Long id, Exception ex) {
    return cacheService.getCachedUser(id);
}

// 3. 返回降级服务
private User getUserFallback(Long id, Exception ex) {
    return backupUserService.getUser(id);
}

// 4. 抛出业务异常
private User getUserFallback(Long id, Exception ex) {
    throw new ServiceUnavailableException("User service is unavailable");
}
```

### 9.4 监控告警

```yaml
# 配置告警阈值
management:
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      application: ${spring.application.name}
      
# Prometheus 告警规则示例
# - alert: CircuitBreakerOpen
#   expr: resilience4j_circuitbreaker_state{state="open"} == 1
#   for: 1m
#   annotations:
#     summary: "Circuit breaker is open"
```

## 10. 常见问题

### 10.1 降级方法不生效

**检查清单:**
- 降级方法签名必须与原方法一致
- 降级方法必须在同一个类中
- 降级方法访问修饰符不能是 private

### 10.2 断路器未打开

**原因:**
- 未达到最小调用次数
- 失败率未达到阈值
- 异常未被记录(被忽略)

**解决方案:**
```yaml
resilience4j:
  circuitbreaker:
    instances:
      myService:
        minimum-number-of-calls: 5  # 降低阈值测试
        failure-rate-threshold: 50
        record-exceptions:
          - java.lang.Exception  # 记录所有异常
```

### 10.3 重试导致性能问题

**原因:** 重试次数过多或间隔过短

**解决方案:**
```yaml
resilience4j:
  retry:
    instances:
      myService:
        max-attempts: 3  # 限制重试次数
        wait-duration: 1s  # 增加重试间隔
        enable-exponential-backoff: true  # 使用指数退避
```

### 10.4 线程池舱壁队列满

**症状:** BulkheadFullException

**解决方案:**
```yaml
resilience4j:
  thread-pool-bulkhead:
    instances:
      myService:
        core-thread-pool-size: 20  # 增加线程数
        queue-capacity: 200  # 增加队列容量
```

## 参考资源

- [Spring Cloud Circuit Breaker 官方文档](https://spring.io/projects/spring-cloud-circuitbreaker)
- [Resilience4j 官方文档](https://resilience4j.readme.io/)
- [Resilience4j GitHub](https://github.com/resilience4j/resilience4j)
- [微服务弹性模式](https://docs.microsoft.com/en-us/azure/architecture/patterns/category/resiliency)
