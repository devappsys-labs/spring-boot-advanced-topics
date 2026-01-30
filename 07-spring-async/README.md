# Spring Async

## Overview

Spring @Async enables asynchronous method execution, allowing methods to run in separate threads without blocking the caller.

## Key Concepts

### 1. Enable Async

```java
@Configuration
@EnableAsync
public class AsyncConfiguration {
    
    @Bean(name = "taskExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(5);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("async-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}
```

### 2. Basic Async Method

```java
@Service
public class EmailService {
    
    private static final Logger log = LoggerFactory.getLogger(EmailService.class);
    
    @Async
    public void sendEmail(String to, String subject, String body) {
        log.info("Sending email in thread: {}", Thread.currentThread().getName());
        // Simulate email sending
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        log.info("Email sent to: {}", to);
    }
}
```

### 3. Async with CompletableFuture

```java
@Service
public class UserService {
    
    @Async
    public CompletableFuture<User> findUserAsync(Long id) {
        User user = userRepository.findById(id).orElse(null);
        return CompletableFuture.completedFuture(user);
    }
    
    @Async
    public CompletableFuture<List<User>> findAllUsersAsync() {
        List<User> users = userRepository.findAll();
        return CompletableFuture.completedFuture(users);
    }
}

// Usage - Parallel execution
@RestController
public class UserController {
    
    @GetMapping("/users/batch")
    public List<User> getBatchUsers() {
        CompletableFuture<User> user1 = userService.findUserAsync(1L);
        CompletableFuture<User> user2 = userService.findUserAsync(2L);
        CompletableFuture<User> user3 = userService.findUserAsync(3L);
        
        // Wait for all to complete
        CompletableFuture.allOf(user1, user2, user3).join();
        
        return Arrays.asList(user1.join(), user2.join(), user3.join());
    }
}
```

### 4. Custom Executor

```java
@Configuration
@EnableAsync
public class MultipleExecutorsConfig {
    
    @Bean(name = "emailExecutor")
    public Executor emailExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(3);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(50);
        executor.setThreadNamePrefix("email-");
        executor.initialize();
        return executor;
    }
    
    @Bean(name = "reportExecutor")
    public Executor reportExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(5);
        executor.setQueueCapacity(25);
        executor.setThreadNamePrefix("report-");
        executor.initialize();
        return executor;
    }
}

@Service
public class NotificationService {
    
    @Async("emailExecutor")
    public void sendEmail(String message) {
        // Uses emailExecutor
    }
    
    @Async("reportExecutor")
    public CompletableFuture<Report> generateReport() {
        // Uses reportExecutor
        return CompletableFuture.completedFuture(new Report());
    }
}
```

### 5. Exception Handling

```java
@Component
public class CustomAsyncExceptionHandler implements AsyncUncaughtExceptionHandler {
    
    private static final Logger log = LoggerFactory.getLogger(CustomAsyncExceptionHandler.class);
    
    @Override
    public void handleUncaughtException(Throwable throwable, Method method, Object... params) {
        log.error("Async method {} threw exception", method.getName(), throwable);
        log.error("Parameters: {}", Arrays.toString(params));
        // Send alert, save to database, etc.
    }
}

@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {
    
    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return new CustomAsyncExceptionHandler();
    }
    
    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("default-async-");
        executor.initialize();
        return executor;
    }
}
```

## POC Requirements

1. Configure custom thread pools
2. Implement async methods with CompletableFuture
3. Handle exceptions properly
4. Use multiple executors
5. Demonstrate parallel execution
6. Monitor thread pool metrics

## Best Practices

1. **Configure Thread Pools**: Don't use default configuration
2. **Return CompletableFuture**: For composable async operations
3. **Handle Exceptions**: Implement AsyncUncaughtExceptionHandler
4. **Avoid Self-Invocation**: @Async doesn't work for self-calls
5. **Monitor Thread Pools**: Track thread pool metrics
6. **Use Appropriate Pool Sizes**: Based on workload type

## Common Pitfalls

1. **Self-Invocation**: Calling async method from same class won't work
2. **Missing @EnableAsync**: Configuration must be enabled
3. **Wrong Return Type**: Use CompletableFuture for return values
4. **Thread Pool Exhaustion**: Configure appropriate pool sizes

## Additional Resources

- [Spring Async Documentation](https://docs.spring.io/spring-framework/reference/integration/scheduling.html)

## Next Steps

Proceed to [Interceptors](../08-interceptors/README.md) to learn about request/response interception.