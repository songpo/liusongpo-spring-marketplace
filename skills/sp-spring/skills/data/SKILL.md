---
name: data
description: Spring Data 数据访问实践，包括 JPA、JDBC、Redis、MongoDB 和查询方法
---

# Spring Data 数据访问实践指南

当你需要使用 Spring Data 进行数据访问时使用此技能，包括 Spring Data JPA、Spring Data JDBC、Spring Data Redis、Spring Data MongoDB 等。

## 何时使用

- 使用 JPA 进行对象关系映射
- 使用 JDBC 进行轻量级数据访问
- 使用 Redis 进行缓存和数据存储
- 使用 MongoDB 进行文档数据库操作
- 实现复杂查询和分页
- 使用规范（Specification）进行动态查询
- 实现审计和版本控制

## 1. Spring Data JPA

### 1.1 Maven 依赖

**Spring Initializr 依赖选择:**
- Spring Data JPA: `data-jpa`
- PostgreSQL Driver: `postgresql` (或其他数据库驱动)

```xml
<dependencies>
    <!-- Spring Data JPA -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    
    <!-- PostgreSQL -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

### 1.2 配置

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: postgres
    password: postgres
    hikari:
      maximum-pool-size: 10
      minimum-idle: 5
  
  jpa:
    hibernate:
      ddl-auto: validate  # 生产环境使用 validate
    show-sql: false
    properties:
      hibernate:
        format_sql: true
        dialect: org.hibernate.dialect.PostgreSQLDialect
        jdbc:
          batch_size: 20
        order_inserts: true
        order_updates: true
```

### 1.3 实体定义

```java
@Entity
@Table(name = "users")
public class User {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false, unique = true, length = 50)
    private String username;
    
    @Column(nullable = false, unique = true)
    private String email;
    
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;
    
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
    
    @PrePersist
    protected void onCreate() {
        createdAt = LocalDateTime.now();
    }
    
    @PreUpdate
    protected void onUpdate() {
        updatedAt = LocalDateTime.now();
    }
    
    // Getters and Setters
}
```

### 1.4 Repository 接口

**基础 Repository：**

```java
public interface UserRepository extends JpaRepository<User, Long> {
    
    // 查询方法命名规则
    Optional<User> findByUsername(String username);
    
    Optional<User> findByEmail(String email);
    
    List<User> findByUsernameContaining(String keyword);
    
    List<User> findByCreatedAtBetween(LocalDateTime start, LocalDateTime end);
    
    boolean existsByUsername(String username);
    
    long countByCreatedAtAfter(LocalDateTime date);
    
    void deleteByUsername(String username);
}
```

**自定义查询：**

```java
public interface UserRepository extends JpaRepository<User, Long> {
    
    // JPQL 查询
    @Query("SELECT u FROM User u WHERE u.username LIKE %:keyword%")
    List<User> searchByUsername(@Param("keyword") String keyword);
    
    // 原生 SQL 查询
    @Query(value = "SELECT * FROM users WHERE email = ?1", nativeQuery = true)
    Optional<User> findByEmailNative(String email);
    
    // 更新操作
    @Modifying
    @Query("UPDATE User u SET u.email = :email WHERE u.id = :id")
    int updateEmail(@Param("id") Long id, @Param("email") String email);
    
    // 投影查询
    @Query("SELECT u.username as username, u.email as email FROM User u WHERE u.id = :id")
    UserProjection findProjectionById(@Param("id") Long id);
}

// 投影接口
public interface UserProjection {
    String getUsername();
    String getEmail();
}
```

### 1.5 分页和排序

```java
@Service
public class UserService {
    
    private final UserRepository userRepository;
    
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
    
    public Page<User> getUsers(int page, int size) {
        Pageable pageable = PageRequest.of(page, size, Sort.by("createdAt").descending());
        return userRepository.findAll(pageable);
    }
    
    public Page<User> searchUsers(String keyword, int page, int size) {
        Pageable pageable = PageRequest.of(page, size);
        return userRepository.findByUsernameContaining(keyword, pageable);
    }
}
```

### 1.6 Specification 动态查询

```java
public class UserSpecifications {
    
    public static Specification<User> hasUsername(String username) {
        return (root, query, cb) -> 
            username == null ? null : cb.equal(root.get("username"), username);
    }
    
    public static Specification<User> emailContains(String email) {
        return (root, query, cb) -> 
            email == null ? null : cb.like(root.get("email"), "%" + email + "%");
    }
    
    public static Specification<User> createdAfter(LocalDateTime date) {
        return (root, query, cb) -> 
            date == null ? null : cb.greaterThan(root.get("createdAt"), date);
    }
}

// Repository
public interface UserRepository extends JpaRepository<User, Long>, JpaSpecificationExecutor<User> {
}

// Service
@Service
public class UserService {
    
    public List<User> searchUsers(String username, String email, LocalDateTime createdAfter) {
        Specification<User> spec = Specification.where(null);
        
        if (username != null) {
            spec = spec.and(UserSpecifications.hasUsername(username));
        }
        if (email != null) {
            spec = spec.and(UserSpecifications.emailContains(email));
        }
        if (createdAfter != null) {
            spec = spec.and(UserSpecifications.createdAfter(createdAfter));
        }
        
        return userRepository.findAll(spec);
    }
}
```

### 1.7 实体关系

**一对多：**

```java
@Entity
@Table(name = "orders")
public class Order {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;
    
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();
    
    // Helper methods
    public void addItem(OrderItem item) {
        items.add(item);
        item.setOrder(this);
    }
    
    public void removeItem(OrderItem item) {
        items.remove(item);
        item.setOrder(null);
    }
}

@Entity
@Table(name = "order_items")
public class OrderItem {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id", nullable = false)
    private Order order;
    
    private String productName;
    private Integer quantity;
    private BigDecimal price;
}
```

**多对多：**

```java
@Entity
@Table(name = "students")
public class Student {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    
    @ManyToMany
    @JoinTable(
        name = "student_courses",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    private Set<Course> courses = new HashSet<>();
}

@Entity
@Table(name = "courses")
public class Course {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    
    @ManyToMany(mappedBy = "courses")
    private Set<Student> students = new HashSet<>();
}
```

### 1.8 审计

```java
@Configuration
@EnableJpaAuditing
public class JpaConfig {
    
    @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> Optional.of(SecurityContextHolder.getContext()
                .getAuthentication()
                .getName());
    }
}

@Entity
@EntityListeners(AuditingEntityListener.class)
public class AuditableEntity {
    
    @CreatedDate
    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;
    
    @LastModifiedDate
    private LocalDateTime updatedAt;
    
    @CreatedBy
    @Column(nullable = false, updatable = false)
    private String createdBy;
    
    @LastModifiedBy
    private String updatedBy;
}
```

## 2. Spring Data JDBC

### 2.1 Maven 依赖

**Spring Initializr 依赖选择:**
- Spring Data JDBC: `data-jdbc`
- PostgreSQL Driver: `postgresql`

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jdbc</artifactId>
    </dependency>
    
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

### 2.2 实体定义

```java
@Table("users")
public class User {
    
    @Id
    private Long id;
    
    private String username;
    private String email;
    
    @CreatedDate
    private LocalDateTime createdAt;
    
    @LastModifiedDate
    private LocalDateTime updatedAt;
    
    // Getters and Setters
}
```

### 2.3 Repository

```java
public interface UserRepository extends CrudRepository<User, Long> {
    
    @Query("SELECT * FROM users WHERE username = :username")
    Optional<User> findByUsername(@Param("username") String username);
    
    @Modifying
    @Query("UPDATE users SET email = :email WHERE id = :id")
    boolean updateEmail(@Param("id") Long id, @Param("email") String email);
}
```

## 3. Spring Data Redis

### 3.1 Maven 依赖

**Spring Initializr 依赖选择:**
- Spring Data Redis: `data-redis`

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
</dependencies>
```

### 3.2 配置

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      password: 
      database: 0
      lettuce:
        pool:
          max-active: 8
          max-idle: 8
          min-idle: 0
```

### 3.3 RedisTemplate 使用

```java
@Configuration
public class RedisConfig {
    
    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        
        // 使用 Jackson 序列化
        Jackson2JsonRedisSerializer<Object> serializer = new Jackson2JsonRedisSerializer<>(Object.class);
        ObjectMapper mapper = new ObjectMapper();
        mapper.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        mapper.activateDefaultTyping(
            mapper.getPolymorphicTypeValidator(),
            ObjectMapper.DefaultTyping.NON_FINAL
        );
        serializer.setObjectMapper(mapper);
        
        // 设置序列化器
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(serializer);
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(serializer);
        
        template.afterPropertiesSet();
        return template;
    }
}
```

### 3.4 Redis 操作

```java
@Service
public class RedisService {
    
    private final RedisTemplate<String, Object> redisTemplate;
    
    public RedisService(RedisTemplate<String, Object> redisTemplate) {
        this.redisTemplate = redisTemplate;
    }
    
    // String 操作
    public void set(String key, Object value) {
        redisTemplate.opsForValue().set(key, value);
    }
    
    public void setWithExpire(String key, Object value, long timeout, TimeUnit unit) {
        redisTemplate.opsForValue().set(key, value, timeout, unit);
    }
    
    public Object get(String key) {
        return redisTemplate.opsForValue().get(key);
    }
    
    // Hash 操作
    public void hSet(String key, String field, Object value) {
        redisTemplate.opsForHash().put(key, field, value);
    }
    
    public Object hGet(String key, String field) {
        return redisTemplate.opsForHash().get(key, field);
    }
    
    // List 操作
    public void lPush(String key, Object value) {
        redisTemplate.opsForList().leftPush(key, value);
    }
    
    public Object lPop(String key) {
        return redisTemplate.opsForList().leftPop(key);
    }
    
    // Set 操作
    public void sAdd(String key, Object... values) {
        redisTemplate.opsForSet().add(key, values);
    }
    
    public Set<Object> sMembers(String key) {
        return redisTemplate.opsForSet().members(key);
    }
    
    // ZSet 操作
    public void zAdd(String key, Object value, double score) {
        redisTemplate.opsForZSet().add(key, value, score);
    }
    
    public Set<Object> zRange(String key, long start, long end) {
        return redisTemplate.opsForZSet().range(key, start, end);
    }
    
    // 删除
    public void delete(String key) {
        redisTemplate.delete(key);
    }
    
    // 设置过期时间
    public void expire(String key, long timeout, TimeUnit unit) {
        redisTemplate.expire(key, timeout, unit);
    }
}
```

### 3.5 Redis Repository

```java
@RedisHash("users")
public class User {
    
    @Id
    private String id;
    
    @Indexed
    private String username;
    
    private String email;
    
    @TimeToLive
    private Long ttl;
    
    // Getters and Setters
}

public interface UserRedisRepository extends CrudRepository<User, String> {
    
    Optional<User> findByUsername(String username);
}
```

## 4. Spring Data MongoDB

### 4.1 Maven 依赖

**Spring Initializr 依赖选择:**
- Spring Data MongoDB: `data-mongodb`

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-mongodb</artifactId>
    </dependency>
</dependencies>
```

### 4.2 配置

```yaml
spring:
  data:
    mongodb:
      uri: mongodb://localhost:27017/mydb
      # 或分开配置
      host: localhost
      port: 27017
      database: mydb
      username: user
      password: password
```

### 4.3 文档定义

```java
@Document(collection = "users")
public class User {
    
    @Id
    private String id;
    
    @Indexed(unique = true)
    private String username;
    
    private String email;
    
    @Field("created_at")
    private LocalDateTime createdAt;
    
    @DBRef
    private List<Order> orders;
    
    // Getters and Setters
}
```

### 4.4 Repository

```java
public interface UserMongoRepository extends MongoRepository<User, String> {
    
    Optional<User> findByUsername(String username);
    
    List<User> findByEmailContaining(String email);
    
    @Query("{ 'username': ?0 }")
    Optional<User> findByUsernameCustom(String username);
    
    @Query("{ 'createdAt': { $gte: ?0, $lte: ?1 } }")
    List<User> findByCreatedAtBetween(LocalDateTime start, LocalDateTime end);
}
```

### 4.5 MongoTemplate 使用

```java
@Service
public class UserMongoService {
    
    private final MongoTemplate mongoTemplate;
    
    public UserMongoService(MongoTemplate mongoTemplate) {
        this.mongoTemplate = mongoTemplate;
    }
    
    public List<User> findByUsernameRegex(String regex) {
        Query query = new Query();
        query.addCriteria(Criteria.where("username").regex(regex));
        return mongoTemplate.find(query, User.class);
    }
    
    public void updateEmail(String id, String email) {
        Query query = new Query(Criteria.where("id").is(id));
        Update update = new Update().set("email", email);
        mongoTemplate.updateFirst(query, update, User.class);
    }
    
    public List<User> findWithPagination(int page, int size) {
        Query query = new Query();
        query.skip((long) page * size);
        query.limit(size);
        return mongoTemplate.find(query, User.class);
    }
}
```

## 5. 查询方法命名规则

Spring Data 支持通过方法名自动生成查询：

| 关键字 | 示例 | JPQL 片段 |
|--------|------|-----------|
| `And` | `findByLastnameAndFirstname` | `where x.lastname = ?1 and x.firstname = ?2` |
| `Or` | `findByLastnameOrFirstname` | `where x.lastname = ?1 or x.firstname = ?2` |
| `Is`, `Equals` | `findByFirstname`, `findByFirstnameIs` | `where x.firstname = ?1` |
| `Between` | `findByStartDateBetween` | `where x.startDate between ?1 and ?2` |
| `LessThan` | `findByAgeLessThan` | `where x.age < ?1` |
| `LessThanEqual` | `findByAgeLessThanEqual` | `where x.age <= ?1` |
| `GreaterThan` | `findByAgeGreaterThan` | `where x.age > ?1` |
| `GreaterThanEqual` | `findByAgeGreaterThanEqual` | `where x.age >= ?1` |
| `After` | `findByStartDateAfter` | `where x.startDate > ?1` |
| `Before` | `findByStartDateBefore` | `where x.startDate < ?1` |
| `IsNull`, `Null` | `findByAge(Is)Null` | `where x.age is null` |
| `IsNotNull`, `NotNull` | `findByAge(Is)NotNull` | `where x.age not null` |
| `Like` | `findByFirstnameLike` | `where x.firstname like ?1` |
| `NotLike` | `findByFirstnameNotLike` | `where x.firstname not like ?1` |
| `StartingWith` | `findByFirstnameStartingWith` | `where x.firstname like ?1` (参数加 %) |
| `EndingWith` | `findByFirstnameEndingWith` | `where x.firstname like ?1` (参数加 %) |
| `Containing` | `findByFirstnameContaining` | `where x.firstname like ?1` (参数加 %%) |
| `OrderBy` | `findByAgeOrderByLastnameDesc` | `where x.age = ?1 order by x.lastname desc` |
| `Not` | `findByLastnameNot` | `where x.lastname <> ?1` |
| `In` | `findByAgeIn(Collection<Age> ages)` | `where x.age in ?1` |
| `NotIn` | `findByAgeNotIn(Collection<Age> ages)` | `where x.age not in ?1` |
| `True` | `findByActiveTrue()` | `where x.active = true` |
| `False` | `findByActiveFalse()` | `where x.active = false` |
| `IgnoreCase` | `findByFirstnameIgnoreCase` | `where UPPER(x.firstname) = UPPER(?1)` |

## 最佳实践

1. **选择合适的 Spring Data 模块**：
   - JPA：复杂对象关系映射
   - JDBC：轻量级、性能优先
   - Redis：缓存和高性能读写
   - MongoDB：文档数据库

2. **使用懒加载**：避免 N+1 查询问题
3. **使用投影**：只查询需要的字段
4. **使用分页**：处理大量数据
5. **使用 Specification**：动态查询
6. **启用审计**：跟踪数据变更
7. **合理使用缓存**：提高查询性能
8. **使用批量操作**：提高写入性能
9. **避免循环引用**：实体关系设计
10. **使用连接池**：优化数据库连接

## 参考资源

- [Spring Data JPA](https://docs.spring.io/spring-data/jpa/reference/)
- [Spring Data JDBC](https://docs.spring.io/spring-data/jdbc/reference/)
- [Spring Data Redis](https://docs.spring.io/spring-data/redis/reference/)
- [Spring Data MongoDB](https://docs.spring.io/spring-data/mongodb/reference/)
