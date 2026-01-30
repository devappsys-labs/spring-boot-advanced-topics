# Spring State Machine

## Overview

Spring State Machine is a framework for building state machine concepts with Spring applications. It provides a powerful way to model complex business logic with well-defined states and transitions, making it easier to manage workflows, order processing, and other stateful processes.

## Key Concepts

### State Machine Components

- **State** - Represents a specific condition or status
- **Event** - Triggers that cause state transitions
- **Transition** - Movement from one state to another
- **Action** - Operations executed during transitions
- **Guard** - Conditions that must be met for transitions

### State Machine Types

- **Simple State Machine** - Basic states and transitions
- **Hierarchical State Machine** - Nested states
- **Concurrent State Machine** - Multiple regions executing simultaneously
- **Junction/Choice** - Conditional branching

## Dependencies

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.statemachine</groupId>
        <artifactId>spring-statemachine-core</artifactId>
        <version>3.2.0</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.statemachine</groupId>
        <artifactId>spring-statemachine-starter</artifactId>
        <version>3.2.0</version>
    </dependency>
</dependencies>
```

## Basic State Machine Example

### Order Processing State Machine

```java
// States
public enum OrderState {
    CREATED,
    PAYMENT_PENDING,
    PAID,
    PROCESSING,
    SHIPPED,
    DELIVERED,
    CANCELLED
}

// Events
public enum OrderEvent {
    SUBMIT_ORDER,
    PAYMENT_RECEIVED,
    PAYMENT_FAILED,
    START_PROCESSING,
    SHIP_ORDER,
    DELIVER_ORDER,
    CANCEL_ORDER
}
```

### State Machine Configuration

```java
@Configuration
@EnableStateMachine
public class OrderStateMachineConfig extends StateMachineConfigurerAdapter<OrderState, OrderEvent> {
    
    @Override
    public void configure(StateMachineStateConfigurer<OrderState, OrderEvent> states) throws Exception {
        states
            .withStates()
                .initial(OrderState.CREATED)
                .state(OrderState.PAYMENT_PENDING)
                .state(OrderState.PAID)
                .state(OrderState.PROCESSING)
                .state(OrderState.SHIPPED)
                .end(OrderState.DELIVERED)
                .end(OrderState.CANCELLED);
    }
    
    @Override
    public void configure(StateMachineTransitionConfigurer<OrderState, OrderEvent> transitions) throws Exception {
        transitions
            .withExternal()
                .source(OrderState.CREATED)
                .target(OrderState.PAYMENT_PENDING)
                .event(OrderEvent.SUBMIT_ORDER)
                .and()
            .withExternal()
                .source(OrderState.PAYMENT_PENDING)
                .target(OrderState.PAID)
                .event(OrderEvent.PAYMENT_RECEIVED)
                .action(paymentReceivedAction())
                .and()
            .withExternal()
                .source(OrderState.PAYMENT_PENDING)
                .target(OrderState.CANCELLED)
                .event(OrderEvent.PAYMENT_FAILED)
                .and()
            .withExternal()
                .source(OrderState.PAID)
                .target(OrderState.PROCESSING)
                .event(OrderEvent.START_PROCESSING)
                .and()
            .withExternal()
                .source(OrderState.PROCESSING)
                .target(OrderState.SHIPPED)
                .event(OrderEvent.SHIP_ORDER)
                .guard(shippingGuard())
                .action(shipOrderAction())
                .and()
            .withExternal()
                .source(OrderState.SHIPPED)
                .target(OrderState.DELIVERED)
                .event(OrderEvent.DELIVER_ORDER)
                .and()
            .withExternal()
                .source(OrderState.CREATED)
                .target(OrderState.CANCELLED)
                .event(OrderEvent.CANCEL_ORDER)
                .and()
            .withExternal()
                .source(OrderState.PAYMENT_PENDING)
                .target(OrderState.CANCELLED)
                .event(OrderEvent.CANCEL_ORDER);
    }
    
    @Bean
    public Action<OrderState, OrderEvent> paymentReceivedAction() {
        return context -> {
            System.out.println("Payment received for order: " + context.getMessage().getHeaders().get("orderId"));
            // Send confirmation email, update database, etc.
        };
    }
    
    @Bean
    public Action<OrderState, OrderEvent> shipOrderAction() {
        return context -> {
            System.out.println("Shipping order: " + context.getMessage().getHeaders().get("orderId"));
            // Generate shipping label, notify logistics, etc.
        };
    }
    
    @Bean
    public Guard<OrderState, OrderEvent> shippingGuard() {
        return context -> {
            // Check if shipping address is valid, items are in stock, etc.
            Boolean canShip = (Boolean) context.getMessage().getHeaders().get("canShip");
            return canShip != null && canShip;
        };
    }
}
```

### State Machine Service

```java
@Service
public class OrderStateMachineService {
    
    @Autowired
    private StateMachine<OrderState, OrderEvent> stateMachine;
    
    public void processOrderEvent(Long orderId, OrderEvent event, Map<String, Object> headers) {
        Message<OrderEvent> message = MessageBuilder
                .withPayload(event)
                .setHeader("orderId", orderId)
                .copyHeaders(headers)
                .build();
        
        stateMachine.sendEvent(message);
    }
    
    public OrderState getCurrentState() {
        return stateMachine.getState().getId();
    }
}
```

## Persistent State Machine

### State Machine Persister Configuration

```java
@Configuration
public class PersistentStateMachineConfig extends StateMachineConfigurerAdapter<OrderState, OrderEvent> {
    
    @Autowired
    private StateMachinePersist<OrderState, OrderEvent, String> stateMachinePersist;
    
    @Override
    public void configure(StateMachineConfigurationConfigurer<OrderState, OrderEvent> config) throws Exception {
        config
            .withConfiguration()
                .autoStartup(false)
            .and()
            .withPersistence()
                .runtimePersister(stateMachinePersist);
    }
    
    @Bean
    public StateMachinePersist<OrderState, OrderEvent, String> stateMachinePersist() {
        return new InMemoryStateMachinePersist<>();
    }
}
```

### Database-Backed Persistence

```java
@Entity
@Table(name = "state_machine_context")
public class StateMachineContextEntity {
    
    @Id
    private String machineId;
    
    @Lob
    private byte[] stateMachineContext;
    
    @Column(name = "created_at")
    private LocalDateTime createdAt;
    
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
    
    // Getters and setters
}

@Repository
public interface StateMachineRepository extends JpaRepository<StateMachineContextEntity, String> {
}

@Component
public class JpaPersistStateMachineHandler implements StateMachinePersist<OrderState, OrderEvent, String> {
    
    @Autowired
    private StateMachineRepository repository;
    
    @Override
    public void write(StateMachineContext<OrderState, OrderEvent> context, String contextObj) {
        StateMachineContextEntity entity = new StateMachineContextEntity();
        entity.setMachineId(contextObj);
        entity.setStateMachineContext(serializeContext(context));
        entity.setUpdatedAt(LocalDateTime.now());
        repository.save(entity);
    }
    
    @Override
    public StateMachineContext<OrderState, OrderEvent> read(String contextObj) {
        return repository.findById(contextObj)
                .map(entity -> deserializeContext(entity.getStateMachineContext()))
                .orElse(null);
    }
    
    private byte[] serializeContext(StateMachineContext<OrderState, OrderEvent> context) {
        // Serialization logic
        return new byte[0];
    }
    
    private StateMachineContext<OrderState, OrderEvent> deserializeContext(byte[] data) {
        // Deserialization logic
        return null;
    }
}
```

## State Machine Factory

### Factory Configuration

```java
@Configuration
@EnableStateMachineFactory
public class OrderStateMachineFactoryConfig extends StateMachineConfigurerAdapter<OrderState, OrderEvent> {
    
    @Override
    public void configure(StateMachineStateConfigurer<OrderState, OrderEvent> states) throws Exception {
        states
            .withStates()
                .initial(OrderState.CREATED)
                .state(OrderState.PAYMENT_PENDING)
                .state(OrderState.PAID)
                .state(OrderState.PROCESSING)
                .state(OrderState.SHIPPED)
                .end(OrderState.DELIVERED)
                .end(OrderState.CANCELLED);
    }
    
    @Override
    public void configure(StateMachineTransitionConfigurer<OrderState, OrderEvent> transitions) throws Exception {
        // Same as before
    }
    
    @Override
    public void configure(StateMachineConfigurationConfigurer<OrderState, OrderEvent> config) throws Exception {
        config
            .withConfiguration()
                .autoStartup(true)
                .listener(stateMachineListener());
    }
    
    @Bean
    public StateMachineListener<OrderState, OrderEvent> stateMachineListener() {
        return new StateMachineListenerAdapter<OrderState, OrderEvent>() {
            @Override
            public void stateChanged(State<OrderState, OrderEvent> from, State<OrderState, OrderEvent> to) {
                System.out.println("State changed from " + (from != null ? from.getId() : "none") + 
                                 " to " + to.getId());
            }
            
            @Override
            public void eventNotAccepted(Message<OrderEvent> event) {
                System.out.println("Event not accepted: " + event.getPayload());
            }
        };
    }
}
```

### Using State Machine Factory

```java
@Service
public class OrderProcessingService {
    
    @Autowired
    private StateMachineFactory<OrderState, OrderEvent> stateMachineFactory;
    
    @Autowired
    private StateMachinePersist<OrderState, OrderEvent, String> stateMachinePersist;
    
    public void processOrder(String orderId, OrderEvent event) {
        StateMachine<OrderState, OrderEvent> stateMachine = stateMachineFactory.getStateMachine(orderId);
        
        // Restore state from persistence
        try {
            StateMachineContext<OrderState, OrderEvent> context = stateMachinePersist.read(orderId);
            if (context != null) {
                stateMachine.getStateMachineAccessor()
                        .doWithAllRegions(access -> access.resetStateMachine(context));
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        
        // Start the state machine
        stateMachine.start();
        
        // Send event
        Message<OrderEvent> message = MessageBuilder
                .withPayload(event)
                .setHeader("orderId", orderId)
                .build();
        
        stateMachine.sendEvent(message);
        
        // Persist state
        try {
            stateMachinePersist.write(
                    stateMachine.getStateMachineAccessor()
                            .doWithAllRegions(access -> access.getStateMachine())
                            .get(0)
                            .getExtendedState()
                            .get("stateMachineContext", StateMachineContext.class),
                    orderId
            );
        } catch (Exception e) {
            e.printStackTrace();
        }
        
        // Stop the state machine
        stateMachine.stop();
    }
    
    public OrderState getOrderState(String orderId) {
        StateMachine<OrderState, OrderEvent> stateMachine = stateMachineFactory.getStateMachine(orderId);
        
        try {
            StateMachineContext<OrderState, OrderEvent> context = stateMachinePersist.read(orderId);
            if (context != null) {
                stateMachine.getStateMachineAccessor()
                        .doWithAllRegions(access -> access.resetStateMachine(context));
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        
        stateMachine.start();
        OrderState state = stateMachine.getState().getId();
        stateMachine.stop();
        
        return state;
    }
}
```

## Advanced Patterns

### Hierarchical State Machine

```java
@Configuration
@EnableStateMachine
public class HierarchicalStateMachineConfig extends StateMachineConfigurerAdapter<String, String> {
    
    @Override
    public void configure(StateMachineStateConfigurer<String, String> states) throws Exception {
        states
            .withStates()
                .initial("IDLE")
                .state("ACTIVE")
                .and()
                .withStates()
                    .parent("ACTIVE")
                    .initial("RUNNING")
                    .state("PAUSED")
                    .state("ERROR");
    }
    
    @Override
    public void configure(StateMachineTransitionConfigurer<String, String> transitions) throws Exception {
        transitions
            .withExternal()
                .source("IDLE").target("ACTIVE").event("START")
                .and()
            .withExternal()
                .source("ACTIVE").target("IDLE").event("STOP")
                .and()
            .withExternal()
                .source("RUNNING").target("PAUSED").event("PAUSE")
                .and()
            .withExternal()
                .source("PAUSED").target("RUNNING").event("RESUME")
                .and()
            .withExternal()
                .source("RUNNING").target("ERROR").event("ERROR");
    }
}
```

### Choice/Junction State

```java
@Override
public void configure(StateMachineStateConfigurer<OrderState, OrderEvent> states) throws Exception {
    states
        .withStates()
            .initial(OrderState.CREATED)
            .choice(OrderState.PAYMENT_PENDING)
            .state(OrderState.PAID)
            .state(OrderState.CANCELLED);
}

@Override
public void configure(StateMachineTransitionConfigurer<OrderState, OrderEvent> transitions) throws Exception {
    transitions
        .withChoice()
            .source(OrderState.PAYMENT_PENDING)
            .first(OrderState.PAID, paymentSuccessGuard())
            .last(OrderState.CANCELLED);
}

@Bean
public Guard<OrderState, OrderEvent> paymentSuccessGuard() {
    return context -> {
        Boolean paymentSuccess = (Boolean) context.getMessageHeader("paymentSuccess");
        return paymentSuccess != null && paymentSuccess;
    };
}
```

### State Machine Actions

```java
public class OrderActions {
    
    @Bean
    public Action<OrderState, OrderEvent> entryAction() {
        return context -> {
            System.out.println("Entering state: " + context.getTarget().getId());
        };
    }
    
    @Bean
    public Action<OrderState, OrderEvent> exitAction() {
        return context -> {
            System.out.println("Exiting state: " + context.getSource().getId());
        };
    }
    
    @Bean
    public Action<OrderState, OrderEvent> stateDoAction() {
        return context -> {
            System.out.println("Doing action in state: " + context.getTarget().getId());
            // Perform state-specific operations
        };
    }
    
    @Bean
    public Action<OrderState, OrderEvent> errorAction() {
        return context -> {
            Exception exception = context.getException();
            System.err.println("Error occurred: " + exception.getMessage());
            // Handle error, rollback, notify, etc.
        };
    }
}

@Override
public void configure(StateMachineStateConfigurer<OrderState, OrderEvent> states) throws Exception {
    states
        .withStates()
            .initial(OrderState.CREATED)
            .state(OrderState.PROCESSING, stateDoAction(), errorAction())
            .state(OrderState.SHIPPED)
            .stateEntry(OrderState.SHIPPED, entryAction())
            .stateExit(OrderState.SHIPPED, exitAction());
}
```

## REST Controller Integration

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    @Autowired
    private OrderProcessingService orderProcessingService;
    
    @PostMapping("/{orderId}/events/{event}")
    public ResponseEntity<String> sendEvent(@PathVariable String orderId, 
                                           @PathVariable OrderEvent event) {
        orderProcessingService.processOrder(orderId, event);
        return ResponseEntity.ok("Event processed");
    }
    
    @GetMapping("/{orderId}/state")
    public ResponseEntity<OrderState> getOrderState(@PathVariable String orderId) {
        OrderState state = orderProcessingService.getOrderState(orderId);
        return ResponseEntity.ok(state);
    }
}
```

## Testing

```java
@SpringBootTest
class OrderStateMachineTest {
    
    @Autowired
    private StateMachineFactory<OrderState, OrderEvent> stateMachineFactory;
    
    @Test
    void testOrderWorkflow() {
        StateMachine<OrderState, OrderEvent> stateMachine = stateMachineFactory.getStateMachine("order-123");
        stateMachine.start();
        
        // Initially in CREATED state
        assertEquals(OrderState.CREATED, stateMachine.getState().getId());
        
        // Submit order
        stateMachine.sendEvent(OrderEvent.SUBMIT_ORDER);
        assertEquals(OrderState.PAYMENT_PENDING, stateMachine.getState().getId());
        
        // Payment received
        stateMachine.sendEvent(OrderEvent.PAYMENT_RECEIVED);
        assertEquals(OrderState.PAID, stateMachine.getState().getId());
        
        // Start processing
        stateMachine.sendEvent(OrderEvent.START_PROCESSING);
        assertEquals(OrderState.PROCESSING, stateMachine.getState().getId());
        
        // Ship order
        Message<OrderEvent> message = MessageBuilder
                .withPayload(OrderEvent.SHIP_ORDER)
                .setHeader("canShip", true)
                .build();
        stateMachine.sendEvent(message);
        assertEquals(OrderState.SHIPPED, stateMachine.getState().getId());
        
        // Deliver order
        stateMachine.sendEvent(OrderEvent.DELIVER_ORDER);
        assertEquals(OrderState.DELIVERED, stateMachine.getState().getId());
        
        stateMachine.stop();
    }
    
    @Test
    void testOrderCancellation() {
        StateMachine<OrderState, OrderEvent> stateMachine = stateMachineFactory.getStateMachine("order-456");
        stateMachine.start();
        
        assertEquals(OrderState.CREATED, stateMachine.getState().getId());
        
        // Cancel order
        stateMachine.sendEvent(OrderEvent.CANCEL_ORDER);
        assertEquals(OrderState.CANCELLED, stateMachine.getState().getId());
        
        stateMachine.stop();
    }
}
```

## Best Practices

1. **Keep states simple** - Each state should have a single responsibility
2. **Use guards wisely** - Validate transitions before they occur
3. **Persist state** - For long-running processes
4. **Handle errors** - Implement error states and recovery mechanisms
5. **Use listeners** - For audit logging and monitoring
6. **Test thoroughly** - Test all state transitions and edge cases
7. **Document states** - Create state diagrams for complex workflows
8. **Use factory pattern** - For managing multiple state machine instances

## Common Use Cases

- Order processing workflows
- Document approval processes
- User authentication flows
- Payment processing
- IoT device state management
- Game state management
- Workflow automation

## Common Pitfalls

1. Over-complicating state machines
2. Not persisting state for long-running processes
3. Ignoring error handling
4. Not using guards for validation
5. Creating circular dependencies
6. Not testing state transitions thoroughly

## Resources

- [Spring State Machine Documentation](https://docs.spring.io/spring-statemachine/docs/current/reference/)
- [Spring State Machine Samples](https://github.com/spring-projects/spring-statemachine/tree/main/spring-statemachine-samples)
