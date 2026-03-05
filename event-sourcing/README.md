# Event Sourcing

## Overview

Event Sourcing is a pattern where state changes are stored as a sequence of immutable events rather than overwriting the current state. Instead of persisting just the latest snapshot of an entity, you persist every event that led to the current state. The current state is derived by replaying all events from the beginning. This gives you a complete audit trail, the ability to reconstruct past states, and natural integration with CQRS and event-driven architectures.

## Key Concepts

### Why Event Sourcing?

- **Complete audit trail** - Every change is recorded as an immutable event
- **Temporal queries** - Reconstruct the state at any point in time
- **Debugging** - Replay events to understand exactly what happened
- **Event replay** - Rebuild read models or create new projections by replaying events
- **Decoupling** - Other services can react to events without coupling to the write model

### Core Components

- **Event** - An immutable fact that something happened (past tense: `OrderPlaced`, `ItemAdded`)
- **Event Store** - A persistent, append-only log of events
- **Aggregate** - A domain entity that applies events to evolve its state
- **Snapshot** - A periodic checkpoint of aggregate state to avoid replaying all events
- **Projection** - A read model built by consuming events

### Architecture

```text
    Command ──► Aggregate ──► Event(s) ──► Event Store
                    ▲                          │
                    │                          │
              Load & Replay                    ▼
              from Events              Event Publisher
                                              │
                              ┌────────────────┼──────────────┐
                              ▼                ▼              ▼
                        Projection A    Projection B    External System
                       (Read Model)    (Read Model)     (via Kafka/MQ)
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
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
    </dependency>
</dependencies>
```

## Event Store

### Event Entity

```java
@Entity
@Table(name = "event_store", indexes = {
    @Index(name = "idx_aggregate", columnList = "aggregateId,aggregateType"),
    @Index(name = "idx_aggregate_version", columnList = "aggregateId,version", unique = true)
})
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class StoredEvent {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String aggregateId;

    @Column(nullable = false)
    private String aggregateType;

    @Column(nullable = false)
    private String eventType;

    @Column(nullable = false, columnDefinition = "TEXT")
    private String payload;

    @Column(nullable = false)
    private int version;

    @Column(nullable = false)
    private LocalDateTime occurredAt;

    private String metadata;
}
```

### Event Store Repository

```java
public interface EventStoreRepository extends JpaRepository<StoredEvent, Long> {

    List<StoredEvent> findByAggregateIdOrderByVersionAsc(String aggregateId);

    List<StoredEvent> findByAggregateIdAndVersionGreaterThanOrderByVersionAsc(
            String aggregateId, int version);

    Optional<StoredEvent> findTopByAggregateIdOrderByVersionDesc(String aggregateId);

    boolean existsByAggregateIdAndVersion(String aggregateId, int version);
}
```

### Event Store Service

```java
@Service
public class EventStore {

    private final EventStoreRepository repository;
    private final ObjectMapper objectMapper;
    private final ApplicationEventPublisher eventPublisher;

    public EventStore(EventStoreRepository repository,
                      ObjectMapper objectMapper,
                      ApplicationEventPublisher eventPublisher) {
        this.repository = repository;
        this.objectMapper = objectMapper;
        this.eventPublisher = eventPublisher;
    }

    @Transactional
    public void saveEvents(String aggregateId, String aggregateType,
                           List<DomainEvent> events, int expectedVersion) {
        // Optimistic concurrency check
        Optional<StoredEvent> lastEvent =
                repository.findTopByAggregateIdOrderByVersionDesc(aggregateId);
        int currentVersion = lastEvent.map(StoredEvent::getVersion).orElse(0);

        if (currentVersion != expectedVersion) {
            throw new OptimisticLockingException(
                    "Expected version " + expectedVersion +
                    " but found " + currentVersion +
                    " for aggregate " + aggregateId);
        }

        int version = expectedVersion;
        for (DomainEvent event : events) {
            version++;
            StoredEvent storedEvent = StoredEvent.builder()
                    .aggregateId(aggregateId)
                    .aggregateType(aggregateType)
                    .eventType(event.getClass().getSimpleName())
                    .payload(serialize(event))
                    .version(version)
                    .occurredAt(event.occurredAt())
                    .build();

            repository.save(storedEvent);
            eventPublisher.publishEvent(event);
        }
    }

    public List<DomainEvent> getEvents(String aggregateId) {
        return repository.findByAggregateIdOrderByVersionAsc(aggregateId)
                .stream()
                .map(this::deserialize)
                .toList();
    }

    public List<DomainEvent> getEventsAfterVersion(String aggregateId, int version) {
        return repository.findByAggregateIdAndVersionGreaterThanOrderByVersionAsc(
                        aggregateId, version)
                .stream()
                .map(this::deserialize)
                .toList();
    }

    private String serialize(DomainEvent event) {
        try {
            return objectMapper.writeValueAsString(event);
        } catch (JsonProcessingException e) {
            throw new RuntimeException("Failed to serialize event", e);
        }
    }

    private DomainEvent deserialize(StoredEvent storedEvent) {
        try {
            Class<?> eventClass = Class.forName(
                    "com.example.eventsourcing.events." + storedEvent.getEventType());
            return (DomainEvent) objectMapper.readValue(
                    storedEvent.getPayload(), eventClass);
        } catch (Exception e) {
            throw new RuntimeException("Failed to deserialize event", e);
        }
    }
}
```

## Domain Events

### Base Event

```java
public interface DomainEvent {
    String aggregateId();
    LocalDateTime occurredAt();
}
```

### Account Events (Banking Example)

```java
public record AccountCreatedEvent(
        String aggregateId,
        String accountHolder,
        String accountType,
        LocalDateTime occurredAt
) implements DomainEvent {
}

public record MoneyDepositedEvent(
        String aggregateId,
        BigDecimal amount,
        String description,
        LocalDateTime occurredAt
) implements DomainEvent {
}

public record MoneyWithdrawnEvent(
        String aggregateId,
        BigDecimal amount,
        String description,
        LocalDateTime occurredAt
) implements DomainEvent {
}

public record AccountClosedEvent(
        String aggregateId,
        String reason,
        LocalDateTime occurredAt
) implements DomainEvent {
}
```

## Aggregate

### Base Aggregate

```java
public abstract class AggregateRoot {

    private String id;
    private int version = 0;
    private final List<DomainEvent> uncommittedEvents = new ArrayList<>();

    protected AggregateRoot() {}

    protected AggregateRoot(String id) {
        this.id = id;
    }

    public String getId() {
        return id;
    }

    protected void setId(String id) {
        this.id = id;
    }

    public int getVersion() {
        return version;
    }

    public List<DomainEvent> getUncommittedEvents() {
        return Collections.unmodifiableList(uncommittedEvents);
    }

    public void clearUncommittedEvents() {
        uncommittedEvents.clear();
    }

    protected void applyChange(DomainEvent event) {
        apply(event);
        uncommittedEvents.add(event);
    }

    public void rehydrate(List<DomainEvent> events) {
        for (DomainEvent event : events) {
            apply(event);
            version++;
        }
    }

    protected abstract void apply(DomainEvent event);
}
```

### Account Aggregate

```java
public class BankAccount extends AggregateRoot {

    private String accountHolder;
    private String accountType;
    private BigDecimal balance;
    private boolean closed;

    public BankAccount() {
        super();
    }

    // --- Command methods (raise events) ---

    public static BankAccount create(String accountId, String accountHolder, String accountType) {
        BankAccount account = new BankAccount();
        account.applyChange(new AccountCreatedEvent(
                accountId, accountHolder, accountType, LocalDateTime.now()));
        return account;
    }

    public void deposit(BigDecimal amount, String description) {
        if (closed) {
            throw new IllegalStateException("Cannot deposit to a closed account");
        }
        if (amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Deposit amount must be positive");
        }
        applyChange(new MoneyDepositedEvent(getId(), amount, description, LocalDateTime.now()));
    }

    public void withdraw(BigDecimal amount, String description) {
        if (closed) {
            throw new IllegalStateException("Cannot withdraw from a closed account");
        }
        if (amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Withdrawal amount must be positive");
        }
        if (balance.compareTo(amount) < 0) {
            throw new IllegalStateException("Insufficient balance. Current: " + balance);
        }
        applyChange(new MoneyWithdrawnEvent(getId(), amount, description, LocalDateTime.now()));
    }

    public void close(String reason) {
        if (closed) {
            throw new IllegalStateException("Account is already closed");
        }
        if (balance.compareTo(BigDecimal.ZERO) != 0) {
            throw new IllegalStateException("Account balance must be zero to close");
        }
        applyChange(new AccountClosedEvent(getId(), reason, LocalDateTime.now()));
    }

    // --- Event handlers (apply state changes) ---

    @Override
    protected void apply(DomainEvent event) {
        switch (event) {
            case AccountCreatedEvent e -> {
                setId(e.aggregateId());
                this.accountHolder = e.accountHolder();
                this.accountType = e.accountType();
                this.balance = BigDecimal.ZERO;
                this.closed = false;
            }
            case MoneyDepositedEvent e -> {
                this.balance = this.balance.add(e.amount());
            }
            case MoneyWithdrawnEvent e -> {
                this.balance = this.balance.subtract(e.amount());
            }
            case AccountClosedEvent e -> {
                this.closed = true;
            }
            default -> throw new IllegalArgumentException(
                    "Unknown event type: " + event.getClass().getSimpleName());
        }
    }

    // Getters
    public String getAccountHolder() { return accountHolder; }
    public String getAccountType() { return accountType; }
    public BigDecimal getBalance() { return balance; }
    public boolean isClosed() { return closed; }
}
```

## Aggregate Repository

```java
@Service
public class BankAccountRepository {

    private final EventStore eventStore;

    public BankAccountRepository(EventStore eventStore) {
        this.eventStore = eventStore;
    }

    public void save(BankAccount account) {
        List<DomainEvent> uncommitted = account.getUncommittedEvents();
        if (uncommitted.isEmpty()) return;

        eventStore.saveEvents(
                account.getId(),
                "BankAccount",
                uncommitted,
                account.getVersion()
        );
        account.clearUncommittedEvents();
    }

    public BankAccount findById(String accountId) {
        List<DomainEvent> events = eventStore.getEvents(accountId);
        if (events.isEmpty()) {
            throw new AccountNotFoundException("Account not found: " + accountId);
        }

        BankAccount account = new BankAccount();
        account.rehydrate(events);
        return account;
    }
}
```

## Snapshots

For aggregates with many events, loading becomes slow. Snapshots store periodic checkpoints.

### Snapshot Entity

```java
@Entity
@Table(name = "aggregate_snapshots")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class AggregateSnapshot {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String aggregateId;

    @Column(nullable = false)
    private String aggregateType;

    @Column(nullable = false, columnDefinition = "TEXT")
    private String payload;

    @Column(nullable = false)
    private int version;

    @Column(nullable = false)
    private LocalDateTime createdAt;
}
```

### Snapshot Repository

```java
public interface SnapshotRepository extends JpaRepository<AggregateSnapshot, Long> {
    Optional<AggregateSnapshot> findTopByAggregateIdOrderByVersionDesc(String aggregateId);
}
```

### Aggregate Repository with Snapshot Support

```java
@Service
public class SnapshottingBankAccountRepository {

    private static final int SNAPSHOT_INTERVAL = 50;

    private final EventStore eventStore;
    private final SnapshotRepository snapshotRepository;
    private final ObjectMapper objectMapper;

    public SnapshottingBankAccountRepository(EventStore eventStore,
                                              SnapshotRepository snapshotRepository,
                                              ObjectMapper objectMapper) {
        this.eventStore = eventStore;
        this.snapshotRepository = snapshotRepository;
        this.objectMapper = objectMapper;
    }

    public void save(BankAccount account) {
        List<DomainEvent> uncommitted = account.getUncommittedEvents();
        if (uncommitted.isEmpty()) return;

        eventStore.saveEvents(
                account.getId(), "BankAccount",
                uncommitted, account.getVersion());
        account.clearUncommittedEvents();

        // Take snapshot every N events
        int newVersion = account.getVersion() + uncommitted.size();
        if (newVersion % SNAPSHOT_INTERVAL == 0) {
            takeSnapshot(account, newVersion);
        }
    }

    public BankAccount findById(String accountId) {
        Optional<AggregateSnapshot> snapshot =
                snapshotRepository.findTopByAggregateIdOrderByVersionDesc(accountId);

        BankAccount account;
        if (snapshot.isPresent()) {
            // Restore from snapshot
            account = deserializeSnapshot(snapshot.get());
            // Apply events after snapshot
            List<DomainEvent> newEvents =
                    eventStore.getEventsAfterVersion(accountId, snapshot.get().getVersion());
            account.rehydrate(newEvents);
        } else {
            // Replay all events
            List<DomainEvent> events = eventStore.getEvents(accountId);
            if (events.isEmpty()) {
                throw new AccountNotFoundException("Account not found: " + accountId);
            }
            account = new BankAccount();
            account.rehydrate(events);
        }

        return account;
    }

    private void takeSnapshot(BankAccount account, int version) {
        try {
            AggregateSnapshot snapshot = AggregateSnapshot.builder()
                    .aggregateId(account.getId())
                    .aggregateType("BankAccount")
                    .payload(objectMapper.writeValueAsString(new BankAccountSnapshot(
                            account.getAccountHolder(),
                            account.getAccountType(),
                            account.getBalance(),
                            account.isClosed())))
                    .version(version)
                    .createdAt(LocalDateTime.now())
                    .build();
            snapshotRepository.save(snapshot);
        } catch (JsonProcessingException e) {
            throw new RuntimeException("Failed to serialize snapshot", e);
        }
    }

    private BankAccount deserializeSnapshot(AggregateSnapshot snapshot) {
        try {
            BankAccountSnapshot state = objectMapper.readValue(
                    snapshot.getPayload(), BankAccountSnapshot.class);
            // Reconstruct aggregate from snapshot state
            BankAccount account = new BankAccount();
            account.rehydrate(List.of(new AccountCreatedEvent(
                    snapshot.getAggregateId(),
                    state.accountHolder(),
                    state.accountType(),
                    LocalDateTime.now()
            )));
            // Apply balance by simulating deposit if needed
            if (state.balance().compareTo(BigDecimal.ZERO) > 0) {
                // Direct state restoration (package-private or reflection)
            }
            return account;
        } catch (JsonProcessingException e) {
            throw new RuntimeException("Failed to deserialize snapshot", e);
        }
    }
}

record BankAccountSnapshot(
        String accountHolder,
        String accountType,
        BigDecimal balance,
        boolean closed
) {
}
```

## Command Handler

```java
@Service
@Transactional
public class BankAccountCommandHandler {

    private final BankAccountRepository accountRepository;

    public BankAccountCommandHandler(BankAccountRepository accountRepository) {
        this.accountRepository = accountRepository;
    }

    public String handle(CreateAccountCommand command) {
        BankAccount account = BankAccount.create(
                UUID.randomUUID().toString(),
                command.accountHolder(),
                command.accountType()
        );
        accountRepository.save(account);
        return account.getId();
    }

    public void handle(DepositMoneyCommand command) {
        BankAccount account = accountRepository.findById(command.accountId());
        account.deposit(command.amount(), command.description());
        accountRepository.save(account);
    }

    public void handle(WithdrawMoneyCommand command) {
        BankAccount account = accountRepository.findById(command.accountId());
        account.withdraw(command.amount(), command.description());
        accountRepository.save(account);
    }

    public void handle(CloseAccountCommand command) {
        BankAccount account = accountRepository.findById(command.accountId());
        account.close(command.reason());
        accountRepository.save(account);
    }
}
```

### Commands

```java
public record CreateAccountCommand(String accountHolder, String accountType) {}
public record DepositMoneyCommand(String accountId, BigDecimal amount, String description) {}
public record WithdrawMoneyCommand(String accountId, BigDecimal amount, String description) {}
public record CloseAccountCommand(String accountId, String reason) {}
```

## Projections

### Account Balance Projection

```java
@Entity
@Table(name = "account_balance_view")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class AccountBalanceView {
    @Id
    private String accountId;
    private String accountHolder;
    private String accountType;
    private BigDecimal balance;
    private boolean closed;
    private LocalDateTime lastUpdated;
}

@Component
public class AccountBalanceProjection {

    private final AccountBalanceViewRepository repository;

    public AccountBalanceProjection(AccountBalanceViewRepository repository) {
        this.repository = repository;
    }

    @EventListener
    public void on(AccountCreatedEvent event) {
        repository.save(AccountBalanceView.builder()
                .accountId(event.aggregateId())
                .accountHolder(event.accountHolder())
                .accountType(event.accountType())
                .balance(BigDecimal.ZERO)
                .closed(false)
                .lastUpdated(event.occurredAt())
                .build());
    }

    @EventListener
    public void on(MoneyDepositedEvent event) {
        repository.findById(event.aggregateId()).ifPresent(view -> {
            view.setBalance(view.getBalance().add(event.amount()));
            view.setLastUpdated(event.occurredAt());
            repository.save(view);
        });
    }

    @EventListener
    public void on(MoneyWithdrawnEvent event) {
        repository.findById(event.aggregateId()).ifPresent(view -> {
            view.setBalance(view.getBalance().subtract(event.amount()));
            view.setLastUpdated(event.occurredAt());
            repository.save(view);
        });
    }

    @EventListener
    public void on(AccountClosedEvent event) {
        repository.findById(event.aggregateId()).ifPresent(view -> {
            view.setClosed(true);
            view.setLastUpdated(event.occurredAt());
            repository.save(view);
        });
    }
}
```

### Transaction History Projection

```java
@Entity
@Table(name = "transaction_history")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class TransactionRecord {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String accountId;
    private String type; // DEPOSIT, WITHDRAWAL
    private BigDecimal amount;
    private String description;
    private LocalDateTime occurredAt;
}

@Component
public class TransactionHistoryProjection {

    private final TransactionRecordRepository repository;

    public TransactionHistoryProjection(TransactionRecordRepository repository) {
        this.repository = repository;
    }

    @EventListener
    public void on(MoneyDepositedEvent event) {
        repository.save(TransactionRecord.builder()
                .accountId(event.aggregateId())
                .type("DEPOSIT")
                .amount(event.amount())
                .description(event.description())
                .occurredAt(event.occurredAt())
                .build());
    }

    @EventListener
    public void on(MoneyWithdrawnEvent event) {
        repository.save(TransactionRecord.builder()
                .accountId(event.aggregateId())
                .type("WITHDRAWAL")
                .amount(event.amount())
                .description(event.description())
                .occurredAt(event.occurredAt())
                .build());
    }
}
```

## REST Controller

```java
@RestController
@RequestMapping("/api/accounts")
public class BankAccountController {

    private final BankAccountCommandHandler commandHandler;
    private final AccountBalanceViewRepository balanceRepository;
    private final TransactionRecordRepository transactionRepository;

    public BankAccountController(BankAccountCommandHandler commandHandler,
                                  AccountBalanceViewRepository balanceRepository,
                                  TransactionRecordRepository transactionRepository) {
        this.commandHandler = commandHandler;
        this.balanceRepository = balanceRepository;
        this.transactionRepository = transactionRepository;
    }

    @PostMapping
    public ResponseEntity<Map<String, String>> createAccount(
            @RequestBody CreateAccountCommand command) {
        String id = commandHandler.handle(command);
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(Map.of("accountId", id));
    }

    @PostMapping("/{id}/deposit")
    public ResponseEntity<Void> deposit(@PathVariable String id,
                                         @RequestBody DepositMoneyCommand command) {
        commandHandler.handle(new DepositMoneyCommand(id, command.amount(), command.description()));
        return ResponseEntity.ok().build();
    }

    @PostMapping("/{id}/withdraw")
    public ResponseEntity<Void> withdraw(@PathVariable String id,
                                          @RequestBody WithdrawMoneyCommand command) {
        commandHandler.handle(new WithdrawMoneyCommand(id, command.amount(), command.description()));
        return ResponseEntity.ok().build();
    }

    @PostMapping("/{id}/close")
    public ResponseEntity<Void> close(@PathVariable String id,
                                       @RequestBody CloseAccountCommand command) {
        commandHandler.handle(new CloseAccountCommand(id, command.reason()));
        return ResponseEntity.ok().build();
    }

    // Query endpoints (read from projections)
    @GetMapping("/{id}")
    public ResponseEntity<AccountBalanceView> getAccount(@PathVariable String id) {
        return balanceRepository.findById(id)
                .map(ResponseEntity::ok)
                .orElse(ResponseEntity.notFound().build());
    }

    @GetMapping("/{id}/transactions")
    public ResponseEntity<List<TransactionRecord>> getTransactions(@PathVariable String id) {
        return ResponseEntity.ok(
                transactionRepository.findByAccountIdOrderByOccurredAtDesc(id));
    }
}
```

## Event Upcasting

When event schemas evolve over time, you need to transform old events to the new format.

```java
@Component
public class EventUpcaster {

    private final Map<String, Function<JsonNode, DomainEvent>> upcasters = new HashMap<>();
    private final ObjectMapper objectMapper;

    public EventUpcaster(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
        registerUpcasters();
    }

    private void registerUpcasters() {
        // V1 MoneyDeposited had no description field
        upcasters.put("MoneyDepositedEvent_v1", node -> new MoneyDepositedEvent(
                node.get("aggregateId").asText(),
                new BigDecimal(node.get("amount").asText()),
                "No description (migrated from v1)",
                LocalDateTime.parse(node.get("occurredAt").asText())
        ));
    }

    public DomainEvent upcast(String eventType, String payload) {
        if (upcasters.containsKey(eventType)) {
            try {
                JsonNode node = objectMapper.readTree(payload);
                return upcasters.get(eventType).apply(node);
            } catch (JsonProcessingException e) {
                throw new RuntimeException("Failed to upcast event", e);
            }
        }
        return null; // No upcasting needed
    }
}
```

## Testing

```java
@SpringBootTest
class EventSourcingTest {

    @Autowired
    private BankAccountCommandHandler commandHandler;

    @Autowired
    private EventStore eventStore;

    @Autowired
    private AccountBalanceViewRepository balanceRepository;

    @Test
    void shouldStoreAndReplayEvents() {
        String accountId = commandHandler.handle(
                new CreateAccountCommand("John Doe", "SAVINGS"));

        commandHandler.handle(new DepositMoneyCommand(accountId,
                new BigDecimal("1000.00"), "Initial deposit"));
        commandHandler.handle(new WithdrawMoneyCommand(accountId,
                new BigDecimal("200.00"), "ATM withdrawal"));

        // Verify events are stored
        List<DomainEvent> events = eventStore.getEvents(accountId);
        assertThat(events).hasSize(3);
        assertThat(events.get(0)).isInstanceOf(AccountCreatedEvent.class);
        assertThat(events.get(1)).isInstanceOf(MoneyDepositedEvent.class);
        assertThat(events.get(2)).isInstanceOf(MoneyWithdrawnEvent.class);

        // Verify aggregate state from replay
        BankAccount account = new BankAccount();
        account.rehydrate(events);
        assertThat(account.getBalance()).isEqualByComparingTo(new BigDecimal("800.00"));
    }

    @Test
    void shouldBuildProjectionFromEvents() {
        String accountId = commandHandler.handle(
                new CreateAccountCommand("Jane Doe", "CHECKING"));

        commandHandler.handle(new DepositMoneyCommand(accountId,
                new BigDecimal("500.00"), "Salary"));

        AccountBalanceView view = balanceRepository.findById(accountId).orElseThrow();
        assertThat(view.getBalance()).isEqualByComparingTo(new BigDecimal("500.00"));
        assertThat(view.getAccountHolder()).isEqualTo("Jane Doe");
    }

    @Test
    void shouldEnforceBusinessRules() {
        String accountId = commandHandler.handle(
                new CreateAccountCommand("Bob", "SAVINGS"));

        commandHandler.handle(new DepositMoneyCommand(accountId,
                new BigDecimal("100.00"), "Deposit"));

        assertThatThrownBy(() -> commandHandler.handle(
                new WithdrawMoneyCommand(accountId, new BigDecimal("200.00"), "Over-withdraw")))
                .isInstanceOf(IllegalStateException.class)
                .hasMessageContaining("Insufficient balance");
    }

    @Test
    void shouldDetectConcurrencyConflicts() {
        String accountId = commandHandler.handle(
                new CreateAccountCommand("Alice", "SAVINGS"));

        commandHandler.handle(new DepositMoneyCommand(accountId,
                new BigDecimal("100.00"), "Deposit"));

        // Simulate concurrent modification by saving events with wrong version
        assertThatThrownBy(() -> eventStore.saveEvents(
                accountId, "BankAccount",
                List.of(new MoneyDepositedEvent(accountId, BigDecimal.TEN, "test",
                        LocalDateTime.now())),
                0 // Wrong expected version
        )).isInstanceOf(OptimisticLockingException.class);
    }
}
```

## Best Practices

1. **Events are immutable facts** - Never modify or delete events; they represent history
2. **Use past tense for event names** - `OrderPlaced`, not `PlaceOrder`
3. **Keep events small and focused** - One event per state change
4. **Version your events** - Plan for schema evolution with upcasting
5. **Implement snapshots** - For aggregates with many events to avoid slow replays
6. **Optimistic concurrency** - Use version checks to prevent conflicting writes
7. **Idempotent event handlers** - Projections should handle duplicate events gracefully
8. **Separate event storage** - The event store should be append-only and not shared with read models

## Common Pitfalls

1. Modifying or deleting events - this breaks the fundamental guarantee of event sourcing
2. Storing too much data in a single event, making evolution difficult
3. Not implementing snapshots, causing degraded performance as events accumulate
4. Tight coupling between events and projection schemas
5. Forgetting concurrency control, leading to corrupted aggregate state
6. Using event sourcing for simple CRUD entities where it adds unnecessary complexity

## Resources

- [Martin Fowler - Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html)
- [Microsoft - Event Sourcing Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing)
- [Greg Young - CQRS and Event Sourcing](https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf)
