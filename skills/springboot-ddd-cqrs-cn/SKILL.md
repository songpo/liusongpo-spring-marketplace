---
name: springboot-ddd-cqrs-cn
description: 在 Spring Boot 应用中实践 CQRS（命令查询职责分离）模式
---

# Spring Boot CQRS 实践指南

当你需要在 Spring Boot 应用中实践 CQRS（Command Query Responsibility Segregation，命令查询职责分离）模式时使用此技能。

## 何时使用

- 实现命令（Command）和查询（Query）的分离
- 设计命令处理器和查询处理器
- 构建读写分离的数据模型
- 实现事件溯源（Event Sourcing）
- 处理最终一致性问题
- 优化复杂查询性能

## CQRS 核心概念

### 1. 命令（Command）

命令表示改变系统状态的意图，遵循任务导向命名。

**原则：**
- 命令不返回数据，只返回成功/失败状态
- 命令是异步处理的（可选）
- 命令包含执行操作所需的所有信息
- 使用领域语言命名（CreateOrder, CancelOrder）

**示例：**
```java
// 命令对象
public record CreateOrderCommand(
    CustomerId customerId,
    List<OrderItemDto> items,
    ShippingAddress shippingAddress
) {}

// 命令处理器
@Service
@Transactional
public class CreateOrderCommandHandler {
    private final OrderRepository orderRepository;
    private final ApplicationEventPublisher eventPublisher;
    
    public OrderId handle(CreateOrderCommand command) {
        // 1. 验证命令
        validateCommand(command);
        
        // 2. 创建聚合
        Order order = Order.create(
            command.customerId(),
            command.items(),
            command.shippingAddress()
        );
        
        // 3. 持久化
        orderRepository.save(order);
        
        // 4. 发布领域事件
        order.getDomainEvents().forEach(eventPublisher::publishEvent);
        
        return order.getId();
    }
    
    private void validateCommand(CreateOrderCommand command) {
        if (command.items().isEmpty()) {
            throw new InvalidCommandException("订单必须包含至少一个商品");
        }
    }
}
```

### 2. 查询（Query）

查询用于读取数据，不改变系统状态。

**原则：**
- 查询只读取数据，不修改状态
- 查询可以直接访问数据库（绕过领域模型）
- 查询模型可以与命令模型完全不同
- 针对特定用例优化查询模型

**示例：**
```java
// 查询对象
public record GetOrderDetailsQuery(OrderId orderId) {}

// 查询结果 DTO
public record OrderDetailsDto(
    String orderId,
    String customerName,
    List<OrderItemDto> items,
    BigDecimal totalAmount,
    String status,
    LocalDateTime createdAt
) {}

// 查询处理器
@Service
@Transactional(readOnly = true)
public class GetOrderDetailsQueryHandler {
    private final JdbcTemplate jdbcTemplate;
    
    public OrderDetailsDto handle(GetOrderDetailsQuery query) {
        // 直接使用 SQL 查询，优化性能
        String sql = """
            SELECT o.id, c.name, o.total_amount, o.status, o.created_at,
                   oi.product_name, oi.quantity, oi.price
            FROM orders o
            JOIN customers c ON o.customer_id = c.id
            LEFT JOIN order_items oi ON o.id = oi.order_id
            WHERE o.id = ?
            """;
        
        return jdbcTemplate.query(sql, 
            new OrderDetailsResultSetExtractor(), 
            query.orderId().value()
        );
    }
}
```

### 3. 读写模型分离

命令侧和查询侧使用不同的数据模型。

**命令侧（写模型）：**
```java
// 领域模型 - 关注业务规则和不变量
@Entity
@Table(name = "orders")
public class Order {
    @Id
    private OrderId id;
    
    @Embedded
    private CustomerId customerId;
    
    @OneToMany(cascade = CascadeType.ALL)
    private List<OrderItem> items;
    
    @Enumerated(EnumType.STRING)
    private OrderStatus status;
    
    // 业务方法
    public void cancel(String reason) {
        if (status == OrderStatus.SHIPPED) {
            throw new OrderCannotBeCancelledException("已发货的订单无法取消");
        }
        this.status = OrderStatus.CANCELLED;
        registerEvent(new OrderCancelledEvent(id, reason));
    }
}
```

**查询侧（读模型）：**
```java
// 查询模型 - 针对展示优化，扁平化结构
@Entity
@Table(name = "order_read_model")
public class OrderReadModel {
    @Id
    private String id;
    
    private String customerName;
    private String customerEmail;
    private BigDecimal totalAmount;
    private String status;
    private int itemCount;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    
    // 只有 getter，没有业务逻辑
}

// 查询仓储
public interface OrderReadModelRepository extends JpaRepository<OrderReadModel, String> {
    List<OrderReadModel> findByCustomerEmailOrderByCreatedAtDesc(String email);
    
    @Query("SELECT o FROM OrderReadModel o WHERE o.status = :status AND o.createdAt > :since")
    List<OrderReadModel> findRecentOrdersByStatus(
        @Param("status") String status, 
        @Param("since") LocalDateTime since
    );
}
```

### 4. 事件驱动同步

使用领域事件同步读写模型。

**发布事件：**
```java
@Service
public class OrderCommandService {
    private final ApplicationEventPublisher eventPublisher;
    
    @Transactional
    public void cancelOrder(CancelOrderCommand command) {
        Order order = orderRepository.findById(command.orderId())
            .orElseThrow(() -> new OrderNotFoundException(command.orderId()));
        
        order.cancel(command.reason());
        orderRepository.save(order);
        
        // 发布事件
        eventPublisher.publishEvent(
            new OrderCancelledEvent(order.getId(), command.reason())
        );
    }
}
```

**更新读模型：**
```java
@Component
public class OrderReadModelUpdater {
    private final OrderReadModelRepository readModelRepository;
    
    @EventListener
    @Transactional
    public void on(OrderCancelledEvent event) {
        OrderReadModel readModel = readModelRepository.findById(event.orderId().value())
            .orElseThrow();
        
        readModel.setStatus("CANCELLED");
        readModel.setUpdatedAt(LocalDateTime.now());
        
        readModelRepository.save(readModel);
    }
    
    @EventListener
    @Transactional
    public void on(OrderCreatedEvent event) {
        OrderReadModel readModel = new OrderReadModel();
        readModel.setId(event.orderId().value());
        readModel.setCustomerName(event.customerName());
        readModel.setCustomerEmail(event.customerEmail());
        readModel.setTotalAmount(event.totalAmount());
        readModel.setStatus("CREATED");
        readModel.setItemCount(event.itemCount());
        readModel.setCreatedAt(event.occurredAt());
        
        readModelRepository.save(readModel);
    }
}
```

### 5. 命令总线模式（可选）

使用命令总线统一处理命令。

```java
// 命令接口
public interface Command {}

// 命令处理器接口
public interface CommandHandler<T extends Command> {
    void handle(T command);
}

// 命令总线
@Service
public class CommandBus {
    private final Map<Class<? extends Command>, CommandHandler> handlers = new HashMap<>();
    
    public <T extends Command> void registerHandler(
        Class<T> commandType, 
        CommandHandler<T> handler
    ) {
        handlers.put(commandType, handler);
    }
    
    @SuppressWarnings("unchecked")
    public <T extends Command> void dispatch(T command) {
        CommandHandler<T> handler = handlers.get(command.getClass());
        if (handler == null) {
            throw new NoHandlerFoundException(command.getClass());
        }
        handler.handle(command);
    }
}

// 使用示例
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    private final CommandBus commandBus;
    
    @PostMapping
    public ResponseEntity<Void> createOrder(@RequestBody CreateOrderRequest request) {
        CreateOrderCommand command = new CreateOrderCommand(
            new CustomerId(request.customerId()),
            request.items(),
            request.shippingAddress()
        );
        
        commandBus.dispatch(command);
        
        return ResponseEntity.accepted().build();
    }
}
```

### 6. 查询总线模式（可选）

```java
// 查询接口
public interface Query<R> {}

// 查询处理器接口
public interface QueryHandler<Q extends Query<R>, R> {
    R handle(Q query);
}

// 查询总线
@Service
public class QueryBus {
    private final Map<Class<? extends Query>, QueryHandler> handlers = new HashMap<>();
    
    public <Q extends Query<R>, R> void registerHandler(
        Class<Q> queryType, 
        QueryHandler<Q, R> handler
    ) {
        handlers.put(queryType, handler);
    }
    
    @SuppressWarnings("unchecked")
    public <Q extends Query<R>, R> R dispatch(Q query) {
        QueryHandler<Q, R> handler = handlers.get(query.getClass());
        if (handler == null) {
            throw new NoHandlerFoundException(query.getClass());
        }
        return handler.handle(query);
    }
}
```

## 项目结构建议

```
src/main/java/com/example/
├── domain/                          # 领域层（命令侧）
│   ├── model/
│   │   ├── Order.java              # 聚合根
│   │   ├── OrderItem.java          # 实体
│   │   └── OrderStatus.java        # 值对象
│   ├── repository/
│   │   └── OrderRepository.java    # 写仓储接口
│   └── event/
│       ├── OrderCreatedEvent.java
│       └── OrderCancelledEvent.java
├── application/                     # 应用层
│   ├── command/
│   │   ├── CreateOrderCommand.java
│   │   ├── CancelOrderCommand.java
│   │   └── handler/
│   │       ├── CreateOrderCommandHandler.java
│   │       └── CancelOrderCommandHandler.java
│   └── query/
│       ├── GetOrderDetailsQuery.java
│       ├── ListOrdersQuery.java
│       └── handler/
│           ├── GetOrderDetailsQueryHandler.java
│           └── ListOrdersQueryHandler.java
├── infrastructure/                  # 基础设施层
│   ├── persistence/
│   │   ├── OrderJpaRepository.java # 写仓储实现
│   │   └── OrderReadModelRepository.java # 读仓储
│   └── event/
│       └── OrderReadModelUpdater.java # 事件监听器
└── interfaces/                      # 接口层
    └── rest/
        ├── OrderCommandController.java
        └── OrderQueryController.java
```

## 最佳实践

### 1. 何时使用 CQRS

**适合使用：**
- 读写比例严重不平衡（读多写少）
- 查询需求复杂，需要多种视图
- 需要独立扩展读写能力
- 需要审计和事件溯源

**不适合使用：**
- 简单的 CRUD 应用
- 读写模式相似
- 团队对 CQRS 不熟悉
- 不需要独立扩展

### 2. 渐进式采用

不必一开始就完全分离读写模型：

**Level 1：** 分离命令和查询方法
```java
@Service
public class OrderService {
    // 命令方法
    public void createOrder(CreateOrderCommand command) { }
    
    // 查询方法
    public OrderDto getOrder(OrderId id) { }
}
```

**Level 2：** 分离命令和查询处理器
```java
@Service
public class CreateOrderCommandHandler { }

@Service
public class GetOrderQueryHandler { }
```

**Level 3：** 分离读写数据模型
```java
// 写模型
@Entity
class Order { }

// 读模型
@Entity
class OrderReadModel { }
```

**Level 4：** 使用事件溯源

### 3. 处理最终一致性

```java
@Service
public class OrderQueryService {
    private final OrderReadModelRepository repository;
    
    public OrderDetailsDto getOrderDetails(OrderId orderId) {
        return repository.findById(orderId.value())
            .map(this::toDto)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
    }
    
    // 提供版本信息，让客户端知道数据的新鲜度
    public OrderDetailsDto getOrderDetailsWithVersion(OrderId orderId) {
        OrderReadModel model = repository.findById(orderId.value())
            .orElseThrow(() -> new OrderNotFoundException(orderId));
        
        return new OrderDetailsDto(
            model.getId(),
            model.getCustomerName(),
            model.getTotalAmount(),
            model.getStatus(),
            model.getVersion(), // 版本号
            model.getUpdatedAt() // 最后更新时间
        );
    }
}
```

### 4. 错误处理

```java
@RestController
public class OrderCommandController {
    private final CommandBus commandBus;
    
    @PostMapping("/api/orders")
    public ResponseEntity<?> createOrder(@RequestBody CreateOrderRequest request) {
        try {
            CreateOrderCommand command = toCommand(request);
            commandBus.dispatch(command);
            
            // 返回 202 Accepted，表示命令已接受但可能尚未完成
            return ResponseEntity.accepted()
                .header("Location", "/api/orders/" + command.orderId())
                .build();
                
        } catch (ValidationException e) {
            return ResponseEntity.badRequest()
                .body(new ErrorResponse(e.getMessage()));
        }
    }
}
```

## 常见陷阱

1. **过度设计**：不是所有场景都需要 CQRS
2. **忽略最终一致性**：客户端需要处理数据延迟
3. **事件丢失**：确保事件发布的可靠性
4. **读模型同步失败**：需要重试和补偿机制
5. **命令返回数据**：违反 CQRS 原则

## 相关技能

- `/springboot-ddd-cn` - DDD 基础概念和聚合设计
- `/springboot-testing-cn` - CQRS 应用的测试策略
