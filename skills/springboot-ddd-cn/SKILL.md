---
name: springboot-ddd-cn
description: 在 Spring Boot 应用中实践领域驱动设计（DDD）时使用
---

# Spring Boot DDD 实践指南

当你需要在 Spring Boot 应用中实践领域驱动设计（DDD）时使用此技能。

## 何时使用

- 设计聚合（Aggregate）、实体（Entity）、值对象（Value Object）
- 实现领域服务（Domain Service）
- 将领域逻辑与基础设施分离
- 设计仓储（Repository）接口

## DDD 核心概念

### 1. 聚合（Aggregate）

聚合是一组相关对象的集合，作为数据修改的单元。

**原则：**
- 每个聚合有一个聚合根（Aggregate Root）
- 外部只能通过聚合根访问聚合内的对象
- 聚合边界内保证事务一致性

**示例：**
```java
@Entity
public class Order {  // 聚合根
    @Id
    private OrderId id;
    
    @Embedded
    private CustomerId customerId;
    
    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();
    
    private OrderStatus status;
    
    // 业务方法
    public void addItem(Product product, int quantity) {
        // 领域逻辑
        if (status != OrderStatus.DRAFT) {
            throw new IllegalStateException("只能向草稿订单添加商品");
        }
        items.add(new OrderItem(product, quantity));
    }
    
    public void submit() {
        if (items.isEmpty()) {
            throw new IllegalStateException("订单不能为空");
        }
        this.status = OrderStatus.SUBMITTED;
    }
}
```

### 2. 实体（Entity）

实体有唯一标识，生命周期中标识不变。

**原则：**
- 使用值对象作为 ID
- 实体相等性基于 ID
- 封装业务逻辑

**示例：**
```java
@Embeddable
public class OrderId {
    private String value;
    
    // 私有构造函数 + 工厂方法
    private OrderId(String value) {
        this.value = Objects.requireNonNull(value);
    }
    
    public static OrderId of(String value) {
        return new OrderId(value);
    }
    
    public static OrderId generate() {
        return new OrderId(UUID.randomUUID().toString());
    }
}
```

### 3. 值对象（Value Object）

值对象没有唯一标识，通过属性值判断相等性。

**原则：**
- 不可变（Immutable）
- 相等性基于属性值
- 包含验证逻辑

**示例：**
```java
@Embeddable
public class Money {
    private BigDecimal amount;
    private String currency;
    
    protected Money() {} // JPA 需要
    
    private Money(BigDecimal amount, String currency) {
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("金额不能为负数");
        }
        this.amount = amount;
        this.currency = Objects.requireNonNull(currency);
    }
    
    public static Money of(BigDecimal amount, String currency) {
        return new Money(amount, currency);
    }
    
    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("货币类型不匹配");
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }
}
```

### 4. 领域服务（Domain Service）

当业务逻辑不属于任何单一实体或值对象时，使用领域服务。

**原则：**
- 无状态
- 操作涉及多个聚合
- 命名体现业务意图

**示例：**
```java
@Service
public class OrderPricingService {
    
    public Money calculateTotal(Order order, List<Discount> discounts) {
        Money subtotal = order.getItems().stream()
            .map(OrderItem::getPrice)
            .reduce(Money.zero(), Money::add);
            
        Money discountAmount = discounts.stream()
            .map(d -> d.calculate(subtotal))
            .reduce(Money.zero(), Money::add);
            
        return subtotal.subtract(discountAmount);
    }
}
```

### 5. 仓储（Repository）

仓储提供聚合的持久化和检索。

**原则：**
- 接口定义在领域层
- 实现在基础设施层
- 只为聚合根提供仓储
- 使用领域对象作为参数和返回值

**示例：**
```java
// 领域层接口
public interface OrderRepository {
    Order findById(OrderId id);
    List<Order> findByCustomerId(CustomerId customerId);
    void save(Order order);
    void delete(Order order);
}

// 基础设施层实现
@Repository
class JpaOrderRepository implements OrderRepository {
    
    @PersistenceContext
    private EntityManager entityManager;
    
    @Override
    public Order findById(OrderId id) {
        return entityManager.find(Order.class, id);
    }
    
    @Override
    public void save(Order order) {
        if (order.getId() == null) {
            entityManager.persist(order);
        } else {
            entityManager.merge(order);
        }
    }
}
```

## 分层架构

```
├── domain/              # 领域层
│   ├── model/          # 聚合、实体、值对象
│   ├── service/        # 领域服务
│   └── repository/     # 仓储接口
├── application/        # 应用层
│   └── service/        # 应用服务（用例）
├── infrastructure/     # 基础设施层
│   ├── persistence/    # 仓储实现
│   └── config/         # 配置
└── interfaces/         # 接口层
    └── rest/           # REST 控制器
```

## 最佳实践

1. **聚合设计要小**：聚合越小，并发冲突越少
2. **通过 ID 引用其他聚合**：不要直接持有其他聚合的引用
3. **在边界内保证一致性**：聚合内强一致性，聚合间最终一致性
4. **使用领域事件**：聚合间通过事件通信
5. **富领域模型**：业务逻辑放在领域对象中，不要贫血模型

## 示例：完整的订单聚合

```java
@Entity
@Table(name = "orders")
public class Order {
    
    @EmbeddedId
    private OrderId id;
    
    @Embedded
    @AttributeOverride(name = "value", column = @Column(name = "customer_id"))
    private CustomerId customerId;
    
    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
    @JoinColumn(name = "order_id")
    private List<OrderItem> items = new ArrayList<>();
    
    @Enumerated(EnumType.STRING)
    private OrderStatus status;
    
    @Embedded
    private Money total;
    
    @Column(name = "created_at")
    private Instant createdAt;
    
    protected Order() {} // JPA
    
    public Order(CustomerId customerId) {
        this.id = OrderId.generate();
        this.customerId = customerId;
        this.status = OrderStatus.DRAFT;
        this.total = Money.zero();
        this.createdAt = Instant.now();
    }
    
    public void addItem(ProductId productId, Money price, int quantity) {
        if (status != OrderStatus.DRAFT) {
            throw new OrderNotEditableException();
        }
        
        OrderItem item = new OrderItem(productId, price, quantity);
        items.add(item);
        recalculateTotal();
    }
    
    public void submit() {
        if (items.isEmpty()) {
            throw new EmptyOrderException();
        }
        if (status != OrderStatus.DRAFT) {
            throw new OrderAlreadySubmittedException();
        }
        
        this.status = OrderStatus.SUBMITTED;
        // 发布领域事件
        DomainEvents.raise(new OrderSubmittedEvent(this.id));
    }
    
    private void recalculateTotal() {
        this.total = items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(Money.zero(), Money::add);
    }
    
    // Getters
    public OrderId getId() { return id; }
    public OrderStatus getStatus() { return status; }
    public Money getTotal() { return total; }
}
```

## 注意事项

- JPA 实体需要无参构造函数，使用 `protected` 访问级别
- 值对象使用 `@Embeddable` 和 `@Embedded`
- 聚合根使用 `@Entity`
- 仓储接口放在领域层，实现放在基础设施层
- 使用领域事件实现聚合间的解耦
