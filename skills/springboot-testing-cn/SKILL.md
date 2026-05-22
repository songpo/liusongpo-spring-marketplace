---
name: springboot-testing-cn
description: 为 Spring Boot 应用设计测试策略、编写单元测试、集成测试
---

# Spring Boot 测试策略与实践

当你需要为 Spring Boot 应用设计测试策略、编写单元测试、集成测试时使用此技能。

## 何时使用

- 设计测试策略
- 编写单元测试
- 编写集成测试
- 在控制器、服务和仓储的测试方法之间做选择

## 测试金字塔

```
        /\
       /  \  E2E 测试（少量）
      /____\
     /      \
    / 集成测试 \ （适量）
   /__________\
  /            \
 /   单元测试    \ （大量）
/________________\
```

**原则：**
- 单元测试：快速、隔离、大量
- 集成测试：验证组件协作、适量
- E2E 测试：验证完整流程、少量

## 1. 单元测试

### 1.1 服务层单元测试

使用 Mockito 模拟依赖。

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    
    @Mock
    private OrderRepository orderRepository;
    
    @Mock
    private ProductRepository productRepository;
    
    @InjectMocks
    private OrderService orderService;
    
    @Test
    void shouldCreateOrder() {
        // Given
        CustomerId customerId = CustomerId.of("customer-1");
        Order order = new Order(customerId);
        
        when(orderRepository.save(any(Order.class)))
            .thenReturn(order);
        
        // When
        Order result = orderService.createOrder(customerId);
        
        // Then
        assertThat(result).isNotNull();
        assertThat(result.getStatus()).isEqualTo(OrderStatus.DRAFT);
        verify(orderRepository).save(any(Order.class));
    }
    
    @Test
    void shouldThrowExceptionWhenProductNotFound() {
        // Given
        OrderId orderId = OrderId.generate();
        ProductId productId = ProductId.of("product-1");
        
        when(orderRepository.findById(orderId))
            .thenReturn(Optional.of(new Order(CustomerId.of("customer-1"))));
        when(productRepository.findById(productId))
            .thenReturn(Optional.empty());
        
        // When & Then
        assertThatThrownBy(() -> 
            orderService.addItemToOrder(orderId, productId, 1))
            .isInstanceOf(ProductNotFoundException.class)
            .hasMessage("Product not found: product-1");
    }
}
```

### 1.2 领域模型单元测试

测试业务逻辑，不需要 Spring 容器。

```java
class OrderTest {
    
    @Test
    void shouldAddItemToOrder() {
        // Given
        Order order = new Order(CustomerId.of("customer-1"));
        ProductId productId = ProductId.of("product-1");
        Money price = Money.of(new BigDecimal("100.00"), "CNY");
        
        // When
        order.addItem(productId, price, 2);
        
        // Then
        assertThat(order.getItems()).hasSize(1);
        assertThat(order.getTotal()).isEqualTo(Money.of(new BigDecimal("200.00"), "CNY"));
    }
    
    @Test
    void shouldNotAddItemToSubmittedOrder() {
        // Given
        Order order = new Order(CustomerId.of("customer-1"));
        order.addItem(ProductId.of("product-1"), Money.of(new BigDecimal("100.00"), "CNY"), 1);
        order.submit();
        
        // When & Then
        assertThatThrownBy(() -> 
            order.addItem(ProductId.of("product-2"), Money.of(new BigDecimal("50.00"), "CNY"), 1))
            .isInstanceOf(OrderNotEditableException.class);
    }
    
    @Test
    void shouldNotSubmitEmptyOrder() {
        // Given
        Order order = new Order(CustomerId.of("customer-1"));
        
        // When & Then
        assertThatThrownBy(() -> order.submit())
            .isInstanceOf(EmptyOrderException.class);
    }
}
```

## 2. 集成测试

### 2.1 仓储层集成测试

使用 `@DataJpaTest` 测试 JPA 仓储。

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class OrderRepositoryTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private TestEntityManager entityManager;
    
    @Test
    void shouldSaveAndFindOrder() {
        // Given
        Order order = new Order(CustomerId.of("customer-1"));
        order.addItem(
            ProductId.of("product-1"), 
            Money.of(new BigDecimal("100.00"), "CNY"), 
            2
        );
        
        // When
        Order saved = orderRepository.save(order);
        entityManager.flush();
        entityManager.clear();
        
        Order found = orderRepository.findById(saved.getId()).orElseThrow();
        
        // Then
        assertThat(found.getId()).isEqualTo(saved.getId());
        assertThat(found.getItems()).hasSize(1);
        assertThat(found.getTotal()).isEqualTo(Money.of(new BigDecimal("200.00"), "CNY"));
    }
    
    @Test
    void shouldFindOrdersByCustomerId() {
        // Given
        CustomerId customerId = CustomerId.of("customer-1");
        Order order1 = new Order(customerId);
        Order order2 = new Order(customerId);
        orderRepository.saveAll(List.of(order1, order2));
        entityManager.flush();
        
        // When
        List<Order> orders = orderRepository.findByCustomerId(customerId);
        
        // Then
        assertThat(orders).hasSize(2);
    }
}
```

### 2.2 服务层集成测试

使用 `@SpringBootTest` 测试完整的应用上下文。

```java
@SpringBootTest
@Testcontainers
@Transactional
class OrderServiceIntegrationTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
    
    @Autowired
    private OrderService orderService;
    
    @Autowired
    private ProductRepository productRepository;
    
    @Test
    void shouldCreateOrderAndAddItems() {
        // Given
        Product product = new Product("Test Product", Money.of(new BigDecimal("100.00"), "CNY"));
        productRepository.save(product);
        
        CustomerId customerId = CustomerId.of("customer-1");
        
        // When
        Order order = orderService.createOrder(customerId);
        orderService.addItemToOrder(order.getId(), product.getId(), 2);
        
        // Then
        Order found = orderService.findById(order.getId());
        assertThat(found.getItems()).hasSize(1);
        assertThat(found.getTotal()).isEqualTo(Money.of(new BigDecimal("200.00"), "CNY"));
    }
}
```

### 2.3 控制器集成测试

使用 `@WebMvcTest` 测试 REST 控制器。

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @MockBean
    private OrderService orderService;
    
    @Test
    void shouldCreateOrder() throws Exception {
        // Given
        CreateOrderRequest request = new CreateOrderRequest("customer-1");
        Order order = new Order(CustomerId.of("customer-1"));
        
        when(orderService.createOrder(any(CustomerId.class)))
            .thenReturn(order);
        
        // When & Then
        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {
                        "customerId": "customer-1"
                    }
                    """))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").exists())
            .andExpect(jsonPath("$.status").value("DRAFT"));
    }
    
    @Test
    void shouldGetOrder() throws Exception {
        // Given
        OrderId orderId = OrderId.of("order-1");
        Order order = new Order(CustomerId.of("customer-1"));
        
        when(orderService.findById(orderId))
            .thenReturn(order);
        
        // When & Then
        mockMvc.perform(get("/api/orders/{id}", "order-1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id").value("order-1"))
            .andExpect(jsonPath("$.status").value("DRAFT"));
    }
    
    @Test
    void shouldReturn404WhenOrderNotFound() throws Exception {
        // Given
        when(orderService.findById(any(OrderId.class)))
            .thenThrow(new OrderNotFoundException("order-1"));
        
        // When & Then
        mockMvc.perform(get("/api/orders/{id}", "order-1"))
            .andExpect(status().isNotFound());
    }
}
```

## 3. E2E 测试

使用 `@SpringBootTest` 和 `TestRestTemplate`。

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class OrderE2ETest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Test
    void shouldCompleteOrderFlow() {
        // 1. 创建订单
        CreateOrderRequest createRequest = new CreateOrderRequest("customer-1");
        ResponseEntity<OrderResponse> createResponse = restTemplate.postForEntity(
            "/api/orders",
            createRequest,
            OrderResponse.class
        );
        
        assertThat(createResponse.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        String orderId = createResponse.getBody().getId();
        
        // 2. 添加商品
        AddItemRequest addItemRequest = new AddItemRequest("product-1", 2);
        ResponseEntity<Void> addItemResponse = restTemplate.postForEntity(
            "/api/orders/{id}/items",
            addItemRequest,
            Void.class,
            orderId
        );
        
        assertThat(addItemResponse.getStatusCode()).isEqualTo(HttpStatus.OK);
        
        // 3. 提交订单
        ResponseEntity<Void> submitResponse = restTemplate.postForEntity(
            "/api/orders/{id}/submit",
            null,
            Void.class,
            orderId
        );
        
        assertThat(submitResponse.getStatusCode()).isEqualTo(HttpStatus.OK);
        
        // 4. 验证订单状态
        ResponseEntity<OrderResponse> getResponse = restTemplate.getForEntity(
            "/api/orders/{id}",
            OrderResponse.class,
            orderId
        );
        
        assertThat(getResponse.getBody().getStatus()).isEqualTo("SUBMITTED");
    }
}
```

## 4. 测试配置

### 4.1 Testcontainers 配置

```java
@TestConfiguration
public class TestcontainersConfiguration {
    
    @Bean
    @ServiceConnection
    public PostgreSQLContainer<?> postgresContainer() {
        return new PostgreSQLContainer<>("postgres:16")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test")
            .withReuse(true);  // 重用容器加速测试
    }
}
```

### 4.2 测试数据构建器

```java
public class OrderTestBuilder {
    
    private CustomerId customerId = CustomerId.of("customer-1");
    private OrderStatus status = OrderStatus.DRAFT;
    private List<OrderItem> items = new ArrayList<>();
    
    public static OrderTestBuilder anOrder() {
        return new OrderTestBuilder();
    }
    
    public OrderTestBuilder withCustomerId(String customerId) {
        this.customerId = CustomerId.of(customerId);
        return this;
    }
    
    public OrderTestBuilder withStatus(OrderStatus status) {
        this.status = status;
        return this;
    }
    
    public OrderTestBuilder withItem(String productId, String price, int quantity) {
        this.items.add(new OrderItem(
            ProductId.of(productId),
            Money.of(new BigDecimal(price), "CNY"),
            quantity
        ));
        return this;
    }
    
    public Order build() {
        Order order = new Order(customerId);
        items.forEach(item -> 
            order.addItem(item.getProductId(), item.getPrice(), item.getQuantity())
        );
        if (status == OrderStatus.SUBMITTED) {
            order.submit();
        }
        return order;
    }
}

// 使用
Order order = anOrder()
    .withCustomerId("customer-1")
    .withItem("product-1", "100.00", 2)
    .withStatus(OrderStatus.SUBMITTED)
    .build();
```

## 5. 最佳实践

### 5.1 测试命名

使用 `should...When...` 或 `given...When...Then` 模式：

```java
@Test
void shouldThrowExceptionWhenOrderIsEmpty() { }

@Test
void givenEmptyOrder_whenSubmit_thenThrowException() { }
```

### 5.2 测试隔离

- 每个测试独立运行
- 使用 `@Transactional` 自动回滚
- 不依赖测试执行顺序

### 5.3 测试数据

- 使用测试构建器创建测试数据
- 避免共享可变状态
- 使用有意义的测试数据

### 5.4 断言

使用 AssertJ 提供流畅的断言：

```java
assertThat(order.getItems())
    .hasSize(2)
    .extracting(OrderItem::getProductId)
    .containsExactly(ProductId.of("product-1"), ProductId.of("product-2"));
```

### 5.5 异常测试

```java
assertThatThrownBy(() -> order.submit())
    .isInstanceOf(EmptyOrderException.class)
    .hasMessage("Order cannot be empty");
```

## 6. 依赖配置

```xml
<dependencies>
    <!-- 测试依赖 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    
    <!-- Testcontainers -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-testcontainers</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>postgresql</artifactId>
        <scope>test</scope>
    </dependency>
    
    <!-- AssertJ -->
    <dependency>
        <groupId>org.assertj</groupId>
        <artifactId>assertj-core</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

## 测试策略选择指南

| 测试类型 | 使用场景 | 注解 | 速度 |
|---------|---------|------|------|
| 单元测试 | 领域模型、工具类 | 无 | 最快 |
| 单元测试（Mock） | 服务层 | `@ExtendWith(MockitoExtension.class)` | 快 |
| 仓储测试 | JPA 仓储 | `@DataJpaTest` | 中等 |
| 控制器测试 | REST API | `@WebMvcTest` | 中等 |
| 集成测试 | 多组件协作 | `@SpringBootTest` | 慢 |
| E2E 测试 | 完整业务流程 | `@SpringBootTest(webEnvironment=RANDOM_PORT)` | 最慢 |
