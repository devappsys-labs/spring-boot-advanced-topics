# Spring Events and Publishing

## Overview

Spring Events provide a powerful mechanism for building loosely coupled, event-driven applications. Components can publish and listen to events without direct dependencies.

## Key Concepts

### 1. Basic Event

```java
// Event class
public class UserCreatedEvent {
    private final Long userId;
    private final String email;
    private final LocalDateTime timestamp;
    
    public UserCreatedEvent(Long userId, String email) {
        this.userId = userId;
        this.email = email;
        this.timestamp = LocalDateTime.now();
    }
    
    // Getters
}
```

### 2. Publishing Events

```java
@Service
public class UserService {
    
    @Autowired
    private ApplicationEventPublisher eventPublisher;
    
    @Autowired
    private UserRepository userRepository;
    
    @Transactional
    public User createUser(CreateUserRequest request) {
        User user = new User();
        user.setEmail(request.getEmail());
        user.setFullName(request.getFullName());
        
        User savedUser = userRepository.save(user);
        
        // Publish event
        eventPublisher.publishEvent(
            new UserCreatedEvent(savedUser.getId(), savedUser.getEmail())
        );
        
        return savedUser;
    }
}
```

### 3. Event Listeners

```java
@Component
public class UserEventListener {
    
    private static final Logger log = LoggerFactory.getLogger(UserEventListener.class);
    
    @EventListener
    public void handleUserCreated(UserCreatedEvent event) {
        log.info("User created: {}", event.getUserId());
        // Send welcome email
        // Create user profile
        // Initialize user preferences
    }
    
    @EventListener
    @Async
    public void handleUserCreatedAsync(UserCreatedEvent event) {
        // Asynchronous processing
        log.info("Processing user creation asynchronously: {}", event.getUserId());
    }
    
    @EventListener(condition = "#event.email.endsWith('@company.com')")
    public void handleCompanyUserCreated(UserCreatedEvent event) {
        log.info("Company user created: {}", event.getEmail());
        // Special handling for company users
    }
}
```

### 4. Transaction-Bound Events

```java
@Component
public class TransactionalEventListener {
    
    // Execute after successful transaction commit
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleAfterCommit(UserCreatedEvent event) {
        // Send notifications after database commit
        notificationService.sendWelcomeEmail(event.getUserId());
    }
    
    // Execute before transaction commit
    @TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)
    public void handleBeforeCommit(UserCreatedEvent event) {
        // Validate before commit
    }
    
    // Execute after transaction rollback
    @TransactionalEventListener(phase = TransactionPhase.AFTER_ROLLBACK)
    public void handleAfterRollback(UserCreatedEvent event) {
        // Cleanup after rollback
        log.warn("User creation rolled back: {}", event.getUserId());
    }
    
    // Execute after transaction completion (commit or rollback)
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMPLETION)
    public void handleAfterCompletion(UserCreatedEvent event) {
        // Cleanup resources
    }
}
```

### 5. Generic Events

```java
// Generic event
public class EntityEvent<T> {
    private final T entity;
    private final EventType type;
    private final LocalDateTime timestamp;
    
    public enum EventType {
        CREATED, UPDATED, DELETED
    }
    
    public EntityEvent(T entity, EventType type) {
        this.entity = entity;
        this.type = type;
        this.timestamp = LocalDateTime.now();
    }
    
    // Getters
}

// Publisher
@Service
public class GenericEntityService<T> {
    
    @Autowired
    private ApplicationEventPublisher eventPublisher;
    
    public void create(T entity) {
        // Save entity
        eventPublisher.publishEvent(new EntityEvent<>(entity, EventType.CREATED));
    }
}

// Listener with generics
@Component
public class EntityEventListener {
    
    @EventListener
    public void handleUserEvent(EntityEvent<User> event) {
        log.info("User event: {} - {}", event.getType(), event.getEntity().getId());
    }
    
    @EventListener
    public void handleOrderEvent(EntityEvent<Order> event) {
        log.info("Order event: {} - {}", event.getType(), event.getEntity().getId());
    }
}
```

### 6. Event Ordering

```java
@Component
public class OrderedEventListeners {
    
    @EventListener
    @Order(1)
    public void handleFirstPriority(UserCreatedEvent event) {
        log.info("First priority listener");
    }
    
    @EventListener
    @Order(2)
    public void handleSecondPriority(UserCreatedEvent event) {
        log.info("Second priority listener");
    }
    
    @EventListener
    @Order(3)
    public void handleThirdPriority(UserCreatedEvent event) {
        log.info("Third priority listener");
    }
}
```

### 7. Custom Event Publisher

```java
@Component
public class CustomEventPublisher {
    
    @Autowired
    private ApplicationEventPublisher eventPublisher;
    
    public void publishUserCreated(User user) {
        UserCreatedEvent event = new UserCreatedEvent(user.getId(), user.getEmail());
        eventPublisher.publishEvent(event);
    }
    
    public void publishUserUpdated(User user, Map<String, Object> changes) {
        UserUpdatedEvent event = new UserUpdatedEvent(user.getId(), changes);
        eventPublisher.publishEvent(event);
    }
    
    public void publishBatchEvent(List<User> users) {
        BatchUserEvent event = new BatchUserEvent(
            users.stream().map(User::getId).collect(Collectors.toList())
        );
        eventPublisher.publishEvent(event);
    }
}
```

### 8. Event Payload

```java
// Using Spring's PayloadApplicationEvent
@Service
public class PayloadEventService {
    
    @Autowired
    private ApplicationEventPublisher eventPublisher;
    
    public void notifyUserStatusChange(Long userId, String newStatus) {
        // Publish simple payload
        eventPublisher.publishEvent(new PayloadApplicationEvent<>(this, 
            Map.of("userId", userId, "status", newStatus)
        ));
    }
}

// Listener for payload events
@Component
public class PayloadEventListener {
    
    @EventListener
    public void handleStatusChange(PayloadApplicationEvent<Map<String, Object>> event) {
        Map<String, Object> payload = event.getPayload();
        log.info("Status changed for user: {}", payload.get("userId"));
    }
}
```

### 9. Async Event Processing

```java
@Configuration
@EnableAsync
public class AsyncConfig {
    
    @Bean(name = "eventExecutor")
    public Executor eventExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("event-");
        executor.initialize();
        return executor;
    }
}

@Component
public class AsyncEventListener {
    
    @Async("eventExecutor")
    @EventListener
    public void handleAsyncEvent(UserCreatedEvent event) {
        log.info("Processing event asynchronously in thread: {}", 
            Thread.currentThread().getName());
        // Long-running task
    }
    
    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleAsyncAfterCommit(UserCreatedEvent event) {
        // Async processing after transaction commit
    }
}
```

### 10. Event Error Handling

```java
@Component
public class EventErrorHandler implements ApplicationListener<UserCreatedEvent> {
    
    @Override
    public void onApplicationEvent(UserCreatedEvent event) {
        try {
            processEvent(event);
        } catch (Exception e) {
            log.error("Error processing event for user: {}", event.getUserId(), e);
            // Publish error event
            eventPublisher.publishEvent(new EventProcessingFailedEvent(event, e));
        }
    }
}

// Error recovery listener
@Component
public class ErrorRecoveryListener {
    
    @EventListener
    @Async
    public void handleEventFailure(EventProcessingFailedEvent event) {
        log.warn("Event processing failed, attempting recovery");
        // Retry logic or dead letter queue
    }
}
```

## POC Requirements

Build an event-driven system that includes:

1. **Multiple Event Types**: User, Order, Payment events
2. **Sync and Async Listeners**: Demonstrate both patterns
3. **Transactional Events**: Use transaction-bound listeners
4. **Event Ordering**: Show prioritized event processing
5. **Error Handling**: Implement failure handling and recovery
6. **Generic Events**: Create reusable event patterns

## Best Practices

1. **Immutable Events**: Make event classes immutable
2. **Descriptive Names**: Use clear event naming
3. **Async for Heavy Operations**: Use @Async for slow tasks
4. **Transaction Awareness**: Use @TransactionalEventListener when needed
5. **Error Handling**: Always handle exceptions in listeners
6. **Avoid Blocking**: Don't block event publisher

## Common Patterns

```java
// Event hierarchy
public abstract class DomainEvent {
    private final String eventId = UUID.randomUUID().toString();
    private final LocalDateTime occurredAt = LocalDateTime.now();
    // Common fields
}

public class UserCreatedEvent extends DomainEvent {
    // Specific fields
}

// Event aggregator
@Component
public class EventAggregator {
    
    private final List<DomainEvent> events = new CopyOnWriteArrayList<>();
    
    @EventListener
    public void onEvent(DomainEvent event) {
        events.add(event);
        if (events.size() >= 100) {
            processBatch();
        }
    }
}
```

## Additional Resources

- [Spring Events Documentation](https://docs.spring.io/spring-framework/reference/core/beans/context-introduction.html#context-functionality-events)
- [Event-Driven Architecture Patterns](https://martinfowler.com/articles/201701-event-driven.html)

## Next Steps

Continue to [Spring Async](../07-spring-async/README.md) to learn about asynchronous processing.