---
name: architecture
description: Spring 应用架构设计实践，包括 DDD、CQRS、模块化单体和分层架构
---

# Spring 应用架构设计实践指南

当你需要设计 Spring 应用架构时使用此技能，包括领域驱动设计（DDD）、CQRS 模式、模块化单体架构和分层架构。

## 何时使用

- 设计新项目的架构
- 选择合适的架构模式
- 实践领域驱动设计（DDD）
- 实现 CQRS 模式
- 构建模块化单体应用
- 设计分层架构
- 定义模块边界和依赖关系

## 1. 架构模式选择

### 1.1 架构模式对比

| 架构模式 | 适用场景 | 优点 | 缺点 |
|---------|---------|------|------|
| **分层架构** | 中小型应用 | 简单易懂 | 层间耦合 |
| **模块化单体** | 中大型应用 | 清晰边界，易于演进 | 需要严格约束 |
| **微服务** | 大型复杂应用 | 独立部署，技术灵活 | 运维复杂 |
| **DDD 分层** | 复杂业务领域 | 业务逻辑清晰 | 学习曲线陡 |
| **CQRS** | 读写分离场景 | 性能优化 | 复杂度高 |

### 1.2 选择建议

**小型项目（< 10 人团队）：**
- 分层架构 + 简单 DDD 概念
- 快速开发，简单维护

**中型项目（10-50 人团队）：**
- 模块化单体 + DDD
- 清晰边界，便于团队协作

**大型项目（> 50 人团队）：**
- 微服务 + DDD + CQRS
- 独立部署，高度自治

## 2. 领域驱动设计（DDD）

### 2.1 DDD 核心概念

**战略设计：**
- **限界上下文（Bounded Context）**：明确的业务边界
- **通用语言（Ubiquitous Language）**：团队统一的术语
- **上下文映射（Context Mapping）**：上下文间的关系

**战术设计：**
- **实体（Entity）**：有唯一标识的对象
- **值对象（Value Object）**：无标识，不可变
- **聚合（Aggregate）**：一组相关对象的集合
- **聚合根（Aggregate Root）**：聚合的入口
- **领域服务（Domain Service）**：跨实体的业务逻辑
- **领域事件（Domain Event）**：领域中发生的事情
- **仓储（Repository）**：聚合的持久化接口

### 2.2 实体（Entity）

**定义：**
- 有唯一标识
- 生命周期内可变
- 通过 ID 判断相等性

**示例：**

```java
@Entity
@Table(name = "orders")
public class Order {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String orderNumber;
    private OrderStatus status;
    private LocalDateTime createdAt;
    
    @Embedded
    private Money totalAmount;
    
    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();
    
    // 业务方法
    public void addItem(Product product, int quantity) {
        OrderItem item = new OrderItem(product, quantity);
        items.add(item);
        recalculateTotal();
    }
    
    public void confirm() {
        if (status != OrderStatus.PENDING) {
            throw new IllegalStateException("只能确认待处理的订单");
        }
        status = OrderStatus.CONFIRMED;
    }
    
    private void recalculateTotal() {
        totalAmount = items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(Money.ZERO, Money::add);
    }
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Order)) return false;
        Order order = (Order) o;
        return Objects.equals(id, order.id);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(id);
    }
}
```

### 2.3 值对象（Value Object）

**定义：**
- 无唯一标识
- 不可变
- 通过属性值判断相等性

**示例：**

```java
@Embeddable
public class Money {
    
    public static final Money ZERO = new Money(BigDecimal.ZERO, Currency.CNY);
    
    private BigDecimal amount;
    
    @Enumerated(EnumType.STRING)
    private Currency currency;
    
    protected Money() {} // JPA 需要
    
    public Money(BigDecimal amount, Currency currency) {
        if (amount == null || currency == null) {
            throw new IllegalArgumentException("金额和货币不能为空");
        }
        this.amount = amount;
        this.currency = currency;
    }
    
    public Money add(Money other) {
        if (!currency.equals(other.currency)) {
            throw new IllegalArgumentException("货币类型不匹配");
        }
        return new Money(amount.add(other.amount), currency);
    }
    
    public Money multiply(int multiplier) {
        return new Money(amount.multiply(BigDecimal.valueOf(multiplier)), currency);
    }
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Money)) return false;
        Money money = (Money) o;
        return amount.compareTo(money.amount) == 0 && currency == money.currency;
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(amount, currency);
    }
}

public enum Currency {
    CNY, USD, EUR
}
```

### 2.4 聚合（Aggregate）

**定义：**
- 一组相关对象的集合
- 有明确的边界
- 通过聚合根访问
- 保证业务不变性

**示例：**

```java
// 聚合根
@Entity
public class Order {
    
    @Id
    private Long id;
    
    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
    @JoinColumn(name = "order_id")
    private List<OrderItem> items = new ArrayList<>();
    
    // 通过聚合根操作聚合内的对象
    public void addItem(Product product, int quantity) {
        // 业务规则：检查库存
        if (!product.hasStock(quantity)) {
            throw new InsufficientStockException();
        }
        
        OrderItem item = new OrderItem(product.getId(), quantity, product.getPrice());
        items.add(item);
    }
    
    public void removeItem(Long itemId) {
        items.removeIf(item -> item.getId().equals(itemId));
    }
    
    // 聚合内的业务不变性
    public Money getTotalAmount() {
        return items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(Money.ZERO, Money::add);
    }
}

// 聚合内的实体
@Entity
public class OrderItem {
    
    @Id
    @GeneratedValue
    private Long id;
    
    private Long productId;
    private int quantity;
    
    @Embedded
    private Money price;
    
    // 不直接暴露给外部，通过聚合根访问
    protected OrderItem() {}
    
    OrderItem(Long productId, int quantity, Money price) {
        this.productId = productId;
        this.quantity = quantity;
        this.price = price;
    }
    
    Money getSubtotal() {
        return price.multiply(quantity);
    }
}
```

### 2.5 领域服务（Domain Service）

**何时使用：**
- 业务逻辑不属于任何实体或值对象
- 需要协调多个聚合
- 无状态的业务操作

**示例：**

```java
@Service
public class OrderPricingService {
    
    private final DiscountRepository discountRepository;
    
    public OrderPricingService(DiscountRepository discountRepository) {
        this.discountRepository = discountRepository;
    }
    
    // 跨聚合的业务逻辑
    public Money calculateFinalPrice(Order order, Customer customer) {
        Money basePrice = order.getTotalAmount();
        
        // 应用客户折扣
        Discount discount = discountRepository.findByCustomerLevel(customer.getLevel());
        Money discountedPrice = discount.apply(basePrice);
        
        // 应用促销规则
        if (order.getItemCount() >= 10) {
            discountedPrice = discountedPrice.multiply(0.95); // 95折
        }
        
        return discountedPrice;
    }
}
```

### 2.6 领域事件（Domain Event）

**定义：**
- 领域中发生的重要事情
- 不可变
- 过去时命名

**示例：**

```java
public class OrderConfirmedEvent {
    
    private final Long orderId;
    private final Long customerId;
    private final Money totalAmount;
    private final LocalDateTime confirmedAt;
    
    public OrderConfirmedEvent(Long orderId, Long customerId, Money totalAmount) {
        this.orderId = orderId;
        this.customerId = customerId;
        this.totalAmount = totalAmount;
        this.confirmedAt = LocalDateTime.now();
    }
    
    // Getters only (不可变)
}

// 在聚合中发布事件
@Entity
public class Order {
    
    @Transient
    private final List<Object> domainEvents = new ArrayList<>();
    
    public void confirm() {
        if (status != OrderStatus.PENDING) {
            throw new IllegalStateException("只能确认待处理的订单");
        }
        status = OrderStatus.CONFIRMED;
        
        // 发布领域事件
        domainEvents.add(new OrderConfirmedEvent(id, customerId, totalAmount));
    }
    
    public List<Object> getDomainEvents() {
        return Collections.unmodifiableList(domainEvents);
    }
    
    public void clearDomainEvents() {
        domainEvents.clear();
    }
}

// 事件监听器
@Component
public class OrderEventListener {
    
    @EventListener
    public void handleOrderConfirmed(OrderConfirmedEvent event) {
        // 发送确认邮件
        // 更新库存
        // 通知物流系统
    }
}
```

### 2.7 仓储（Repository）

**定义：**
- 聚合的持久化接口
- 只为聚合根提供仓储
- 封装数据访问细节

**示例：**

```java
public interface OrderRepository {
    
    Order findById(Long id);
    
    List<Order> findByCustomerId(Long customerId);
    
    void save(Order order);
    
    void delete(Order order);
}

// Spring Data JPA 实现
public interface OrderJpaRepository extends JpaRepository<Order, Long> {
    
    List<Order> findByCustomerId(Long customerId);
}

// 适配器模式
@Repository
public class OrderRepositoryAdapter implements OrderRepository {
    
    private final OrderJpaRepository jpaRepository;
    
    public OrderRepositoryAdapter(OrderJpaRepository jpaRepository) {
        this.jpaRepository = jpaRepository;
    }
    
    @Override
    public Order findById(Long id) {
        return jpaRepository.findById(id)
            .orElseThrow(() -> new OrderNotFoundException(id));
    }
    
    @Override
    public List<Order> findByCustomerId(Long customerId) {
        return jpaRepository.findByCustomerId(customerId);
    }
    
    @Override
    public void save(Order order) {
        jpaRepository.save(order);
    }
    
    @Override
    public void delete(Order order) {
        jpaRepository.delete(order);
    }
}
```

## 3. 分层架构

### 3.1 经典四层架构

```
┌─────────────────────────────────┐
│     接口层 (Interface Layer)     │  ← REST API, GraphQL, gRPC
├─────────────────────────────────┤
│     应用层 (Application Layer)   │  ← 用例、应用服务、DTO
├─────────────────────────────────┤
│     领域层 (Domain Layer)        │  ← 实体、值对象、领域服务
├─────────────────────────────────┤
│  基础设施层 (Infrastructure)     │  ← 数据库、消息队列、外部服务
└─────────────────────────────────┘
```

**依赖方向：** 接口层 → 应用层 → 领域层 ← 基础设施层

### 3.2 接口层（Interface Layer）

**职责：**
- 接收外部请求
- 参数验证
- 调用应用服务
- 返回响应

**示例：**

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    private final OrderApplicationService orderService;
    
    public OrderController(OrderApplicationService orderService) {
        this.orderService = orderService;
    }
    
    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(@Valid @RequestBody CreateOrderRequest request) {
        OrderResponse response = orderService.createOrder(request);
        return ResponseEntity.ok(response);
    }
    
    @PostMapping("/{id}/confirm")
    public ResponseEntity<Void> confirmOrder(@PathVariable Long id) {
        orderService.confirmOrder(id);
        return ResponseEntity.ok().build();
    }
}
```

### 3.3 应用层（Application Layer）

**职责：**
- 编排业务流程
- 事务管理
- DTO 转换
- 发布领域事件

**示例：**

```java
@Service
@Transactional
public class OrderApplicationService {
    
    private final OrderRepository orderRepository;
    private final ProductRepository productRepository;
    private final ApplicationEventPublisher eventPublisher;
    
    public OrderApplicationService(
            OrderRepository orderRepository,
            ProductRepository productRepository,
            ApplicationEventPublisher eventPublisher) {
        this.orderRepository = orderRepository;
        this.productRepository = productRepository;
        this.eventPublisher = eventPublisher;
    }
    
    public OrderResponse createOrder(CreateOrderRequest request) {
        // 1. 加载聚合
        Product product = productRepository.findById(request.getProductId());
        
        // 2. 执行业务逻辑（委托给领域层）
        Order order = new Order(request.getCustomerId());
        order.addItem(product, request.getQuantity());
        
        // 3. 持久化
        orderRepository.save(order);
        
        // 4. 发布事件
        order.getDomainEvents().forEach(eventPublisher::publishEvent);
        order.clearDomainEvents();
        
        // 5. 返回 DTO
        return OrderResponse.from(order);
    }
    
    public void confirmOrder(Long orderId) {
        Order order = orderRepository.findById(orderId);
        order.confirm();
        orderRepository.save(order);
        
        order.getDomainEvents().forEach(eventPublisher::publishEvent);
        order.clearDomainEvents();
    }
}
```

### 3.4 领域层（Domain Layer）

**职责：**
- 核心业务逻辑
- 业务规则验证
- 领域模型

**示例：** 见上面的实体、值对象、聚合示例

### 3.5 基础设施层（Infrastructure Layer）

**职责：**
- 数据持久化
- 外部服务调用
- 消息队列
- 缓存

**示例：**

```java
@Repository
public class OrderRepositoryImpl implements OrderRepository {
    
    private final OrderJpaRepository jpaRepository;
    
    @Override
    public Order findById(Long id) {
        return jpaRepository.findById(id)
            .orElseThrow(() -> new OrderNotFoundException(id));
    }
    
    @Override
    public void save(Order order) {
        jpaRepository.save(order);
    }
}
```

## 4. CQRS 模式

### 4.1 什么是 CQRS

**CQRS（Command Query Responsibility Segregation）：** 命令查询职责分离

- **命令（Command）**：修改状态，不返回数据
- **查询（Query）**：返回数据，不修改状态

### 4.2 为什么使用 CQRS

**优点：**
- 读写分离，性能优化
- 读模型可以针对查询优化
- 写模型专注业务逻辑
- 可以使用不同的数据存储

**缺点：**
- 复杂度增加
- 数据最终一致性
- 需要同步机制

### 4.3 实现 CQRS

**命令端（写模型）：**

```java
// 命令
public class CreateOrderCommand {
    private final Long customerId;
    private final Long productId;
    private final int quantity;
    
    // Constructor, Getters
}

// 命令处理器
@Service
@Transactional
public class CreateOrderCommandHandler {
    
    private final OrderRepository orderRepository;
    private final ApplicationEventPublisher eventPublisher;
    
    public void handle(CreateOrderCommand command) {
        Order order = new Order(command.getCustomerId());
        order.addItem(command.getProductId(), command.getQuantity());
        
        orderRepository.save(order);
        
        // 发布事件用于同步读模型
        eventPublisher.publishEvent(new OrderCreatedEvent(order.getId(), order));
    }
}
```

**查询端（读模型）：**

```java
// 查询 DTO
public class OrderQueryDto {
    private Long id;
    private String orderNumber;
    private String customerName;
    private BigDecimal totalAmount;
    private String status;
    
    // Getters, Setters
}

// 查询服务
@Service
@Transactional(readOnly = true)
public class OrderQueryService {
    
    private final JdbcTemplate jdbcTemplate;
    
    public OrderQueryDto findById(Long id) {
        String sql = """
            SELECT o.id, o.order_number, c.name as customer_name,
                   o.total_amount, o.status
            FROM orders o
            JOIN customers c ON o.customer_id = c.id
            WHERE o.id = ?
            """;
        
        return jdbcTemplate.queryForObject(sql, 
            (rs, rowNum) -> new OrderQueryDto(
                rs.getLong("id"),
                rs.getString("order_number"),
                rs.getString("customer_name"),
                rs.getBigDecimal("total_amount"),
                rs.getString("status")
            ), id);
    }
    
    public List<OrderQueryDto> findByCustomerId(Long customerId) {
        // 优化的查询逻辑
    }
}
```

**事件同步：**

```java
@Component
public class OrderReadModelUpdater {
    
    @EventListener
    @Async
    public void handleOrderCreated(OrderCreatedEvent event) {
        // 更新读模型（可以是不同的数据库）
        updateReadModel(event);
    }
    
    private void updateReadModel(OrderCreatedEvent event) {
        // 同步到 Elasticsearch、MongoDB 等
    }
}
```

## 5. 模块化单体架构（Spring Modulith）

### 5.1 什么是 Spring Modulith

Spring Modulith 帮助构建模块化的单体应用：
- 明确的模块边界
- 模块间通过事件通信
- 架构验证
- 自动生成文档

**相关技能：** `/spring-modulith-cn`

### 5.2 模块划分

```
src/main/java/com/example/
├── order/              # 订单模块
│   ├── Order.java
│   ├── OrderService.java
│   └── OrderRepository.java
├── customer/           # 客户模块
│   ├── Customer.java
│   └── CustomerService.java
├── product/            # 商品模块
│   ├── Product.java
│   └── ProductService.java
└── payment/            # 支付模块
    ├── Payment.java
    └── PaymentService.java
```

### 5.3 模块间通信

**通过事件：**

```java
// 订单模块发布事件
@Service
public class OrderService {
    
    private final ApplicationEventPublisher events;
    
    public void confirmOrder(Long orderId) {
        Order order = orderRepository.findById(orderId);
        order.confirm();
        
        events.publishEvent(new OrderConfirmedEvent(orderId));
    }
}

// 支付模块监听事件
@Service
public class PaymentService {
    
    @ApplicationModuleListener
    void handleOrderConfirmed(OrderConfirmedEvent event) {
        // 创建支付记录
        createPayment(event.getOrderId());
    }
}
```

## 6. 最佳实践

1. **从简单开始**：不要过度设计，根据需要逐步演进
2. **明确边界**：清晰的模块/聚合边界
3. **通用语言**：团队统一的术语
4. **小聚合**：聚合尽量小，减少锁竞争
5. **事件驱动**：模块间通过事件解耦
6. **测试驱动**：先写测试，再写实现
7. **持续重构**：随着理解加深，不断优化模型
8. **文档化**：记录架构决策和设计原理

## 参考资源

- [Domain-Driven Design (Eric Evans)](https://www.domainlanguage.com/ddd/)
- [Implementing Domain-Driven Design (Vaughn Vernon)](https://vaughnvernon.com/)
- [Spring Modulith](https://spring.io/projects/spring-modulith)
- [CQRS Pattern](https://martinfowler.com/bliki/CQRS.html)
