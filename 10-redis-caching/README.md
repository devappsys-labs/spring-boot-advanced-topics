# Redis and Spring Caching

## Overview

Redis is an in-memory data structure store used as a database, cache, and message broker. Spring Boot provides seamless integration with Redis for caching, session management, and distributed data storage.

## Key Concepts

### Redis Data Structures

- **Strings** - Simple key-value pairs
- **Hashes** - Maps between string fields and values
- **Lists** - Collections of strings sorted by insertion order
- **Sets** - Unordered collections of unique strings
- **Sorted Sets** - Sets ordered by score
- **Streams** - Log-like data structure
- **Bitmaps** - Bit-level operations
- **HyperLogLogs** - Probabilistic data structure

### Spring Cache Abstraction

Spring provides a declarative caching mechanism using annotations:

- `@Cacheable` - Cache method results
- `@CachePut` - Update cache
- `@CacheEvict` - Remove from cache
- `@Caching` - Group multiple cache operations
- `@CacheConfig` - Class-level cache configuration

## Dependencies

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-cache</artifactId>
    </dependency>
    <dependency>
        <groupId>redis.clients</groupId>
        <artifactId>jedis</artifactId>
    </dependency>
</dependencies>
```

## Configuration

### Application Properties

```properties
# Redis Configuration
spring.data.redis.host=localhost
spring.data.redis.port=6379
spring.data.redis.password=
spring.data.redis.database=0
spring.data.redis.timeout=60000
spring.data.redis.jedis.pool.max-active=8
spring.data.redis.jedis.pool.max-idle=8
spring.data.redis.jedis.pool.min-idle=0

# Cache Configuration
spring.cache.type=redis
spring.cache.redis.time-to-live=600000
spring.cache.redis.cache-null-values=true
spring.cache.redis.use-key-prefix=true
```

### Redis Configuration Class

```java
@Configuration
@EnableCaching
public class RedisConfig {
    
    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory connectionFactory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);
        
        Jackson2JsonRedisSerializer<Object> serializer = new Jackson2JsonRedisSerializer<>(Object.class);
        ObjectMapper mapper = new ObjectMapper();
        mapper.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        mapper.activateDefaultTyping(mapper.getPolymorphicTypeValidator(), 
                                     ObjectMapper.DefaultTyping.NON_FINAL);
        serializer.setObjectMapper(mapper);
        
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(serializer);
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(serializer);
        template.afterPropertiesSet();
        
        return template;
    }
    
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(10))
                .serializeKeysWith(RedisSerializationContext.SerializationPair
                        .fromSerializer(new StringRedisSerializer()))
                .serializeValuesWith(RedisSerializationContext.SerializationPair
                        .fromSerializer(new GenericJackson2JsonRedisSerializer()))
                .disableCachingNullValues();
        
        return RedisCacheManager.builder(connectionFactory)
                .cacheDefaults(config)
                .transactionAware()
                .build();
    }
}
```

## Basic Caching Examples

### Service with Caching

```java
@Service
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Cacheable(value = "users", key = "#id")
    public User getUserById(Long id) {
        System.out.println("Fetching user from database: " + id);
        return userRepository.findById(id)
                .orElseThrow(() -> new EntityNotFoundException("User not found"));
    }
    
    @Cacheable(value = "users", key = "#email")
    public User getUserByEmail(String email) {
        System.out.println("Fetching user by email from database: " + email);
        return userRepository.findByEmail(email)
                .orElseThrow(() -> new EntityNotFoundException("User not found"));
    }
    
    @CachePut(value = "users", key = "#user.id")
    public User updateUser(User user) {
        System.out.println("Updating user in database: " + user.getId());
        return userRepository.save(user);
    }
    
    @CacheEvict(value = "users", key = "#id")
    public void deleteUser(Long id) {
        System.out.println("Deleting user from database: " + id);
        userRepository.deleteById(id);
    }
    
    @CacheEvict(value = "users", allEntries = true)
    public void clearCache() {
        System.out.println("Clearing all users cache");
    }
}
```

### Conditional Caching

```java
@Service
public class ProductService {
    
    @Cacheable(value = "products", key = "#id", condition = "#id > 0", unless = "#result == null")
    public Product getProduct(Long id) {
        return productRepository.findById(id).orElse(null);
    }
    
    @Cacheable(value = "products", key = "#category", condition = "#category != 'uncategorized'")
    public List<Product> getProductsByCategory(String category) {
        return productRepository.findByCategory(category);
    }
}
```

### Multiple Cache Operations

```java
@Service
public class OrderService {
    
    @Caching(
        cacheable = {
            @Cacheable(value = "orders", key = "#id")
        },
        put = {
            @CachePut(value = "recentOrders", key = "#id")
        }
    )
    public Order getOrder(Long id) {
        return orderRepository.findById(id).orElse(null);
    }
    
    @Caching(evict = {
        @CacheEvict(value = "orders", key = "#id"),
        @CacheEvict(value = "recentOrders", key = "#id"),
        @CacheEvict(value = "orderStats", allEntries = true)
    })
    public void deleteOrder(Long id) {
        orderRepository.deleteById(id);
    }
}
```

## Direct Redis Operations

### Using RedisTemplate

```java
@Service
public class RedisService {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    // String Operations
    public void setValue(String key, Object value) {
        redisTemplate.opsForValue().set(key, value);
    }
    
    public void setValueWithExpiry(String key, Object value, long timeout, TimeUnit unit) {
        redisTemplate.opsForValue().set(key, value, timeout, unit);
    }
    
    public Object getValue(String key) {
        return redisTemplate.opsForValue().get(key);
    }
    
    public Boolean deleteKey(String key) {
        return redisTemplate.delete(key);
    }
    
    // Hash Operations
    public void setHashValue(String key, String hashKey, Object value) {
        redisTemplate.opsForHash().put(key, hashKey, value);
    }
    
    public Object getHashValue(String key, String hashKey) {
        return redisTemplate.opsForHash().get(key, hashKey);
    }
    
    public Map<Object, Object> getHashEntries(String key) {
        return redisTemplate.opsForHash().entries(key);
    }
    
    // List Operations
    public Long pushToList(String key, Object value) {
        return redisTemplate.opsForList().rightPush(key, value);
    }
    
    public Object popFromList(String key) {
        return redisTemplate.opsForList().leftPop(key);
    }
    
    public List<Object> getListRange(String key, long start, long end) {
        return redisTemplate.opsForList().range(key, start, end);
    }
    
    // Set Operations
    public Long addToSet(String key, Object... values) {
        return redisTemplate.opsForSet().add(key, values);
    }
    
    public Set<Object> getSetMembers(String key) {
        return redisTemplate.opsForSet().members(key);
    }
    
    public Boolean isMember(String key, Object value) {
        return redisTemplate.opsForSet().isMember(key, value);
    }
    
    // Sorted Set Operations
    public Boolean addToSortedSet(String key, Object value, double score) {
        return redisTemplate.opsForZSet().add(key, value, score);
    }
    
    public Set<Object> getSortedSetRange(String key, long start, long end) {
        return redisTemplate.opsForZSet().range(key, start, end);
    }
    
    public Set<Object> getSortedSetRangeByScore(String key, double min, double max) {
        return redisTemplate.opsForZSet().rangeByScore(key, min, max);
    }
}
```

### Using StringRedisTemplate

```java
@Service
public class StringRedisService {
    
    @Autowired
    private StringRedisTemplate stringRedisTemplate;
    
    public void increment(String key) {
        stringRedisTemplate.opsForValue().increment(key);
    }
    
    public void incrementBy(String key, long delta) {
        stringRedisTemplate.opsForValue().increment(key, delta);
    }
    
    public Long getCounter(String key) {
        String value = stringRedisTemplate.opsForValue().get(key);
        return value != null ? Long.parseLong(value) : 0L;
    }
    
    public Boolean setIfAbsent(String key, String value, long timeout, TimeUnit unit) {
        return stringRedisTemplate.opsForValue().setIfAbsent(key, value, timeout, unit);
    }
}
```

## Advanced Patterns

### Distributed Lock

```java
@Component
public class RedisDistributedLock {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    public boolean tryLock(String lockKey, String requestId, long expireTime) {
        Boolean result = redisTemplate.opsForValue()
                .setIfAbsent(lockKey, requestId, expireTime, TimeUnit.MILLISECONDS);
        return Boolean.TRUE.equals(result);
    }
    
    public boolean releaseLock(String lockKey, String requestId) {
        String script = "if redis.call('get', KEYS[1]) == ARGV[1] then " +
                       "return redis.call('del', KEYS[1]) else return 0 end";
        
        Long result = redisTemplate.execute(
                new DefaultRedisScript<>(script, Long.class),
                Collections.singletonList(lockKey),
                requestId
        );
        
        return Long.valueOf(1L).equals(result);
    }
    
    public <T> T executeWithLock(String lockKey, long waitTime, long leaseTime, Supplier<T> supplier) {
        String requestId = UUID.randomUUID().toString();
        long start = System.currentTimeMillis();
        
        try {
            while (System.currentTimeMillis() - start < waitTime) {
                if (tryLock(lockKey, requestId, leaseTime)) {
                    try {
                        return supplier.get();
                    } finally {
                        releaseLock(lockKey, requestId);
                    }
                }
                Thread.sleep(100);
            }
            throw new RuntimeException("Failed to acquire lock");
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException("Interrupted while waiting for lock", e);
        }
    }
}
```

### Cache-Aside Pattern

```java
@Service
public class CacheAsideService {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Autowired
    private DatabaseRepository repository;
    
    public Object getData(String key) {
        // Try cache first
        Object cached = redisTemplate.opsForValue().get(key);
        if (cached != null) {
            return cached;
        }
        
        // Cache miss - fetch from database
        Object data = repository.findByKey(key);
        
        // Update cache
        if (data != null) {
            redisTemplate.opsForValue().set(key, data, 10, TimeUnit.MINUTES);
        }
        
        return data;
    }
    
    public void updateData(String key, Object data) {
        // Update database
        repository.save(data);
        
        // Invalidate cache
        redisTemplate.delete(key);
    }
}
```

### Rate Limiting with Redis

```java
@Component
public class RedisRateLimiter {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    public boolean isAllowed(String key, int maxRequests, int windowSeconds) {
        String redisKey = "rate_limit:" + key;
        long currentTime = System.currentTimeMillis();
        long windowStart = currentTime - (windowSeconds * 1000L);
        
        // Remove old entries
        redisTemplate.opsForZSet().removeRangeByScore(redisKey, 0, windowStart);
        
        // Count requests in current window
        Long count = redisTemplate.opsForZSet().count(redisKey, windowStart, currentTime);
        
        if (count != null && count < maxRequests) {
            // Add current request
            redisTemplate.opsForZSet().add(redisKey, String.valueOf(currentTime), currentTime);
            redisTemplate.expire(redisKey, windowSeconds, TimeUnit.SECONDS);
            return true;
        }
        
        return false;
    }
}
```

### Session Management

```java
@Configuration
@EnableRedisHttpSession(maxInactiveIntervalInSeconds = 3600)
public class RedisSessionConfig {
    // Spring Session with Redis
}

@RestController
@RequestMapping("/session")
public class SessionController {
    
    @GetMapping("/set")
    public String setSession(HttpSession session) {
        session.setAttribute("user", "John Doe");
        return "Session attribute set";
    }
    
    @GetMapping("/get")
    public String getSession(HttpSession session) {
        return (String) session.getAttribute("user");
    }
}
```

## Cache Configuration Strategies

### Custom Cache Names with Different TTLs

```java
@Configuration
@EnableCaching
public class CacheConfig {
    
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        Map<String, RedisCacheConfiguration> cacheConfigurations = new HashMap<>();
        
        // Short-lived cache (1 minute)
        cacheConfigurations.put("shortCache", 
                createCacheConfiguration(Duration.ofMinutes(1)));
        
        // Medium-lived cache (10 minutes)
        cacheConfigurations.put("mediumCache", 
                createCacheConfiguration(Duration.ofMinutes(10)));
        
        // Long-lived cache (1 hour)
        cacheConfigurations.put("longCache", 
                createCacheConfiguration(Duration.ofHours(1)));
        
        return RedisCacheManager.builder(connectionFactory)
                .cacheDefaults(createCacheConfiguration(Duration.ofMinutes(5)))
                .withInitialCacheConfigurations(cacheConfigurations)
                .transactionAware()
                .build();
    }
    
    private RedisCacheConfiguration createCacheConfiguration(Duration ttl) {
        return RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(ttl)
                .serializeKeysWith(RedisSerializationContext.SerializationPair
                        .fromSerializer(new StringRedisSerializer()))
                .serializeValuesWith(RedisSerializationContext.SerializationPair
                        .fromSerializer(new GenericJackson2JsonRedisSerializer()))
                .disableCachingNullValues();
    }
}
```

## Monitoring and Metrics

### Redis Health Indicator

```java
@Component
public class RedisHealthIndicator implements HealthIndicator {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Override
    public Health health() {
        try {
            redisTemplate.execute((RedisCallback<String>) connection -> {
                return connection.ping();
            });
            return Health.up().withDetail("redis", "Available").build();
        } catch (Exception e) {
            return Health.down().withException(e).build();
        }
    }
}
```

### Cache Statistics

```java
@Service
public class CacheStatisticsService {
    
    @Autowired
    private CacheManager cacheManager;
    
    public Map<String, CacheStatistics> getCacheStatistics() {
        Map<String, CacheStatistics> stats = new HashMap<>();
        
        cacheManager.getCacheNames().forEach(cacheName -> {
            Cache cache = cacheManager.getCache(cacheName);
            if (cache instanceof RedisCache) {
                RedisCache redisCache = (RedisCache) cache;
                // Collect statistics
                stats.put(cacheName, collectStats(redisCache));
            }
        });
        
        return stats;
    }
    
    private CacheStatistics collectStats(RedisCache cache) {
        // Implementation to collect cache statistics
        return new CacheStatistics();
    }
}
```

## Best Practices

1. **Set appropriate TTL** - Avoid memory bloat
2. **Use cache namespaces** - Organize keys logically
3. **Implement cache warming** - Pre-load frequently accessed data
4. **Monitor cache hit rates** - Optimize caching strategy
5. **Handle cache failures gracefully** - Fallback to database
6. **Use serialization wisely** - Balance between speed and size
7. **Implement cache invalidation** - Keep data consistent
8. **Use Redis clustering** - For high availability

## Testing

```java
@SpringBootTest
@TestPropertySource(properties = {
    "spring.cache.type=redis",
    "spring.data.redis.host=localhost",
    "spring.data.redis.port=6379"
})
class RedisCacheTest {
    
    @Autowired
    private UserService userService;
    
    @Autowired
    private CacheManager cacheManager;
    
    @Test
    void testCacheable() {
        User user1 = userService.getUserById(1L);
        User user2 = userService.getUserById(1L);
        
        assertSame(user1, user2);
        
        Cache cache = cacheManager.getCache("users");
        assertNotNull(cache.get(1L));
    }
    
    @Test
    void testCacheEvict() {
        userService.getUserById(1L);
        userService.deleteUser(1L);
        
        Cache cache = cacheManager.getCache("users");
        assertNull(cache.get(1L));
    }
}
```

## Docker Compose Setup

```yaml
version: '3.8'
services:
  redis:
    image: redis:7-alpine
    container_name: redis
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    command: redis-server --appendonly yes

volumes:
  redis-data:
```

## Common Pitfalls

1. Not setting TTL - Memory leaks
2. Caching large objects - Performance issues
3. Not handling cache misses - Poor user experience
4. Over-caching - Stale data
5. Not monitoring Redis memory - Out of memory errors
6. Improper serialization - Data loss or corruption

## Resources

- [Spring Data Redis Documentation](https://docs.spring.io/spring-data/redis/docs/current/reference/html/)
- [Redis Documentation](https://redis.io/documentation)
- [Spring Cache Abstraction](https://docs.spring.io/spring-framework/reference/integration/cache.html)