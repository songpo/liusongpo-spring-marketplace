---
name: integration
description: 在 Spring Boot 应用中使用 Spring Integration 实现企业集成模式(EIP)、消息通道、端点、转换器、路由、聚合时使用
---

# Spring Integration 企业集成模式指南

当你需要在应用中实现企业集成模式(EIP)时使用此技能,包括消息通道、消息端点、消息转换、消息路由、消息聚合、消息分割等功能。

## 何时使用

- 系统间消息传递和集成
- 异步消息处理
- 消息路由和转换
- 文件处理和监控
- 与外部系统集成(FTP/SFTP/HTTP/JMS)
- 事件驱动架构
- 批处理和调度任务

## Maven 依赖

**Spring Initializr 依赖选择:**
- Spring Integration: `integration`
- Spring Boot Actuator: `actuator` (可选,用于监控)

**注意:** Spring Initializr 提供基础的 Spring Integration 依赖,其他适配器(文件、HTTP、JMS等)需要手动添加。

```xml
<dependencies>
    <!-- Spring Integration 核心 -->
    <dependency>
        <groupId>org.springframework.integration</groupId>
        <artifactId>spring-integration-core</artifactId>
    </dependency>
    
    <!-- 文件支持 -->
    <dependency>
        <groupId>org.springframework.integration</groupId>
        <artifactId>spring-integration-file</artifactId>
    </dependency>
    
    <!-- HTTP 支持 -->
    <dependency>
        <groupId>org.springframework.integration</groupId>
        <artifactId>spring-integration-http</artifactId>
    </dependency>
    
    <!-- JMS 支持 -->
    <dependency>
        <groupId>org.springframework.integration</groupId>
        <artifactId>spring-integration-jms</artifactId>
    </dependency>
    
    <!-- JDBC 支持 -->
    <dependency>
        <groupId>org.springframework.integration</groupId>
        <artifactId>spring-integration-jdbc</artifactId>
    </dependency>
    
    <!-- Mail 支持 -->
    <dependency>
        <groupId>org.springframework.integration</groupId>
        <artifactId>spring-integration-mail</artifactId>
    </dependency>
    
    <!-- FTP/SFTP 支持 -->
    <dependency>
        <groupId>org.springframework.integration</groupId>
        <artifactId>spring-integration-ftp</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.integration</groupId>
        <artifactId>spring-integration-sftp</artifactId>
    </dependency>
</dependencies>
```

## 1. 核心概念

### 1.1 消息(Message)

```java
package com.example.integration;

import org.springframework.messaging.Message;
import org.springframework.messaging.support.MessageBuilder;

public class MessageExample {
    
    public void createMessage() {
        // 创建消息
        Message<String> message = MessageBuilder
            .withPayload("Hello Spring Integration")
            .setHeader("userId", 123L)
            .setHeader("timestamp", System.currentTimeMillis())
            .build();
            
        // 获取消息内容
        String payload = message.getPayload();
        
        // 获取消息头
        Long userId = (Long) message.getHeaders().get("userId");
    }
}
```

### 1.2 消息通道(Message Channel)

```java
package com.example.integration.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.channel.DirectChannel;
import org.springframework.integration.channel.PublishSubscribeChannel;
import org.springframework.integration.channel.QueueChannel;
import org.springframework.messaging.MessageChannel;

@Configuration
public class ChannelConfig {
    
    // 点对点通道(同步)
    @Bean
    public MessageChannel directChannel() {
        return new DirectChannel();
    }
    
    // 发布订阅通道(多个订阅者)
    @Bean
    public MessageChannel pubSubChannel() {
        return new PublishSubscribeChannel();
    }
    
    // 队列通道(异步,带缓冲)
    @Bean
    public MessageChannel queueChannel() {
        return new QueueChannel(100);  // 容量100
    }
}
```

### 1.3 消息端点(Message Endpoint)

```java
package com.example.integration.endpoint;

import org.springframework.integration.annotation.ServiceActivator;
import org.springframework.messaging.Message;
import org.springframework.stereotype.Component;

@Component
public class MessageEndpoint {
    
    // 服务激活器
    @ServiceActivator(inputChannel = "inputChannel", outputChannel = "outputChannel")
    public String processMessage(String payload) {
        System.out.println("Processing: " + payload);
        return payload.toUpperCase();
    }
    
    // 处理完整消息
    @ServiceActivator(inputChannel = "messageChannel")
    public void handleMessage(Message<?> message) {
        System.out.println("Payload: " + message.getPayload());
        System.out.println("Headers: " + message.getHeaders());
    }
}
```

## 2. 消息网关(Gateway)

### 2.1 定义网关接口

```java
package com.example.integration.gateway;

import org.springframework.integration.annotation.Gateway;
import org.springframework.integration.annotation.MessagingGateway;

// 消息网关
@MessagingGateway(defaultRequestChannel = "requestChannel")
public interface OrderGateway {
    
    // 同步调用
    @Gateway
    String processOrder(String order);
    
    // 指定请求和响应通道
    @Gateway(requestChannel = "orderInputChannel", replyChannel = "orderOutputChannel")
    OrderResponse submitOrder(OrderRequest request);
    
    // 异步调用(返回 Future)
    @Gateway(requestChannel = "asyncChannel")
    java.util.concurrent.Future<String> processAsync(String data);
    
    // 单向消息(无返回值)
    @Gateway(requestChannel = "notificationChannel")
    void sendNotification(String message);
}
```

### 2.2 使用网关

```java
package com.example.integration.service;

import com.example.integration.gateway.OrderGateway;
import org.springframework.stereotype.Service;

@Service
public class OrderService {
    
    private final OrderGateway orderGateway;
    
    public OrderService(OrderGateway orderGateway) {
        this.orderGateway = orderGateway;
    }
    
    public void createOrder(String orderData) {
        // 通过网关发送消息
        String result = orderGateway.processOrder(orderData);
        System.out.println("Order result: " + result);
    }
}
```

## 3. 消息转换(Transformer)

### 3.1 基础转换器

```java
package com.example.integration.transformer;

import org.springframework.integration.annotation.Transformer;
import org.springframework.stereotype.Component;

@Component
public class MessageTransformer {
    
    // 简单转换
    @Transformer(inputChannel = "inputChannel", outputChannel = "outputChannel")
    public String transform(String input) {
        return input.toUpperCase();
    }
    
    // 对象转换
    @Transformer(inputChannel = "orderChannel", outputChannel = "processedOrderChannel")
    public ProcessedOrder transformOrder(Order order) {
        ProcessedOrder processed = new ProcessedOrder();
        processed.setOrderId(order.getId());
        processed.setTotalAmount(order.getAmount() * 1.1);  // 加税
        return processed;
    }
}
```

### 3.2 JSON 转换

```java
package com.example.integration.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.annotation.Transformer;
import org.springframework.integration.channel.DirectChannel;
import org.springframework.integration.json.JsonToObjectTransformer;
import org.springframework.integration.json.ObjectToJsonTransformer;
import org.springframework.messaging.MessageChannel;

@Configuration
public class JsonTransformerConfig {
    
    @Bean
    public MessageChannel jsonInputChannel() {
        return new DirectChannel();
    }
    
    @Bean
    public MessageChannel objectChannel() {
        return new DirectChannel();
    }
    
    // JSON 转对象
    @Bean
    @Transformer(inputChannel = "jsonInputChannel", outputChannel = "objectChannel")
    public JsonToObjectTransformer jsonToObjectTransformer() {
        return new JsonToObjectTransformer(Order.class);
    }
    
    // 对象转 JSON
    @Bean
    @Transformer(inputChannel = "objectChannel", outputChannel = "jsonOutputChannel")
    public ObjectToJsonTransformer objectToJsonTransformer() {
        return new ObjectToJsonTransformer();
    }
}
```

## 4. 消息路由(Router)

### 4.1 基于内容的路由

```java
package com.example.integration.router;

import org.springframework.integration.annotation.Router;
import org.springframework.stereotype.Component;

@Component
public class MessageRouter {
    
    // 基于返回值路由
    @Router(inputChannel = "routerInputChannel")
    public String route(Order order) {
        if (order.getAmount() > 1000) {
            return "highValueChannel";
        } else if (order.getAmount() > 100) {
            return "mediumValueChannel";
        } else {
            return "lowValueChannel";
        }
    }
    
    // 路由到多个通道
    @Router(inputChannel = "multiRouterChannel")
    public java.util.List<String> routeToMultiple(String message) {
        java.util.List<String> channels = new java.util.ArrayList<>();
        
        if (message.contains("urgent")) {
            channels.add("urgentChannel");
        }
        if (message.contains("notification")) {
            channels.add("notificationChannel");
        }
        channels.add("defaultChannel");
        
        return channels;
    }
}
```

### 4.2 基于消息头路由

```java
package com.example.integration.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.annotation.Router;
import org.springframework.integration.router.HeaderValueRouter;

@Configuration
public class HeaderRouterConfig {
    
    @Bean
    @Router(inputChannel = "headerRouterInput")
    public HeaderValueRouter headerRouter() {
        HeaderValueRouter router = new HeaderValueRouter("orderType");
        router.setChannelMapping("ONLINE", "onlineOrderChannel");
        router.setChannelMapping("OFFLINE", "offlineOrderChannel");
        router.setDefaultOutputChannel(defaultOrderChannel());
        return router;
    }
    
    @Bean
    public org.springframework.messaging.MessageChannel defaultOrderChannel() {
        return new org.springframework.integration.channel.DirectChannel();
    }
}
```

## 5. 消息过滤(Filter)

```java
package com.example.integration.filter;

import org.springframework.integration.annotation.Filter;
import org.springframework.stereotype.Component;

@Component
public class MessageFilter {
    
    // 简单过滤
    @Filter(inputChannel = "filterInputChannel", outputChannel = "filterOutputChannel")
    public boolean filter(String message) {
        // 返回 true 通过,false 丢弃
        return message != null && !message.isEmpty();
    }
    
    // 对象过滤
    @Filter(inputChannel = "orderFilterChannel", outputChannel = "validOrderChannel")
    public boolean filterOrder(Order order) {
        return order.getAmount() > 0 && order.getCustomerId() != null;
    }
    
    // 丢弃的消息发送到其他通道
    @Filter(inputChannel = "inputChannel", 
            outputChannel = "validChannel",
            discardChannel = "invalidChannel")
    public boolean validateMessage(String message) {
        return message.matches("^[A-Z0-9]+$");
    }
}
```

## 6. 消息分割和聚合

### 6.1 消息分割(Splitter)

```java
package com.example.integration.splitter;

import org.springframework.integration.annotation.Splitter;
import org.springframework.stereotype.Component;

import java.util.Arrays;
import java.util.List;

@Component
public class MessageSplitter {
    
    // 分割字符串
    @Splitter(inputChannel = "splitInputChannel", outputChannel = "splitOutputChannel")
    public List<String> split(String message) {
        return Arrays.asList(message.split(","));
    }
    
    // 分割订单项
    @Splitter(inputChannel = "orderSplitChannel", outputChannel = "orderItemChannel")
    public List<OrderItem> splitOrder(Order order) {
        return order.getItems();
    }
}
```

### 6.2 消息聚合(Aggregator)

```java
package com.example.integration.aggregator;

import org.springframework.integration.annotation.Aggregator;
import org.springframework.integration.annotation.CorrelationStrategy;
import org.springframework.integration.annotation.ReleaseStrategy;
import org.springframework.messaging.Message;
import org.springframework.stereotype.Component;

import java.util.List;

@Component
public class MessageAggregator {
    
    // 聚合消息
    @Aggregator(inputChannel = "aggregateInputChannel", 
                outputChannel = "aggregateOutputChannel")
    public String aggregate(List<String> messages) {
        return String.join(", ", messages);
    }
    
    // 关联策略(决定哪些消息属于同一组)
    @CorrelationStrategy
    public Object correlationKey(Message<?> message) {
        return message.getHeaders().get("orderId");
    }
    
    // 释放策略(决定何时聚合完成)
    @ReleaseStrategy
    public boolean release(List<Message<?>> messages) {
        // 收集到5条消息或包含结束标记
        return messages.size() >= 5 || 
               messages.stream().anyMatch(m -> 
                   Boolean.TRUE.equals(m.getHeaders().get("last")));
    }
}
```

## 7. 文件处理

### 7.1 文件入站适配器(读取文件)

```java
package com.example.integration.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.annotation.InboundChannelAdapter;
import org.springframework.integration.annotation.Poller;
import org.springframework.integration.annotation.ServiceActivator;
import org.springframework.integration.core.MessageSource;
import org.springframework.integration.file.FileReadingMessageSource;
import org.springframework.integration.file.filters.SimplePatternFileListFilter;
import org.springframework.messaging.Message;

import java.io.File;

@Configuration
public class FileInboundConfig {
    
    // 文件入站适配器
    @Bean
    @InboundChannelAdapter(value = "fileInputChannel", 
                          poller = @Poller(fixedDelay = "5000"))
    public MessageSource<File> fileReadingMessageSource() {
        FileReadingMessageSource source = new FileReadingMessageSource();
        source.setDirectory(new File("/data/input"));
        source.setFilter(new SimplePatternFileListFilter("*.txt"));
        source.setAutoCreateDirectory(true);
        return source;
    }
    
    // 处理文件
    @Bean
    @ServiceActivator(inputChannel = "fileInputChannel")
    public void processFile(Message<File> message) {
        File file = message.getPayload();
        System.out.println("Processing file: " + file.getName());
        // 处理文件内容
    }
}
```

### 7.2 文件出站适配器(写入文件)

```java
package com.example.integration.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.annotation.ServiceActivator;
import org.springframework.integration.file.FileWritingMessageHandler;
import org.springframework.integration.file.support.FileExistsMode;
import org.springframework.messaging.MessageHandler;

import java.io.File;

@Configuration
public class FileOutboundConfig {
    
    @Bean
    @ServiceActivator(inputChannel = "fileOutputChannel")
    public MessageHandler fileWritingMessageHandler() {
        FileWritingMessageHandler handler = 
            new FileWritingMessageHandler(new File("/data/output"));
        handler.setAutoCreateDirectory(true);
        handler.setFileExistsMode(FileExistsMode.REPLACE);
        handler.setExpectReply(false);
        return handler;
    }
}
```

### 7.3 文件转换和移动

```java
package com.example.integration.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.dsl.IntegrationFlow;
import org.springframework.integration.file.dsl.Files;
import org.springframework.integration.file.transformer.FileToStringTransformer;

import java.io.File;

@Configuration
public class FileProcessingFlow {
    
    @Bean
    public IntegrationFlow fileFlow() {
        return IntegrationFlow
            // 读取文件
            .from(Files.inboundAdapter(new File("/data/input"))
                .patternFilter("*.csv")
                .autoCreateDirectory(true),
                e -> e.poller(p -> p.fixedDelay(5000)))
            // 转换为字符串
            .transform(new FileToStringTransformer())
            // 处理内容
            .handle(String.class, (payload, headers) -> {
                System.out.println("File content: " + payload);
                return payload.toUpperCase();
            })
            // 写入输出文件
            .handle(Files.outboundAdapter(new File("/data/output"))
                .autoCreateDirectory(true)
                .fileExistsMode(FileExistsMode.REPLACE))
            .get();
    }
}
```

## 8. HTTP 集成

### 8.1 HTTP 入站网关

```java
package com.example.integration.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.dsl.IntegrationFlow;
import org.springframework.integration.http.dsl.Http;

@Configuration
public class HttpInboundConfig {
    
    @Bean
    public IntegrationFlow httpInboundFlow() {
        return IntegrationFlow
            .from(Http.inboundGateway("/api/orders")
                .requestMapping(m -> m.methods(org.springframework.http.HttpMethod.POST))
                .requestPayloadType(Order.class))
            .handle(Order.class, (payload, headers) -> {
                // 处理订单
                System.out.println("Received order: " + payload);
                return "Order processed: " + payload.getId();
            })
            .get();
    }
}
```

### 8.2 HTTP 出站网关

```java
package com.example.integration.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.dsl.IntegrationFlow;
import org.springframework.integration.http.dsl.Http;
import org.springframework.http.HttpMethod;

@Configuration
public class HttpOutboundConfig {
    
    @Bean
    public IntegrationFlow httpOutboundFlow() {
        return IntegrationFlow
            .from("httpRequestChannel")
            .handle(Http.outboundGateway("http://external-api.com/api/data")
                .httpMethod(HttpMethod.POST)
                .expectedResponseType(String.class))
            .channel("httpResponseChannel")
            .get();
    }
}
```

## 9. DSL 配置(Java DSL)

### 9.1 完整流程示例

```java
package com.example.integration.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.dsl.IntegrationFlow;
import org.springframework.integration.dsl.Pollers;

@Configuration
public class IntegrationFlowConfig {
    
    @Bean
    public IntegrationFlow orderProcessingFlow() {
        return IntegrationFlow
            // 从网关接收
            .from("orderInputChannel")
            // 过滤
            .filter(Order.class, order -> order.getAmount() > 0)
            // 转换
            .transform(Order.class, order -> {
                order.setStatus("PROCESSING");
                return order;
            })
            // 路由
            .route(Order.class, order -> {
                if (order.getAmount() > 1000) {
                    return "highValueChannel";
                } else {
                    return "normalChannel";
                }
            })
            .get();
    }
    
    @Bean
    public IntegrationFlow highValueFlow() {
        return IntegrationFlow
            .from("highValueChannel")
            .handle(Order.class, (payload, headers) -> {
                System.out.println("High value order: " + payload);
                // 特殊处理
                return payload;
            })
            .channel("orderOutputChannel")
            .get();
    }
    
    @Bean
    public IntegrationFlow normalFlow() {
        return IntegrationFlow
            .from("normalChannel")
            .handle(Order.class, (payload, headers) -> {
                System.out.println("Normal order: " + payload);
                return payload;
            })
            .channel("orderOutputChannel")
            .get();
    }
}
```

### 9.2 轮询和调度

```java
package com.example.integration.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.dsl.IntegrationFlow;
import org.springframework.integration.dsl.Pollers;

@Configuration
public class PollingConfig {
    
    @Bean
    public IntegrationFlow pollingFlow() {
        return IntegrationFlow
            .from(() -> "Scheduled message",
                e -> e.poller(Pollers.fixedDelay(5000)))
            .handle(message -> {
                System.out.println("Polling: " + message.getPayload());
            })
            .get();
    }
    
    @Bean
    public IntegrationFlow cronFlow() {
        return IntegrationFlow
            .from(() -> java.time.LocalDateTime.now(),
                e -> e.poller(Pollers.cron("0 0 * * * *")))  // 每小时
            .handle(message -> {
                System.out.println("Cron job executed at: " + message.getPayload());
            })
            .get();
    }
}
```

## 10. 错误处理

### 10.1 全局错误处理

```java
package com.example.integration.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.annotation.ServiceActivator;
import org.springframework.integration.channel.DirectChannel;
import org.springframework.messaging.Message;
import org.springframework.messaging.MessageChannel;
import org.springframework.messaging.support.ErrorMessage;

@Configuration
public class ErrorHandlingConfig {
    
    @Bean
    public MessageChannel errorChannel() {
        return new DirectChannel();
    }
    
    @ServiceActivator(inputChannel = "errorChannel")
    public void handleError(ErrorMessage errorMessage) {
        Throwable throwable = errorMessage.getPayload();
        Message<?> originalMessage = errorMessage.getOriginalMessage();
        
        System.err.println("Error occurred: " + throwable.getMessage());
        System.err.println("Original message: " + originalMessage);
        
        // 记录错误、发送告警等
    }
}
```

### 10.2 流程级错误处理

```java
package com.example.integration.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.dsl.IntegrationFlow;

@Configuration
public class FlowErrorHandlingConfig {
    
    @Bean
    public IntegrationFlow flowWithErrorHandling() {
        return IntegrationFlow
            .from("inputChannel")
            .handle(String.class, (payload, headers) -> {
                if (payload.contains("error")) {
                    throw new RuntimeException("Processing error");
                }
                return payload;
            })
            // 错误处理
            .handle((payload, headers) -> {
                System.out.println("Success: " + payload);
                return payload;
            }, e -> e.advice(retryAdvice()))
            .get();
    }
    
    @Bean
    public org.springframework.integration.handler.advice.RequestHandlerRetryAdvice retryAdvice() {
        org.springframework.integration.handler.advice.RequestHandlerRetryAdvice advice = 
            new org.springframework.integration.handler.advice.RequestHandlerRetryAdvice();
        advice.setRetryTemplate(retryTemplate());
        return advice;
    }
    
    @Bean
    public org.springframework.retry.support.RetryTemplate retryTemplate() {
        org.springframework.retry.support.RetryTemplate template = 
            new org.springframework.retry.support.RetryTemplate();
        
        org.springframework.retry.policy.SimpleRetryPolicy retryPolicy = 
            new org.springframework.retry.policy.SimpleRetryPolicy();
        retryPolicy.setMaxAttempts(3);
        template.setRetryPolicy(retryPolicy);
        
        return template;
    }
}
```

## 11. 监控和管理

### 11.1 启用 JMX

```yaml
spring:
  jmx:
    enabled: true
    
management:
  endpoints:
    web:
      exposure:
        include: integrationgraph,health,metrics
```

### 11.2 集成图可视化

```java
package com.example.integration.controller;

import org.springframework.integration.graph.IntegrationGraphServer;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class IntegrationGraphController {
    
    private final IntegrationGraphServer integrationGraphServer;
    
    public IntegrationGraphController(IntegrationGraphServer integrationGraphServer) {
        this.integrationGraphServer = integrationGraphServer;
    }
    
    @GetMapping("/integration/graph")
    public String getGraph() {
        return integrationGraphServer.getGraph();
    }
}
```

## 12. 最佳实践

### 12.1 通道选择

| 场景 | 通道类型 | 说明 |
|-----|---------|------|
| 同步处理 | DirectChannel | 调用者线程执行 |
| 异步处理 | QueueChannel | 独立线程池执行 |
| 发布订阅 | PublishSubscribeChannel | 多个订阅者 |
| 优先级处理 | PriorityChannel | 按优先级排序 |

### 12.2 错误处理策略

```java
// 1. 重试
@ServiceActivator(inputChannel = "input", adviceChain = "retryAdvice")
public String process(String message) {
    // 可能失败的操作
}

// 2. 降级
@ServiceActivator(inputChannel = "input")
public String processWithFallback(String message) {
    try {
        return externalService.call(message);
    } catch (Exception e) {
        return fallbackService.call(message);
    }
}

// 3. 死信队列
@Router(inputChannel = "errorChannel")
public String routeError(ErrorMessage error) {
    return "deadLetterChannel";
}
```

### 12.3 性能优化

```yaml
# 配置线程池
spring:
  integration:
    channel:
      auto-create: true
      max-unicast-subscribers: 10
      max-broadcast-subscribers: 10
    poller:
      fixed-delay: 1000
      max-messages-per-poll: 10
```

## 参考资源

- [Spring Integration 官方文档](https://docs.spring.io/spring-integration/docs/current/reference/html/)
- [Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/)
- [Spring Integration GitHub](https://github.com/spring-projects/spring-integration)
