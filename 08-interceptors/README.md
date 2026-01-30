# 08. Interceptors

## Overview

Interceptors in Spring Boot allow you to intercept HTTP requests and responses before they reach the controller or after the controller has processed them. They are powerful tools for implementing cross-cutting concerns like logging, authentication, authorization, and request/response modification.

## Key Concepts

### HandlerInterceptor Interface

```java
public interface HandlerInterceptor {
    boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler);
    void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView);
    void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex);
}
```

### Execution Flow

1. **preHandle** - Before controller execution
2. **Controller Method** - Actual request processing
3. **postHandle** - After controller, before view rendering
4. **afterCompletion** - After complete request processing

## Implementation Examples

### Basic Logging Interceptor

```java
@Component
public class LoggingInterceptor implements HandlerInterceptor {
    
    private static final Logger logger = LoggerFactory.getLogger(LoggingInterceptor.class);
    
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        logger.info("Request URI: {}, Method: {}", request.getRequestURI(), request.getMethod());
        request.setAttribute("startTime", System.currentTimeMillis());
        return true;
    }
    
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) {
        long startTime = (Long) request.getAttribute("startTime");
        long endTime = System.currentTimeMillis();
        logger.info("Request completed in {} ms", (endTime - startTime));
    }
}
```

### Authentication Interceptor

```java
@Component
public class AuthenticationInterceptor implements HandlerInterceptor {
    
    @Autowired
    private TokenService tokenService;
    
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        String token = request.getHeader("Authorization");
        
        if (token == null || !tokenService.validateToken(token)) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            return false;
        }
        
        UserContext userContext = tokenService.getUserContext(token);
        request.setAttribute("userContext", userContext);
        return true;
    }
}
```

### Rate Limiting Interceptor

```java
@Component
public class RateLimitInterceptor implements HandlerInterceptor {
    
    private final Map<String, List<Long>> requestCounts = new ConcurrentHashMap<>();
    private static final int MAX_REQUESTS = 100;
    private static final long TIME_WINDOW = 60000; // 1 minute
    
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        String clientId = request.getRemoteAddr();
        long currentTime = System.currentTimeMillis();
        
        requestCounts.putIfAbsent(clientId, new ArrayList<>());
        List<Long> timestamps = requestCounts.get(clientId);
        
        timestamps.removeIf(timestamp -> currentTime - timestamp > TIME_WINDOW);
        
        if (timestamps.size() >= MAX_REQUESTS) {
            response.setStatus(HttpServletResponse.SC_TOO_MANY_REQUESTS);
            return false;
        }
        
        timestamps.add(currentTime);
        return true;
    }
}
```

## Configuration

### WebMvcConfigurer Registration

```java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {
    
    @Autowired
    private LoggingInterceptor loggingInterceptor;
    
    @Autowired
    private AuthenticationInterceptor authenticationInterceptor;
    
    @Autowired
    private RateLimitInterceptor rateLimitInterceptor;
    
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(loggingInterceptor)
                .addPathPatterns("/**");
        
        registry.addInterceptor(authenticationInterceptor)
                .addPathPatterns("/api/**")
                .excludePathPatterns("/api/public/**", "/api/auth/**");
        
        registry.addInterceptor(rateLimitInterceptor)
                .addPathPatterns("/api/**")
                .order(1);
    }
}
```

## Advanced Use Cases

### Request/Response Modification

```java
@Component
public class HeaderModificationInterceptor implements HandlerInterceptor {
    
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        response.setHeader("X-Custom-Header", "CustomValue");
        response.setHeader("X-Request-ID", UUID.randomUUID().toString());
        return true;
    }
    
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) {
        response.setHeader("X-Response-Time", String.valueOf(System.currentTimeMillis()));
    }
}
```

### Tenant Resolution Interceptor

```java
@Component
public class TenantInterceptor implements HandlerInterceptor {
    
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        String tenantId = request.getHeader("X-Tenant-ID");
        
        if (tenantId == null) {
            tenantId = "default";
        }
        
        TenantContext.setCurrentTenant(tenantId);
        return true;
    }
    
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) {
        TenantContext.clear();
    }
}
```

## Best Practices

1. **Return false to stop processing** - If authentication fails
2. **Clean up in afterCompletion** - Always clear ThreadLocal variables
3. **Order matters** - Use `order()` method for interceptor precedence
4. **Use path patterns wisely** - Include/exclude paths appropriately
5. **Keep interceptors lightweight** - Avoid heavy processing
6. **Exception handling** - Handle exceptions gracefully in afterCompletion

## Testing

```java
@WebMvcTest
class InterceptorTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Test
    void testLoggingInterceptor() throws Exception {
        mockMvc.perform(get("/api/test"))
                .andExpect(status().isOk())
                .andExpect(header().exists("X-Request-ID"));
    }
    
    @Test
    void testAuthenticationInterceptor_Unauthorized() throws Exception {
        mockMvc.perform(get("/api/protected"))
                .andExpect(status().isUnauthorized());
    }
    
    @Test
    void testRateLimitInterceptor() throws Exception {
        for (int i = 0; i < 101; i++) {
            mockMvc.perform(get("/api/test"));
        }
        
        mockMvc.perform(get("/api/test"))
                .andExpect(status().isTooManyRequests());
    }
}
```

## Common Pitfalls

1. Not clearing ThreadLocal variables
2. Blocking operations in preHandle
3. Incorrect interceptor ordering
4. Not handling exceptions properly
5. Modifying request/response incorrectly

## Resources

- [Spring Framework Documentation](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-config/interceptors.html)
- [Baeldung - Spring MVC Handler Interceptors](https://www.baeldung.com/spring-mvc-handlerinterceptor)
