---
name: springboot-messaging-cn
description: Spring Boot 消息队列实践，包括 RabbitMQ、Kafka、Spring Cloud Stream 和消息可靠性保证
---

# Spring Boot 消息队列实践指南

当你需要在 Spring Boot 应用中使用消息队列实现异步通信和解耦时使用此技能，包括 RabbitMQ、Kafka 和 Spring Cloud Stream。

## 何时使用

- 实现系统解耦
- 异步处理任务
- 削峰填谷
- 实现事件驱动架构
- 分布式事务（最终一致性）
- 日志收集和数据同步

## Maven 依赖

```xml
<dependencies>
    <!-- RabbitMQ -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-amqp</artifactId>
    </dependency>
    
    <!-- Kafka -->
    <dependency>
        <groupId>org.springframework.kafka</groupId>
        <artifactId>spring-kafka</artifactId>
    </dependency>
    
    <!-- Spring Cloud Stream -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
    </dependency>
    <!-- 或者 Kafka -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-stream-kafka</artifactId>
    </dependency>
</dependencies>
```

## 1. RabbitMQ

### 1.1 RabbitMQ 配置

```yaml
# application.yml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
    virtual-host: /
    # 连接池配置
    listener:
      simple:
        # 消费者数量
        concurrency: 5
        max-concurrency: 10
        # 手动确认
        acknowledge-mode: manual
        # 重试配置
        retry:
          enabled: true
          initial-interval: 1000
          max-attempts: 3
          multiplier: 2
    # 发布者确认
    publisher-confirm-type: correlated
    publisher-returns: true
    template:
      mandatory: true
```

### 1.2 RabbitMQ 配置类

```java
package com.example.config;

import org.springframework.amqp.core.*;
import org.springframework.amqp.rabbit.config.SimpleRabbitListenerContainerFactory;
import org.springframework.amqp.rabbit.connection.ConnectionFactory;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.amqp.support.converter.Jackson2JsonMessageConverter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RabbitMQConfig {
    
    // 队列名称
    public static final String ORDER_QUEUE = "order.queue";
    public static final String ORDER_EXCHANGE = "order.exchange";
    public static final String ORDER_ROUTING_KEY = "order.create";
    
    // 死信队列
    public static final String DLX_QUEUE = "order.dlx.queue";
    public static final String DLX_EXCHANGE = "order.dlx.exchange";
    public static final String DLX_ROUTING_KEY = "order.dlx";
    
    // 声明队列
    @Bean
    public Queue orderQueue() {
        return QueueBuilder.durable(ORDER_QUEUE)
            .withArgument("x-dead-letter-exchange", DLX_EXCHANGE)
            .withArgument("x-dead-letter-routing-key", DLX_ROUTING_KEY)
            .withArgument("x-message-ttl", 60000) // 消息 TTL 60 秒
            .build();
    }
    
    // 声明交换机
    @Bean
    public DirectExchange orderExchange() {
        return new DirectExchange(ORDER_EXCHANGE, true, false);
    }
    
    // 绑定队列到交换机
    @Bean
    public Binding orderBinding() {
        return BindingBuilder
            .bind(orderQueue())
            .to(orderExchange())
            .with(ORDER_ROUTING_KEY);
    }
    
    // 死信队列
    @Bean
    public Queue dlxQueue() {
        return QueueBuilder.durable(DLX_QUEUE).build();
    }
    
    @Bean
    public DirectExchange dlxExchange() {
        return new DirectExchange(DLX_EXCHANGE, true, false);
    }
    
    @Bean
    public Binding dlxBinding() {
        return BindingBuilder
            .bind(dlxQueue())
            .to(dlxExchange())
            .with(DLX_ROUTING_KEY);
    }
    
    // JSON 消息转换器
    @Bean
    public Jackson2JsonMessageConverter messageConverter() {
        return new Jackson2JsonMessageConverter();
    }
    
    // RabbitTemplate 配置
    @Bean
    public RabbitTemplate rabbitTemplate(
        ConnectionFactory connectionFactory,
        Jackson2JsonMessageConverter messageConverter
    ) {
        RabbitTemplate template = new RabbitTemplate(connectionFactory);
        template.setMessageConverter(messageConverter);
        
        // 发布者确认回调
        template.setConfirmCallback((correlationData, ack, cause) -> {
            if (ack) {
                System.out.println("消息发送成功");
            } else {
                System.err.println("消息发送失败: " + cause);
            }
        });
        
        // 消息返回回调（消息无法路由时）
        template.setReturnsCallback(returned -> {
            System.err.println("消息被退回: " + returned.getMessage());
        });
        
        return template;
    }
    
    // 监听器容器工厂
    @Bean
    public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory(
        ConnectionFactory connectionFactory,
        Jackson2JsonMessageConverter messageConverter
    ) {
        SimpleRabbitListenerContainerFactory factory = 
            new SimpleRabbitListenerContainerFactory();
        factory.setConnectionFactory(connectionFactory);
        factory.setMessageConverter(messageConverter);
        factory.setConcurrentConsumers(5);
        factory.setMaxConcurrentConsumers(10);
        factory.setPrefetchCount(10);
        return factory;
    }
}
```

### 1.3 发送消息

```java
package com.example.service;

import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.stereotype.Service;

@Service
public class OrderMessageProducer {
    
    private final RabbitTemplate rabbitTemplate;
    
    public OrderMessageProducer(RabbitTemplate rabbitTemplate) {
        this.rabbitTemplate = rabbitTemplate;
    }
    
    public void sendOrderMessage(OrderMessage message) {
        rabbitTemplate.convertAndSend(
            RabbitMQConfig.ORDER_EXCHANGE,
            RabbitMQConfig.ORDER_ROUTING_KEY,
            message
        );
    }
    
    // 带延迟的消息
    public void sendDelayedMessage(OrderMessage message, int delayMs) {
        rabbitTemplate.convertAndSend(
            RabbitMQConfig.ORDER_EXCHANGE,
            RabbitMQConfig.ORDER_ROUTING_KEY,
            message,
            msg -> {
                msg.getMessageProperties().setDelay(delayMs);
                return msg;
            }
        );
    }
}

record OrderMessage(String orderId, String customerId, double amount) {}
```

### 1.4 接收消息

```java
package com.example.listener;

import com.rabbitmq.client.Channel;
import org.springframework.amqp.core.Message;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

import java.io.IOException;

@Component
public class OrderMessageConsumer {
    
    // 基本消费
    @RabbitListener(queues = RabbitMQConfig.ORDER_QUEUE)
    public void handleOrderMessage(OrderMessage message) {
        System.out.println("收到订单消息: " + message);
        // 处理业务逻辑
    }
    
    // 手动确认
    @RabbitListener(queues = RabbitMQConfig.ORDER_QUEUE)
    public void handleOrderMessageWithAck(
        OrderMessage message,
        Channel channel,
        Message amqpMessage
    ) throws IOException {
        try {
            System.out.println("处理订单: " + message);
            
            // 业务逻辑
            processOrder(message);
            
            // 手动确认
            channel.basicAck(amqpMessage.getMessageProperties().getDeliveryTag(), false);
            
        } catch (Exception e) {
            // 拒绝消息，重新入队
            channel.basicNack(
                amqpMessage.getMessageProperties().getDeliveryTag(),
                false,
                true  // requeue
            );
        }
    }
    
    // 死信队列消费
    @RabbitListener(queues = RabbitMQConfig.DLX_QUEUE)
    public void handleDlxMessage(OrderMessage message) {
        System.err.println("处理死信消息: " + message);
        // 记录日志、发送告警等
    }
    
    private void processOrder(OrderMessage message) {
        // 业务处理
    }
}
```

### 1.5 延迟队列

```java
package com.example.config;

import org.springframework.amqp.core.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class DelayQueueConfig {
    
    public static final String DELAY_QUEUE = "order.delay.queue";
    public static final String DELAY_EXCHANGE = "order.delay.exchange";
    public static final String DELAY_ROUTING_KEY = "order.delay";
    
    // 延迟队列（通过 TTL + 死信实现）
    @Bean
    public Queue delayQueue() {
        return QueueBuilder.durable(DELAY_QUEUE)
            .withArgument("x-dead-letter-exchange", RabbitMQConfig.ORDER_EXCHANGE)
            .withArgument("x-dead-letter-routing-key", RabbitMQConfig.ORDER_ROUTING_KEY)
            .withArgument("x-message-ttl", 30000) // 30 秒后转发到业务队列
            .build();
    }
    
    @Bean
    public DirectExchange delayExchange() {
        return new DirectExchange(DELAY_EXCHANGE, true, false);
    }
    
    @Bean
    public Binding delayBinding() {
        return BindingBuilder
            .bind(delayQueue())
            .to(delayExchange())
            .with(DELAY_ROUTING_KEY);
    }
}
```

## 2. Kafka

### 2.1 Kafka 配置

```yaml
# application.yml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    # 生产者配置
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all  # 所有副本确认
      retries: 3
      batch-size: 16384
      buffer-memory: 33554432
      properties:
        linger.ms: 10
        max.in.flight.requests.per.connection: 5
    # 消费者配置
    consumer:
      group-id: order-service
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      auto-offset-reset: earliest
      enable-auto-commit: false
      properties:
        spring.json.trusted.packages: com.example.dto
        max.poll.records: 100
        max.poll.interval.ms: 300000
    # 监听器配置
    listener:
      ack-mode: manual
      concurrency: 3
```

### 2.2 Kafka 配置类

```java
package com.example.config;

import org.apache.kafka.clients.admin.NewTopic;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.config.TopicBuilder;

@Configuration
public class KafkaConfig {
    
    public static final String ORDER_TOPIC = "order-events";
    public static final String PAYMENT_TOPIC = "payment-events";
    
    @Bean
    public NewTopic orderTopic() {
        return TopicBuilder.name(ORDER_TOPIC)
            .partitions(3)
            .replicas(1)
            .build();
    }
    
    @Bean
    public NewTopic paymentTopic() {
        return TopicBuilder.name(PAYMENT_TOPIC)
            .partitions(3)
            .replicas(1)
            .build();
    }
}
```

### 2.3 发送消息

```java
package com.example.service;

import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.support.SendResult;
import org.springframework.stereotype.Service;

import java.util.concurrent.CompletableFuture;

@Service
public class KafkaProducerService {
    
    private final KafkaTemplate<String, Object> kafkaTemplate;
    
    public KafkaProducerService(KafkaTemplate<String, Object> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }
    
    public void sendMessage(String topic, String key, Object message) {
        kafkaTemplate.send(topic, key, message);
    }
    
    // 带回调的发送
    public void sendMessageWithCallback(String topic, String key, Object message) {
        CompletableFuture<SendResult<String, Object>> future = 
            kafkaTemplate.send(topic, key, message);
        
        future.whenComplete((result, ex) -> {
            if (ex == null) {
                System.out.println("消息发送成功: " + result.getRecordMetadata());
            } else {
                System.err.println("消息发送失败: " + ex.getMessage());
            }
        });
    }
    
    // 事务发送
    public void sendInTransaction(String topic, String key, Object message) {
        kafkaTemplate.executeInTransaction(operations -> {
            operations.send(topic, key, message);
            // 可以发送多条消息
            return true;
        });
    }
}
```

### 2.4 接收消息

```java
package com.example.listener;

import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.support.Acknowledgment;
import org.springframework.stereotype.Component;

@Component
public class KafkaConsumerService {
    
    // 基本消费
    @KafkaListener(topics = KafkaConfig.ORDER_TOPIC, groupId = "order-service")
    public void handleOrderEvent(OrderEvent event) {
        System.out.println("收到订单事件: " + event);
        // 处理业务逻辑
    }
    
    // 手动确认
    @KafkaListener(topics = KafkaConfig.ORDER_TOPIC, groupId = "order-service")
    public void handleOrderEventWithAck(
        ConsumerRecord<String, OrderEvent> record,
        Acknowledgment acknowledgment
    ) {
        try {
            OrderEvent event = record.value();
            System.out.println("处理订单: " + event);
            
            // 业务逻辑
            processOrder(event);
            
            // 手动确认
            acknowledgment.acknowledge();
            
        } catch (Exception e) {
            System.err.println("处理失败: " + e.getMessage());
            // 不确认，消息会重新消费
        }
    }
    
    // 批量消费
    @KafkaListener(topics = KafkaConfig.ORDER_TOPIC, groupId = "order-service")
    public void handleBatch(List<ConsumerRecord<String, OrderEvent>> records) {
        System.out.println("批量处理 " + records.size() + " 条消息");
        for (ConsumerRecord<String, OrderEvent> record : records) {
            processOrder(record.value());
        }
    }
    
    private void processOrder(OrderEvent event) {
        // 业务处理
    }
}

record OrderEvent(String orderId, String type, String data) {}
```

## 3. Spring Cloud Stream

### 3.1 配置

```yaml
# application.yml
spring:
  cloud:
    stream:
      bindings:
        # 输出通道
        orderOutput:
          destination: order-events
          content-type: application/json
        # 输入通道
        orderInput:
          destination: order-events
          group: order-service
          content-type: application/json
      # RabbitMQ 配置
      rabbit:
        bindings:
          orderOutput:
            producer:
              routing-key-expression: headers['routingKey']
          orderInput:
            consumer:
              acknowledge-mode: manual
      # Kafka 配置
      kafka:
        binder:
          brokers: localhost:9092
        bindings:
          orderOutput:
            producer:
              configuration:
                acks: all
          orderInput:
            consumer:
              configuration:
                max.poll.records: 100
```

### 3.2 定义通道

```java
package com.example.stream;

import org.springframework.cloud.stream.function.StreamBridge;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.Message;

import java.util.function.Consumer;
import java.util.function.Function;

@Configuration
public class StreamConfig {
    
    // 消费者
    @Bean
    public Consumer<Message<OrderEvent>> orderInput() {
        return message -> {
            OrderEvent event = message.getPayload();
            System.out.println("收到订单事件: " + event);
            // 处理业务逻辑
        };
    }
    
    // 处理器（输入 -> 输出）
    @Bean
    public Function<OrderEvent, PaymentEvent> orderToPayment() {
        return orderEvent -> {
            // 转换逻辑
            return new PaymentEvent(
                orderEvent.orderId(),
                orderEvent.amount()
            );
        };
    }
}

record PaymentEvent(String orderId, double amount) {}
```

### 3.3 发送消息

```java
package com.example.service;

import org.springframework.cloud.stream.function.StreamBridge;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.stereotype.Service;

@Service
public class StreamProducerService {
    
    private final StreamBridge streamBridge;
    
    public StreamProducerService(StreamBridge streamBridge) {
        this.streamBridge = streamBridge;
    }
    
    public void sendOrderEvent(OrderEvent event) {
        streamBridge.send("orderOutput", event);
    }
    
    // 带 Header 的消息
    public void sendWithHeaders(OrderEvent event, String routingKey) {
        streamBridge.send(
            "orderOutput",
            MessageBuilder.withPayload(event)
                .setHeader("routingKey", routingKey)
                .build()
        );
    }
}
```

## 4. 消息可靠性保证

### 4.1 生产者确认

```java
package com.example.service;

import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.stereotype.Service;

@Service
public class ReliableProducer {
    
    private final RabbitTemplate rabbitTemplate;
    
    public void sendWithConfirm(OrderMessage message) {
        rabbitTemplate.invoke(operations -> {
            operations.convertAndSend(
                RabbitMQConfig.ORDER_EXCHANGE,
                RabbitMQConfig.ORDER_ROUTING_KEY,
                message
            );
            
            // 等待确认
            operations.waitForConfirms(5000);
            
            return null;
        });
    }
}
```

### 4.2 消费者幂等性

```java
package com.example.service;

import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.util.concurrent.TimeUnit;

@Service
public class IdempotentConsumer {
    
    private final RedisTemplate<String, String> redisTemplate;
    
    public boolean isProcessed(String messageId) {
        String key = "msg:processed:" + messageId;
        return Boolean.TRUE.equals(redisTemplate.hasKey(key));
    }
    
    public void markAsProcessed(String messageId) {
        String key = "msg:processed:" + messageId;
        redisTemplate.opsForValue().set(key, "1", 24, TimeUnit.HOURS);
    }
    
    public void handleMessage(String messageId, OrderMessage message) {
        // 检查是否已处理
        if (isProcessed(messageId)) {
            System.out.println("消息已处理，跳过: " + messageId);
            return;
        }
        
        try {
            // 处理业务逻辑
            processOrder(message);
            
            // 标记为已处理
            markAsProcessed(messageId);
            
        } catch (Exception e) {
            System.err.println("处理失败: " + e.getMessage());
            throw e;
        }
    }
    
    private void processOrder(OrderMessage message) {
        // 业务处理
    }
}
```

### 4.3 消息重试

```java
package com.example.config;

import org.springframework.amqp.rabbit.config.RetryInterceptorBuilder;
import org.springframework.amqp.rabbit.retry.RejectAndDontRequeueRecoverer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.retry.interceptor.RetryOperationsInterceptor;

@Configuration
public class RetryConfig {
    
    @Bean
    public RetryOperationsInterceptor retryInterceptor() {
        return RetryInterceptorBuilder.stateless()
            .maxAttempts(3)
            .backOffOptions(1000, 2.0, 10000)
            .recoverer(new RejectAndDontRequeueRecoverer())
            .build();
    }
}
```

## 5. 最佳实践

### 5.1 消息顺序性

```java
// Kafka 保证分区内有序
public void sendOrderedMessage(String orderId, OrderEvent event) {
    // 使用 orderId 作为 key，确保同一订单的消息发送到同一分区
    kafkaTemplate.send(KafkaConfig.ORDER_TOPIC, orderId, event);
}
```

### 5.2 消息去重

```java
@Component
public class DeduplicationConsumer {
    
    private final IdempotentConsumer idempotentConsumer;
    
    @RabbitListener(queues = RabbitMQConfig.ORDER_QUEUE)
    public void handleMessage(Message message, OrderMessage payload) {
        String messageId = message.getMessageProperties().getMessageId();
        
        if (messageId != null) {
            idempotentConsumer.handleMessage(messageId, payload);
        } else {
            System.err.println("消息缺少 messageId");
        }
    }
}
```

### 5.3 监控和告警

```java
package com.example.monitoring;

import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.stereotype.Component;

@Component
public class MessageMetrics {
    
    private final Counter messagesSent;
    private final Counter messagesFailed;
    
    public MessageMetrics(MeterRegistry registry) {
        this.messagesSent = Counter.builder("messages.sent")
            .description("发送的消息总数")
            .register(registry);
        
        this.messagesFailed = Counter.builder("messages.failed")
            .description("失败的消息总数")
            .register(registry);
    }
    
    public void recordSent() {
        messagesSent.increment();
    }
    
    public void recordFailed() {
        messagesFailed.increment();
    }
}
```

## 相关技能

- `/springboot-project-planner-cn` - 规划消息队列需求
- `/springboot-observability-cn` - 监控消息队列
- `/springboot-testing-cn` - 测试消息队列功能
