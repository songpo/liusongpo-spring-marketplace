---
name: springboot-ddd-architecture-cn
description: 从零开始创建完整的 Spring Boot DDD 架构实现
---

# Spring Boot DDD 架构完整指南

当你需要从零开始创建一个完整的 DDD（领域驱动设计）架构的 Spring Boot 应用时使用此技能。

## 何时使用

- 启动新的 Spring Boot 项目，采用 DDD 架构
- 重构现有项目为 DDD 架构
- 需要清晰的分层和模块划分
- 构建复杂业务逻辑的应用
- 需要长期维护和演进的系统

## 完整项目结构

### Maven 多模块结构（推荐）

```
order-service/
├── pom.xml                          # 父 POM
├── order-domain/                    # 领域模块
│   ├── pom.xml
│   └── src/main/java/com/example/order/domain/
│       ├── model/                   # 领域模型
│       │   ├── Order.java          # 聚合根
│       │   ├── OrderItem.java      # 实体
│       │   ├── OrderId.java        # 值对象
│       │   ├── CustomerId.java
│       │   ├── Money.java
│       │   └── OrderStatus.java
│       ├── service/                 # 领域服务
│       │   ├── OrderPricingService.java
│       │   └── OrderValidationService.java
│       ├── repository/              # 仓储接口
│       │   └── OrderRepository.java
│       └── event/                   # 领域事件
│           ├── OrderCreatedEvent.java
│           ├── OrderCancelledEvent.java
│           └── DomainEvent.java
├── order-application/               # 应用模块
│   ├── pom.xml
│   └── src/main/java/com/example/order/application/
│       ├── command/                 # 命令
│       │   ├── CreateOrderCommand.java
│       │   ├── CancelOrderCommand.java
│       │   └── UpdateOrderCommand.java
│       ├── query/                   # 查询
│       │   ├── GetOrderQuery.java
│       │   └── ListOrdersQuery.java
│       ├── service/                 # 应用服务
│       │   ├── OrderCommandService.java
│       │   └── OrderQueryService.java
│       └── dto/                     # 数据传输对象
│           ├── OrderDto.java
│           ├── CreateOrderRequest.java
│           └── OrderResponse.java
├── order-infrastructure/            # 基础设施模块
│   ├── pom.xml
│   └── src/main/java/com/example/order/infrastructure/
│       ├── persistence/             # 持久化
│       │   ├── OrderJpaRepository.java
│       │   ├── OrderRepositoryImpl.java
│       │   └── entity/
│       │       ├── OrderEntity.java
│       │       └── OrderItemEntity.java
│       ├── messaging/               # 消息
│       │   ├── OrderEventPublisher.java
│       │   └── OrderEventListener.java
│       └── config/                  # 配置
│           ├── JpaConfig.java
│           └── MessagingConfig.java
└── order-interfaces/                # 接口模块
    ├── pom.xml
    └── src/main/java/com/example/order/interfaces/
        ├── rest/                    # REST API
        │   ├── OrderController.java
        │   ├── OrderQueryController.java
        │   └── dto/
        │       ├── CreateOrderRequest.java
        │       └── OrderResponse.java
        ├── web/                     # Web 页面（可选）
        └── OrderServiceApplication.java  # 启动类
```

### 单模块结构（小型项目）

```
order-service/
├── pom.xml
└── src/main/java/com/example/order/
    ├── OrderServiceApplication.java
    ├── domain/                      # 领域层
    │   ├── model/
    │   ├── service/
    │   ├── repository/
    │   └── event/
    ├── application/                 # 应用层
    │   ├── command/
    │   ├── query/
    │   ├── service/
    │   └── dto/
    ├── infrastructure/              # 基础设施层
    │   ├── persistence/
    │   ├── messaging/
    │   └── config/
    └── interfaces/                  # 接口层
        └── rest/
```

## 分层架构详解

### 1. 领域层（Domain Layer）

**职责：** 核心业务逻辑和规则

**依赖：** 不依赖任何其他层

**包含：**
- 聚合根、实体、值对象
- 领域服务
- 仓储接口（不是实现）
- 领域事件

**pom.xml 依赖：**
```xml
<dependencies>
    <!-- 只依赖 Java 标准库和少量工具库 -->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
        <scope>provided</scope> <!-- 仅用于 @Service 等注解 -->
    </dependency>
</dependencies>
```

**示例代码：**
```java
// 聚合根
package com.example.order.domain.model;

public class Order {
    private OrderId id;
    private CustomerId customerId;
    private List<OrderItem> items;
    private OrderStatus status;
    private Money totalAmount;
    private LocalDateTime createdAt;
    
    // 私有构造函数，强制使用工厂方法
    private Order() {}
    
    // 工厂方法
    public static Order create(CustomerId customerId, List<OrderItem> items) {
        Order order = new Order();
        order.id = OrderId.generate();
        order.customerId = customerId;
        order.items = new ArrayList<>(items);
        order.status = OrderStatus.DRAFT;
        order.createdAt = LocalDateTime.now();
        order.calculateTotal();
        
        // 验证不变量
        order.validate();
        
        return order;
    }
    
    // 业务方法
    public void submit() {
        if (status != OrderStatus.DRAFT) {
            throw new IllegalStateException("只能提交草稿状态的订单");
        }
        if (items.isEmpty()) {
            throw new IllegalStateException("订单必须包含至少一个商品");
        }
        
        this.status = OrderStatus.SUBMITTED;
    }
    
    public void cancel(String reason) {
        if (status == OrderStatus.SHIPPED || status == OrderStatus.DELIVERED) {
            throw new OrderCannotBeCancelledException("已发货或已送达的订单无法取消");
        }
        
        this.status = OrderStatus.CANCELLED;
        // 发布领域事件
        DomainEventPublisher.publish(new OrderCancelledEvent(this.id, reason));
    }
    
    public void addItem(Product product, int quantity) {
        if (status != OrderStatus.DRAFT) {
            throw new IllegalStateException("只能向草稿订单添加商品");
        }
        
        OrderItem item = new OrderItem(product, quantity);
        items.add(item);
        calculateTotal();
    }
    
    // 私有辅助方法
    private void calculateTotal() {
        this.totalAmount = items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(Money.ZERO, Money::add);
    }
    
    private void validate() {
        if (customerId == null) {
            throw new IllegalArgumentException("客户ID不能为空");
        }
        if (totalAmount.isNegative()) {
            throw new IllegalArgumentException("订单总额不能为负数");
        }
    }
    
    // Getter 方法（只读）
    public OrderId getId() { return id; }
    public CustomerId getCustomerId() { return customerId; }
    public List<OrderItem> getItems() { return Collections.unmodifiableList(items); }
    public OrderStatus getStatus() { return status; }
    public Money getTotalAmount() { return totalAmount; }
}

// 值对象
public record OrderId(String value) {
    public OrderId {
        if (value == null || value.isBlank()) {
            throw new IllegalArgumentException("订单ID不能为空");
        }
    }
    
    public static OrderId generate() {
        return new OrderId(UUID.randomUUID().toString());
    }
}

// 仓储接口（在领域层定义）
package com.example.order.domain.repository;

public interface OrderRepository {
    Order save(Order order);
    Optional<Order> findById(OrderId id);
    List<Order> findByCustomerId(CustomerId customerId);
    void delete(OrderId id);
}
```

### 2. 应用层（Application Layer）

**职责：** 协调领域对象完成用例

**依赖：** 领域层

**包含：**
- 应用服务
- 命令和查询对象
- DTO（数据传输对象）
- 事务边界

**pom.xml 依赖：**
```xml
<dependencies>
    <dependency>
        <groupId>com.example</groupId>
        <artifactId>order-domain</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-tx</artifactId>
    </dependency>
</dependencies>
```

**示例代码：**
```java
// 应用服务
package com.example.order.application.service;

@Service
@Transactional
public class OrderCommandService {
    private final OrderRepository orderRepository;
    private final CustomerRepository customerRepository;
    private final ProductRepository productRepository;
    private final ApplicationEventPublisher eventPublisher;
    
    public OrderCommandService(
        OrderRepository orderRepository,
        CustomerRepository customerRepository,
        ProductRepository productRepository,
        ApplicationEventPublisher eventPublisher
    ) {
        this.orderRepository = orderRepository;
        this.customerRepository = customerRepository;
        this.productRepository = productRepository;
        this.eventPublisher = eventPublisher;
    }
    
    public OrderId createOrder(CreateOrderCommand command) {
        // 1. 验证命令
        validateCommand(command);
        
        // 2. 加载依赖的聚合
        Customer customer = customerRepository.findById(command.customerId())
            .orElseThrow(() -> new CustomerNotFoundException(command.customerId()));
        
        List<OrderItem> items = command.items().stream()
            .map(this::toOrderItem)
            .toList();
        
        // 3. 创建聚合
        Order order = Order.create(customer.getId(), items);
        
        // 4. 持久化
        orderRepository.save(order);
        
        // 5. 发布事件
        eventPublisher.publishEvent(new OrderCreatedEvent(
            order.getId(),
            order.getCustomerId(),
            order.getTotalAmount()
        ));
        
        return order.getId();
    }
    
    public void cancelOrder(CancelOrderCommand command) {
        Order order = orderRepository.findById(command.orderId())
            .orElseThrow(() -> new OrderNotFoundException(command.orderId()));
        
        order.cancel(command.reason());
        
        orderRepository.save(order);
        
        eventPublisher.publishEvent(new OrderCancelledEvent(
            order.getId(),
            command.reason()
        ));
    }
    
    private void validateCommand(CreateOrderCommand command) {
        if (command.items().isEmpty()) {
            throw new InvalidCommandException("订单必须包含至少一个商品");
        }
    }
    
    private OrderItem toOrderItem(OrderItemDto dto) {
        Product product = productRepository.findById(dto.productId())
            .orElseThrow(() -> new ProductNotFoundException(dto.productId()));
        return new OrderItem(product, dto.quantity());
    }
}

// 查询服务
@Service
@Transactional(readOnly = true)
public class OrderQueryService {
    private final OrderRepository orderRepository;
    
    public OrderDto getOrder(OrderId orderId) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
        
        return OrderDto.from(order);
    }
    
    public List<OrderDto> listCustomerOrders(CustomerId customerId) {
        return orderRepository.findByCustomerId(customerId).stream()
            .map(OrderDto::from)
            .toList();
    }
}

// DTO
package com.example.order.application.dto;

public record OrderDto(
    String id,
    String customerId,
    List<OrderItemDto> items,
    String status,
    BigDecimal totalAmount,
    LocalDateTime createdAt
) {
    public static OrderDto from(Order order) {
        return new OrderDto(
            order.getId().value(),
            order.getCustomerId().value(),
            order.getItems().stream()
                .map(OrderItemDto::from)
                .toList(),
            order.getStatus().name(),
            order.getTotalAmount().amount(),
            order.getCreatedAt()
        );
    }
}
```

### 3. 基础设施层（Infrastructure Layer）

**职责：** 提供技术实现

**依赖：** 领域层、应用层

**包含：**
- 仓储实现
- 消息发布/订阅
- 外部服务集成
- 配置

**pom.xml 依赖：**
```xml
<dependencies>
    <dependency>
        <groupId>com.example</groupId>
        <artifactId>order-domain</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-amqp</artifactId>
    </dependency>
</dependencies>
```

**示例代码：**
```java
// 仓储实现
package com.example.order.infrastructure.persistence;

@Repository
public class OrderRepositoryImpl implements OrderRepository {
    private final OrderJpaRepository jpaRepository;
    private final OrderMapper mapper;
    
    @Override
    public Order save(Order order) {
        OrderEntity entity = mapper.toEntity(order);
        OrderEntity saved = jpaRepository.save(entity);
        return mapper.toDomain(saved);
    }
    
    @Override
    public Optional<Order> findById(OrderId id) {
        return jpaRepository.findById(id.value())
            .map(mapper::toDomain);
    }
    
    @Override
    public List<Order> findByCustomerId(CustomerId customerId) {
        return jpaRepository.findByCustomerId(customerId.value()).stream()
            .map(mapper::toDomain)
            .toList();
    }
}

// JPA 仓储
interface OrderJpaRepository extends JpaRepository<OrderEntity, String> {
    List<OrderEntity> findByCustomerId(String customerId);
}

// JPA 实体（与领域模型分离）
@Entity
@Table(name = "orders")
class OrderEntity {
    @Id
    private String id;
    
    @Column(name = "customer_id", nullable = false)
    private String customerId;
    
    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
    @JoinColumn(name = "order_id")
    private List<OrderItemEntity> items = new ArrayList<>();
    
    @Enumerated(EnumType.STRING)
    private OrderStatus status;
    
    @Column(name = "total_amount")
    private BigDecimal totalAmount;
    
    @Column(name = "created_at")
    private LocalDateTime createdAt;
    
    // Getter/Setter
}

// 映射器
@Component
class OrderMapper {
    public OrderEntity toEntity(Order order) {
        OrderEntity entity = new OrderEntity();
        entity.setId(order.getId().value());
        entity.setCustomerId(order.getCustomerId().value());
        entity.setStatus(order.getStatus());
        entity.setTotalAmount(order.getTotalAmount().amount());
        entity.setCreatedAt(order.getCreatedAt());
        entity.setItems(order.getItems().stream()
            .map(this::toItemEntity)
            .toList());
        return entity;
    }
    
    public Order toDomain(OrderEntity entity) {
        // 使用反射或构造器重建领域对象
        // 注意：这里需要特殊处理，因为领域对象通常是不可变的
    }
}
```

#### 使用 MyBatis 实现仓储（替代方案）

如果你更喜欢使用 MyBatis 而不是 JPA，可以这样实现：

**pom.xml 依赖：**
```xml
<dependencies>
    <dependency>
        <groupId>com.example</groupId>
        <artifactId>order-domain</artifactId>
    </dependency>
    <dependency>
        <groupId>org.mybatis.spring.boot</groupId>
        <artifactId>mybatis-spring-boot-starter</artifactId>
        <version>3.0.3</version>
    </dependency>
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
    </dependency>
</dependencies>
```

**MyBatis 仓储实现：**
```java
// 仓储实现
package com.example.order.infrastructure.persistence;

@Repository
public class OrderRepositoryImpl implements OrderRepository {
    private final OrderMapper orderMapper;
    private final OrderItemMapper orderItemMapper;
    private final OrderDomainMapper domainMapper;
    
    public OrderRepositoryImpl(
        OrderMapper orderMapper,
        OrderItemMapper orderItemMapper,
        OrderDomainMapper domainMapper
    ) {
        this.orderMapper = orderMapper;
        this.orderItemMapper = orderItemMapper;
        this.domainMapper = domainMapper;
    }
    
    @Override
    @Transactional
    public Order save(Order order) {
        OrderPO orderPO = domainMapper.toPO(order);
        
        if (orderMapper.existsById(orderPO.getId())) {
            // 更新
            orderMapper.update(orderPO);
            // 删除旧的订单项
            orderItemMapper.deleteByOrderId(orderPO.getId());
        } else {
            // 插入
            orderMapper.insert(orderPO);
        }
        
        // 插入订单项
        for (OrderItemPO item : orderPO.getItems()) {
            item.setOrderId(orderPO.getId());
            orderItemMapper.insert(item);
        }
        
        return order;
    }
    
    @Override
    public Optional<Order> findById(OrderId id) {
        OrderPO orderPO = orderMapper.selectById(id.value());
        if (orderPO == null) {
            return Optional.empty();
        }
        
        List<OrderItemPO> items = orderItemMapper.selectByOrderId(id.value());
        orderPO.setItems(items);
        
        return Optional.of(domainMapper.toDomain(orderPO));
    }
    
    @Override
    public List<Order> findByCustomerId(CustomerId customerId) {
        List<OrderPO> orderPOs = orderMapper.selectByCustomerId(customerId.value());
        
        return orderPOs.stream()
            .map(po -> {
                List<OrderItemPO> items = orderItemMapper.selectByOrderId(po.getId());
                po.setItems(items);
                return domainMapper.toDomain(po);
            })
            .toList();
    }
    
    @Override
    @Transactional
    public void delete(OrderId id) {
        orderItemMapper.deleteByOrderId(id.value());
        orderMapper.deleteById(id.value());
    }
}

// MyBatis Mapper 接口
package com.example.order.infrastructure.persistence.mapper;

@Mapper
public interface OrderMapper {
    @Insert("""
        INSERT INTO orders (id, customer_id, status, total_amount, created_at)
        VALUES (#{id}, #{customerId}, #{status}, #{totalAmount}, #{createdAt})
        """)
    void insert(OrderPO order);
    
    @Update("""
        UPDATE orders 
        SET customer_id = #{customerId}, 
            status = #{status}, 
            total_amount = #{totalAmount}
        WHERE id = #{id}
        """)
    void update(OrderPO order);
    
    @Select("SELECT * FROM orders WHERE id = #{id}")
    @Results({
        @Result(property = "id", column = "id"),
        @Result(property = "customerId", column = "customer_id"),
        @Result(property = "status", column = "status"),
        @Result(property = "totalAmount", column = "total_amount"),
        @Result(property = "createdAt", column = "created_at")
    })
    OrderPO selectById(String id);
    
    @Select("SELECT * FROM orders WHERE customer_id = #{customerId}")
    @Results({
        @Result(property = "id", column = "id"),
        @Result(property = "customerId", column = "customer_id"),
        @Result(property = "status", column = "status"),
        @Result(property = "totalAmount", column = "total_amount"),
        @Result(property = "createdAt", column = "created_at")
    })
    List<OrderPO> selectByCustomerId(String customerId);
    
    @Select("SELECT COUNT(*) FROM orders WHERE id = #{id}")
    boolean existsById(String id);
    
    @Delete("DELETE FROM orders WHERE id = #{id}")
    void deleteById(String id);
}

package com.example.order.infrastructure.persistence.mapper;

@Mapper
public interface OrderItemMapper {
    @Insert("""
        INSERT INTO order_items (id, order_id, product_id, product_name, quantity, price)
        VALUES (#{id}, #{orderId}, #{productId}, #{productName}, #{quantity}, #{price})
        """)
    void insert(OrderItemPO item);
    
    @Select("SELECT * FROM order_items WHERE order_id = #{orderId}")
    @Results({
        @Result(property = "id", column = "id"),
        @Result(property = "orderId", column = "order_id"),
        @Result(property = "productId", column = "product_id"),
        @Result(property = "productName", column = "product_name"),
        @Result(property = "quantity", column = "quantity"),
        @Result(property = "price", column = "price")
    })
    List<OrderItemPO> selectByOrderId(String orderId);
    
    @Delete("DELETE FROM order_items WHERE order_id = #{orderId}")
    void deleteByOrderId(String orderId);
}

// 持久化对象（PO）
public class OrderPO {
    private String id;
    private String customerId;
    private String status;
    private BigDecimal totalAmount;
    private LocalDateTime createdAt;
    private List<OrderItemPO> items;
    
    // Getter/Setter
}

public class OrderItemPO {
    private String id;
    private String orderId;
    private String productId;
    private String productName;
    private Integer quantity;
    private BigDecimal price;
    
    // Getter/Setter
}

// 领域对象映射器
@Component
public class OrderDomainMapper {
    public OrderPO toPO(Order order) {
        OrderPO po = new OrderPO();
        po.setId(order.getId().value());
        po.setCustomerId(order.getCustomerId().value());
        po.setStatus(order.getStatus().name());
        po.setTotalAmount(order.getTotalAmount().amount());
        po.setCreatedAt(order.getCreatedAt());
        
        List<OrderItemPO> itemPOs = order.getItems().stream()
            .map(this::toItemPO)
            .toList();
        po.setItems(itemPOs);
        
        return po;
    }
    
    public Order toDomain(OrderPO po) {
        // 重建领域对象
        List<OrderItem> items = po.getItems().stream()
            .map(this::toItemDomain)
            .toList();
        
        // 使用反射或特殊构造器重建 Order
        return Order.reconstruct(
            new OrderId(po.getId()),
            new CustomerId(po.getCustomerId()),
            items,
            OrderStatus.valueOf(po.getStatus()),
            new Money(po.getTotalAmount()),
            po.getCreatedAt()
        );
    }
    
    private OrderItemPO toItemPO(OrderItem item) {
        OrderItemPO po = new OrderItemPO();
        po.setId(UUID.randomUUID().toString());
        po.setProductId(item.getProductId().value());
        po.setProductName(item.getProductName());
        po.setQuantity(item.getQuantity());
        po.setPrice(item.getPrice().amount());
        return po;
    }
    
    private OrderItem toItemDomain(OrderItemPO po) {
        return OrderItem.reconstruct(
            new ProductId(po.getProductId()),
            po.getProductName(),
            po.getQuantity(),
            new Money(po.getPrice())
        );
    }
}
```

**使用 XML 配置（推荐用于复杂查询）：**

```xml
<!-- resources/mapper/OrderMapper.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" 
    "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.order.infrastructure.persistence.mapper.OrderMapper">
    
    <resultMap id="OrderResultMap" type="com.example.order.infrastructure.persistence.OrderPO">
        <id property="id" column="id"/>
        <result property="customerId" column="customer_id"/>
        <result property="status" column="status"/>
        <result property="totalAmount" column="total_amount"/>
        <result property="createdAt" column="created_at"/>
        <collection property="items" ofType="com.example.order.infrastructure.persistence.OrderItemPO">
            <id property="id" column="item_id"/>
            <result property="orderId" column="order_id"/>
            <result property="productId" column="product_id"/>
            <result property="productName" column="product_name"/>
            <result property="quantity" column="quantity"/>
            <result property="price" column="price"/>
        </collection>
    </resultMap>
    
    <select id="selectByIdWithItems" resultMap="OrderResultMap">
        SELECT 
            o.id, o.customer_id, o.status, o.total_amount, o.created_at,
            oi.id as item_id, oi.order_id, oi.product_id, 
            oi.product_name, oi.quantity, oi.price
        FROM orders o
        LEFT JOIN order_items oi ON o.id = oi.order_id
        WHERE o.id = #{id}
    </select>
    
    <select id="selectByCustomerIdWithItems" resultMap="OrderResultMap">
        SELECT 
            o.id, o.customer_id, o.status, o.total_amount, o.created_at,
            oi.id as item_id, oi.order_id, oi.product_id, 
            oi.product_name, oi.quantity, oi.price
        FROM orders o
        LEFT JOIN order_items oi ON o.id = oi.order_id
        WHERE o.customer_id = #{customerId}
        ORDER BY o.created_at DESC
    </select>
    
</mapper>
```

**MyBatis 配置：**

```java
// application.yml
mybatis:
  mapper-locations: classpath:mapper/*.xml
  type-aliases-package: com.example.order.infrastructure.persistence
  configuration:
    map-underscore-to-camel-case: true
    default-fetch-size: 100
    default-statement-timeout: 30

// 或者 Java 配置
@Configuration
@MapperScan("com.example.order.infrastructure.persistence.mapper")
public class MyBatisConfig {
    
    @Bean
    public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
        SqlSessionFactoryBean sessionFactory = new SqlSessionFactoryBean();
        sessionFactory.setDataSource(dataSource);
        sessionFactory.setMapperLocations(
            new PathMatchingResourcePatternResolver()
                .getResources("classpath:mapper/*.xml")
        );
        
        org.apache.ibatis.session.Configuration configuration = 
            new org.apache.ibatis.session.Configuration();
        configuration.setMapUnderscoreToCamelCase(true);
        sessionFactory.setConfiguration(configuration);
        
        return sessionFactory.getObject();
    }
}
```

**MyBatis vs JPA 选择建议：**

| 场景 | 推荐 |
|------|------|
| 复杂查询、多表关联 | MyBatis |
| 简单 CRUD | JPA |
| 需要完全控制 SQL | MyBatis |
| 快速开发原型 | JPA |
| 性能优化要求高 | MyBatis |
| 数据库无关性 | JPA |
| 动态 SQL | MyBatis |
| 领域模型复杂 | JPA + 映射器 |

**混合使用：**

可以在同一个项目中同时使用 JPA 和 MyBatis：
- 写操作（命令）：使用 JPA，利用其事务管理和对象映射
- 读操作（查询）：使用 MyBatis，优化复杂查询性能

```java
@Service
@Transactional
public class OrderCommandService {
    private final OrderJpaRepository jpaRepository; // 写操作用 JPA
    
    public OrderId createOrder(CreateOrderCommand command) {
        // 使用 JPA 保存
    }
}

@Service
@Transactional(readOnly = true)
public class OrderQueryService {
    private final OrderMapper mapper; // 读操作用 MyBatis
    
    public List<OrderDto> searchOrders(OrderSearchCriteria criteria) {
        // 使用 MyBatis 复杂查询
    }
}
```

#### 使用 Spring Data JDBC 实现仓储（推荐的轻量级方案）

Spring Data JDBC 提供类似 JPA 的编程模型，但更简单、更透明，没有延迟加载、脏检查等复杂特性。

**pom.xml 依赖：**
```xml
<dependencies>
    <dependency>
        <groupId>com.example</groupId>
        <artifactId>order-domain</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jdbc</artifactId>
    </dependency>
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
    </dependency>
</dependencies>
```

**Spring Data JDBC 实体：**
```java
// 持久化实体（与领域模型分离）
package com.example.order.infrastructure.persistence;

import org.springframework.data.annotation.*;
import org.springframework.data.relational.core.mapping.*;

@Table("orders")
class OrderEntity {
    @Id
    private String id;
    
    @Column("customer_id")
    private String customerId;
    
    private String status;
    
    @Column("total_amount")
    private BigDecimal totalAmount;
    
    @Column("created_at")
    private LocalDateTime createdAt;
    
    // Spring Data JDBC 使用 Set 管理一对多关系
    @MappedCollection(idColumn = "order_id")
    private Set<OrderItemEntity> items = new HashSet<>();
    
    // Getter/Setter
}

@Table("order_items")
class OrderItemEntity {
    @Id
    private Long id; // 自增主键
    
    @Column("product_id")
    private String productId;
    
    @Column("product_name")
    private String productName;
    
    private Integer quantity;
    
    private BigDecimal price;
    
    // Getter/Setter
}
```

**Spring Data JDBC 仓储：**
```java
// Spring Data JDBC 仓储接口
package com.example.order.infrastructure.persistence;

import org.springframework.data.repository.CrudRepository;
import org.springframework.data.jdbc.repository.query.Query;

interface OrderJdbcRepository extends CrudRepository<OrderEntity, String> {
    
    @Query("SELECT * FROM orders WHERE customer_id = :customerId ORDER BY created_at DESC")
    List<OrderEntity> findByCustomerId(String customerId);
    
    @Query("""
        SELECT * FROM orders 
        WHERE status = :status 
        AND created_at > :since
        """)
    List<OrderEntity> findByStatusAndCreatedAtAfter(String status, LocalDateTime since);
}

// 仓储实现（领域仓储接口的实现）
@Repository
public class OrderRepositoryImpl implements OrderRepository {
    private final OrderJdbcRepository jdbcRepository;
    private final OrderMapper mapper;
    
    public OrderRepositoryImpl(OrderJdbcRepository jdbcRepository, OrderMapper mapper) {
        this.jdbcRepository = jdbcRepository;
        this.mapper = mapper;
    }
    
    @Override
    public Order save(Order order) {
        OrderEntity entity = mapper.toEntity(order);
        OrderEntity saved = jdbcRepository.save(entity);
        return mapper.toDomain(saved);
    }
    
    @Override
    public Optional<Order> findById(OrderId id) {
        return jdbcRepository.findById(id.value())
            .map(mapper::toDomain);
    }
    
    @Override
    public List<Order> findByCustomerId(CustomerId customerId) {
        return jdbcRepository.findByCustomerId(customerId.value()).stream()
            .map(mapper::toDomain)
            .toList();
    }
    
    @Override
    public void delete(OrderId id) {
        jdbcRepository.deleteById(id.value());
    }
}

// 映射器
@Component
class OrderMapper {
    public OrderEntity toEntity(Order order) {
        OrderEntity entity = new OrderEntity();
        entity.setId(order.getId().value());
        entity.setCustomerId(order.getCustomerId().value());
        entity.setStatus(order.getStatus().name());
        entity.setTotalAmount(order.getTotalAmount().amount());
        entity.setCreatedAt(order.getCreatedAt());
        
        Set<OrderItemEntity> items = order.getItems().stream()
            .map(this::toItemEntity)
            .collect(Collectors.toSet());
        entity.setItems(items);
        
        return entity;
    }
    
    public Order toDomain(OrderEntity entity) {
        List<OrderItem> items = entity.getItems().stream()
            .map(this::toItemDomain)
            .toList();
        
        return Order.reconstruct(
            new OrderId(entity.getId()),
            new CustomerId(entity.getCustomerId()),
            items,
            OrderStatus.valueOf(entity.getStatus()),
            new Money(entity.getTotalAmount()),
            entity.getCreatedAt()
        );
    }
    
    private OrderItemEntity toItemEntity(OrderItem item) {
        OrderItemEntity entity = new OrderItemEntity();
        entity.setProductId(item.getProductId().value());
        entity.setProductName(item.getProductName());
        entity.setQuantity(item.getQuantity());
        entity.setPrice(item.getPrice().amount());
        return entity;
    }
    
    private OrderItem toItemDomain(OrderItemEntity entity) {
        return OrderItem.reconstruct(
            new ProductId(entity.getProductId()),
            entity.getProductName(),
            entity.getQuantity(),
            new Money(entity.getPrice())
        );
    }
}
```

**使用 JdbcClient（Spring 6.1+）：**

```java
// JdbcClient 是 Spring 6.1 引入的现代化 JDBC API
@Repository
public class OrderRepositoryImpl implements OrderRepository {
    private final JdbcClient jdbcClient;
    
    public OrderRepositoryImpl(JdbcClient jdbcClient) {
        this.jdbcClient = jdbcClient;
    }
    
    @Override
    @Transactional
    public Order save(Order order) {
        String sql = """
            INSERT INTO orders (id, customer_id, status, total_amount, created_at)
            VALUES (:id, :customerId, :status, :totalAmount, :createdAt)
            ON DUPLICATE KEY UPDATE
                status = VALUES(status),
                total_amount = VALUES(total_amount)
            """;
        
        jdbcClient.sql(sql)
            .param("id", order.getId().value())
            .param("customerId", order.getCustomerId().value())
            .param("status", order.getStatus().name())
            .param("totalAmount", order.getTotalAmount().amount())
            .param("createdAt", order.getCreatedAt())
            .update();
        
        // 保存订单项
        saveOrderItems(order);
        
        return order;
    }
    
    @Override
    public Optional<Order> findById(OrderId id) {
        String sql = """
            SELECT o.id, o.customer_id, o.status, o.total_amount, o.created_at,
                   oi.id as item_id, oi.product_id, oi.product_name, 
                   oi.quantity, oi.price
            FROM orders o
            LEFT JOIN order_items oi ON o.id = oi.order_id
            WHERE o.id = :id
            """;
        
        List<OrderRow> rows = jdbcClient.sql(sql)
            .param("id", id.value())
            .query(OrderRow.class)
            .list();
        
        if (rows.isEmpty()) {
            return Optional.empty();
        }
        
        return Optional.of(mapToOrder(rows));
    }
    
    @Override
    public List<Order> findByCustomerId(CustomerId customerId) {
        String sql = """
            SELECT o.id, o.customer_id, o.status, o.total_amount, o.created_at,
                   oi.id as item_id, oi.product_id, oi.product_name, 
                   oi.quantity, oi.price
            FROM orders o
            LEFT JOIN order_items oi ON o.id = oi.order_id
            WHERE o.customer_id = :customerId
            ORDER BY o.created_at DESC
            """;
        
        List<OrderRow> rows = jdbcClient.sql(sql)
            .param("customerId", customerId.value())
            .query(OrderRow.class)
            .list();
        
        // 按订单 ID 分组
        Map<String, List<OrderRow>> grouped = rows.stream()
            .collect(Collectors.groupingBy(OrderRow::orderId));
        
        return grouped.values().stream()
            .map(this::mapToOrder)
            .toList();
    }
    
    private void saveOrderItems(Order order) {
        // 删除旧的订单项
        jdbcClient.sql("DELETE FROM order_items WHERE order_id = :orderId")
            .param("orderId", order.getId().value())
            .update();
        
        // 插入新的订单项
        String sql = """
            INSERT INTO order_items (id, order_id, product_id, product_name, quantity, price)
            VALUES (:id, :orderId, :productId, :productName, :quantity, :price)
            """;
        
        for (OrderItem item : order.getItems()) {
            jdbcClient.sql(sql)
                .param("id", UUID.randomUUID().toString())
                .param("orderId", order.getId().value())
                .param("productId", item.getProductId().value())
                .param("productName", item.getProductName())
                .param("quantity", item.getQuantity())
                .param("price", item.getPrice().amount())
                .update();
        }
    }
    
    private Order mapToOrder(List<OrderRow> rows) {
        OrderRow first = rows.get(0);
        
        List<OrderItem> items = rows.stream()
            .filter(row -> row.itemId() != null)
            .map(row -> OrderItem.reconstruct(
                new ProductId(row.productId()),
                row.productName(),
                row.quantity(),
                new Money(row.price())
            ))
            .toList();
        
        return Order.reconstruct(
            new OrderId(first.orderId()),
            new CustomerId(first.customerId()),
            items,
            OrderStatus.valueOf(first.status()),
            new Money(first.totalAmount()),
            first.createdAt()
        );
    }
}
```

**Spring Data JDBC 配置：**

```java
@Configuration
@EnableJdbcRepositories(basePackages = "com.example.order.infrastructure.persistence")
public class JdbcConfig extends AbstractJdbcConfiguration {
    
    // 自定义命名策略
    @Bean
    NamingStrategy namingStrategy() {
        return new NamingStrategy() {
            @Override
            public String getColumnName(RelationalPersistentProperty property) {
                // 驼峰转下划线
                return CaseFormat.LOWER_CAMEL.to(CaseFormat.LOWER_UNDERSCORE, 
                    property.getName());
            }
        };
    }
    
    // 自定义转换器
    @Override
    protected List<?> userConverters() {
        return List.of(
            new OrderStatusReadingConverter(),
            new OrderStatusWritingConverter()
        );
    }
}

// 自定义转换器
@ReadingConverter
class OrderStatusReadingConverter implements Converter<String, OrderStatus> {
    @Override
    public OrderStatus convert(String source) {
        return OrderStatus.valueOf(source);
    }
}

@WritingConverter
class OrderStatusWritingConverter implements Converter<OrderStatus, String> {
    @Override
    public String convert(OrderStatus source) {
        return source.name();
    }
}
```

**Spring Data JDBC vs JPA 对比：**

| 特性 | Spring Data JPA | Spring Data JDBC |
|------|----------------|------------------|
| 复杂度 | 高 | 低 |
| 学习曲线 | 陡峭 | 平缓 |
| 延迟加载 | 支持 | 不支持 |
| 脏检查 | 自动 | 无 |
| 缓存 | 一级/二级缓存 | 无 |
| 关系映射 | 复杂 | 简单（聚合） |
| SQL 控制 | 低 | 高 |
| 性能 | 中等 | 高 |
| 透明度 | 低（魔法多） | 高 |
| DDD 友好 | 中等 | 高 |

**Spring Data JDBC 的优势：**

1. **简单透明**：没有延迟加载、脏检查等隐藏行为
2. **性能好**：没有缓存和代理开销
3. **DDD 友好**：聚合根概念天然契合
4. **易于理解**：保存时完整更新聚合，删除时级联删除
5. **无魔法**：行为可预测

**Spring Data JDBC 的限制：**

1. **不支持延迟加载**：总是立即加载整个聚合
2. **关系映射简单**：主要支持聚合内的一对多
3. **无缓存**：每次查询都访问数据库
4. **无脏检查**：需要显式调用 save()

**适用场景：**

- ✅ DDD 项目（聚合根模式）
- ✅ 需要简单透明的持久化
- ✅ 不需要复杂的对象关系映射
- ✅ 追求性能和可预测性
- ❌ 需要延迟加载大对象图
- ❌ 需要复杂的多对多关系
- ❌ 需要二级缓存

#### 使用 Spring JDBC（JdbcTemplate/JdbcClient）实现仓储

如果你不需要 ORM 的任何功能，可以使用原生的 Spring JDBC：

**pom.xml 依赖：**
```xml
<dependencies>
    <dependency>
        <groupId>com.example</groupId>
        <artifactId>order-domain</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-jdbc</artifactId>
    </dependency>
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
    </dependency>
</dependencies>
```

**JdbcTemplate 仓储实现：**
```java
// 仓储实现
package com.example.order.infrastructure.persistence;

@Repository
public class OrderRepositoryImpl implements OrderRepository {
    private final JdbcTemplate jdbcTemplate;
    private final NamedParameterJdbcTemplate namedJdbcTemplate;
    
    public OrderRepositoryImpl(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
        this.namedJdbcTemplate = new NamedParameterJdbcTemplate(jdbcTemplate);
    }
    
    @Override
    @Transactional
    public Order save(Order order) {
        String orderSql = """
            INSERT INTO orders (id, customer_id, status, total_amount, created_at)
            VALUES (?, ?, ?, ?, ?)
            ON DUPLICATE KEY UPDATE
                status = VALUES(status),
                total_amount = VALUES(total_amount)
            """;
        
        jdbcTemplate.update(orderSql,
            order.getId().value(),
            order.getCustomerId().value(),
            order.getStatus().name(),
            order.getTotalAmount().amount(),
            Timestamp.valueOf(order.getCreatedAt())
        );
        
        // 删除旧的订单项
        jdbcTemplate.update("DELETE FROM order_items WHERE order_id = ?", 
            order.getId().value());
        
        // 插入订单项
        String itemSql = """
            INSERT INTO order_items (id, order_id, product_id, product_name, quantity, price)
            VALUES (?, ?, ?, ?, ?, ?)
            """;
        
        for (OrderItem item : order.getItems()) {
            jdbcTemplate.update(itemSql,
                UUID.randomUUID().toString(),
                order.getId().value(),
                item.getProductId().value(),
                item.getProductName(),
                item.getQuantity(),
                item.getPrice().amount()
            );
        }
        
        return order;
    }
    
    @Override
    public Optional<Order> findById(OrderId id) {
        String sql = """
            SELECT o.id, o.customer_id, o.status, o.total_amount, o.created_at,
                   oi.id as item_id, oi.product_id, oi.product_name, 
                   oi.quantity, oi.price
            FROM orders o
            LEFT JOIN order_items oi ON o.id = oi.order_id
            WHERE o.id = ?
            """;
        
        try {
            List<OrderRow> rows = jdbcTemplate.query(sql, 
                new OrderRowMapper(), 
                id.value()
            );
            
            if (rows.isEmpty()) {
                return Optional.empty();
            }
            
            return Optional.of(mapToOrder(rows));
        } catch (EmptyResultDataAccessException e) {
            return Optional.empty();
        }
    }
    
    @Override
    public List<Order> findByCustomerId(CustomerId customerId) {
        String sql = """
            SELECT o.id, o.customer_id, o.status, o.total_amount, o.created_at,
                   oi.id as item_id, oi.product_id, oi.product_name, 
                   oi.quantity, oi.price
            FROM orders o
            LEFT JOIN order_items oi ON o.id = oi.order_id
            WHERE o.customer_id = ?
            ORDER BY o.created_at DESC
            """;
        
        List<OrderRow> rows = jdbcTemplate.query(sql, 
            new OrderRowMapper(), 
            customerId.value()
        );
        
        // 按订单 ID 分组
        Map<String, List<OrderRow>> grouped = rows.stream()
            .collect(Collectors.groupingBy(OrderRow::orderId));
        
        return grouped.values().stream()
            .map(this::mapToOrder)
            .toList();
    }
    
    @Override
    @Transactional
    public void delete(OrderId id) {
        jdbcTemplate.update("DELETE FROM order_items WHERE order_id = ?", id.value());
        jdbcTemplate.update("DELETE FROM orders WHERE id = ?", id.value());
    }
    
    // 辅助方法：将查询结果映射为领域对象
    private Order mapToOrder(List<OrderRow> rows) {
        OrderRow first = rows.get(0);
        
        List<OrderItem> items = rows.stream()
            .filter(row -> row.itemId() != null)
            .map(row -> OrderItem.reconstruct(
                new ProductId(row.productId()),
                row.productName(),
                row.quantity(),
                new Money(row.price())
            ))
            .toList();
        
        return Order.reconstruct(
            new OrderId(first.orderId()),
            new CustomerId(first.customerId()),
            items,
            OrderStatus.valueOf(first.status()),
            new Money(first.totalAmount()),
            first.createdAt()
        );
    }
}

// RowMapper 实现
class OrderRowMapper implements RowMapper<OrderRow> {
    @Override
    public OrderRow mapRow(ResultSet rs, int rowNum) throws SQLException {
        return new OrderRow(
            rs.getString("id"),
            rs.getString("customer_id"),
            rs.getString("status"),
            rs.getBigDecimal("total_amount"),
            rs.getTimestamp("created_at").toLocalDateTime(),
            rs.getString("item_id"),
            rs.getString("product_id"),
            rs.getString("product_name"),
            rs.getObject("quantity", Integer.class),
            rs.getBigDecimal("price")
        );
    }
}

// 查询结果记录
record OrderRow(
    String orderId,
    String customerId,
    String status,
    BigDecimal totalAmount,
    LocalDateTime createdAt,
    String itemId,
    String productId,
    String productName,
    Integer quantity,
    BigDecimal price
) {}
```

**使用 NamedParameterJdbcTemplate（推荐）：**

```java
@Repository
public class OrderRepositoryImpl implements OrderRepository {
    private final NamedParameterJdbcTemplate jdbcTemplate;
    
    @Override
    @Transactional
    public Order save(Order order) {
        String sql = """
            INSERT INTO orders (id, customer_id, status, total_amount, created_at)
            VALUES (:id, :customerId, :status, :totalAmount, :createdAt)
            ON DUPLICATE KEY UPDATE
                status = VALUES(status),
                total_amount = VALUES(total_amount)
            """;
        
        MapSqlParameterSource params = new MapSqlParameterSource()
            .addValue("id", order.getId().value())
            .addValue("customerId", order.getCustomerId().value())
            .addValue("status", order.getStatus().name())
            .addValue("totalAmount", order.getTotalAmount().amount())
            .addValue("createdAt", Timestamp.valueOf(order.getCreatedAt()));
        
        jdbcTemplate.update(sql, params);
        
        // 批量插入订单项
        saveOrderItems(order);
        
        return order;
    }
    
    private void saveOrderItems(Order order) {
        // 先删除旧的
        jdbcTemplate.update(
            "DELETE FROM order_items WHERE order_id = :orderId",
            Map.of("orderId", order.getId().value())
        );
        
        // 批量插入新的
        String sql = """
            INSERT INTO order_items (id, order_id, product_id, product_name, quantity, price)
            VALUES (:id, :orderId, :productId, :productName, :quantity, :price)
            """;
        
        SqlParameterSource[] batchParams = order.getItems().stream()
            .map(item -> new MapSqlParameterSource()
                .addValue("id", UUID.randomUUID().toString())
                .addValue("orderId", order.getId().value())
                .addValue("productId", item.getProductId().value())
                .addValue("productName", item.getProductName())
                .addValue("quantity", item.getQuantity())
                .addValue("price", item.getPrice().amount())
            )
            .toArray(SqlParameterSource[]::new);
        
        jdbcTemplate.batchUpdate(sql, batchParams);
    }
    
    @Override
    public Optional<Order> findById(OrderId id) {
        String sql = """
            SELECT o.id, o.customer_id, o.status, o.total_amount, o.created_at,
                   oi.id as item_id, oi.product_id, oi.product_name, 
                   oi.quantity, oi.price
            FROM orders o
            LEFT JOIN order_items oi ON o.id = oi.order_id
            WHERE o.id = :id
            """;
        
        try {
            List<OrderRow> rows = jdbcTemplate.query(sql, 
                Map.of("id", id.value()),
                new OrderRowMapper()
            );
            
            if (rows.isEmpty()) {
                return Optional.empty();
            }
            
            return Optional.of(mapToOrder(rows));
        } catch (EmptyResultDataAccessException e) {
            return Optional.empty();
        }
    }
}
```

**使用 SimpleJdbcInsert（简化插入）：**

```java
@Repository
public class OrderRepositoryImpl implements OrderRepository {
    private final JdbcTemplate jdbcTemplate;
    private final SimpleJdbcInsert orderInsert;
    private final SimpleJdbcInsert orderItemInsert;
    
    public OrderRepositoryImpl(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
        
        this.orderInsert = new SimpleJdbcInsert(jdbcTemplate)
            .withTableName("orders")
            .usingColumns("id", "customer_id", "status", "total_amount", "created_at");
        
        this.orderItemInsert = new SimpleJdbcInsert(jdbcTemplate)
            .withTableName("order_items")
            .usingGeneratedKeyColumns("id");
    }
    
    @Override
    @Transactional
    public Order save(Order order) {
        Map<String, Object> orderParams = Map.of(
            "id", order.getId().value(),
            "customer_id", order.getCustomerId().value(),
            "status", order.getStatus().name(),
            "total_amount", order.getTotalAmount().amount(),
            "created_at", Timestamp.valueOf(order.getCreatedAt())
        );
        
        orderInsert.execute(orderParams);
        
        // 插入订单项
        for (OrderItem item : order.getItems()) {
            Map<String, Object> itemParams = Map.of(
                "order_id", order.getId().value(),
                "product_id", item.getProductId().value(),
                "product_name", item.getProductName(),
                "quantity", item.getQuantity(),
                "price", item.getPrice().amount()
            );
            
            orderItemInsert.execute(itemParams);
        }
        
        return order;
    }
}
```

**Spring JDBC 配置：**

```java
@Configuration
public class JdbcConfig {
    
    @Bean
    public JdbcTemplate jdbcTemplate(DataSource dataSource) {
        JdbcTemplate template = new JdbcTemplate(dataSource);
        template.setQueryTimeout(30);
        return template;
    }
    
    @Bean
    public NamedParameterJdbcTemplate namedParameterJdbcTemplate(JdbcTemplate jdbcTemplate) {
        return new NamedParameterJdbcTemplate(jdbcTemplate);
    }
    
    @Bean
    public TransactionTemplate transactionTemplate(PlatformTransactionManager transactionManager) {
        return new TransactionTemplate(transactionManager);
    }
}
```

**持久化方案对比：**

| 特性 | JPA | Spring Data JDBC | MyBatis | Spring JDBC |
|------|-----|------------------|---------|-------------|
| 学习曲线 | 陡峭 | 平缓 | 中等 | 低 |
| 代码量 | 少 | 少 | 中等 | 多 |
| SQL 控制 | 低 | 中等 | 高 | 高 |
| 性能 | 中等 | 高 | 高 | 高 |
| 复杂查询 | 困难 | 中等 | 容易 | 容易 |
| 对象映射 | 自动 | 自动（简单） | 半自动 | 手动 |
| DDD 友好 | 中等 | 高 | 中等 | 低 |
| 延迟加载 | 支持 | 不支持 | 支持 | 不支持 |
| 缓存 | 一级/二级 | 无 | 二级（可选） | 无 |
| 透明度 | 低 | 高 | 高 | 高 |
| 事务管理 | 自动 | 自动 | 自动 | 自动 |
| 批量操作 | 支持 | 支持 | 优秀 | 优秀 |
| 适用场景 | 复杂对象图 | DDD 聚合 | 复杂查询 | 轻量级项目 |

**选择建议：**

1. **Spring Data JDBC**（推荐）：DDD 项目，需要简单透明的持久化
2. **JPA**：复杂对象关系映射，需要延迟加载和缓存
3. **MyBatis**：需要完全控制 SQL，复杂查询多
4. **Spring JDBC**：极简项目，不需要任何 ORM 功能

**混合使用策略：**

```java
// 写操作：使用 Spring Data JDBC（简单透明）
@Service
@Transactional
public class OrderCommandService {
    private final OrderJdbcRepository jdbcRepository;
    
    public OrderId createOrder(CreateOrderCommand command) {
        Order order = Order.create(command);
        OrderEntity entity = mapper.toEntity(order);
        jdbcRepository.save(entity);
        return order.getId();
    }
}

// 简单查询：使用 Spring Data JDBC
@Service
@Transactional(readOnly = true)
public class OrderQueryService {
    private final OrderJdbcRepository jdbcRepository;
    
    public Order getOrder(OrderId id) {
        return jdbcRepository.findById(id.value())
            .map(mapper::toDomain)
            .orElseThrow(() -> new OrderNotFoundException(id));
    }
}

// 复杂查询/报表：使用 JdbcClient
@Service
@Transactional(readOnly = true)
public class OrderReportService {
    private final JdbcClient jdbcClient;
    
    public List<OrderStatistics> getOrderStatistics(LocalDate from, LocalDate to) {
        String sql = """
            SELECT 
                DATE(o.created_at) as order_date,
                COUNT(*) as order_count,
                SUM(o.total_amount) as total_revenue,
                AVG(o.total_amount) as avg_order_value
            FROM orders o
            WHERE o.created_at BETWEEN :from AND :to
            GROUP BY DATE(o.created_at)
            ORDER BY order_date
            """;
        
        return jdbcClient.sql(sql)
            .param("from", from)
            .param("to", to)
            .query(OrderStatistics.class)
            .list();
    }
}
```

### 4. 接口层（Interfaces Layer）

**职责：** 暴露应用功能

**依赖：** 应用层

**包含：**
- REST Controller
- GraphQL Resolver
- gRPC Service
- Web 页面

**pom.xml 依赖：**
```xml
<dependencies>
    <dependency>
        <groupId>com.example</groupId>
        <artifactId>order-application</artifactId>
    </dependency>
    <dependency>
        <groupId>com.example</groupId>
        <artifactId>order-infrastructure</artifactId>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

**示例代码：**
```java
// REST Controller
package com.example.order.interfaces.rest;

@RestController
@RequestMapping("/api/orders")
public class OrderController {
    private final OrderCommandService commandService;
    private final OrderQueryService queryService;
    
    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(
        @RequestBody @Valid CreateOrderRequest request
    ) {
        CreateOrderCommand command = new CreateOrderCommand(
            new CustomerId(request.customerId()),
            request.items()
        );
        
        OrderId orderId = commandService.createOrder(command);
        OrderDto order = queryService.getOrder(orderId);
        
        return ResponseEntity
            .created(URI.create("/api/orders/" + orderId.value()))
            .body(OrderResponse.from(order));
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<OrderResponse> getOrder(@PathVariable String id) {
        OrderDto order = queryService.getOrder(new OrderId(id));
        return ResponseEntity.ok(OrderResponse.from(order));
    }
    
    @PostMapping("/{id}/cancel")
    public ResponseEntity<Void> cancelOrder(
        @PathVariable String id,
        @RequestBody CancelOrderRequest request
    ) {
        CancelOrderCommand command = new CancelOrderCommand(
            new OrderId(id),
            request.reason()
        );
        
        commandService.cancelOrder(command);
        
        return ResponseEntity.noContent().build();
    }
}
```

## 完整实战案例：电商订单系统

### 需求分析

**用户故事：**
1. 作为客户，我想创建订单，以便购买商品
2. 作为客户，我想查看我的订单列表
3. 作为客户，我想取消未发货的订单
4. 作为系统，订单创建后需要扣减库存

### 步骤 1：识别聚合

**Order 聚合：**
- 聚合根：Order
- 实体：OrderItem
- 值对象：OrderId, Money, OrderStatus

**边界：** 一个订单及其订单项

### 步骤 2：定义领域模型

```java
// domain/model/Order.java
public class Order {
    private OrderId id;
    private CustomerId customerId;
    private List<OrderItem> items;
    private OrderStatus status;
    private Money totalAmount;
    
    public static Order create(CustomerId customerId, List<OrderItem> items) {
        // 实现
    }
    
    public void submit() { /* 实现 */ }
    public void cancel(String reason) { /* 实现 */ }
}
```

### 步骤 3：定义仓储接口

```java
// domain/repository/OrderRepository.java
public interface OrderRepository {
    Order save(Order order);
    Optional<Order> findById(OrderId id);
}
```

### 步骤 4：实现应用服务

```java
// application/service/OrderCommandService.java
@Service
@Transactional
public class OrderCommandService {
    public OrderId createOrder(CreateOrderCommand command) {
        // 实现
    }
}
```

### 步骤 5：实现基础设施

```java
// infrastructure/persistence/OrderRepositoryImpl.java
@Repository
public class OrderRepositoryImpl implements OrderRepository {
    // 实现
}
```

### 步骤 6：暴露 REST API

```java
// interfaces/rest/OrderController.java
@RestController
public class OrderController {
    // 实现
}
```

## Maven 配置

### 父 POM

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project>
    <modelVersion>4.0.0</modelVersion>
    
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
    </parent>
    
    <groupId>com.example</groupId>
    <artifactId>order-service</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>pom</packaging>
    
    <modules>
        <module>order-domain</module>
        <module>order-application</module>
        <module>order-infrastructure</module>
        <module>order-interfaces</module>
    </modules>
    
    <properties>
        <java.version>17</java.version>
    </properties>
</project>
```

### 领域模块 POM

```xml
<project>
    <parent>
        <groupId>com.example</groupId>
        <artifactId>order-service</artifactId>
        <version>1.0.0-SNAPSHOT</version>
    </parent>
    
    <artifactId>order-domain</artifactId>
    
    <dependencies>
        <!-- 最小依赖 -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <scope>provided</scope>
        </dependency>
    </dependencies>
</project>
```

## 最佳实践

### 1. 依赖方向

```
接口层 → 应用层 → 领域层 ← 基础设施层
```

领域层不依赖任何其他层！

### 2. 事务边界

事务边界在应用服务层：

```java
@Service
@Transactional  // 事务边界
public class OrderCommandService {
    public OrderId createOrder(CreateOrderCommand command) {
        // 整个方法是一个事务
    }
}
```

### 3. 领域事件

```java
// 在聚合中发布事件
public class Order {
    public void cancel(String reason) {
        this.status = OrderStatus.CANCELLED;
        registerEvent(new OrderCancelledEvent(this.id, reason));
    }
}

// 在应用服务中发布到消息总线
@Service
public class OrderCommandService {
    public void cancelOrder(CancelOrderCommand command) {
        Order order = orderRepository.findById(command.orderId())
            .orElseThrow();
        
        order.cancel(command.reason());
        orderRepository.save(order);
        
        // 发布事件
        order.getDomainEvents().forEach(eventPublisher::publishEvent);
    }
}
```

### 4. 防腐层

当集成外部系统时，使用防腐层：

```java
// 外部服务接口（在领域层定义）
public interface PaymentService {
    PaymentResult processPayment(OrderId orderId, Money amount);
}

// 防腐层实现（在基础设施层）
@Service
public class StripePaymentServiceAdapter implements PaymentService {
    private final StripeClient stripeClient;
    
    @Override
    public PaymentResult processPayment(OrderId orderId, Money amount) {
        // 调用 Stripe API
        StripePaymentResponse response = stripeClient.charge(
            amount.amount(),
            "usd"
        );
        
        // 转换为领域对象
        return new PaymentResult(
            response.isSuccess(),
            response.getTransactionId()
        );
    }
}
```

## 常见问题

### Q: 领域模型和 JPA 实体应该分离吗？

**A:** 推荐分离，原因：
- 领域模型关注业务规则
- JPA 实体关注持久化
- 避免 JPA 注解污染领域模型
- 更容易测试

### Q: 什么时候使用多模块？

**A:** 
- 团队规模 > 5 人
- 项目预期长期维护
- 需要强制依赖方向
- 需要独立部署某些模块

小型项目可以使用单模块 + 包结构。

### Q: DTO 和领域对象如何转换？

**A:** 使用专门的 Mapper 或工厂方法：

```java
public record OrderDto(...) {
    public static OrderDto from(Order order) {
        // 转换逻辑
    }
}
```

## 相关技能

- `/springboot-ddd-cn` - DDD 核心概念详解
- `/springboot-ddd-cqrs-cn` - CQRS 模式实践
- `/springboot-testing-cn` - DDD 应用的测试策略
