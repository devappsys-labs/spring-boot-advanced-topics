# CQRS (Command Query Responsibility Segregation)

## Overview

CQRS is an architectural pattern that separates read (Query) and write (Command) operations into distinct models. Instead of a single model that handles both reads and writes, you build two separate models optimized for their specific purpose. The write model focuses on business rules and data integrity, while the read model is optimized for query performance and UI projections.

## Key Concepts

### Why CQRS?

- **Independent scaling** - Scale read and write sides separately
- **Optimized models** - Each side uses a model best suited for its job
- **Simplified commands** - Write model focuses purely on business logic
- **Flexible read models** - Create multiple projections for different views
- **Event-driven friendly** - Pairs naturally with Event Sourcing and messaging

### Core Components

- **Command** - An intent to change state (e.g., `PlaceOrderCommand`)
- **Command Handler** - Processes a command and applies business rules
- **Query** - A request for data (e.g., `GetOrderByIdQuery`)
- **Query Handler** - Reads and returns data from the read model
- **Event** - A record of something that happened (e.g., `OrderPlacedEvent`)
- **Projection** - A read-optimized view built from events

### Architecture

```text
                    ┌──────────────┐
   Write Side       │   Command    │     Read Side
                    │   Gateway    │
                    └──────┬───────┘
                           │
              ┌────────────┴────────────┐
              │                         │
      ┌───────▼───────┐        ┌───────▼───────┐
      │   Command     │        │    Query      │
      │   Handler     │        │    Handler    │
      └───────┬───────┘        └───────┬───────┘
              │                        │
      ┌───────▼───────┐        ┌───────▼───────┐
      │  Write Model  │        │  Read Model   │
      │  (Normalized) │        │ (Denormalized)│
      └───────┬───────┘        └───────────────┘
              │                        ▲
              │    ┌──────────┐        │
              └───►│  Events  ├────────┘
                   └──────────┘
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
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
    <!-- Optional: For async event-driven sync between models -->
    <dependency>
        <groupId>org.springframework.kafka</groupId>
        <artifactId>spring-kafka</artifactId>
    </dependency>
</dependencies>
```

## Domain Model - Order Example

### Write Model (Command Side)

#### Order Aggregate

```java
@Entity
@Table(name = "orders")
@Data
@NoArgsConstructor
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    @Column(nullable = false)
    private String customerId;

    @Enumerated(EnumType.STRING)
    private OrderStatus status;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();

    private BigDecimal totalAmount;

    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;

    public static Order create(String customerId, List<OrderItem> items) {
        Order order = new Order();
        order.customerId = customerId;
        order.status = OrderStatus.PENDING;
        order.items = items;
        order.totalAmount = items.stream()
                .map(OrderItem::getSubtotal)
                .reduce(BigDecimal.ZERO, BigDecimal::add);
        order.createdAt = LocalDateTime.now();
        order.updatedAt = LocalDateTime.now();
        items.forEach(item -> item.setOrder(order));
        return order;
    }

    public void confirm() {
        if (status != OrderStatus.PENDING) {
            throw new IllegalStateException("Only PENDING orders can be confirmed");
        }
        this.status = OrderStatus.CONFIRMED;
        this.updatedAt = LocalDateTime.now();
    }

    public void ship() {
        if (status != OrderStatus.CONFIRMED) {
            throw new IllegalStateException("Only CONFIRMED orders can be shipped");
        }
        this.status = OrderStatus.SHIPPED;
        this.updatedAt = LocalDateTime.now();
    }

    public void cancel() {
        if (status == OrderStatus.SHIPPED || status == OrderStatus.DELIVERED) {
            throw new IllegalStateException("Cannot cancel shipped/delivered orders");
        }
        this.status = OrderStatus.CANCELLED;
        this.updatedAt = LocalDateTime.now();
    }
}

public enum OrderStatus {
    PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED
}

@Entity
@Table(name = "order_items")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class OrderItem {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String productId;
    private String productName;
    private int quantity;
    private BigDecimal unitPrice;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id")
    private Order order;

    public BigDecimal getSubtotal() {
        return unitPrice.multiply(BigDecimal.valueOf(quantity));
    }
}
```

### Read Model (Query Side)

```java
@Entity
@Table(name = "order_summary_view")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class OrderSummaryView {

    @Id
    private String orderId;

    private String customerId;
    private String customerName;
    private String status;
    private BigDecimal totalAmount;
    private int itemCount;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}

@Entity
@Table(name = "order_detail_view")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class OrderDetailView {

    @Id
    private String orderId;

    private String customerId;
    private String customerName;
    private String customerEmail;
    private String status;
    private BigDecimal totalAmount;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;

    @ElementCollection(fetch = FetchType.EAGER)
    @CollectionTable(name = "order_detail_items", joinColumns = @JoinColumn(name = "order_id"))
    private List<OrderItemView> items = new ArrayList<>();
}

@Embeddable
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class OrderItemView {
    private String productId;
    private String productName;
    private int quantity;
    private BigDecimal unitPrice;
    private BigDecimal subtotal;
}
```

## Commands and Command Handlers

### Command Definitions

```java
public sealed interface OrderCommand permits
        PlaceOrderCommand, ConfirmOrderCommand, ShipOrderCommand, CancelOrderCommand {
}

public record PlaceOrderCommand(
        String customerId,
        List<OrderItemRequest> items
) implements OrderCommand {
}

public record ConfirmOrderCommand(String orderId) implements OrderCommand {
}

public record ShipOrderCommand(String orderId) implements OrderCommand {
}

public record CancelOrderCommand(String orderId, String reason) implements OrderCommand {
}

public record OrderItemRequest(
        String productId,
        String productName,
        int quantity,
        BigDecimal unitPrice
) {
}
```

### Command Handler

```java
@Service
@Transactional
public class OrderCommandHandler {

    private final OrderRepository orderRepository;
    private final ApplicationEventPublisher eventPublisher;

    public OrderCommandHandler(OrderRepository orderRepository,
                                ApplicationEventPublisher eventPublisher) {
        this.orderRepository = orderRepository;
        this.eventPublisher = eventPublisher;
    }

    public String handle(PlaceOrderCommand command) {
        List<OrderItem> items = command.items().stream()
                .map(req -> OrderItem.builder()
                        .productId(req.productId())
                        .productName(req.productName())
                        .quantity(req.quantity())
                        .unitPrice(req.unitPrice())
                        .build())
                .toList();

        Order order = Order.create(command.customerId(), items);
        Order saved = orderRepository.save(order);

        eventPublisher.publishEvent(new OrderPlacedEvent(
                saved.getId(),
                saved.getCustomerId(),
                saved.getTotalAmount(),
                saved.getItems().stream()
                        .map(i -> new OrderItemDetail(i.getProductId(), i.getProductName(),
                                i.getQuantity(), i.getUnitPrice(), i.getSubtotal()))
                        .toList(),
                saved.getCreatedAt()
        ));

        return saved.getId();
    }

    public void handle(ConfirmOrderCommand command) {
        Order order = findOrder(command.orderId());
        order.confirm();
        orderRepository.save(order);

        eventPublisher.publishEvent(new OrderConfirmedEvent(
                order.getId(), LocalDateTime.now()));
    }

    public void handle(ShipOrderCommand command) {
        Order order = findOrder(command.orderId());
        order.ship();
        orderRepository.save(order);

        eventPublisher.publishEvent(new OrderShippedEvent(
                order.getId(), LocalDateTime.now()));
    }

    public void handle(CancelOrderCommand command) {
        Order order = findOrder(command.orderId());
        order.cancel();
        orderRepository.save(order);

        eventPublisher.publishEvent(new OrderCancelledEvent(
                order.getId(), command.reason(), LocalDateTime.now()));
    }

    private Order findOrder(String orderId) {
        return orderRepository.findById(orderId)
                .orElseThrow(() -> new OrderNotFoundException("Order not found: " + orderId));
    }
}
```

## Events

### Domain Events

```java
public sealed interface OrderEvent permits
        OrderPlacedEvent, OrderConfirmedEvent, OrderShippedEvent, OrderCancelledEvent {
}

public record OrderPlacedEvent(
        String orderId,
        String customerId,
        BigDecimal totalAmount,
        List<OrderItemDetail> items,
        LocalDateTime occurredAt
) implements OrderEvent {
}

public record OrderConfirmedEvent(
        String orderId,
        LocalDateTime occurredAt
) implements OrderEvent {
}

public record OrderShippedEvent(
        String orderId,
        LocalDateTime occurredAt
) implements OrderEvent {
}

public record OrderCancelledEvent(
        String orderId,
        String reason,
        LocalDateTime occurredAt
) implements OrderEvent {
}

public record OrderItemDetail(
        String productId,
        String productName,
        int quantity,
        BigDecimal unitPrice,
        BigDecimal subtotal
) {
}
```

## Projections (Read Model Updaters)

### Event Listeners that Build Read Models

```java
@Component
@Transactional
public class OrderProjection {

    private final OrderSummaryViewRepository summaryRepository;
    private final OrderDetailViewRepository detailRepository;
    private final CustomerService customerService;

    public OrderProjection(OrderSummaryViewRepository summaryRepository,
                           OrderDetailViewRepository detailRepository,
                           CustomerService customerService) {
        this.summaryRepository = summaryRepository;
        this.detailRepository = detailRepository;
        this.customerService = customerService;
    }

    @EventListener
    public void on(OrderPlacedEvent event) {
        CustomerInfo customer = customerService.getCustomer(event.customerId());

        // Build summary view
        OrderSummaryView summary = OrderSummaryView.builder()
                .orderId(event.orderId())
                .customerId(event.customerId())
                .customerName(customer.name())
                .status("PENDING")
                .totalAmount(event.totalAmount())
                .itemCount(event.items().size())
                .createdAt(event.occurredAt())
                .updatedAt(event.occurredAt())
                .build();
        summaryRepository.save(summary);

        // Build detail view
        List<OrderItemView> itemViews = event.items().stream()
                .map(item -> OrderItemView.builder()
                        .productId(item.productId())
                        .productName(item.productName())
                        .quantity(item.quantity())
                        .unitPrice(item.unitPrice())
                        .subtotal(item.subtotal())
                        .build())
                .toList();

        OrderDetailView detail = OrderDetailView.builder()
                .orderId(event.orderId())
                .customerId(event.customerId())
                .customerName(customer.name())
                .customerEmail(customer.email())
                .status("PENDING")
                .totalAmount(event.totalAmount())
                .items(itemViews)
                .createdAt(event.occurredAt())
                .updatedAt(event.occurredAt())
                .build();
        detailRepository.save(detail);
    }

    @EventListener
    public void on(OrderConfirmedEvent event) {
        updateStatus(event.orderId(), "CONFIRMED", event.occurredAt());
    }

    @EventListener
    public void on(OrderShippedEvent event) {
        updateStatus(event.orderId(), "SHIPPED", event.occurredAt());
    }

    @EventListener
    public void on(OrderCancelledEvent event) {
        updateStatus(event.orderId(), "CANCELLED", event.occurredAt());
    }

    private void updateStatus(String orderId, String status, LocalDateTime updatedAt) {
        summaryRepository.findById(orderId).ifPresent(summary -> {
            summary.setStatus(status);
            summary.setUpdatedAt(updatedAt);
            summaryRepository.save(summary);
        });

        detailRepository.findById(orderId).ifPresent(detail -> {
            detail.setStatus(status);
            detail.setUpdatedAt(updatedAt);
            detailRepository.save(detail);
        });
    }
}
```

## Queries and Query Handlers

### Query Definitions

```java
public sealed interface OrderQuery permits
        GetOrderByIdQuery, GetOrdersByCustomerQuery, GetOrderSummariesQuery {
}

public record GetOrderByIdQuery(String orderId) implements OrderQuery {
}

public record GetOrdersByCustomerQuery(
        String customerId,
        int page,
        int size
) implements OrderQuery {
}

public record GetOrderSummariesQuery(
        String status,
        int page,
        int size
) implements OrderQuery {
}
```

### Query Handler

```java
@Service
@Transactional(readOnly = true)
public class OrderQueryHandler {

    private final OrderSummaryViewRepository summaryRepository;
    private final OrderDetailViewRepository detailRepository;

    public OrderQueryHandler(OrderSummaryViewRepository summaryRepository,
                              OrderDetailViewRepository detailRepository) {
        this.summaryRepository = summaryRepository;
        this.detailRepository = detailRepository;
    }

    public OrderDetailView handle(GetOrderByIdQuery query) {
        return detailRepository.findById(query.orderId())
                .orElseThrow(() -> new OrderNotFoundException(
                        "Order not found: " + query.orderId()));
    }

    public Page<OrderSummaryView> handle(GetOrdersByCustomerQuery query) {
        return summaryRepository.findByCustomerId(
                query.customerId(),
                PageRequest.of(query.page(), query.size(), Sort.by("createdAt").descending())
        );
    }

    public Page<OrderSummaryView> handle(GetOrderSummariesQuery query) {
        if (query.status() != null) {
            return summaryRepository.findByStatus(
                    query.status(),
                    PageRequest.of(query.page(), query.size(), Sort.by("createdAt").descending())
            );
        }
        return summaryRepository.findAll(
                PageRequest.of(query.page(), query.size(), Sort.by("createdAt").descending())
        );
    }
}
```

## Repositories

```java
// Write side
public interface OrderRepository extends JpaRepository<Order, String> {
}

// Read side
public interface OrderSummaryViewRepository extends JpaRepository<OrderSummaryView, String> {
    Page<OrderSummaryView> findByCustomerId(String customerId, Pageable pageable);
    Page<OrderSummaryView> findByStatus(String status, Pageable pageable);
}

public interface OrderDetailViewRepository extends JpaRepository<OrderDetailView, String> {
}
```

## REST Controllers

### Command Controller (Write)

```java
@RestController
@RequestMapping("/api/orders")
public class OrderCommandController {

    private final OrderCommandHandler commandHandler;

    public OrderCommandController(OrderCommandHandler commandHandler) {
        this.commandHandler = commandHandler;
    }

    @PostMapping
    public ResponseEntity<Map<String, String>> placeOrder(
            @RequestBody @Valid PlaceOrderRequest request) {
        String orderId = commandHandler.handle(new PlaceOrderCommand(
                request.customerId(), request.items()));
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(Map.of("orderId", orderId));
    }

    @PostMapping("/{orderId}/confirm")
    public ResponseEntity<Void> confirmOrder(@PathVariable String orderId) {
        commandHandler.handle(new ConfirmOrderCommand(orderId));
        return ResponseEntity.ok().build();
    }

    @PostMapping("/{orderId}/ship")
    public ResponseEntity<Void> shipOrder(@PathVariable String orderId) {
        commandHandler.handle(new ShipOrderCommand(orderId));
        return ResponseEntity.ok().build();
    }

    @PostMapping("/{orderId}/cancel")
    public ResponseEntity<Void> cancelOrder(@PathVariable String orderId,
                                             @RequestParam(required = false) String reason) {
        commandHandler.handle(new CancelOrderCommand(orderId, reason));
        return ResponseEntity.ok().build();
    }
}
```

### Query Controller (Read)

```java
@RestController
@RequestMapping("/api/orders")
public class OrderQueryController {

    private final OrderQueryHandler queryHandler;

    public OrderQueryController(OrderQueryHandler queryHandler) {
        this.queryHandler = queryHandler;
    }

    @GetMapping("/{orderId}")
    public ResponseEntity<OrderDetailView> getOrder(@PathVariable String orderId) {
        return ResponseEntity.ok(queryHandler.handle(new GetOrderByIdQuery(orderId)));
    }

    @GetMapping("/customer/{customerId}")
    public ResponseEntity<Page<OrderSummaryView>> getOrdersByCustomer(
            @PathVariable String customerId,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size) {
        return ResponseEntity.ok(queryHandler.handle(
                new GetOrdersByCustomerQuery(customerId, page, size)));
    }

    @GetMapping
    public ResponseEntity<Page<OrderSummaryView>> getOrders(
            @RequestParam(required = false) String status,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size) {
        return ResponseEntity.ok(queryHandler.handle(
                new GetOrderSummariesQuery(status, page, size)));
    }
}
```

## Async Projection with Kafka

For eventual consistency between write and read models across services.

### Publishing Events to Kafka

```java
@Component
public class OrderEventKafkaPublisher {

    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public OrderEventKafkaPublisher(KafkaTemplate<String, OrderEvent> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void on(OrderPlacedEvent event) {
        kafkaTemplate.send("order-events", event.orderId(), event);
    }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void on(OrderConfirmedEvent event) {
        kafkaTemplate.send("order-events", event.orderId(), event);
    }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void on(OrderShippedEvent event) {
        kafkaTemplate.send("order-events", event.orderId(), event);
    }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void on(OrderCancelledEvent event) {
        kafkaTemplate.send("order-events", event.orderId(), event);
    }
}
```

### Consuming Events for Read Model

```java
@Component
public class OrderEventKafkaConsumer {

    private final OrderSummaryViewRepository summaryRepository;
    private final OrderDetailViewRepository detailRepository;

    public OrderEventKafkaConsumer(OrderSummaryViewRepository summaryRepository,
                                    OrderDetailViewRepository detailRepository) {
        this.summaryRepository = summaryRepository;
        this.detailRepository = detailRepository;
    }

    @KafkaListener(topics = "order-events", groupId = "order-projection")
    public void consume(OrderEvent event) {
        switch (event) {
            case OrderPlacedEvent placed -> handlePlaced(placed);
            case OrderConfirmedEvent confirmed -> handleConfirmed(confirmed);
            case OrderShippedEvent shipped -> handleShipped(shipped);
            case OrderCancelledEvent cancelled -> handleCancelled(cancelled);
        }
    }

    private void handlePlaced(OrderPlacedEvent event) {
        // Build and save summary + detail views
    }

    private void handleConfirmed(OrderConfirmedEvent event) {
        updateStatus(event.orderId(), "CONFIRMED", event.occurredAt());
    }

    private void handleShipped(OrderShippedEvent event) {
        updateStatus(event.orderId(), "SHIPPED", event.occurredAt());
    }

    private void handleCancelled(OrderCancelledEvent event) {
        updateStatus(event.orderId(), "CANCELLED", event.occurredAt());
    }

    private void updateStatus(String orderId, String status, LocalDateTime updatedAt) {
        summaryRepository.findById(orderId).ifPresent(summary -> {
            summary.setStatus(status);
            summary.setUpdatedAt(updatedAt);
            summaryRepository.save(summary);
        });
    }
}
```

## Separate Databases for Read/Write

### Configuration

```yaml
spring:
  datasource:
    write:
      url: jdbc:postgresql://localhost:5432/orders_write
      username: write_user
      password: write_pass
    read:
      url: jdbc:postgresql://localhost:5433/orders_read
      username: read_user
      password: read_pass
```

### DataSource Configuration

```java
@Configuration
public class DataSourceConfig {

    @Bean
    @Primary
    @ConfigurationProperties("spring.datasource.write")
    public DataSource writeDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    @ConfigurationProperties("spring.datasource.read")
    public DataSource readDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    @Primary
    public LocalContainerEntityManagerFactoryBean writeEntityManagerFactory(
            @Qualifier("writeDataSource") DataSource dataSource,
            EntityManagerFactoryBuilder builder) {
        return builder.dataSource(dataSource)
                .packages("com.example.cqrs.command.model")
                .persistenceUnit("write")
                .build();
    }

    @Bean
    public LocalContainerEntityManagerFactoryBean readEntityManagerFactory(
            @Qualifier("readDataSource") DataSource dataSource,
            EntityManagerFactoryBuilder builder) {
        return builder.dataSource(dataSource)
                .packages("com.example.cqrs.query.model")
                .persistenceUnit("read")
                .build();
    }

    @Bean
    @Primary
    public PlatformTransactionManager writeTransactionManager(
            @Qualifier("writeEntityManagerFactory")
            LocalContainerEntityManagerFactoryBean emf) {
        return new JpaTransactionManager(emf.getObject());
    }

    @Bean
    public PlatformTransactionManager readTransactionManager(
            @Qualifier("readEntityManagerFactory")
            LocalContainerEntityManagerFactoryBean emf) {
        return new JpaTransactionManager(emf.getObject());
    }
}
```

## Testing

```java
@SpringBootTest
class OrderCQRSTest {

    @Autowired
    private OrderCommandHandler commandHandler;

    @Autowired
    private OrderQueryHandler queryHandler;

    @Autowired
    private OrderSummaryViewRepository summaryRepository;

    @Test
    void shouldPlaceOrderAndQueryIt() {
        // Command: Place order
        String orderId = commandHandler.handle(new PlaceOrderCommand(
                "customer-1",
                List.of(new OrderItemRequest("prod-1", "Widget", 2, new BigDecimal("29.99")))
        ));

        // Query: Fetch order detail
        OrderDetailView detail = queryHandler.handle(new GetOrderByIdQuery(orderId));

        assertThat(detail.getOrderId()).isEqualTo(orderId);
        assertThat(detail.getStatus()).isEqualTo("PENDING");
        assertThat(detail.getTotalAmount()).isEqualByComparingTo(new BigDecimal("59.98"));
        assertThat(detail.getItems()).hasSize(1);
    }

    @Test
    void shouldReflectStatusChangesInReadModel() {
        String orderId = commandHandler.handle(new PlaceOrderCommand(
                "customer-1",
                List.of(new OrderItemRequest("prod-1", "Widget", 1, new BigDecimal("10.00")))
        ));

        commandHandler.handle(new ConfirmOrderCommand(orderId));

        OrderSummaryView summary = summaryRepository.findById(orderId).orElseThrow();
        assertThat(summary.getStatus()).isEqualTo("CONFIRMED");
    }

    @Test
    void shouldPreventInvalidStateTransitions() {
        String orderId = commandHandler.handle(new PlaceOrderCommand(
                "customer-1",
                List.of(new OrderItemRequest("prod-1", "Widget", 1, new BigDecimal("10.00")))
        ));

        // Cannot ship a PENDING order
        assertThatThrownBy(() -> commandHandler.handle(new ShipOrderCommand(orderId)))
                .isInstanceOf(IllegalStateException.class);
    }
}
```

## Best Practices

1. **Keep commands task-based** - Model commands around user intents, not CRUD operations
2. **Use eventual consistency** - Accept that the read model may lag behind the write model
3. **Idempotent projections** - Ensure replaying events produces the same read model
4. **Separate packages** - Keep command and query code in distinct packages
5. **Thin controllers** - Controllers should only map HTTP to commands/queries
6. **Validate in commands** - Enforce business rules before persisting state changes
7. **Use `@TransactionalEventListener`** - Ensure events publish only after successful commit

## Common Pitfalls

1. Applying CQRS everywhere - it adds complexity; use only where read/write patterns diverge significantly
2. Sharing entities between command and query sides, defeating the purpose of separation
3. Synchronous projections causing write latency - use async for performance-critical paths
4. Not handling projection failures - if the read model update fails, the data becomes stale
5. Treating CQRS and Event Sourcing as the same thing - CQRS can work without Event Sourcing

## Resources

- [Martin Fowler - CQRS](https://martinfowler.com/bliki/CQRS.html)
- [Microsoft - CQRS Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs)
- [Spring Events Documentation](https://docs.spring.io/spring-framework/reference/core/beans/context-introduction.html#context-functionality-events)
