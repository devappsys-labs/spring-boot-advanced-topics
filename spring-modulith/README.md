# Spring Modulith

## Overview

Spring Modulith is a framework for building well-structured, modular Spring Boot applications. It enforces logical module boundaries within a single deployable unit (a modular monolith), provides inter-module communication via events, generates documentation from your module structure, and validates architectural rules at test time. It gives you the separation of concerns of microservices without the distributed system complexity.

## Key Concepts

### Why Spring Modulith?

- **Enforced boundaries** - Modules can only access each other's public APIs
- **Modular monolith** - Single deployable with clear internal structure
- **Event-driven communication** - Modules interact via events, not direct calls
- **Architecture verification** - Tests that validate module dependencies
- **Auto-documentation** - Generates module diagrams from code
- **Gradual extraction** - Easy path to microservices if/when needed

### Module Structure

```text
com.example.app/
├── Application.java                    ← Root package
├── order/                              ← Order module
│   ├── Order.java                      ← Public API (accessible)
│   ├── OrderService.java              ← Public API (accessible)
│   ├── internal/                       ← Internal (hidden from other modules)
│   │   ├── OrderRepository.java
│   │   ├── OrderValidator.java
│   │   └── OrderMapper.java
│   └── package-info.java
├── inventory/                          ← Inventory module
│   ├── InventoryService.java          ← Public API
│   ├── internal/
│   │   ├── InventoryRepository.java
│   │   └── StockCalculator.java
│   └── package-info.java
├── shipping/                           ← Shipping module
│   ├── ShippingService.java
│   ├── internal/
│   │   └── ShippingRepository.java
│   └── package-info.java
└── notification/                       ← Notification module
    ├── NotificationService.java
    └── internal/
        ├── EmailSender.java
        └── SmsSender.java
```

### Module Visibility Rules

```text
order/OrderService.java        ← Accessible by other modules
order/internal/OrderRepo.java  ← ONLY accessible within order module

inventory/InventoryService     ← Can call order/OrderService ✓
inventory/InventoryService     ← Cannot call order/internal/OrderRepo ✗
```

## Dependencies

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.modulith</groupId>
        <artifactId>spring-modulith-starter-core</artifactId>
    </dependency>
    <!-- For event externalization (optional) -->
    <dependency>
        <groupId>org.springframework.modulith</groupId>
        <artifactId>spring-modulith-events-api</artifactId>
    </dependency>
    <!-- For persisting events in a transactional outbox -->
    <dependency>
        <groupId>org.springframework.modulith</groupId>
        <artifactId>spring-modulith-starter-jpa</artifactId>
    </dependency>
    <!-- For Kafka event externalization -->
    <dependency>
        <groupId>org.springframework.modulith</groupId>
        <artifactId>spring-modulith-events-kafka</artifactId>
    </dependency>
    <!-- For documentation generation -->
    <dependency>
        <groupId>org.springframework.modulith</groupId>
        <artifactId>spring-modulith-docs</artifactId>
        <scope>test</scope>
    </dependency>
    <!-- For architecture verification -->
    <dependency>
        <groupId>org.springframework.modulith</groupId>
        <artifactId>spring-modulith-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.modulith</groupId>
            <artifactId>spring-modulith-bom</artifactId>
            <version>1.2.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

## Module Definition

### Application Root

```java
@SpringBootApplication
public class ShopApplication {
    public static void main(String[] args) {
        SpringApplication.run(ShopApplication.class, args);
    }
}
```

Spring Modulith automatically detects modules as direct sub-packages of the application root package.

### Named Module with package-info.java

```java
// com/example/app/order/package-info.java
@org.springframework.modulith.ApplicationModule(
    allowedDependencies = {"inventory", "notification :: NotificationService"}
)
package com.example.app.order;
```

### Named Module Annotation

```java
@ApplicationModule(
    displayName = "Order Management",
    allowedDependencies = {"inventory", "notification"}
)
package com.example.app.order;
```

## Module APIs (Public Surface)

### Order Module - Public API

```java
// com.example.app.order.OrderService - PUBLIC
@Service
@Transactional
public class OrderService {

    private final OrderRepository orderRepository;
    private final ApplicationEventPublisher events;

    OrderService(OrderRepository orderRepository, ApplicationEventPublisher events) {
        this.orderRepository = orderRepository;
        this.events = events;
    }

    public OrderDto placeOrder(PlaceOrderRequest request) {
        Order order = Order.create(request.customerId(), request.items());
        Order saved = orderRepository.save(order);

        events.publishEvent(new OrderPlaced(
                saved.getId(),
                saved.getCustomerId(),
                saved.getTotalAmount(),
                saved.getItemProductIds()
        ));

        return OrderDto.from(saved);
    }

    public OrderDto getOrder(String orderId) {
        return orderRepository.findById(orderId)
                .map(OrderDto::from)
                .orElseThrow(() -> new OrderNotFoundException(orderId));
    }

    public void completeOrder(String orderId) {
        Order order = orderRepository.findById(orderId)
                .orElseThrow(() -> new OrderNotFoundException(orderId));
        order.complete();
        orderRepository.save(order);

        events.publishEvent(new OrderCompleted(order.getId()));
    }
}
```

### Order Module - Public DTOs and Events

```java
// com.example.app.order.OrderDto - PUBLIC
public record OrderDto(
        String id,
        String customerId,
        BigDecimal totalAmount,
        String status,
        List<OrderItemDto> items
) {
    static OrderDto from(Order order) {
        return new OrderDto(
                order.getId(),
                order.getCustomerId(),
                order.getTotalAmount(),
                order.getStatus().name(),
                order.getItems().stream().map(OrderItemDto::from).toList()
        );
    }
}

// com.example.app.order.OrderPlaced - PUBLIC event
public record OrderPlaced(
        String orderId,
        String customerId,
        BigDecimal totalAmount,
        List<String> productIds
) {
}

// com.example.app.order.OrderCompleted - PUBLIC event
public record OrderCompleted(String orderId) {
}
```

### Order Module - Internal Implementation

```java
// com.example.app.order.internal.Order - INTERNAL
@Entity
@Table(name = "orders")
class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    private String customerId;
    private BigDecimal totalAmount;

    @Enumerated(EnumType.STRING)
    private OrderStatus status;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    private List<OrderItem> items = new ArrayList<>();

    static Order create(String customerId, List<OrderItemRequest> items) {
        Order order = new Order();
        order.customerId = customerId;
        order.status = OrderStatus.PENDING;
        // ... build items and calculate total
        return order;
    }

    void complete() {
        if (status != OrderStatus.CONFIRMED) {
            throw new IllegalStateException("Cannot complete order in state: " + status);
        }
        this.status = OrderStatus.COMPLETED;
    }

    List<String> getItemProductIds() {
        return items.stream().map(OrderItem::getProductId).toList();
    }

    // getters...
}

// com.example.app.order.internal.OrderRepository - INTERNAL
interface OrderRepository extends JpaRepository<Order, String> {
}
```

## Inter-Module Communication via Events

### Inventory Module Listening to Order Events

```java
// com.example.app.inventory.internal.OrderEventListener - INTERNAL
@Component
class OrderEventListener {

    private final InventoryService inventoryService;

    OrderEventListener(InventoryService inventoryService) {
        this.inventoryService = inventoryService;
    }

    @ApplicationModuleListener
    void on(OrderPlaced event) {
        event.productIds().forEach(productId ->
                inventoryService.reserveStock(productId, 1));
    }

    @ApplicationModuleListener
    void on(OrderCompleted event) {
        // Finalize stock deduction
    }
}
```

### Notification Module Listening to Events

```java
// com.example.app.notification.internal.OrderNotificationListener
@Component
class OrderNotificationListener {

    private final NotificationService notificationService;

    OrderNotificationListener(NotificationService notificationService) {
        this.notificationService = notificationService;
    }

    @ApplicationModuleListener
    void on(OrderPlaced event) {
        notificationService.sendOrderConfirmation(event.orderId(), event.customerId());
    }
}
```

### Shipping Module

```java
// com.example.app.shipping.internal.OrderShippingListener
@Component
class OrderShippingListener {

    private final ShippingService shippingService;

    OrderShippingListener(ShippingService shippingService) {
        this.shippingService = shippingService;
    }

    @ApplicationModuleListener
    void on(OrderCompleted event) {
        shippingService.scheduleShipment(event.orderId());
    }
}
```

## Event Publication Registry (Transactional Outbox)

Spring Modulith can persist events to ensure delivery even if a listener fails.

### Configuration

```yaml
spring:
  modulith:
    events:
      jdbc:
        schema-initialization:
          enabled: true    # Auto-create event publication table
    republish-outstanding-events-on-restart: true
```

### How It Works

```text
1. OrderService.placeOrder() starts a transaction
2. Order is saved to database
3. OrderPlaced event is published
4. Event is stored in event_publication table (same transaction)
5. Transaction commits
6. Listeners process the event asynchronously
7. On success → event marked as completed in event_publication
8. On failure → event stays incomplete, retried on restart
```

### Incomplete Event Republication

```java
@Configuration
public class EventConfig {

    @Bean
    ApplicationRunner resubmitIncompletePublications(IncompleteEventPublications publications) {
        return args -> publications.resubmitIncompletePublications(
                event -> true // resubmit all incomplete events
        );
    }
}
```

## Async Event Processing

```java
@Component
class AsyncOrderEventListener {

    @ApplicationModuleListener
    @Async
    void on(OrderPlaced event) {
        // Processed asynchronously, outside the publishing transaction
        // If this fails, the event publication registry tracks it
    }
}
```

## Event Externalization (Kafka/RabbitMQ)

Spring Modulith can automatically externalize events to a message broker.

### Configuration

```yaml
spring:
  modulith:
    events:
      externalization:
        enabled: true
```

### Marking Events for Externalization

```java
// Option 1: @Externalized annotation
@Externalized("order-events::#{#this.orderId()}")
public record OrderPlaced(
        String orderId,
        String customerId,
        BigDecimal totalAmount,
        List<String> productIds
) {
}

// Option 2: Programmatic configuration
@Configuration
class EventExternalizationConfig {

    @Bean
    EventExternalizationConfiguration eventExternalizationConfiguration() {
        return EventExternalizationConfiguration.externalizing()
                .select(EventExternalizationConfiguration.annotatedAsExternalized())
                .route(OrderPlaced.class,
                        it -> RoutingTarget.forTarget("order-events").andKey(it.orderId()))
                .build();
    }
}
```

## Architecture Verification

### Verify Module Structure

```java
@Test
void shouldHaveValidModuleStructure() {
    ApplicationModules modules = ApplicationModules.of(ShopApplication.class);
    modules.verify();
}
```

This test verifies:
- No cyclic dependencies between modules
- No access to internal packages from other modules
- All declared dependencies are valid

### Inspect Module Details

```java
@Test
void shouldPrintModuleStructure() {
    ApplicationModules modules = ApplicationModules.of(ShopApplication.class);

    // Print all modules and their dependencies
    modules.forEach(System.out::println);

    // Access specific module
    ApplicationModule orderModule = modules.getModuleByName("order")
            .orElseThrow();

    System.out.println("Order module dependencies: " +
            orderModule.getDependencies(modules));
}
```

### Output Example

```text
## com.example.app.order ##
> Logical name: order
> Base package: com.example.app.order
> Spring beans:
  + ….OrderService
  + ….internal.OrderEventListener
> Published events:
  + OrderPlaced
  + OrderCompleted
> Listened events:
  (none)
> Dependencies:
  (none)

## com.example.app.inventory ##
> Logical name: inventory
> Dependencies:
  - order (via event listener for OrderPlaced)
```

## Documentation Generation

### Generate Module Diagrams

```java
@Test
void shouldGenerateDocumentation() {
    ApplicationModules modules = ApplicationModules.of(ShopApplication.class);

    // Generate PlantUML and Asciidoc documentation
    new Documenter(modules)
            .writeModulesAsPlantUml()
            .writeIndividualModulesAsPlantUml()
            .writeModuleCanvases();
}
```

This generates:
- `target/modulith-docs/components.puml` - Full module diagram
- `target/modulith-docs/module-order.puml` - Per-module diagram
- `target/modulith-docs/module-order.adoc` - Module canvas documentation

### Generated Diagram (PlantUML)

```text
@startuml
package "order" {
  [OrderService]
}
package "inventory" {
  [InventoryService]
}
package "shipping" {
  [ShippingService]
}
package "notification" {
  [NotificationService]
}

[order] ..> [inventory] : OrderPlaced
[order] ..> [shipping] : OrderCompleted
[order] ..> [notification] : OrderPlaced
@enduml
```

## Integration Testing per Module

### Test a Single Module in Isolation

```java
@ApplicationModuleTest
class OrderModuleTest {

    @Autowired
    private OrderService orderService;

    @Test
    void shouldPlaceOrder(Scenario scenario) {
        scenario.stimulate(() -> orderService.placeOrder(new PlaceOrderRequest(
                        "customer-1",
                        List.of(new OrderItemRequest("prod-1", 2, new BigDecimal("29.99"))))))
                .andWaitForEventOfType(OrderPlaced.class)
                .toArriveAndVerify(event -> {
                    assertThat(event.customerId()).isEqualTo("customer-1");
                    assertThat(event.totalAmount()).isEqualByComparingTo(new BigDecimal("59.98"));
                });
    }
}
```

### Test Module Interaction

```java
@ApplicationModuleTest
class InventoryModuleTest {

    @Autowired
    private InventoryService inventoryService;

    @Test
    void shouldReserveStockOnOrderPlaced(Scenario scenario) {
        // Given
        inventoryService.addStock("prod-1", 10);

        // When: Simulate an OrderPlaced event
        scenario.publish(new OrderPlaced("order-1", "customer-1",
                        new BigDecimal("29.99"), List.of("prod-1")))
                .andWaitForStateChange(() ->
                        inventoryService.getAvailableStock("prod-1"))
                .andVerify(stock -> assertThat(stock).isEqualTo(9));
    }
}
```

### Scenario API

```java
@ApplicationModuleTest
class ShippingModuleTest {

    @Test
    void shouldScheduleShipmentOnOrderCompleted(Scenario scenario) {
        scenario.publish(new OrderCompleted("order-1"))
                .andWaitForEventOfType(ShipmentScheduled.class)
                .toArriveAndVerify(event -> {
                    assertThat(event.orderId()).isEqualTo("order-1");
                });
    }
}
```

## Named Interfaces (Exposing Specific Types)

```java
// Expose only specific types from a module
@NamedInterface("api")
package com.example.app.order;

// In package-info.java of a sub-package
@NamedInterface("events")
package com.example.app.order.events;
```

```java
// Another module can depend on specific named interfaces
@ApplicationModule(
    allowedDependencies = {"order :: api", "order :: events"}
)
package com.example.app.inventory;
```

## Open Modules

For gradual migration, you can temporarily relax module boundaries.

```java
@ApplicationModule(type = ApplicationModule.Type.OPEN)
package com.example.app.legacy;
```

Open modules allow access to their internal packages. Use this temporarily during migration, then tighten to closed modules.

## Best Practices

1. **Start with a modular monolith** - Easier to split into microservices later than to merge services
2. **Keep module APIs minimal** - Expose DTOs and events, not entities
3. **Communicate via events** - Avoid direct service-to-service calls between modules
4. **Verify architecture in CI** - Run `modules.verify()` in every build
5. **Use `internal` packages** - Hide implementation details from other modules
6. **Generate documentation** - Keep module diagrams up to date automatically
7. **Use the Scenario API for testing** - Test event flows between modules
8. **Enable event publication registry** - Ensure no events are lost

## Common Pitfalls

1. Putting everything in the module root package instead of using `internal` sub-packages
2. Sharing JPA entities across modules instead of using DTOs/events
3. Circular dependencies between modules (detected by `verify()`)
4. Not using `@ApplicationModuleListener` (which includes `@Async` + `@Transactional`)
5. Testing modules together instead of in isolation with the Scenario API
6. Forgetting to enable event publication table initialization for the outbox pattern

## Resources

- [Spring Modulith Documentation](https://docs.spring.io/spring-modulith/reference/)
- [Spring Modulith GitHub](https://github.com/spring-projects/spring-modulith)
- [Modular Monoliths by Simon Brown](https://www.youtube.com/watch?v=5OjqD-ow8GE)
- [Oliver Drotbohm - Spring Modulith Talk](https://www.youtube.com/watch?v=MuGmzcKfPcw)
