---
name: spring-framework-cn
description: Spring Framework 核心框架实践，包括 IoC/DI、AOP、事件机制、资源管理和数据绑定
---

# Spring Framework 核心框架实践指南

当你需要深入理解和使用 Spring Framework 核心功能时使用此技能，包括依赖注入、面向切面编程、事件机制、资源管理等。

## 何时使用

- 理解 Spring IoC 容器和依赖注入
- 使用 Spring AOP 实现横切关注点
- 实现应用内事件驱动架构
- 管理应用资源和配置
- 数据绑定和验证
- 国际化和本地化
- 任务调度和异步执行

## Maven 依赖

```xml
<dependencies>
    <!-- Spring Context（包含 Core、Beans、Context、Expression） -->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
        <version>6.1.0</version>
    </dependency>
    
    <!-- Spring AOP -->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-aop</artifactId>
        <version>6.1.0</version>
    </dependency>
    
    <!-- AspectJ（用于 AOP） -->
    <dependency>
        <groupId>org.aspectj</groupId>
        <artifactId>aspectjweaver</artifactId>
        <version>1.9.20</version>
    </dependency>
</dependencies>
```

## 1. IoC 容器和依赖注入

### 1.1 Bean 定义方式

**方式 1：基于注解（推荐）**

```java
@Configuration
public class AppConfig {
    
    @Bean
    public UserService userService(UserRepository userRepository) {
        return new UserService(userRepository);
    }
    
    @Bean
    public UserRepository userRepository() {
        return new JpaUserRepository();
    }
}
```

**方式 2：组件扫描**

```java
@Configuration
@ComponentScan(basePackages = "com.example")
public class AppConfig {
}

@Service
public class UserService {
    
    private final UserRepository userRepository;
    
    // 构造器注入（推荐）
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}

@Repository
public class JpaUserRepository implements UserRepository {
    // ...
}
```

**方式 3：Java Config**

```java
@Configuration
public class DataSourceConfig {
    
    @Bean
    public DataSource dataSource(
            @Value("${db.url}") String url,
            @Value("${db.username}") String username,
            @Value("${db.password}") String password) {
        
        HikariDataSource dataSource = new HikariDataSource();
        dataSource.setJdbcUrl(url);
        dataSource.setUsername(username);
        dataSource.setPassword(password);
        return dataSource;
    }
}
```

### 1.2 依赖注入方式

**构造器注入（推荐）：**

```java
@Service
public class OrderService {
    
    private final OrderRepository orderRepository;
    private final PaymentService paymentService;
    
    // 单个构造器时 @Autowired 可省略
    public OrderService(OrderRepository orderRepository, 
                       PaymentService paymentService) {
        this.orderRepository = orderRepository;
        this.paymentService = paymentService;
    }
}
```

**Setter 注入：**

```java
@Service
public class NotificationService {
    
    private EmailService emailService;
    
    @Autowired
    public void setEmailService(EmailService emailService) {
        this.emailService = emailService;
    }
}
```

**字段注入（不推荐）：**

```java
@Service
public class UserService {
    
    @Autowired
    private UserRepository userRepository;  // 难以测试
}
```

### 1.3 Bean 作用域

```java
@Configuration
public class BeanScopeConfig {
    
    // 单例（默认）
    @Bean
    @Scope("singleton")
    public SingletonBean singletonBean() {
        return new SingletonBean();
    }
    
    // 原型
    @Bean
    @Scope("prototype")
    public PrototypeBean prototypeBean() {
        return new PrototypeBean();
    }
    
    // Web 作用域（需要 Web 环境）
    @Bean
    @Scope(value = WebApplicationContext.SCOPE_REQUEST, proxyMode = ScopedProxyMode.TARGET_CLASS)
    public RequestBean requestBean() {
        return new RequestBean();
    }
    
    @Bean
    @Scope(value = WebApplicationContext.SCOPE_SESSION, proxyMode = ScopedProxyMode.TARGET_CLASS)
    public SessionBean sessionBean() {
        return new SessionBean();
    }
}
```

### 1.4 Bean 生命周期

```java
@Component
public class LifecycleBean implements InitializingBean, DisposableBean {
    
    // 1. 构造器
    public LifecycleBean() {
        System.out.println("1. 构造器");
    }
    
    // 2. 依赖注入
    @Autowired
    public void setDependency(SomeDependency dependency) {
        System.out.println("2. 依赖注入");
    }
    
    // 3. @PostConstruct
    @PostConstruct
    public void postConstruct() {
        System.out.println("3. @PostConstruct");
    }
    
    // 4. InitializingBean.afterPropertiesSet()
    @Override
    public void afterPropertiesSet() {
        System.out.println("4. afterPropertiesSet");
    }
    
    // 5. @Bean(initMethod)
    public void customInit() {
        System.out.println("5. customInit");
    }
    
    // 销毁前：@PreDestroy
    @PreDestroy
    public void preDestroy() {
        System.out.println("销毁前：@PreDestroy");
    }
    
    // 销毁：DisposableBean.destroy()
    @Override
    public void destroy() {
        System.out.println("销毁：destroy");
    }
    
    // 销毁：@Bean(destroyMethod)
    public void customDestroy() {
        System.out.println("销毁：customDestroy");
    }
}
```

### 1.5 条件化配置

```java
@Configuration
public class ConditionalConfig {
    
    // 当类存在时
    @Bean
    @ConditionalOnClass(name = "com.example.SomeClass")
    public SomeService someService() {
        return new SomeService();
    }
    
    // 当 Bean 不存在时
    @Bean
    @ConditionalOnMissingBean(DataSource.class)
    public DataSource defaultDataSource() {
        return new EmbeddedDataSource();
    }
    
    // 当属性匹配时
    @Bean
    @ConditionalOnProperty(name = "feature.enabled", havingValue = "true")
    public FeatureService featureService() {
        return new FeatureService();
    }
}
```

## 2. 面向切面编程（AOP）

### 2.1 启用 AOP

```java
@Configuration
@EnableAspectJAutoProxy
public class AopConfig {
}
```

### 2.2 定义切面

```java
@Aspect
@Component
public class LoggingAspect {
    
    private static final Logger log = LoggerFactory.getLogger(LoggingAspect.class);
    
    // 切点表达式
    @Pointcut("execution(* com.example.service.*.*(..))")
    public void serviceMethods() {}
    
    // 前置通知
    @Before("serviceMethods()")
    public void logBefore(JoinPoint joinPoint) {
        log.info("调用方法: {}", joinPoint.getSignature().getName());
    }
    
    // 后置通知
    @AfterReturning(pointcut = "serviceMethods()", returning = "result")
    public void logAfterReturning(JoinPoint joinPoint, Object result) {
        log.info("方法返回: {}, 结果: {}", joinPoint.getSignature().getName(), result);
    }
    
    // 异常通知
    @AfterThrowing(pointcut = "serviceMethods()", throwing = "error")
    public void logAfterThrowing(JoinPoint joinPoint, Throwable error) {
        log.error("方法异常: {}, 错误: {}", joinPoint.getSignature().getName(), error.getMessage());
    }
    
    // 环绕通知
    @Around("serviceMethods()")
    public Object logAround(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.currentTimeMillis();
        
        try {
            Object result = joinPoint.proceed();
            long duration = System.currentTimeMillis() - start;
            log.info("方法执行: {}, 耗时: {}ms", joinPoint.getSignature().getName(), duration);
            return result;
        } catch (Throwable e) {
            log.error("方法执行失败: {}", joinPoint.getSignature().getName(), e);
            throw e;
        }
    }
}
```

### 2.3 切点表达式

```java
@Aspect
@Component
public class PointcutExamples {
    
    // 匹配所有 public 方法
    @Pointcut("execution(public * *(..))")
    public void publicMethods() {}
    
    // 匹配特定包下的所有方法
    @Pointcut("execution(* com.example.service..*.*(..))")
    public void servicePackage() {}
    
    // 匹配特定注解
    @Pointcut("@annotation(com.example.annotation.Loggable)")
    public void loggableMethods() {}
    
    // 匹配特定类
    @Pointcut("within(com.example.service.UserService)")
    public void userServiceMethods() {}
    
    // 匹配带有特定注解的类
    @Pointcut("@within(org.springframework.stereotype.Service)")
    public void serviceBeans() {}
    
    // 组合切点
    @Pointcut("serviceMethods() && loggableMethods()")
    public void loggableServiceMethods() {}
}
```

### 2.4 自定义注解 + AOP

**定义注解：**

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Auditable {
    String operation() default "";
}
```

**定义切面：**

```java
@Aspect
@Component
public class AuditAspect {
    
    @Around("@annotation(auditable)")
    public Object audit(ProceedingJoinPoint joinPoint, Auditable auditable) throws Throwable {
        String operation = auditable.operation();
        String user = SecurityContextHolder.getContext().getAuthentication().getName();
        
        log.info("审计: 用户={}, 操作={}, 方法={}", 
                user, operation, joinPoint.getSignature().getName());
        
        Object result = joinPoint.proceed();
        
        log.info("审计完成: 用户={}, 操作={}", user, operation);
        
        return result;
    }
}
```

**使用：**

```java
@Service
public class UserService {
    
    @Auditable(operation = "创建用户")
    public User createUser(User user) {
        // 创建用户逻辑
        return user;
    }
}
```

## 3. 事件机制

### 3.1 定义事件

```java
public class UserRegisteredEvent extends ApplicationEvent {
    
    private final String username;
    private final String email;
    
    public UserRegisteredEvent(Object source, String username, String email) {
        super(source);
        this.username = username;
        this.email = email;
    }
    
    // Getters
}
```

### 3.2 发布事件

```java
@Service
public class UserService {
    
    private final ApplicationEventPublisher eventPublisher;
    
    public UserService(ApplicationEventPublisher eventPublisher) {
        this.eventPublisher = eventPublisher;
    }
    
    public void registerUser(String username, String email) {
        // 注册用户逻辑
        
        // 发布事件
        eventPublisher.publishEvent(new UserRegisteredEvent(this, username, email));
    }
}
```

### 3.3 监听事件

**方式 1：@EventListener**

```java
@Component
public class UserEventListener {
    
    @EventListener
    public void handleUserRegistered(UserRegisteredEvent event) {
        System.out.println("用户注册: " + event.getUsername());
        // 发送欢迎邮件
    }
    
    // 异步处理
    @EventListener
    @Async
    public void handleUserRegisteredAsync(UserRegisteredEvent event) {
        // 异步处理
    }
    
    // 条件监听
    @EventListener(condition = "#event.email.endsWith('@example.com')")
    public void handleInternalUserRegistered(UserRegisteredEvent event) {
        // 只处理内部用户
    }
}
```

**方式 2：ApplicationListener 接口**

```java
@Component
public class UserRegisteredListener implements ApplicationListener<UserRegisteredEvent> {
    
    @Override
    public void onApplicationEvent(UserRegisteredEvent event) {
        System.out.println("用户注册: " + event.getUsername());
    }
}
```

### 3.4 事务事件

```java
@Component
public class TransactionalEventListener {
    
    // 事务提交后执行
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleAfterCommit(UserRegisteredEvent event) {
        // 事务成功提交后发送通知
    }
    
    // 事务回滚后执行
    @TransactionalEventListener(phase = TransactionPhase.AFTER_ROLLBACK)
    public void handleAfterRollback(UserRegisteredEvent event) {
        // 事务回滚后的处理
    }
    
    // 事务完成后执行（无论成功或失败）
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMPLETION)
    public void handleAfterCompletion(UserRegisteredEvent event) {
        // 事务完成后的清理工作
    }
}
```

## 4. 资源管理

### 4.1 Resource 接口

```java
@Service
public class ResourceService {
    
    @Autowired
    private ResourceLoader resourceLoader;
    
    public void loadResource() throws IOException {
        // 类路径资源
        Resource resource1 = resourceLoader.getResource("classpath:config.properties");
        
        // 文件系统资源
        Resource resource2 = resourceLoader.getResource("file:/path/to/file.txt");
        
        // URL 资源
        Resource resource3 = resourceLoader.getResource("https://example.com/data.json");
        
        // 读取资源
        try (InputStream is = resource1.getInputStream()) {
            // 处理输入流
        }
    }
}
```

### 4.2 @Value 注入资源

```java
@Component
public class ConfigComponent {
    
    @Value("${app.name}")
    private String appName;
    
    @Value("${app.timeout:30}")
    private int timeout;  // 默认值 30
    
    @Value("#{systemProperties['user.home']}")
    private String userHome;
    
    @Value("classpath:data.txt")
    private Resource dataFile;
}
```

## 5. 数据绑定和验证

### 5.1 数据绑定

```java
@RestController
public class UserController {
    
    @PostMapping("/users")
    public User createUser(@RequestBody @Valid UserDto userDto) {
        // Spring 自动将 JSON 绑定到 UserDto
        return userService.create(userDto);
    }
}
```

### 5.2 Bean Validation

```java
public class UserDto {
    
    @NotBlank(message = "用户名不能为空")
    @Size(min = 3, max = 20, message = "用户名长度必须在 3-20 之间")
    private String username;
    
    @NotBlank(message = "邮箱不能为空")
    @Email(message = "邮箱格式不正确")
    private String email;
    
    @NotNull(message = "年龄不能为空")
    @Min(value = 18, message = "年龄必须大于等于 18")
    @Max(value = 100, message = "年龄必须小于等于 100")
    private Integer age;
    
    @Pattern(regexp = "^1[3-9]\\d{9}$", message = "手机号格式不正确")
    private String phone;
    
    // Getters and Setters
}
```

### 5.3 自定义验证器

**定义注解：**

```java
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = UsernameValidator.class)
public @interface ValidUsername {
    String message() default "用户名格式不正确";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

**实现验证器：**

```java
public class UsernameValidator implements ConstraintValidator<ValidUsername, String> {
    
    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        if (value == null) {
            return false;
        }
        // 自定义验证逻辑
        return value.matches("^[a-zA-Z0-9_]{3,20}$");
    }
}
```

**使用：**

```java
public class UserDto {
    
    @ValidUsername
    private String username;
}
```

## 6. 任务调度

### 6.1 启用调度

```java
@Configuration
@EnableScheduling
public class SchedulingConfig {
}
```

### 6.2 定时任务

```java
@Component
public class ScheduledTasks {
    
    // 固定延迟（上次执行完成后延迟）
    @Scheduled(fixedDelay = 5000)
    public void taskWithFixedDelay() {
        System.out.println("固定延迟任务: " + System.currentTimeMillis());
    }
    
    // 固定频率（上次执行开始后延迟）
    @Scheduled(fixedRate = 5000)
    public void taskWithFixedRate() {
        System.out.println("固定频率任务: " + System.currentTimeMillis());
    }
    
    // 初始延迟
    @Scheduled(initialDelay = 1000, fixedRate = 5000)
    public void taskWithInitialDelay() {
        System.out.println("初始延迟任务: " + System.currentTimeMillis());
    }
    
    // Cron 表达式
    @Scheduled(cron = "0 0 * * * ?")  // 每小时执行
    public void taskWithCron() {
        System.out.println("Cron 任务: " + System.currentTimeMillis());
    }
    
    // 从配置读取
    @Scheduled(cron = "${task.cron}")
    public void taskWithConfigurableCron() {
        System.out.println("可配置 Cron 任务");
    }
}
```

## 7. 异步执行

### 7.1 启用异步

```java
@Configuration
@EnableAsync
public class AsyncConfig {
    
    @Bean
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("async-");
        executor.initialize();
        return executor;
    }
}
```

### 7.2 异步方法

```java
@Service
public class AsyncService {
    
    @Async
    public void asyncMethod() {
        System.out.println("异步执行: " + Thread.currentThread().getName());
    }
    
    @Async
    public CompletableFuture<String> asyncMethodWithReturn() {
        // 模拟耗时操作
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        return CompletableFuture.completedFuture("结果");
    }
}
```

## 8. 国际化

### 8.1 配置消息源

```java
@Configuration
public class I18nConfig {
    
    @Bean
    public MessageSource messageSource() {
        ResourceBundleMessageSource messageSource = new ResourceBundleMessageSource();
        messageSource.setBasename("messages");
        messageSource.setDefaultEncoding("UTF-8");
        return messageSource;
    }
}
```

### 8.2 消息文件

**messages_zh_CN.properties：**
```properties
user.created=用户创建成功
user.notfound=用户不存在
```

**messages_en_US.properties：**
```properties
user.created=User created successfully
user.notfound=User not found
```

### 8.3 使用消息

```java
@Service
public class MessageService {
    
    @Autowired
    private MessageSource messageSource;
    
    public String getMessage(String code, Locale locale) {
        return messageSource.getMessage(code, null, locale);
    }
    
    public String getMessageWithArgs(String code, Object[] args, Locale locale) {
        return messageSource.getMessage(code, args, locale);
    }
}
```

## 最佳实践

1. **优先使用构造器注入**：更易测试，依赖明确
2. **使用 @Component 系列注解**：@Service、@Repository、@Controller
3. **合理使用 AOP**：日志、事务、安全等横切关注点
4. **使用事件解耦**：模块间通过事件通信
5. **Bean 作用域选择**：默认单例，按需使用其他作用域
6. **异步处理**：耗时操作使用 @Async
7. **资源管理**：使用 Resource 接口统一资源访问
8. **数据验证**：使用 Bean Validation 验证输入
9. **国际化**：支持多语言的应用使用 MessageSource
10. **任务调度**：使用 @Scheduled 实现定时任务

## 参考资源

- [Spring Framework 官方文档](https://docs.spring.io/spring-framework/reference/)
- [Spring IoC 容器](https://docs.spring.io/spring-framework/reference/core/beans.html)
- [Spring AOP](https://docs.spring.io/spring-framework/reference/core/aop.html)
- [Spring 事件](https://docs.spring.io/spring-framework/reference/core/beans/context-introduction.html#context-functionality-events)
