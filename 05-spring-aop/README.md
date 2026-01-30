# Spring AOP

## Overview

Aspect-Oriented Programming (AOP) enables modularization of cross-cutting concerns like logging, security, transactions, and monitoring. Spring AOP uses proxy-based approach to weave aspects into your application.

## Key Concepts

### 1. Core Terminology

- **Aspect**: Module that encapsulates cross-cutting concern
- **Join Point**: Point in program execution (method call, exception throw)
- **Advice**: Action taken at join point
- **Pointcut**: Expression that selects join points
- **Weaving**: Linking aspects with objects

### 2. Basic Aspect

```java
@Aspect
@Component
public class LoggingAspect {
    
    private static final Logger log = LoggerFactory.getLogger(LoggingAspect.class);
    
    @Before("execution(* com.example.service.*.*(..))")
    public void logBefore(JoinPoint joinPoint) {
        log.info("Executing: {}.{}",
            joinPoint.getSignature().getDeclaringTypeName(),
            joinPoint.getSignature().getName());
    }
    
    @AfterReturning(
        pointcut = "execution(* com.example.service.*.*(..))",
        returning = "result"
    )
    public void logAfterReturning(JoinPoint joinPoint, Object result) {
        log.info("Method {} returned: {}",
            joinPoint.getSignature().getName(),
            result);
    }
}
```

### 3. Pointcut Expressions

```java
@Aspect
@Component
public class PointcutExamples {
    
    // Match all methods in service package
    @Pointcut("execution(* com.example.service.*.*(..))")
    public void serviceMethods() {}
    
    // Match specific method
    @Pointcut("execution(* com.example.service.UserService.createUser(..))")
    public void createUserMethod() {}
    
    // Match methods with specific annotation
    @Pointcut("@annotation(com.example.annotation.Audited)")
    public void auditedMethods() {}
    
    // Match all repository methods
    @Pointcut("execution(* com.example.repository.*.*(..))")
    public void repositoryMethods() {}
    
    // Combine pointcuts
    @Pointcut("serviceMethods() || repositoryMethods()")
    public void allDataAccessMethods() {}
    
    // Match methods with specific parameter
    @Pointcut("execution(* *..*.*(Long)) && args(userId)")
    public void methodsWithUserId(Long userId) {}
}
```

### 4. Advice Types

```java
@Aspect
@Component
public class AdviceTypesAspect {
    
    // Before advice
    @Before("execution(* com.example.service.*.*(..))")
    public void beforeAdvice(JoinPoint joinPoint) {
        log.info("Before method: {}", joinPoint.getSignature().getName());
    }
    
    // After returning advice
    @AfterReturning(
        pointcut = "execution(* com.example.service.*.*(..))",
        returning = "result"
    )
    public void afterReturningAdvice(JoinPoint joinPoint, Object result) {
        log.info("Method returned: {}", result);
    }
    
    // After throwing advice
    @AfterThrowing(
        pointcut = "execution(* com.example.service.*.*(..))",
        throwing = "error"
    )
    public void afterThrowingAdvice(JoinPoint joinPoint, Throwable error) {
        log.error("Method {} threw exception: {}", 
            joinPoint.getSignature().getName(), 
            error.getMessage());
    }
    
    // After advice (finally)
    @After("execution(* com.example.service.*.*(..))")
    public void afterAdvice(JoinPoint joinPoint) {
        log.info("Method completed: {}", joinPoint.getSignature().getName());
    }
    
    // Around advice (most powerful)
    @Around("execution(* com.example.service.*.*(..))")
    public Object aroundAdvice(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.currentTimeMillis();
        
        try {
            Object result = joinPoint.proceed();
            return result;
        } finally {
            long executionTime = System.currentTimeMillis() - start;
            log.info("Method {} executed in {}ms",
                joinPoint.getSignature().getName(),
                executionTime);
        }
    }
}
```

### 5. Custom Annotation-Based AOP

```java
// Custom annotation
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface PerformanceMonitor {
    String value() default "";
}

// Aspect
@Aspect
@Component
public class PerformanceAspect {
    
    @Around("@annotation(performanceMonitor)")
    public Object monitorPerformance(ProceedingJoinPoint joinPoint, 
                                     PerformanceMonitor performanceMonitor) throws Throwable {
        String operationName = performanceMonitor.value().isEmpty() 
            ? joinPoint.getSignature().getName() 
            : performanceMonitor.value();
        
        long start = System.currentTimeMillis();
        try {
            return joinPoint.proceed();
        } finally {
            long duration = System.currentTimeMillis() - start;
            log.info("Operation '{}' took {}ms", operationName, duration);
            
            // Send to monitoring system
            metricsService.recordDuration(operationName, duration);
        }
    }
}

// Usage
@Service
public class UserService {
    
    @PerformanceMonitor("user-creation")
    public User createUser(CreateUserRequest request) {
        // Implementation
    }
}
```

### 6. Audit Logging Aspect

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Audited {
    String action();
    String resourceType();
}

@Aspect
@Component
public class AuditAspect {
    
    @Autowired
    private AuditLogRepository auditLogRepository;
    
    @Around("@annotation(audited)")
    public Object audit(ProceedingJoinPoint joinPoint, Audited audited) throws Throwable {
        String username = SecurityContextHolder.getContext()
            .getAuthentication()
            .getName();
        
        AuditLog auditLog = new AuditLog();
        auditLog.setUsername(username);
        auditLog.setAction(audited.action());
        auditLog.setResourceType(audited.resourceType());
        auditLog.setTimestamp(LocalDateTime.now());
        
        try {
            Object result = joinPoint.proceed();
            auditLog.setStatus("SUCCESS");
            auditLog.setDetails(extractDetails(result));
            return result;
        } catch (Exception e) {
            auditLog.setStatus("FAILURE");
            auditLog.setErrorMessage(e.getMessage());
            throw e;
        } finally {
            auditLogRepository.save(auditLog);
        }
    }
}

// Usage
@Audited(action = "CREATE", resourceType = "USER")
public User createUser(CreateUserRequest request) {
    // Implementation
}
```

### 7. Caching Aspect

```java
@Aspect
@Component
public class CachingAspect {
    
    @Autowired
    private CacheManager cacheManager;
    
    @Around("@annotation(cacheable)")
    public Object handleCaching(ProceedingJoinPoint joinPoint, Cacheable cacheable) throws Throwable {
        String cacheName = cacheable.value()[0];
        String key = generateKey(joinPoint);
        
        Cache cache = cacheManager.getCache(cacheName);
        if (cache != null) {
            Cache.ValueWrapper cachedValue = cache.get(key);
            if (cachedValue != null) {
                log.debug("Cache hit for key: {}", key);
                return cachedValue.get();
            }
        }
        
        log.debug("Cache miss for key: {}", key);
        Object result = joinPoint.proceed();
        
        if (cache != null && result != null) {
            cache.put(key, result);
        }
        
        return result;
    }
}
```

### 8. Retry Aspect

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Retry {
    int maxAttempts() default 3;
    long delay() default 1000;
    Class<? extends Exception>[] retryFor() default {Exception.class};
}

@Aspect
@Component
public class RetryAspect {
    
    @Around("@annotation(retry)")
    public Object handleRetry(ProceedingJoinPoint joinPoint, Retry retry) throws Throwable {
        int attempts = 0;
        Throwable lastException = null;
        
        while (attempts < retry.maxAttempts()) {
            try {
                return joinPoint.proceed();
            } catch (Throwable e) {
                lastException = e;
                attempts++;
                
                if (attempts >= retry.maxAttempts() || !shouldRetry(e, retry.retryFor())) {
                    throw e;
                }
                
                log.warn("Method {} failed (attempt {}/{}). Retrying in {}ms",
                    joinPoint.getSignature().getName(),
                    attempts,
                    retry.maxAttempts(),
                    retry.delay());
                
                Thread.sleep(retry.delay());
            }
        }
        
        throw lastException;
    }
}
```

### 9. Transaction Management Aspect

```java
@Aspect
@Component
public class TransactionAspect {
    
    @Around("@annotation(transactional)")
    public Object handleTransaction(ProceedingJoinPoint joinPoint, 
                                    Transactional transactional) throws Throwable {
        log.info("Starting transaction for: {}", joinPoint.getSignature().getName());
        
        try {
            Object result = joinPoint.proceed();
            log.info("Transaction committed for: {}", joinPoint.getSignature().getName());
            return result;
        } catch (Exception e) {
            log.error("Transaction rolled back for: {}", 
                joinPoint.getSignature().getName(), e);
            throw e;
        }
    }
}
```

### 10. Security Aspect

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RequireRole {
    String[] value();
}

@Aspect
@Component
public class SecurityAspect {
    
    @Before("@annotation(requireRole)")
    public void checkRole(JoinPoint joinPoint, RequireRole requireRole) {
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        
        if (authentication == null || !authentication.isAuthenticated()) {
            throw new AccessDeniedException("User not authenticated");
        }
        
        boolean hasRole = Arrays.stream(requireRole.value())
            .anyMatch(role -> authentication.getAuthorities().stream()
                .anyMatch(auth -> auth.getAuthority().equals("ROLE_" + role)));
        
        if (!hasRole) {
            throw new AccessDeniedException("Insufficient permissions");
        }
    }
}
```

## Configuration

```java
@Configuration
@EnableAspectJAutoProxy
public class AopConfig {
    // Aspects are auto-detected as Spring beans
}
```

## POC Requirements

Create aspects that demonstrate:

1. **Logging Aspect**: Log method entry, exit, and exceptions
2. **Performance Monitoring**: Track execution time
3. **Audit Logging**: Record user actions
4. **Retry Logic**: Automatic retry on failure
5. **Security Checks**: Role-based access control
6. **Custom Annotations**: Create and use custom AOP annotations

## Best Practices

1. **Keep Aspects Focused**: One concern per aspect
2. **Use Specific Pointcuts**: Avoid overly broad pointcuts
3. **Consider Performance**: Around advice has overhead
4. **Handle Exceptions**: Properly handle exceptions in aspects
5. **Test Thoroughly**: AOP can be tricky to debug
6. **Document Pointcuts**: Explain what methods are affected

## Common Pitfalls

1. **Self-Invocation**: AOP doesn't work for internal method calls
2. **Final Methods**: Can't be proxied
3. **Private Methods**: Not visible to proxies
4. **Performance Impact**: Too many aspects can slow application

## Additional Resources

- [Spring AOP Documentation](https://docs.spring.io/spring-framework/reference/core/aop.html)
- [AspectJ Documentation](https://www.eclipse.org/aspectj/doc/released/progguide/index.html)

## Next Steps

Continue to [Spring Events](../06-spring-events/README.md) to learn about event-driven programming.