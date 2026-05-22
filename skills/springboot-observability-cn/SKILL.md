---
name: springboot-observability-cn
description: Spring Boot 可观测性实践，包括监控、日志、追踪、告警和性能分析
---

# Spring Boot 可观测性实践指南

当你需要为 Spring Boot 应用添加可观测性能力时使用此技能，包括监控指标、日志聚合、分布式追踪和告警。

## 何时使用

- 监控应用健康状态和性能指标
- 实现日志聚合和分析
- 实现分布式追踪
- 配置告警规则
- 性能分析和优化
- 生产环境问题排查

## 可观测性三大支柱

1. **Metrics（指标）**：数值型数据，如 CPU、内存、请求数、响应时间
2. **Logs（日志）**：事件记录，用于问题排查
3. **Traces（追踪）**：请求在系统中的完整路径

## Maven 依赖

```xml
<dependencies>
    <!-- Spring Boot Actuator -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    
    <!-- Micrometer Prometheus -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>
    
    <!-- Micrometer Tracing -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-tracing-bridge-brave</artifactId>
    </dependency>
    
    <!-- Zipkin Reporter -->
    <dependency>
        <groupId>io.zipkin.reporter2</groupId>
        <artifactId>zipkin-reporter-brave</artifactId>
    </dependency>
    
    <!-- Logback（Spring Boot 默认） -->
    <dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-classic</artifactId>
    </dependency>
    
    <!-- Logstash Encoder（可选） -->
    <dependency>
        <groupId>net.logstash.logback</groupId>
        <artifactId>logstash-logback-encoder</artifactId>
        <version>7.4</version>
    </dependency>
</dependencies>
```

## 1. Metrics（指标监控）

### 1.1 Actuator 配置

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
      base-path: /actuator
  endpoint:
    health:
      show-details: always
      probes:
        enabled: true
    metrics:
      enabled: true
    prometheus:
      enabled: true
  metrics:
    tags:
      application: ${spring.application.name}
      environment: ${spring.profiles.active}
    distribution:
      percentiles-histogram:
        http.server.requests: true
      slo:
        http.server.requests: 50ms,100ms,200ms,500ms,1s
```

### 1.2 自定义指标

```java
package com.example.metrics;

import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import org.springframework.stereotype.Component;

@Component
public class CustomMetrics {
    
    private final Counter orderCreatedCounter;
    private final Counter orderFailedCounter;
    private final Timer orderProcessingTimer;
    
    public CustomMetrics(MeterRegistry registry) {
        this.orderCreatedCounter = Counter.builder("orders.created")
            .description("订单创建总数")
            .tag("type", "business")
            .register(registry);
        
        this.orderFailedCounter = Counter.builder("orders.failed")
            .description("订单失败总数")
            .tag("type", "business")
            .register(registry);
        
        this.orderProcessingTimer = Timer.builder("orders.processing.time")
            .description("订单处理时间")
            .tag("type", "business")
            .register(registry);
    }
    
    public void recordOrderCreated() {
        orderCreatedCounter.increment();
    }
    
    public void recordOrderFailed() {
        orderFailedCounter.increment();
    }
    
    public void recordOrderProcessingTime(Runnable task) {
        orderProcessingTimer.record(task);
    }
}
```

### 1.3 使用 @Timed 注解

```java
package com.example.service;

import io.micrometer.core.annotation.Timed;
import org.springframework.stereotype.Service;

@Service
public class OrderService {
    
    @Timed(
        value = "order.create",
        description = "创建订单耗时",
        percentiles = {0.5, 0.95, 0.99}
    )
    public Order createOrder(CreateOrderRequest request) {
        // 业务逻辑
    }
}
```

### 1.4 Prometheus 配置

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'spring-boot-app'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['localhost:8080']
        labels:
          application: 'order-service'
          environment: 'production'
```

### 1.5 Grafana Dashboard

常用的 Grafana Dashboard ID：
- **Spring Boot 2.1 System Monitor**: 11378
- **JVM (Micrometer)**: 4701
- **Spring Boot Statistics**: 6756

## 2. Logs（日志管理）

### 2.1 Logback 配置

```xml
<!-- logback-spring.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>
    
    <springProperty scope="context" name="APP_NAME" source="spring.application.name"/>
    <springProperty scope="context" name="LOG_PATH" source="logging.file.path" defaultValue="logs"/>
    
    <!-- 控制台输出 -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
            <charset>UTF-8</charset>
        </encoder>
    </appender>
    
    <!-- 文件输出 -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_PATH}/${APP_NAME}.log</file>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
            <charset>UTF-8</charset>
        </encoder>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${LOG_PATH}/${APP_NAME}-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>100MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
    </appender>
    
    <!-- JSON 格式输出（用于 ELK） -->
    <appender name="JSON" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_PATH}/${APP_NAME}-json.log</file>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <customFields>{"app":"${APP_NAME}"}</customFields>
        </encoder>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${LOG_PATH}/${APP_NAME}-json-%d{yyyy-MM-dd}.log</fileNamePattern>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
    </appender>
    
    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="FILE"/>
        <appender-ref ref="JSON"/>
    </root>
    
    <!-- 特定包的日志级别 -->
    <logger name="com.example" level="DEBUG"/>
    <logger name="org.springframework.web" level="INFO"/>
    <logger name="org.hibernate.SQL" level="DEBUG"/>
    <logger name="org.hibernate.type.descriptor.sql.BasicBinder" level="TRACE"/>
</configuration>
```

### 2.2 结构化日志

```java
package com.example.logging;

import net.logstash.logback.argument.StructuredArguments;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

@Component
public class StructuredLogging {
    
    private static final Logger log = LoggerFactory.getLogger(StructuredLogging.class);
    
    public void logOrderCreated(String orderId, String customerId, double amount) {
        log.info("订单创建成功",
            StructuredArguments.kv("orderId", orderId),
            StructuredArguments.kv("customerId", customerId),
            StructuredArguments.kv("amount", amount),
            StructuredArguments.kv("eventType", "ORDER_CREATED")
        );
    }
    
    public void logWithContext(String message, Map<String, Object> context) {
        log.info(message, StructuredArguments.entries(context));
    }
}
```

### 2.3 MDC（Mapped Diagnostic Context）

```java
package com.example.logging;

import jakarta.servlet.*;
import jakarta.servlet.http.HttpServletRequest;
import org.slf4j.MDC;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.util.UUID;

@Component
public class MdcFilter implements Filter {
    
    private static final String REQUEST_ID = "requestId";
    private static final String USER_ID = "userId";
    
    @Override
    public void doFilter(
        ServletRequest request,
        ServletResponse response,
        FilterChain chain
    ) throws IOException, ServletException {
        try {
            HttpServletRequest httpRequest = (HttpServletRequest) request;
            
            // 生成请求 ID
            String requestId = UUID.randomUUID().toString();
            MDC.put(REQUEST_ID, requestId);
            
            // 从请求头或 Security Context 获取用户 ID
            String userId = getUserIdFromRequest(httpRequest);
            if (userId != null) {
                MDC.put(USER_ID, userId);
            }
            
            chain.doFilter(request, response);
        } finally {
            MDC.clear();
        }
    }
    
    private String getUserIdFromRequest(HttpServletRequest request) {
        // 实现获取用户 ID 的逻辑
        return null;
    }
}
```

### 2.4 ELK Stack 配置

**Filebeat 配置：**
```yaml
# filebeat.yml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/app/*-json.log
    json.keys_under_root: true
    json.add_error_key: true

output.elasticsearch:
  hosts: ["localhost:9200"]
  index: "app-logs-%{+yyyy.MM.dd}"

setup.kibana:
  host: "localhost:5601"
```

## 3. Traces（分布式追踪）

### 3.1 Micrometer Tracing 配置

```yaml
# application.yml
management:
  tracing:
    sampling:
      probability: 1.0  # 采样率（生产环境建议 0.1）
  zipkin:
    tracing:
      endpoint: http://localhost:9411/api/v2/spans

logging:
  pattern:
    level: '%5p [${spring.application.name:},%X{traceId:-},%X{spanId:-}]'
```

### 3.2 自定义 Span

```java
package com.example.tracing;

import io.micrometer.tracing.Span;
import io.micrometer.tracing.Tracer;
import org.springframework.stereotype.Service;

@Service
public class TracingService {
    
    private final Tracer tracer;
    
    public TracingService(Tracer tracer) {
        this.tracer = tracer;
    }
    
    public void processOrder(String orderId) {
        Span span = tracer.nextSpan().name("process-order").start();
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            span.tag("orderId", orderId);
            span.tag("operation", "process");
            
            // 业务逻辑
            validateOrder(orderId);
            saveOrder(orderId);
            
            span.event("order-processed");
        } catch (Exception e) {
            span.error(e);
            throw e;
        } finally {
            span.end();
        }
    }
    
    private void validateOrder(String orderId) {
        Span span = tracer.nextSpan().name("validate-order").start();
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            // 验证逻辑
        } finally {
            span.end();
        }
    }
    
    private void saveOrder(String orderId) {
        Span span = tracer.nextSpan().name("save-order").start();
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            // 保存逻辑
        } finally {
            span.end();
        }
    }
}
```

### 3.3 Zipkin 部署

```bash
# Docker 方式
docker run -d -p 9411:9411 openzipkin/zipkin
```

## 4. 健康检查

### 4.1 自定义健康检查

```java
package com.example.health;

import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;

@Component
public class CustomHealthIndicator implements HealthIndicator {
    
    @Override
    public Health health() {
        // 检查外部依赖
        boolean databaseUp = checkDatabase();
        boolean redisUp = checkRedis();
        
        if (databaseUp && redisUp) {
            return Health.up()
                .withDetail("database", "UP")
                .withDetail("redis", "UP")
                .build();
        }
        
        return Health.down()
            .withDetail("database", databaseUp ? "UP" : "DOWN")
            .withDetail("redis", redisUp ? "UP" : "DOWN")
            .build();
    }
    
    private boolean checkDatabase() {
        // 实现数据库检查
        return true;
    }
    
    private boolean checkRedis() {
        // 实现 Redis 检查
        return true;
    }
}
```

### 4.2 Kubernetes 探针

```yaml
# application.yml
management:
  endpoint:
    health:
      probes:
        enabled: true
      group:
        liveness:
          include: livenessState
        readiness:
          include: readinessState,db,redis
```

```yaml
# k8s-deployment.yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: app
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
```

## 5. 告警配置

### 5.1 Prometheus 告警规则

```yaml
# alert-rules.yml
groups:
  - name: application
    interval: 30s
    rules:
      # 高错误率告警
      - alert: HighErrorRate
        expr: rate(http_server_requests_seconds_count{status=~"5.."}[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "应用错误率过高"
          description: "{{ $labels.application }} 错误率超过 5%"
      
      # 高响应时间告警
      - alert: HighResponseTime
        expr: histogram_quantile(0.95, rate(http_server_requests_seconds_bucket[5m])) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "应用响应时间过长"
          description: "{{ $labels.application }} P95 响应时间超过 1 秒"
      
      # 内存使用率告警
      - alert: HighMemoryUsage
        expr: (jvm_memory_used_bytes / jvm_memory_max_bytes) > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "JVM 内存使用率过高"
          description: "{{ $labels.application }} 内存使用率超过 90%"
      
      # 应用不可用告警
      - alert: ApplicationDown
        expr: up{job="spring-boot-app"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "应用不可用"
          description: "{{ $labels.application }} 无法访问"
```

### 5.2 Alertmanager 配置

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname', 'cluster']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  receiver: 'default'
  routes:
    - match:
        severity: critical
      receiver: 'critical'
    - match:
        severity: warning
      receiver: 'warning'

receivers:
  - name: 'default'
    webhook_configs:
      - url: 'http://localhost:8080/webhook/alert'
  
  - name: 'critical'
    webhook_configs:
      - url: 'http://localhost:8080/webhook/alert'
    email_configs:
      - to: 'ops@example.com'
        from: 'alert@example.com'
        smarthost: 'smtp.example.com:587'
  
  - name: 'warning'
    webhook_configs:
      - url: 'http://localhost:8080/webhook/alert'
```

### 5.3 钉钉告警

```java
package com.example.alert;

import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

import java.util.HashMap;
import java.util.Map;

@Service
public class DingTalkAlertService {
    
    private final RestTemplate restTemplate;
    private final String webhookUrl;
    
    public DingTalkAlertService(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
        this.webhookUrl = "https://oapi.dingtalk.com/robot/send?access_token=YOUR_TOKEN";
    }
    
    public void sendAlert(String title, String content) {
        Map<String, Object> message = new HashMap<>();
        message.put("msgtype", "markdown");
        
        Map<String, String> markdown = new HashMap<>();
        markdown.put("title", title);
        markdown.put("text", content);
        message.put("markdown", markdown);
        
        restTemplate.postForObject(webhookUrl, message, String.class);
    }
}
```

## 6. 性能分析

### 6.1 JVM 监控

```java
package com.example.monitoring;

import org.springframework.boot.actuate.metrics.MetricsEndpoint;
import org.springframework.stereotype.Component;

import java.lang.management.ManagementFactory;
import java.lang.management.MemoryMXBean;
import java.lang.management.ThreadMXBean;

@Component
public class JvmMonitor {
    
    private final MemoryMXBean memoryMXBean = ManagementFactory.getMemoryMXBean();
    private final ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();
    
    public JvmMetrics getJvmMetrics() {
        long heapUsed = memoryMXBean.getHeapMemoryUsage().getUsed();
        long heapMax = memoryMXBean.getHeapMemoryUsage().getMax();
        long nonHeapUsed = memoryMXBean.getNonHeapMemoryUsage().getUsed();
        int threadCount = threadMXBean.getThreadCount();
        
        return new JvmMetrics(heapUsed, heapMax, nonHeapUsed, threadCount);
    }
}

record JvmMetrics(
    long heapUsed,
    long heapMax,
    long nonHeapUsed,
    int threadCount
) {}
```

### 6.2 慢查询监控

```java
package com.example.monitoring;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

@Aspect
@Component
public class SlowQueryMonitor {
    
    private static final Logger log = LoggerFactory.getLogger(SlowQueryMonitor.class);
    private static final long SLOW_QUERY_THRESHOLD = 1000; // 1 秒
    
    @Around("execution(* com.example.repository..*(..))")
    public Object monitorQuery(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.currentTimeMillis();
        
        try {
            return joinPoint.proceed();
        } finally {
            long duration = System.currentTimeMillis() - start;
            
            if (duration > SLOW_QUERY_THRESHOLD) {
                log.warn("慢查询检测: {} 耗时 {}ms",
                    joinPoint.getSignature().toShortString(),
                    duration
                );
            }
        }
    }
}
```

## 7. 完整配置示例

```yaml
# application.yml
spring:
  application:
    name: order-service

management:
  endpoints:
    web:
      exposure:
        include: '*'
      base-path: /actuator
  endpoint:
    health:
      show-details: always
      probes:
        enabled: true
    metrics:
      enabled: true
    prometheus:
      enabled: true
  metrics:
    tags:
      application: ${spring.application.name}
      environment: ${spring.profiles.active:dev}
    distribution:
      percentiles-histogram:
        http.server.requests: true
      slo:
        http.server.requests: 50ms,100ms,200ms,500ms,1s
  tracing:
    sampling:
      probability: 0.1
  zipkin:
    tracing:
      endpoint: http://localhost:9411/api/v2/spans

logging:
  level:
    root: INFO
    com.example: DEBUG
  pattern:
    console: '%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} [%X{traceId:-},%X{spanId:-}] - %msg%n'
  file:
    path: logs
```

## 相关技能

- `/springboot-project-planner-cn` - 规划可观测性需求
- `/springboot-security-cn` - 保护监控端点
- `/springboot-testing-cn` - 测试监控功能
