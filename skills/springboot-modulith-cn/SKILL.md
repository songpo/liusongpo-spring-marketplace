---
name: springboot-modulith-cn
description: 使用 Spring Modulith 构建模块化单体应用
---

# Spring Modulith 模块化单体架构指南

当你需要构建模块化单体应用（Modular Monolith）时使用此技能。Spring Modulith 帮助你在单体应用中实现清晰的模块边界，为未来可能的微服务拆分做准备。

## 何时使用

- 构建新的单体应用，但希望保持良好的模块化
- 避免过早的微服务化
- 需要强制模块边界和依赖规则
- 希望使用事件驱动的模块间通信
- 需要自动化的架构验证和文档生成
- 为未来可能的微服务拆分做准备

## Spring Modulith 核心概念

### 1. 什么是模块化单体？

模块化单体是一种架构风格，在单个部署单元中组织代码为松耦合的模块。

**优势：**
- 简单的部署和运维
- 避免分布式系统的复杂性
- 保持模块边界清晰
- 易于重构和演进
- 可以逐步拆分为微服务

**Spring Modulith 提供：**
- 基于包结构的模块定义
- 模块边界验证
- 事件驱动的模块通信
- 自动化文档生成
- 可观测性支持

### 2. 项目结构

```
order-service/
├── pom.xml
└── src/main/java/com/example/orderservice/
    ├── OrderServiceApplication.java
    ├── order/                       # 订单模块（应用模块）
    │   ├── Order.java
    │   ├── OrderService.java
    │   ├── OrderController.java
    │   ├── OrderRepository.java
    │   └── internal/                # 内部实现（不对外暴露）
    │       ├── OrderEntity.java
    │       └── OrderMapper.java
    ├── inventory/                   # 库存模块
    │   ├── InventoryService.java
    │   ├── Product.java
    │   └── internal/
    │       ├── InventoryRepository.java
    │       └── ProductEntity.java
    ├── customer/                    # 客户模块
    │   ├── Customer.java
    │   ├── CustomerService.java
    │   └── internal/
    │       └── CustomerRepository.java
    └── notification/                # 通知模块
        ├── NotificationService.java
        └── internal/
            └── EmailSender.java
```

**模块规则：**
- 顶层包（如 `order`、`inventory`）是应用模块
- `internal` 包中的类只能被同模块访问
- 模块间只能通过公开的 API 通信

### 3. Maven 依赖

```xml
<dependencies>
    <!-- Spring Boot -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    
    <!-- Spring Modulith -->
    <dependency>
        <groupId>org.springframework.modulith</groupId>
        <artifactId>spring-modulith-starter-core</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.modulith</groupId>
        <artifactId>spring-modulith-starter-jpa</artifactId>
    </dependency>
    
    <!-- 测试支持 -->
    <dependency>
        <groupId>org.springframework.modulith</groupId>
        <artifactId>spring-modulith-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    
    <!-- 文档生成（可选） -->
    <dependency>
        <groupId>org.springframework.modulith</groupId>
        <artifactId>spring-modulith-docs</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.modulith</groupId>
            <artifactId>spring-modulith-bom</artifactId>
            <version>1.1.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

## 模块定义和边界

### 1. 应用模块（Application Module）

默认情况下，主包下的直接子包被视为应用模块。

```java
// com.example.orderservice.order - 订单模块
package com.example.orderservice.order;

// 公开 API - 可以被其他模块访问
public interface OrderManagement {
    OrderId createOrder(CreateOrderCommand command);
    void cancelOrder(OrderId orderId);
}

@Service
public class OrderService implements OrderManagement {
    private final OrderRepository repository;
    private final ApplicationEventPublisher events;
    
    @Override
    @Transactional
    public OrderId createOrder(CreateOrderCommand command) {
        Order order = Order.create(command);
        repository.save(order);
        
        // 发布领域事件
        events.publishEvent(new OrderCreatedEvent(order.getId(), order.getCustomerId()));
        
        return order.getId();
    }
    
    @Override
    @Transactional
    public void cancelOrder(OrderId orderId) {
        Order order = repository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
        
        order.cancel();
        repository.save(order);
        
        events.publishEvent(new OrderCancelledEvent(orderId));
    }
}

// 领域模型 - 公开
public class Order {
    private OrderId id;
    private CustomerId customerId;
    private OrderStatus status;
    
    public static Order create(CreateOrderCommand command) {
        // 创建逻辑
    }
    
    public void cancel() {
        if (status == OrderStatus.SHIPPED) {
            throw new IllegalStateException("已发货订单无法取消");
        }
        this.status = OrderStatus.CANCELLED;
    }
}

// 命令对象 - 公开
public record CreateOrderCommand(
    CustomerId customerId,
    List<OrderItem> items
) {}

// 事件 - 公开（其他模块可以监听）
public record OrderCreatedEvent(OrderId orderId, CustomerId customerId) {}
public record OrderCancelledEvent(OrderId orderId) {}
```

### 2. 内部实现（Internal）

`internal` 包中的类不能被其他模块访问。

```java
// com.example.orderservice.order.internal - 内部实现
package com.example.orderservice.order.internal;

// 仓储接口 - 内部
interface OrderRepository extends JpaRepository<OrderEntity, String> {
    // 内部使用
}

// JPA 实体 - 内部
@Entity
@Table(name = "orders")
class OrderEntity {
    @Id
    private String id;
    private String customerId;
    private String status;
    // ...
}

// 映射器 - 内部
@Component
class OrderMapper {
    OrderEntity toEntity(Order order) {
        // 映射逻辑
    }
    
    Order toDomain(OrderEntity entity) {
        // 映射逻辑
    }
}
```

### 3. 显式模块定义

使用 `package-info.java` 显式定义模块：

```java
// com.example.orderservice.order.package-info.java
@org.springframework.modulith.ApplicationModule(
    displayName = "Order Management",
    allowedDependencies = {"customer", "inventory"}
)
package com.example.orderservice.order;
```

**配置选项：**
- `displayName`: 模块显示名称
- `allowedDependencies`: 允许依赖的其他模块
- `type`: 模块类型（OPEN 或 CLOSED）

## 模块间通信

### 1. 事件驱动通信（推荐）

模块间通过事件进行异步通信，保持松耦合。

**发布事件：**
```java
// 订单模块
package com.example.orderservice.order;

@Service
public class OrderService {
    private final ApplicationEventPublisher events;
    
    @Transactional
    public OrderId createOrder(CreateOrderCommand command) {
        Order order = Order.create(command);
        repository.save(order);
        
        // 发布事件
        events.publishEvent(new OrderCreatedEvent(
            order.getId(),
            order.getCustomerId(),
            order.getTotalAmount()
        ));
        
        return order.getId();
    }
}

// 事件定义（公开 API）
public record OrderCreatedEvent(
    OrderId orderId,
    CustomerId customerId,
    Money totalAmount
) {
    public Instant occurredAt() {
        return Instant.now();
    }
}
```

**监听事件：**
```java
// 库存模块
package com.example.orderservice.inventory;

@Service
class InventoryEventListener {
    private final InventoryService inventoryService;
    
    @ApplicationModuleListener
    void on(OrderCreatedEvent event) {
        // 扣减库存
        inventoryService.reserveStock(event.orderId());
    }
    
    @ApplicationModuleListener
    void on(OrderCancelledEvent event) {
        // 释放库存
        inventoryService.releaseStock(event.orderId());
    }
}

// 通知模块
package com.example.orderservice.notification;

@Service
class NotificationEventListener {
    private final NotificationService notificationService;
    
    @ApplicationModuleListener
    void on(OrderCreatedEvent event) {
        // 发送订单确认邮件
        notificationService.sendOrderConfirmation(event.customerId(), event.orderId());
    }
}
```

**@ApplicationModuleListener 特性：**
- 自动事务管理
- 异步执行（可配置）
- 事件持久化（可选）
- 失败重试

### 2. 直接依赖（谨慎使用）

如果必须直接调用其他模块，只能使用公开的 API。

```java
// 订单模块依赖客户模块
package com.example.orderservice.order;

@Service
public class OrderService {
    private final CustomerManagement customerManagement; // 客户模块的公开接口
    
    public OrderId createOrder(CreateOrderCommand command) {
        // 验证客户
        Customer customer = customerManagement.getCustomer(command.customerId());
        if (!customer.isActive()) {
            throw new InactiveCustomerException();
        }
        
        // 创建订单
        Order order = Order.create(command);
        repository.save(order);
        
        return order.getId();
    }
}
```

**注意：** 直接依赖会增加耦合，优先使用事件驱动通信。

## 架构验证

### 1. 自动化测试

Spring Modulith 提供测试支持，验证模块边界。

```java
package com.example.orderservice;

import org.junit.jupiter.api.Test;
import org.springframework.modulith.core.ApplicationModules;
import org.springframework.modulith.docs.Documenter;

class ModularityTests {
    
    ApplicationModules modules = ApplicationModules.of(OrderServiceApplication.class);
    
    @Test
    void verifiesModularStructure() {
        // 验证模块结构是否符合规则
        modules.verify();
    }
    
    @Test
    void createModuleDocumentation() {
        // 生成模块文档
        new Documenter(modules)
            .writeDocumentation()
            .writeIndividualModulesAsPlantUml();
    }
    
    @Test
    void verifyNoCyclicDependencies() {
        // 验证没有循环依赖
        modules.verify();
    }
}
```

### 2. 模块测试

测试单个模块，只加载该模块及其依赖。

```java
package com.example.orderservice.order;

import org.junit.jupiter.api.Test;
import org.springframework.modulith.test.ApplicationModuleTest;
import org.springframework.beans.factory.annotation.Autowired;

@ApplicationModuleTest
class OrderModuleTests {
    
    @Autowired
    OrderService orderService;
    
    @Test
    void shouldCreateOrder() {
        // 只加载 order 模块进行测试
        CreateOrderCommand command = new CreateOrderCommand(
            new CustomerId("C001"),
            List.of(new OrderItem("P001", 2))
        );
        
        OrderId orderId = orderService.createOrder(command);
        
        assertThat(orderId).isNotNull();
    }
}
```

### 3. 场景测试

测试跨模块的业务场景。

```java
@ApplicationModuleTest
class OrderScenarioTests {
    
    @Autowired
    OrderService orderService;
    
    @Test
    void shouldPublishEventWhenOrderCreated(Scenario scenario) {
        CreateOrderCommand command = new CreateOrderCommand(
            new CustomerId("C001"),
            List.of(new OrderItem("P001", 2))
        );
        
        scenario.stimulate(() -> orderService.createOrder(command))
            .andWaitForEventOfType(OrderCreatedEvent.class)
            .matching(event -> event.customerId().equals(new CustomerId("C001")))
            .toArriveAndVerify((event, context) -> {
                // 验证事件被正确处理
                assertThat(event.orderId()).isNotNull();
            });
    }
}
```

## 事件持久化

Spring Modulith 支持事件持久化，确保事件不丢失。

### 1. 配置事件持久化

```java
@Configuration
@EnableJpaEventPublication
public class EventPublicationConfiguration {
    // 自动配置事件发布表
}
```

### 2. 事件表

Spring Modulith 自动创建事件发布表：

```sql
CREATE TABLE event_publication (
    id UUID PRIMARY KEY,
    event_type VARCHAR(255) NOT NULL,
    listener_id VARCHAR(255) NOT NULL,
    publication_date TIMESTAMP NOT NULL,
    serialized_event TEXT NOT NULL,
    completion_date TIMESTAMP
);
```

### 3. 事件重放

```java
@Service
public class EventReplayService {
    private final EventPublicationRegistry registry;
    
    public void replayFailedEvents() {
        // 重新发布失败的事件
        registry.findIncompletePublications()
            .forEach(publication -> {
                // 重试逻辑
            });
    }
}
```

## 可观测性

### 1. 模块依赖可视化

```java
@Test
void createModuleDiagram() {
    ApplicationModules modules = ApplicationModules.of(OrderServiceApplication.class);
    
    new Documenter(modules)
        .writeModulesAsPlantUml()
        .writeIndividualModulesAsPlantUml();
}
```

生成的 PlantUML 图展示模块依赖关系。

### 2. 运行时监控

```java
@Configuration
public class ObservabilityConfiguration {
    
    @Bean
    ApplicationModuleInitializer moduleInitializer(ApplicationModules modules) {
        return new ApplicationModuleInitializer(modules, 
            (module, context) -> {
                log.info("Initializing module: {}", module.getName());
            }
        );
    }
}
```

## 完整示例：电商系统

### 模块划分

```
order-service/
├── order/              # 订单模块
├── inventory/          # 库存模块
├── customer/           # 客户模块
├── payment/            # 支付模块
└── notification/       # 通知模块
```

### 订单模块

```java
package com.example.orderservice.order;

// 公开 API
public interface OrderManagement {
    OrderId createOrder(CreateOrderCommand command);
    void cancelOrder(OrderId orderId);
    Order getOrder(OrderId orderId);
}

// 服务实现
@Service
@Transactional
class OrderService implements OrderManagement {
    private final OrderRepository repository;
    private final ApplicationEventPublisher events;
    
    @Override
    public OrderId createOrder(CreateOrderCommand command) {
        Order order = Order.create(command);
        repository.save(order);
        
        events.publishEvent(new OrderCreatedEvent(
            order.getId(),
            order.getCustomerId(),
            order.getItems()
        ));
        
        return order.getId();
    }
    
    @Override
    public void cancelOrder(OrderId orderId) {
        Order order = repository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
        
        order.cancel();
        repository.save(order);
        
        events.publishEvent(new OrderCancelledEvent(orderId));
    }
    
    @Override
    @Transactional(readOnly = true)
    public Order getOrder(OrderId orderId) {
        return repository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
    }
}

// 领域模型
public class Order {
    private OrderId id;
    private CustomerId customerId;
    private List<OrderItem> items;
    private OrderStatus status;
    
    public static Order create(CreateOrderCommand command) {
        Order order = new Order();
        order.id = OrderId.generate();
        order.customerId = command.customerId();
        order.items = new ArrayList<>(command.items());
        order.status = OrderStatus.PENDING;
        return order;
    }
    
    public void cancel() {
        if (status == OrderStatus.SHIPPED) {
            throw new IllegalStateException("已发货订单无法取消");
        }
        this.status = OrderStatus.CANCELLED;
    }
}

// 事件
public record OrderCreatedEvent(
    OrderId orderId,
    CustomerId customerId,
    List<OrderItem> items
) {}

public record OrderCancelledEvent(OrderId orderId) {}
```

### 库存模块

```java
package com.example.orderservice.inventory;

// 事件监听器
@Service
class InventoryEventListener {
    private final InventoryService inventoryService;
    
    @ApplicationModuleListener
    @Transactional
    void on(OrderCreatedEvent event) {
        // 扣减库存
        for (OrderItem item : event.items()) {
            inventoryService.reserveStock(item.productId(), item.quantity());
        }
    }
    
    @ApplicationModuleListener
    @Transactional
    void on(OrderCancelledEvent event) {
        // 释放库存
        inventoryService.releaseStock(event.orderId());
    }
}

// 库存服务
@Service
class InventoryService {
    private final ProductRepository repository;
    
    void reserveStock(ProductId productId, int quantity) {
        Product product = repository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));
        
        product.reserve(quantity);
        repository.save(product);
    }
    
    void releaseStock(OrderId orderId) {
        // 释放逻辑
    }
}
```

### 支付模块

```java
package com.example.orderservice.payment;

@Service
class PaymentEventListener {
    private final PaymentService paymentService;
    
    @ApplicationModuleListener
    @Transactional
    void on(OrderCreatedEvent event) {
        // 创建支付订单
        paymentService.createPayment(event.orderId(), event.totalAmount());
    }
}
```

### 通知模块

```java
package com.example.orderservice.notification;

@Service
class NotificationEventListener {
    private final EmailService emailService;
    
    @ApplicationModuleListener
    @Async
    void on(OrderCreatedEvent event) {
        // 发送订单确认邮件
        emailService.sendOrderConfirmation(event.customerId(), event.orderId());
    }
    
    @ApplicationModuleListener
    @Async
    void on(OrderCancelledEvent event) {
        // 发送取消通知
        emailService.sendCancellationNotice(event.orderId());
    }
}
```

## 最佳实践

### 1. 模块设计原则

**高内聚：**
- 相关功能放在同一模块
- 模块内部紧密协作

**低耦合：**
- 模块间通过事件通信
- 避免直接依赖
- 公开最小化的 API

### 2. 事件设计

**事件命名：**
- 使用过去时（OrderCreated, PaymentCompleted）
- 描述已发生的事实

**事件内容：**
- 包含足够的信息供监听者使用
- 避免包含敏感信息
- 使用不可变对象（record）

### 3. 模块边界

**什么应该公开：**
- 接口定义
- 命令和查询对象
- 领域事件
- DTO

**什么应该隐藏：**
- 实现细节
- 数据库实体
- 仓储实现
- 内部服务

### 4. 渐进式演进

**阶段 1：** 单体应用，无模块划分
```
src/main/java/com/example/app/
├── controller/
├── service/
├── repository/
└── entity/
```

**阶段 2：** 按功能划分包
```
src/main/java/com/example/app/
├── order/
├── inventory/
└── customer/
```

**阶段 3：** 引入 Spring Modulith
```
src/main/java/com/example/app/
├── order/
│   ├── OrderService.java
│   └── internal/
├── inventory/
│   ├── InventoryService.java
│   └── internal/
└── customer/
    ├── CustomerService.java
    └── internal/
```

**阶段 4：** 拆分为微服务（如需要）

### 5. 何时拆分为微服务

**保持单体的信号：**
- 团队规模 < 10 人
- 部署频率 < 每周一次
- 模块间耦合度低
- 性能满足需求

**考虑拆分的信号：**
- 不同模块需要独立扩展
- 不同模块使用不同技术栈
- 团队按模块划分
- 需要独立部署

## 常见问题

### Q: Spring Modulith vs 微服务？

**A:** Spring Modulith 适合：
- 早期项目
- 中小型团队
- 不确定是否需要微服务
- 希望保持简单

微服务适合：
- 大型团队
- 需要独立扩展
- 不同技术栈
- 成熟的 DevOps 能力

### Q: 如何处理跨模块事务？

**A:** 使用 Saga 模式或最终一致性：
```java
@ApplicationModuleListener
@Transactional
void on(OrderCreatedEvent event) {
    try {
        inventoryService.reserveStock(event.items());
    } catch (InsufficientStockException e) {
        // 发布补偿事件
        events.publishEvent(new StockReservationFailedEvent(event.orderId()));
    }
}
```

### Q: 模块可以共享数据库表吗？

**A:** 不推荐。每个模块应该拥有自己的表，通过事件同步数据。

## 相关技能

- `/springboot-ddd-architecture-cn` - DDD 分层架构
- `/springboot-ddd-cn` - DDD 核心概念
- `/springboot-ddd-cqrs-cn` - CQRS 模式
- `/springboot-testing-cn` - 模块化应用的测试策略
