# Rate Limiting

## Overview

Rate limiting controls how many requests a client can make to your API within a given time window. It protects your services from abuse, prevents resource exhaustion, enforces fair usage, and ensures stability under load. Spring Boot offers multiple approaches: Bucket4j for in-app token bucket algorithms, Spring Cloud Gateway filters for API gateway-level limiting, and Redis-backed distributed rate limiting for clustered deployments.

## Key Concepts

### Rate Limiting Algorithms

- **Token Bucket** - Tokens are added at a fixed rate; each request consumes a token. Allows bursts up to bucket capacity
- **Sliding Window** - Tracks requests in a moving time window for smooth distribution
- **Fixed Window** - Counts requests in fixed time intervals (e.g., 100 req/min). Simple but allows boundary bursts
- **Leaky Bucket** - Requests enter a queue and are processed at a fixed rate. Smoothest output

### Token Bucket Visualization

```text
Bucket capacity: 10 tokens
Refill rate: 5 tokens/second

Time 0s:  [##########]  10/10 tokens
Request → [#########.]   9/10 tokens consumed 1
Request → [########..]   8/10 tokens consumed 1
...
Time 1s:  [##########]  refilled to 10/10

Burst of 12 requests:
[##########] → 10 served, 2 rejected (429 Too Many Requests)
```

## Approach 1: Bucket4j (In-Process Rate Limiting)

### Dependencies

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>com.bucket4j</groupId>
        <artifactId>bucket4j-core</artifactId>
        <version>8.10.1</version>
    </dependency>
    <!-- For Redis-backed distributed buckets -->
    <dependency>
        <groupId>com.bucket4j</groupId>
        <artifactId>bucket4j-redis</artifactId>
        <version>8.10.1</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
</dependencies>
```

### Simple In-Memory Rate Limiter

```java
@Component
public class RateLimiterService {

    private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();

    public Bucket resolveBucket(String key) {
        return buckets.computeIfAbsent(key, this::createBucket);
    }

    private Bucket createBucket(String key) {
        return Bucket.builder()
                .addLimit(BandwidthBuilder.builder()
                        .capacity(100)
                        .refillGreedy(100, Duration.ofMinutes(1))
                        .build())
                .build();
    }
}
```

### Rate Limiting Filter

```java
@Component
@Order(1)
public class RateLimitFilter extends OncePerRequestFilter {

    private final RateLimiterService rateLimiterService;

    public RateLimitFilter(RateLimiterService rateLimiterService) {
        this.rateLimiterService = rateLimiterService;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain filterChain)
            throws ServletException, IOException {

        String clientId = resolveClientId(request);
        Bucket bucket = rateLimiterService.resolveBucket(clientId);

        ConsumptionProbe probe = bucket.tryConsumeAndReturnRemaining(1);

        if (probe.isConsumed()) {
            response.setHeader("X-Rate-Limit-Remaining",
                    String.valueOf(probe.getRemainingTokens()));
            filterChain.doFilter(request, response);
        } else {
            long waitTime = probe.getNanosToWaitForRefill() / 1_000_000_000;
            response.setHeader("X-Rate-Limit-Retry-After-Seconds", String.valueOf(waitTime));
            response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            response.setContentType(MediaType.APPLICATION_JSON_VALUE);
            response.getWriter().write("""
                    {"error": "Too many requests", "retryAfterSeconds": %d}
                    """.formatted(waitTime));
        }
    }

    private String resolveClientId(HttpServletRequest request) {
        // Prefer API key, then authenticated user, then IP
        String apiKey = request.getHeader("X-API-Key");
        if (apiKey != null) return "api:" + apiKey;

        Principal principal = request.getUserPrincipal();
        if (principal != null) return "user:" + principal.getName();

        return "ip:" + request.getRemoteAddr();
    }
}
```

### Tiered Rate Limits by Plan

```java
@Component
public class TieredRateLimiterService {

    private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();
    private final UserPlanService userPlanService;

    public TieredRateLimiterService(UserPlanService userPlanService) {
        this.userPlanService = userPlanService;
    }

    public Bucket resolveBucket(String userId) {
        return buckets.computeIfAbsent(userId, key -> {
            UserPlan plan = userPlanService.getPlan(key);
            return createBucket(plan);
        });
    }

    private Bucket createBucket(UserPlan plan) {
        return switch (plan) {
            case FREE -> Bucket.builder()
                    .addLimit(BandwidthBuilder.builder()
                            .capacity(20)
                            .refillGreedy(20, Duration.ofMinutes(1))
                            .build())
                    .addLimit(BandwidthBuilder.builder()
                            .capacity(500)
                            .refillGreedy(500, Duration.ofHours(1))
                            .build())
                    .build();

            case BASIC -> Bucket.builder()
                    .addLimit(BandwidthBuilder.builder()
                            .capacity(100)
                            .refillGreedy(100, Duration.ofMinutes(1))
                            .build())
                    .addLimit(BandwidthBuilder.builder()
                            .capacity(5000)
                            .refillGreedy(5000, Duration.ofHours(1))
                            .build())
                    .build();

            case PREMIUM -> Bucket.builder()
                    .addLimit(BandwidthBuilder.builder()
                            .capacity(500)
                            .refillGreedy(500, Duration.ofMinutes(1))
                            .build())
                    .build();
        };
    }
}

public enum UserPlan {
    FREE, BASIC, PREMIUM
}
```

## Approach 2: Annotation-Based Rate Limiting

### Custom Annotation

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RateLimit {
    int capacity() default 100;
    int refillTokens() default 100;
    int refillSeconds() default 60;
    String key() default "";  // SpEL expression for dynamic key
}
```

### Rate Limit Aspect

```java
@Aspect
@Component
public class RateLimitAspect {

    private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();
    private final HttpServletRequest request;

    public RateLimitAspect(HttpServletRequest request) {
        this.request = request;
    }

    @Around("@annotation(rateLimit)")
    public Object enforce(ProceedingJoinPoint joinPoint, RateLimit rateLimit) throws Throwable {
        String key = resolveKey(rateLimit, joinPoint);
        Bucket bucket = buckets.computeIfAbsent(key, k ->
                Bucket.builder()
                        .addLimit(BandwidthBuilder.builder()
                                .capacity(rateLimit.capacity())
                                .refillGreedy(rateLimit.refillTokens(),
                                        Duration.ofSeconds(rateLimit.refillSeconds()))
                                .build())
                        .build()
        );

        ConsumptionProbe probe = bucket.tryConsumeAndReturnRemaining(1);
        if (!probe.isConsumed()) {
            throw new RateLimitExceededException("Rate limit exceeded. Try again later.");
        }

        return joinPoint.proceed();
    }

    private String resolveKey(RateLimit rateLimit, ProceedingJoinPoint joinPoint) {
        String methodName = joinPoint.getSignature().toShortString();
        if (!rateLimit.key().isEmpty()) {
            return rateLimit.key() + ":" + request.getRemoteAddr();
        }
        return methodName + ":" + request.getRemoteAddr();
    }
}
```

### Usage

```java
@RestController
@RequestMapping("/api")
public class ApiController {

    @GetMapping("/data")
    @RateLimit(capacity = 50, refillTokens = 50, refillSeconds = 60)
    public ResponseEntity<String> getData() {
        return ResponseEntity.ok("data");
    }

    @PostMapping("/upload")
    @RateLimit(capacity = 10, refillTokens = 10, refillSeconds = 60, key = "upload")
    public ResponseEntity<String> upload() {
        return ResponseEntity.ok("uploaded");
    }

    @GetMapping("/search")
    @RateLimit(capacity = 30, refillTokens = 30, refillSeconds = 60, key = "search")
    public ResponseEntity<String> search(@RequestParam String query) {
        return ResponseEntity.ok("results for: " + query);
    }
}
```

## Approach 3: Distributed Rate Limiting with Redis

### Redis-Backed Rate Limiter

```java
@Service
public class RedisRateLimiter {

    private final StringRedisTemplate redisTemplate;

    public RedisRateLimiter(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    public RateLimitResult tryConsume(String key, int maxRequests, Duration window) {
        String redisKey = "rate_limit:" + key;

        List<Object> results = redisTemplate.execute(new SessionCallback<>() {
            @Override
            public List<Object> execute(RedisOperations operations) throws DataAccessException {
                operations.multi();
                operations.opsForValue().increment(redisKey);
                operations.expire(redisKey, window);
                return operations.exec();
            }
        });

        long currentCount = (Long) results.get(0);

        if (currentCount <= maxRequests) {
            return new RateLimitResult(true, maxRequests - currentCount, 0);
        } else {
            Long ttl = redisTemplate.getExpire(redisKey, TimeUnit.SECONDS);
            return new RateLimitResult(false, 0, ttl != null ? ttl : 0);
        }
    }
}

public record RateLimitResult(
        boolean allowed,
        long remaining,
        long retryAfterSeconds
) {
}
```

### Sliding Window with Redis (Sorted Set)

```java
@Service
public class SlidingWindowRateLimiter {

    private final StringRedisTemplate redisTemplate;

    public SlidingWindowRateLimiter(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    public boolean isAllowed(String key, int maxRequests, Duration window) {
        String redisKey = "sliding_rate:" + key;
        long now = System.currentTimeMillis();
        long windowStart = now - window.toMillis();

        // Use Redis pipeline for atomicity
        redisTemplate.execute(new SessionCallback<>() {
            @Override
            public Object execute(RedisOperations operations) throws DataAccessException {
                operations.multi();
                // Remove expired entries
                operations.opsForZSet().removeRangeByScore(redisKey, 0, windowStart);
                // Add current request
                operations.opsForZSet().add(redisKey, UUID.randomUUID().toString(), now);
                // Set TTL
                operations.expire(redisKey, window);
                return operations.exec();
            }
        });

        Long count = redisTemplate.opsForZSet().zCard(redisKey);
        return count != null && count <= maxRequests;
    }
}
```

### Redis Rate Limit Filter

```java
@Component
@Order(1)
public class RedisRateLimitFilter extends OncePerRequestFilter {

    private final RedisRateLimiter rateLimiter;

    public RedisRateLimitFilter(RedisRateLimiter rateLimiter) {
        this.rateLimiter = rateLimiter;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain filterChain)
            throws ServletException, IOException {

        String clientId = request.getRemoteAddr();
        RateLimitResult result = rateLimiter.tryConsume(clientId, 100, Duration.ofMinutes(1));

        response.setHeader("X-Rate-Limit-Remaining", String.valueOf(result.remaining()));

        if (result.allowed()) {
            filterChain.doFilter(request, response);
        } else {
            response.setHeader("Retry-After", String.valueOf(result.retryAfterSeconds()));
            response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            response.setContentType(MediaType.APPLICATION_JSON_VALUE);
            response.getWriter().write("""
                    {"error": "Rate limit exceeded", "retryAfterSeconds": %d}
                    """.formatted(result.retryAfterSeconds()));
        }
    }
}
```

## Per-Endpoint Rate Limiting

### Configuration-Driven

```yaml
rate-limiting:
  endpoints:
    - path: /api/auth/login
      capacity: 5
      refill-tokens: 5
      refill-seconds: 60
    - path: /api/search
      capacity: 30
      refill-tokens: 30
      refill-seconds: 60
    - path: /api/**
      capacity: 100
      refill-tokens: 100
      refill-seconds: 60
```

```java
@ConfigurationProperties(prefix = "rate-limiting")
@Data
public class RateLimitProperties {
    private List<EndpointLimit> endpoints = new ArrayList<>();

    @Data
    public static class EndpointLimit {
        private String path;
        private int capacity;
        private int refillTokens;
        private int refillSeconds;
    }
}

@Component
@Order(1)
public class EndpointRateLimitFilter extends OncePerRequestFilter {

    private final RateLimitProperties properties;
    private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();
    private final AntPathMatcher pathMatcher = new AntPathMatcher();

    public EndpointRateLimitFilter(RateLimitProperties properties) {
        this.properties = properties;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain filterChain)
            throws ServletException, IOException {

        String path = request.getRequestURI();
        String clientIp = request.getRemoteAddr();

        Optional<RateLimitProperties.EndpointLimit> matchedLimit = properties.getEndpoints()
                .stream()
                .filter(limit -> pathMatcher.match(limit.getPath(), path))
                .findFirst();

        if (matchedLimit.isEmpty()) {
            filterChain.doFilter(request, response);
            return;
        }

        RateLimitProperties.EndpointLimit limit = matchedLimit.get();
        String bucketKey = path + ":" + clientIp;
        Bucket bucket = buckets.computeIfAbsent(bucketKey, k ->
                Bucket.builder()
                        .addLimit(BandwidthBuilder.builder()
                                .capacity(limit.getCapacity())
                                .refillGreedy(limit.getRefillTokens(),
                                        Duration.ofSeconds(limit.getRefillSeconds()))
                                .build())
                        .build()
        );

        if (bucket.tryConsume(1)) {
            filterChain.doFilter(request, response);
        } else {
            response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            response.setContentType(MediaType.APPLICATION_JSON_VALUE);
            response.getWriter().write("{\"error\": \"Rate limit exceeded for this endpoint\"}");
        }
    }
}
```

## Exception Handling

```java
public class RateLimitExceededException extends RuntimeException {
    private final long retryAfterSeconds;

    public RateLimitExceededException(String message) {
        this(message, 60);
    }

    public RateLimitExceededException(String message, long retryAfterSeconds) {
        super(message);
        this.retryAfterSeconds = retryAfterSeconds;
    }

    public long getRetryAfterSeconds() {
        return retryAfterSeconds;
    }
}

@RestControllerAdvice
public class RateLimitExceptionHandler {

    @ExceptionHandler(RateLimitExceededException.class)
    public ResponseEntity<Map<String, Object>> handleRateLimit(RateLimitExceededException ex) {
        return ResponseEntity.status(HttpStatus.TOO_MANY_REQUESTS)
                .header("Retry-After", String.valueOf(ex.getRetryAfterSeconds()))
                .body(Map.of(
                        "error", "Too Many Requests",
                        "message", ex.getMessage(),
                        "retryAfterSeconds", ex.getRetryAfterSeconds()
                ));
    }
}
```

## Testing

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class RateLimitTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void shouldAllowRequestsWithinLimit() {
        for (int i = 0; i < 50; i++) {
            ResponseEntity<String> response = restTemplate.getForEntity("/api/data", String.class);
            assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        }
    }

    @Test
    void shouldRejectRequestsExceedingLimit() {
        // Exhaust the rate limit
        for (int i = 0; i < 100; i++) {
            restTemplate.getForEntity("/api/data", String.class);
        }

        // Next request should be rate limited
        ResponseEntity<String> response = restTemplate.getForEntity("/api/data", String.class);
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.TOO_MANY_REQUESTS);
        assertThat(response.getHeaders().getFirst("Retry-After")).isNotNull();
    }

    @Test
    void shouldIncludeRateLimitHeaders() {
        ResponseEntity<String> response = restTemplate.getForEntity("/api/data", String.class);
        assertThat(response.getHeaders().getFirst("X-Rate-Limit-Remaining")).isNotNull();
    }

    @Test
    void shouldRateLimitPerClient() {
        // Client A
        HttpHeaders headersA = new HttpHeaders();
        headersA.set("X-API-Key", "client-a");
        HttpEntity<Void> entityA = new HttpEntity<>(headersA);

        // Client B
        HttpHeaders headersB = new HttpHeaders();
        headersB.set("X-API-Key", "client-b");
        HttpEntity<Void> entityB = new HttpEntity<>(headersB);

        // Both clients should have independent limits
        for (int i = 0; i < 50; i++) {
            restTemplate.exchange("/api/data", HttpMethod.GET, entityA, String.class);
        }

        // Client B should still be within limits
        ResponseEntity<String> response =
                restTemplate.exchange("/api/data", HttpMethod.GET, entityB, String.class);
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
    }
}
```

## Best Practices

1. **Return standard headers** - `X-Rate-Limit-Remaining`, `Retry-After` to help clients back off
2. **Rate limit by API key, not just IP** - IPs can be shared (NAT) or spoofed
3. **Use tiered limits** - Different limits for free vs paid plans
4. **Distribute state with Redis** - In-memory buckets don't work across multiple instances
5. **Apply at multiple levels** - Global + per-endpoint + per-user
6. **Return HTTP 429** - Standard status code for rate limit exceeded
7. **Log rate limit events** - Monitor for abuse patterns
8. **Don't rate limit health checks** - Exclude internal/monitoring endpoints

## Common Pitfalls

1. In-memory rate limiters not working in clustered/multi-instance deployments
2. Rate limiting by IP only, which breaks for users behind shared NATs
3. Not returning `Retry-After` header, causing clients to retry immediately
4. Setting limits too aggressively, blocking legitimate users
5. Bucket cleanup - in-memory maps growing unbounded without eviction
6. Race conditions in Redis-based limiting without proper atomicity

## Resources

- [Bucket4j Documentation](https://bucket4j.com/)
- [IETF RFC 6585 - HTTP 429](https://www.rfc-editor.org/rfc/rfc6585#section-4)
- [Token Bucket Algorithm](https://en.wikipedia.org/wiki/Token_bucket)
- [Redis Rate Limiting Patterns](https://redis.io/glossary/rate-limiting/)
