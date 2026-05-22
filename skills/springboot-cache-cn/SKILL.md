---
name: springboot-cache-cn
description: Spring Boot 缓存实践，包括 Spring Cache、Redis、Caffeine 和缓存策略
---

# Spring Boot 缓存实践指南

当你需要在 Spring Boot 应用中使用缓存提升性能时使用此技能，包括 Spring Cache 抽象、Redis、Caffeine 本地缓存和各种缓存策略。

## 何时使用

- 减少数据库查询压力
- 提升应用响应速度
- 缓存热点数据
- 实现分布式缓存
- 实现多级缓存
- 处理缓存一致性问题

## Maven 依赖

```xml
<dependencies>
    <!-- Spring Cache -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-cache</artifactId>
    </dependency>
    
    <!-- Redis -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    
    <!-- Lettuce（Redis 客户端，Spring Boot 默认） -->
    <dependency>
        <groupId>io.lettuce</groupId>
        <artifactId>lettuce-core</artifactId>
    </dependency>
    
    <!-- Caffeine（本地缓存） -->
    <dependency>
        <groupId>com.github.ben-manes.caffeine</groupId>
        <artifactId>caffeine</artifactId>
    </dependency>
    
    <!-- Redisson（可选，高级 Redis 客户端） -->
    <dependency>
        <groupId>org.redisson</groupId>
        <artifactId>redisson-spring-boot-starter</artifactId>
        <version>3.24.3</version>
    </dependency>
</dependencies>
```

## 1. Spring Cache 基础

### 1.1 启用缓存

```java
package com.example.config;

import org.springframework.cache.annotation.EnableCaching;
import org.springframework.context.annotation.Configuration;

@Configuration
@EnableCaching
public class CacheConfig {
}
```

### 1.2 基本注解使用

```java
package com.example.service;

import org.springframework.cache.annotation.*;
import org.springframework.stereotype.Service;

@Service
public class UserService {
    
    // 缓存结果
    @Cacheable(value = "users", key = "#id")
    public User getUserById(Long id) {
        System.out.println("从数据库查询用户: " + id);
        return userRepository.findById(id).orElse(null);
    }
    
    // 条件缓存
    @Cacheable(value = "users", key = "#id", condition = "#id > 0")
    public User getUserByIdWithCondition(Long id) {
        return userRepository.findById(id).orElse(null);
    }
    
    // 排除某些结果不缓存
    @Cacheable(value = "users", key = "#id", unless = "#result == null")
    public User getUserByIdUnlessNull(Long id) {
        return userRepository.findById(id).orElse(null);
    }
    
    // 更新缓存
    @CachePut(value = "users", key = "#user.id")
    public User updateUser(User user) {
        return userRepository.save(user);
    }
    
    // 删除缓存
    @CacheEvict(value = "users", key = "#id")
    public void deleteUser(Long id) {
        userRepository.deleteById(id);
    }
    
    // 删除所有缓存
    @CacheEvict(value = "users", allEntries = true)
    public void deleteAllUsers() {
        userRepository.deleteAll();
    }
    
    // 组合操作
    @Caching(
        evict = {
            @CacheEvict(value = "users", key = "#user.id"),
            @CacheEvict(value = "userList", allEntries = true)
        },
        put = @CachePut(value = "users", key = "#user.id")
    )
    public User updateUserWithMultiCache(User user) {
        return userRepository.save(user);
    }
}
```

### 1.3 自定义缓存 Key

```java
package com.example.config;

import org.springframework.cache.interceptor.KeyGenerator;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.lang.reflect.Method;
import java.util.Arrays;

@Configuration
public class CacheKeyConfig {
    
    @Bean("customKeyGenerator")
    public KeyGenerator customKeyGenerator() {
        return (target, method, params) -> {
            return target.getClass().getSimpleName() + "_"
                + method.getName() + "_"
                + Arrays.toString(params);
        };
    }
}

// 使用自定义 Key Generator
@Service
public class ProductService {
    
    @Cacheable(value = "products", keyGenerator = "customKeyGenerator")
    public Product getProduct(Long id, String category) {
        return productRepository.findByIdAndCategory(id, category);
    }
}
```

## 2. Redis 缓存

### 2.1 Redis 配置

```yaml
# application.yml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      password: ${REDIS_PASSWORD:}
      database: 0
      timeout: 3000ms
      lettuce:
        pool:
          max-active: 8
          max-idle: 8
          min-idle: 0
          max-wait: -1ms
  cache:
    type: redis
    redis:
      time-to-live: 600000  # 10 分钟
      cache-null-values: false
      key-prefix: "app:"
      use-key-prefix: true
```

### 2.2 Redis 配置类

```java
package com.example.config;

import com.fasterxml.jackson.annotation.JsonTypeInfo;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.jsontype.impl.LaissezFaireSubTypeValidator;
import org.springframework.cache.CacheManager;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.*;

import java.time.Duration;
import java.util.HashMap;
import java.util.Map;

@Configuration
public class RedisCacheConfig {
    
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        // 默认配置
        RedisCacheConfiguration defaultConfig = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .serializeKeysWith(
                RedisSerializationContext.SerializationPair.fromSerializer(
                    new StringRedisSerializer()
                )
            )
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair.fromSerializer(
                    new GenericJackson2JsonRedisSerializer()
                )
            )
            .disableCachingNullValues();
        
        // 针对不同缓存的个性化配置
        Map<String, RedisCacheConfiguration> cacheConfigurations = new HashMap<>();
        
        // 用户缓存：1 小时
        cacheConfigurations.put("users",
            defaultConfig.entryTtl(Duration.ofHours(1))
        );
        
        // 产品缓存：30 分钟
        cacheConfigurations.put("products",
            defaultConfig.entryTtl(Duration.ofMinutes(30))
        );
        
        // 配置缓存：永不过期
        cacheConfigurations.put("config",
            defaultConfig.entryTtl(Duration.ZERO)
        );
        
        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(defaultConfig)
            .withInitialCacheConfigurations(cacheConfigurations)
            .transactionAware()
            .build();
    }
    
    @Bean
    public RedisTemplate<String, Object> redisTemplate(
        RedisConnectionFactory connectionFactory
    ) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);
        
        // Key 序列化
        template.setKeySerializer(new StringRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());
        
        // Value 序列化
        GenericJackson2JsonRedisSerializer serializer = 
            new GenericJackson2JsonRedisSerializer();
        template.setValueSerializer(serializer);
        template.setHashValueSerializer(serializer);
        
        template.afterPropertiesSet();
        return template;
    }
}
```

### 2.3 RedisTemplate 使用

```java
package com.example.service;

import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.util.concurrent.TimeUnit;

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
    
    public Boolean delete(String key) {
        return redisTemplate.delete(key);
    }
    
    public Boolean expire(String key, long timeout, TimeUnit unit) {
        return redisTemplate.expire(key, timeout, unit);
    }
    
    // Hash 操作
    public void hSet(String key, String field, Object value) {
        redisTemplate.opsForHash().put(key, field, value);
    }
    
    public Object hGet(String key, String field) {
        return redisTemplate.opsForHash().get(key, field);
    }
    
    // List 操作
    public Long lPush(String key, Object value) {
        return redisTemplate.opsForList().leftPush(key, value);
    }
    
    public Object lPop(String key) {
        return redisTemplate.opsForList().leftPop(key);
    }
    
    // Set 操作
    public Long sAdd(String key, Object... values) {
        return redisTemplate.opsForSet().add(key, values);
    }
    
    public Boolean sIsMember(String key, Object value) {
        return redisTemplate.opsForSet().isMember(key, value);
    }
    
    // ZSet 操作
    public Boolean zAdd(String key, Object value, double score) {
        return redisTemplate.opsForZSet().add(key, value, score);
    }
    
    public Set<Object> zRange(String key, long start, long end) {
        return redisTemplate.opsForZSet().range(key, start, end);
    }
}
```

## 3. Caffeine 本地缓存

### 3.1 Caffeine 配置

```yaml
# application.yml
spring:
  cache:
    type: caffeine
    caffeine:
      spec: maximumSize=1000,expireAfterWrite=10m
    cache-names:
      - users
      - products
      - config
```

### 3.2 Caffeine 配置类

```java
package com.example.config;

import com.github.benmanes.caffeine.cache.Caffeine;
import org.springframework.cache.CacheManager;
import org.springframework.cache.caffeine.CaffeineCacheManager;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.concurrent.TimeUnit;

@Configuration
public class CaffeineCacheConfig {
    
    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager(
            "users", "products", "config"
        );
        
        cacheManager.setCaffeine(Caffeine.newBuilder()
            .maximumSize(1000)
            .expireAfterWrite(10, TimeUnit.MINUTES)
            .recordStats()
        );
        
        return cacheManager;
    }
    
    // 针对不同缓存的个性化配置
    @Bean
    public CacheManager customCacheManager() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager();
        
        // 用户缓存
        cacheManager.registerCustomCache("users",
            Caffeine.newBuilder()
                .maximumSize(500)
                .expireAfterWrite(1, TimeUnit.HOURS)
                .build()
        );
        
        // 产品缓存
        cacheManager.registerCustomCache("products",
            Caffeine.newBuilder()
                .maximumSize(1000)
                .expireAfterWrite(30, TimeUnit.MINUTES)
                .build()
        );
        
        // 配置缓存（永不过期）
        cacheManager.registerCustomCache("config",
            Caffeine.newBuilder()
                .maximumSize(100)
                .build()
        );
        
        return cacheManager;
    }
}
```

## 4. 多级缓存

### 4.1 本地缓存 + Redis

```java
package com.example.config;

import org.springframework.cache.Cache;
import org.springframework.cache.CacheManager;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.cache.caffeine.CaffeineCacheManager;

import java.util.Collection;
import java.util.stream.Collectors;

public class MultiLevelCacheManager implements CacheManager {
    
    private final CaffeineCacheManager l1CacheManager;  // 一级缓存（本地）
    private final RedisCacheManager l2CacheManager;     // 二级缓存（Redis）
    
    public MultiLevelCacheManager(
        CaffeineCacheManager l1CacheManager,
        RedisCacheManager l2CacheManager
    ) {
        this.l1CacheManager = l1CacheManager;
        this.l2CacheManager = l2CacheManager;
    }
    
    @Override
    public Cache getCache(String name) {
        Cache l1Cache = l1CacheManager.getCache(name);
        Cache l2Cache = l2CacheManager.getCache(name);
        
        return new MultiLevelCache(name, l1Cache, l2Cache);
    }
    
    @Override
    public Collection<String> getCacheNames() {
        return l1CacheManager.getCacheNames().stream()
            .collect(Collectors.toSet());
    }
}

class MultiLevelCache implements Cache {
    
    private final String name;
    private final Cache l1Cache;
    private final Cache l2Cache;
    
    public MultiLevelCache(String name, Cache l1Cache, Cache l2Cache) {
        this.name = name;
        this.l1Cache = l1Cache;
        this.l2Cache = l2Cache;
    }
    
    @Override
    public String getName() {
        return name;
    }
    
    @Override
    public Object getNativeCache() {
        return this;
    }
    
    @Override
    public ValueWrapper get(Object key) {
        // 先查一级缓存
        ValueWrapper value = l1Cache.get(key);
        if (value != null) {
            return value;
        }
        
        // 再查二级缓存
        value = l2Cache.get(key);
        if (value != null) {
            // 回写到一级缓存
            l1Cache.put(key, value.get());
        }
        
        return value;
    }
    
    @Override
    public void put(Object key, Object value) {
        l1Cache.put(key, value);
        l2Cache.put(key, value);
    }
    
    @Override
    public void evict(Object key) {
        l1Cache.evict(key);
        l2Cache.evict(key);
    }
    
    @Override
    public void clear() {
        l1Cache.clear();
        l2Cache.clear();
    }
}
```

## 5. 缓存策略

### 5.1 Cache-Aside（旁路缓存）

```java
package com.example.service;

import org.springframework.stereotype.Service;

@Service
public class CacheAsideService {
    
    private final RedisService redisService;
    private final UserRepository userRepository;
    
    public User getUser(Long id) {
        String key = "user:" + id;
        
        // 1. 先查缓存
        User user = (User) redisService.get(key);
        if (user != null) {
            return user;
        }
        
        // 2. 缓存未命中，查数据库
        user = userRepository.findById(id).orElse(null);
        
        // 3. 写入缓存
        if (user != null) {
            redisService.setWithExpire(key, user, 10, TimeUnit.MINUTES);
        }
        
        return user;
    }
    
    public void updateUser(User user) {
        // 1. 更新数据库
        userRepository.save(user);
        
        // 2. 删除缓存
        String key = "user:" + user.getId();
        redisService.delete(key);
    }
}
```

### 5.2 Read-Through / Write-Through

```java
package com.example.cache;

import org.springframework.cache.annotation.CachePut;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Component;

@Component
public class ReadWriteThroughCache {
    
    private final UserRepository userRepository;
    
    // Read-Through：缓存自动加载
    @Cacheable(value = "users", key = "#id")
    public User getUser(Long id) {
        return userRepository.findById(id).orElse(null);
    }
    
    // Write-Through：同时更新缓存和数据库
    @CachePut(value = "users", key = "#user.id")
    public User updateUser(User user) {
        return userRepository.save(user);
    }
}
```

### 5.3 Write-Behind（异步写入）

```java
package com.example.cache;

import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Component;

import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;

@Component
public class WriteBehindCache {
    
    private final ConcurrentMap<Long, User> writeBuffer = new ConcurrentHashMap<>();
    private final UserRepository userRepository;
    
    public User getUser(Long id) {
        // 先查缓存
        User user = writeBuffer.get(id);
        if (user != null) {
            return user;
        }
        
        // 查数据库
        return userRepository.findById(id).orElse(null);
    }
    
    public void updateUser(User user) {
        // 立即更新缓存
        writeBuffer.put(user.getId(), user);
        
        // 异步写入数据库
        asyncWriteToDatabase(user);
    }
    
    @Async
    private void asyncWriteToDatabase(User user) {
        userRepository.save(user);
        writeBuffer.remove(user.getId());
    }
}
```

## 6. 缓存一致性

### 6.1 延迟双删

```java
package com.example.service;

import org.springframework.stereotype.Service;

import java.util.concurrent.TimeUnit;

@Service
public class DelayedDoubleDeleteService {
    
    private final RedisService redisService;
    private final UserRepository userRepository;
    
    public void updateUser(User user) {
        String key = "user:" + user.getId();
        
        // 1. 删除缓存
        redisService.delete(key);
        
        // 2. 更新数据库
        userRepository.save(user);
        
        // 3. 延迟后再次删除缓存
        try {
            TimeUnit.MILLISECONDS.sleep(500);
            redisService.delete(key);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

### 6.2 分布式锁

```java
package com.example.cache;

import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Component;

import java.util.concurrent.TimeUnit;

@Component
public class DistributedLockCache {
    
    private final RedisTemplate<String, Object> redisTemplate;
    private final UserRepository userRepository;
    
    public User getUserWithLock(Long id) {
        String key = "user:" + id;
        String lockKey = "lock:user:" + id;
        
        // 尝试获取锁
        Boolean locked = redisTemplate.opsForValue()
            .setIfAbsent(lockKey, "1", 10, TimeUnit.SECONDS);
        
        if (Boolean.TRUE.equals(locked)) {
            try {
                // 查缓存
                User user = (User) redisTemplate.opsForValue().get(key);
                if (user != null) {
                    return user;
                }
                
                // 查数据库
                user = userRepository.findById(id).orElse(null);
                
                // 写缓存
                if (user != null) {
                    redisTemplate.opsForValue().set(key, user, 10, TimeUnit.MINUTES);
                }
                
                return user;
            } finally {
                // 释放锁
                redisTemplate.delete(lockKey);
            }
        }
        
        // 获取锁失败，直接查数据库
        return userRepository.findById(id).orElse(null);
    }
}
```

### 6.3 使用 Redisson 分布式锁

```java
package com.example.cache;

import org.redisson.api.RLock;
import org.redisson.api.RedissonClient;
import org.springframework.stereotype.Component;

import java.util.concurrent.TimeUnit;

@Component
public class RedissonLockCache {
    
    private final RedissonClient redissonClient;
    private final RedisService redisService;
    private final UserRepository userRepository;
    
    public User getUserWithRedissonLock(Long id) {
        String key = "user:" + id;
        String lockKey = "lock:user:" + id;
        
        RLock lock = redissonClient.getLock(lockKey);
        
        try {
            // 尝试加锁，最多等待 3 秒，锁定 10 秒后自动释放
            if (lock.tryLock(3, 10, TimeUnit.SECONDS)) {
                try {
                    // 查缓存
                    User user = (User) redisService.get(key);
                    if (user != null) {
                        return user;
                    }
                    
                    // 查数据库
                    user = userRepository.findById(id).orElse(null);
                    
                    // 写缓存
                    if (user != null) {
                        redisService.setWithExpire(key, user, 10, TimeUnit.MINUTES);
                    }
                    
                    return user;
                } finally {
                    lock.unlock();
                }
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        return userRepository.findById(id).orElse(null);
    }
}
```

## 7. 缓存预热和更新

### 7.1 应用启动时预热

```java
package com.example.cache;

import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.stereotype.Component;

@Component
public class CacheWarmer implements ApplicationRunner {
    
    private final RedisService redisService;
    private final ConfigRepository configRepository;
    
    @Override
    public void run(ApplicationArguments args) {
        // 预热配置缓存
        List<Config> configs = configRepository.findAll();
        for (Config config : configs) {
            redisService.set("config:" + config.getKey(), config.getValue());
        }
        
        System.out.println("缓存预热完成");
    }
}
```

### 7.2 定时更新缓存

```java
package com.example.cache;

import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

@Component
public class CacheRefresher {
    
    private final RedisService redisService;
    private final ProductRepository productRepository;
    
    // 每小时更新热门产品缓存
    @Scheduled(cron = "0 0 * * * ?")
    public void refreshHotProducts() {
        List<Product> hotProducts = productRepository.findHotProducts();
        for (Product product : hotProducts) {
            redisService.setWithExpire(
                "product:" + product.getId(),
                product,
                1,
                TimeUnit.HOURS
            );
        }
    }
}
```

## 8. 缓存监控

```java
package com.example.cache;

import com.github.benmanes.caffeine.cache.stats.CacheStats;
import org.springframework.cache.CacheManager;
import org.springframework.cache.caffeine.CaffeineCache;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.HashMap;
import java.util.Map;

@RestController
@RequestMapping("/api/cache")
public class CacheMonitorController {
    
    private final CacheManager cacheManager;
    
    public CacheMonitorController(CacheManager cacheManager) {
        this.cacheManager = cacheManager;
    }
    
    @GetMapping("/stats")
    public Map<String, Object> getCacheStats() {
        Map<String, Object> stats = new HashMap<>();
        
        for (String cacheName : cacheManager.getCacheNames()) {
            CaffeineCache cache = (CaffeineCache) cacheManager.getCache(cacheName);
            if (cache != null) {
                CacheStats cacheStats = cache.getNativeCache().stats();
                
                Map<String, Object> cacheInfo = new HashMap<>();
                cacheInfo.put("hitCount", cacheStats.hitCount());
                cacheInfo.put("missCount", cacheStats.missCount());
                cacheInfo.put("hitRate", cacheStats.hitRate());
                cacheInfo.put("evictionCount", cacheStats.evictionCount());
                
                stats.put(cacheName, cacheInfo);
            }
        }
        
        return stats;
    }
}
```

## 配置文件

```yaml
# application.yml
spring:
  cache:
    type: redis
  data:
    redis:
      host: localhost
      port: 6379
      password: ${REDIS_PASSWORD:}
      database: 0
      timeout: 3000ms
      lettuce:
        pool:
          max-active: 8
          max-idle: 8
          min-idle: 0
          max-wait: -1ms
```

## 相关技能

- `/springboot-project-planner-cn` - 规划缓存需求
- `/springboot-observability-cn` - 监控缓存性能
- `/springboot-testing-cn` - 测试缓存功能
