# Resilience4j

## Overview

Resilience4j is a lightweight, easy-to-use fault tolerance library designed for Java. It provides decorators for Circuit Breaker, Rate Limiter, Retry, Bulkhead, and Time Limiter patterns. In a microservices architecture, services depend on each other and any downstream failure can cascade. Resilience4j helps you build applications that gracefully handle failures, degrade functionality, and recover automatically.

## Key Concepts

### Resilience Patterns

- **Circuit Breaker** - Stops calling a failing service after a threshold, allowing it to recover
- **Retry** - Automatically retries failed operations with configurable backoff
- **Rate Limiter** - Limits the number of calls in a time period
- **Bulkhead** - Limits concurrent calls to prevent resource exhaustion
- **Time Limiter** - Sets a timeout for operations to prevent indefinite waiting

### Circuit Breaker States

```text
         ┌───────────┐
         │           │
         │   CLOSED  │◄──── Normal operation
         │           │
         └─────┬─────┘
               │ Failure threshold exceeded
               ▼
         ┌───────────┐
         │           │
         │   OPEN    │◄──── All calls rejected with fallback
         │           │
         └─────┬─────┘
               │ Wait duration elapsed
               ▼
         ┌───────────┐
         │           │
         │ HALF_OPEN │◄──── Limited calls allowed to test recovery
         │           │
         └─────┬─────┘
               │ Success threshold met → CLOSED
               │ Failure threshold met → OPEN
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
        <artifactId>spring-boot-starter-aop</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-spring-boot3</artifactId>
    </dependency>
</dependencies>
```

## Configuration

### application.yml

```yaml
resilience4j:
  circuitbreaker:
    configs:
      default:
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 10
        minimumNumberOfCalls: 5
        failureRateThreshold: 50
        waitDurationInOpenState: 10s
        permittedNumberOfCallsInHalfOpenState: 3
        automaticTransitionFromOpenToHalfOpenEnabled: true
        recordExceptions:
          - java.io.IOException
          - java.net.ConnectException
          - java.util.concurrent.TimeoutException
        ignoreExceptions:
          - com.example.exception.BusinessException
    instances:
      paymentService:
        baseConfig: default
        slidingWindowSize: 20
        failureRateThreshold: 60
      inventoryService:
        baseConfig: default
        waitDurationInOpenState: 30s

  retry:
    configs:
      default:
        maxAttempts: 3
        waitDuration: 1s
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2
        retryExceptions:
          - java.io.IOException
          - java.net.ConnectException
        ignoreExceptions:
          - com.example.exception.BusinessException
    instances:
      paymentService:
        baseConfig: default
        maxAttempts: 5
      emailService:
        baseConfig: default
        maxAttempts: 3
        waitDuration: 2s

  bulkhead:
    configs:
      default:
        maxConcurrentCalls: 25
        maxWaitDuration: 0ms
    instances:
      paymentService:
        maxConcurrentCalls: 10
        maxWaitDuration: 500ms
      reportService:
        maxConcurrentCalls: 5

  timelimiter:
    configs:
      default:
        timeoutDuration: 5s
        cancelRunningFuture: true
    instances:
      paymentService:
        timeoutDuration: 3s
      reportService:
        timeoutDuration: 30s

  ratelimiter:
    configs:
      default:
        limitForPeriod: 50
        limitRefreshPeriod: 1s
        timeoutDuration: 0ms
    instances:
      apiEndpoint:
        limitForPeriod: 100
        limitRefreshPeriod: 1s
      externalApi:
        limitForPeriod: 10
        limitRefreshPeriod: 1s

# Expose resilience4j metrics
management:
  endpoints:
    web:
      exposure:
        include: health,circuitbreakers,retries,ratelimiters,bulkheads
  health:
    circuitbreakers:
      enabled: true
```

## Circuit Breaker

### Annotation-Based

```java
@Service
public class PaymentService {

    private final PaymentGatewayClient paymentGateway;

    public PaymentService(PaymentGatewayClient paymentGateway) {
        this.paymentGateway = paymentGateway;
    }

    @CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
    public PaymentResponse processPayment(PaymentRequest request) {
        return paymentGateway.charge(request);
    }

    private PaymentResponse paymentFallback(PaymentRequest request, Throwable throwable) {
        // Log the failure
        log.warn("Payment service circuit breaker triggered: {}", throwable.getMessage());

        // Return a queued/pending response
        return PaymentResponse.builder()
                .status(PaymentStatus.QUEUED)
                .message("Payment queued for retry. Reference: " + UUID.randomUUID())
                .build();
    }
}
```

### Programmatic

```java
@Service
public class PaymentServiceProgrammatic {

    private final CircuitBreakerRegistry circuitBreakerRegistry;
    private final PaymentGatewayClient paymentGateway;

    public PaymentServiceProgrammatic(CircuitBreakerRegistry circuitBreakerRegistry,
                                       PaymentGatewayClient paymentGateway) {
        this.circuitBreakerRegistry = circuitBreakerRegistry;
        this.paymentGateway = paymentGateway;
    }

    public PaymentResponse processPayment(PaymentRequest request) {
        CircuitBreaker circuitBreaker = circuitBreakerRegistry.circuitBreaker("paymentService");

        return circuitBreaker.executeSupplier(() -> paymentGateway.charge(request));
    }

    public PaymentResponse processPaymentWithFallback(PaymentRequest request) {
        CircuitBreaker circuitBreaker = circuitBreakerRegistry.circuitBreaker("paymentService");

        return Try.ofSupplier(
                CircuitBreaker.decorateSupplier(circuitBreaker,
                        () -> paymentGateway.charge(request)))
                .recover(throwable -> PaymentResponse.builder()
                        .status(PaymentStatus.QUEUED)
                        .message("Queued for retry")
                        .build())
                .get();
    }
}
```

### Circuit Breaker Event Listener

```java
@Component
public class CircuitBreakerEventListener {

    private static final Logger log = LoggerFactory.getLogger(CircuitBreakerEventListener.class);

    public CircuitBreakerEventListener(CircuitBreakerRegistry registry) {
        registry.getAllCircuitBreakers().forEach(cb -> {
            cb.getEventPublisher()
                    .onStateTransition(event ->
                            log.warn("CircuitBreaker '{}' state transition: {} -> {}",
                                    event.getCircuitBreakerName(),
                                    event.getStateTransition().getFromState(),
                                    event.getStateTransition().getToState()))
                    .onFailureRateExceeded(event ->
                            log.warn("CircuitBreaker '{}' failure rate exceeded: {}%",
                                    event.getCircuitBreakerName(),
                                    event.getFailureRate()))
                    .onCallNotPermitted(event ->
                            log.warn("CircuitBreaker '{}' call not permitted",
                                    event.getCircuitBreakerName()));
        });
    }
}
```

## Retry

```java
@Service
public class NotificationService {

    private final EmailClient emailClient;

    public NotificationService(EmailClient emailClient) {
        this.emailClient = emailClient;
    }

    @Retry(name = "emailService", fallbackMethod = "emailFallback")
    public void sendEmail(String to, String subject, String body) {
        emailClient.send(to, subject, body);
    }

    private void emailFallback(String to, String subject, String body, Throwable throwable) {
        log.error("Failed to send email to {} after retries: {}", to, throwable.getMessage());
        // Store for later retry via scheduled job
        failedEmailRepository.save(new FailedEmail(to, subject, body));
    }
}
```

### Retry with Custom Configuration

```java
@Configuration
public class RetryConfig {

    @Bean
    public RetryRegistry retryRegistry() {
        io.github.resilience4j.retry.RetryConfig config =
                io.github.resilience4j.retry.RetryConfig.custom()
                        .maxAttempts(4)
                        .waitDuration(Duration.ofMillis(500))
                        .retryOnException(e -> e instanceof IOException)
                        .retryOnResult(response ->
                                response instanceof HttpResponse r && r.statusCode() >= 500)
                        .intervalFunction(IntervalFunction.ofExponentialBackoff(
                                Duration.ofMillis(500), 2.0))
                        .build();

        return RetryRegistry.of(config);
    }
}
```

## Bulkhead

### Semaphore Bulkhead (Default)

```java
@Service
public class ReportService {

    @Bulkhead(name = "reportService", fallbackMethod = "reportFallback")
    public Report generateReport(ReportRequest request) {
        // CPU/memory intensive operation
        return reportGenerator.generate(request);
    }

    private Report reportFallback(ReportRequest request, Throwable throwable) {
        if (throwable instanceof BulkheadFullException) {
            throw new ServiceOverloadedException(
                    "Report generation is at capacity. Please try again later.");
        }
        throw new RuntimeException("Report generation failed", throwable);
    }
}
```

### Thread Pool Bulkhead

```java
@Service
public class AsyncReportService {

    @Bulkhead(name = "reportService", type = Bulkhead.Type.THREADPOOL,
              fallbackMethod = "reportFallback")
    public CompletableFuture<Report> generateReportAsync(ReportRequest request) {
        return CompletableFuture.supplyAsync(() -> reportGenerator.generate(request));
    }

    private CompletableFuture<Report> reportFallback(ReportRequest request, Throwable throwable) {
        return CompletableFuture.completedFuture(
                Report.placeholder("Report generation at capacity"));
    }
}
```

Thread pool bulkhead configuration:

```yaml
resilience4j:
  thread-pool-bulkhead:
    instances:
      reportService:
        maxThreadPoolSize: 10
        coreThreadPoolSize: 5
        queueCapacity: 20
        keepAliveDuration: 20ms
```

## Time Limiter

```java
@Service
public class ExternalApiService {

    @TimeLimiter(name = "externalApi", fallbackMethod = "timeoutFallback")
    @CircuitBreaker(name = "externalApi", fallbackMethod = "circuitBreakerFallback")
    public CompletableFuture<ApiResponse> callExternalApi(String endpoint) {
        return CompletableFuture.supplyAsync(() -> externalApiClient.call(endpoint));
    }

    private CompletableFuture<ApiResponse> timeoutFallback(String endpoint,
                                                            TimeoutException e) {
        log.warn("External API call timed out: {}", endpoint);
        return CompletableFuture.completedFuture(ApiResponse.timeout());
    }

    private CompletableFuture<ApiResponse> circuitBreakerFallback(String endpoint,
                                                                    Throwable throwable) {
        log.warn("External API circuit breaker open: {}", endpoint);
        return CompletableFuture.completedFuture(ApiResponse.unavailable());
    }
}
```

## Combining Multiple Patterns

### Stacking Annotations

The order of execution (outermost to innermost):
`Retry → CircuitBreaker → RateLimiter → TimeLimiter → Bulkhead → Function`

```java
@Service
public class OrderService {

    private final InventoryClient inventoryClient;

    public OrderService(InventoryClient inventoryClient) {
        this.inventoryClient = inventoryClient;
    }

    @CircuitBreaker(name = "inventoryService", fallbackMethod = "inventoryFallback")
    @Retry(name = "inventoryService")
    @Bulkhead(name = "inventoryService")
    @RateLimiter(name = "externalApi")
    public InventoryResponse checkInventory(String productId) {
        return inventoryClient.getAvailability(productId);
    }

    private InventoryResponse inventoryFallback(String productId, Throwable throwable) {
        log.warn("Inventory check failed for {}: {}", productId, throwable.getMessage());
        // Return cached/default inventory status
        return InventoryResponse.builder()
                .productId(productId)
                .available(true) // Optimistic fallback
                .source("CACHE")
                .build();
    }
}
```

### Programmatic Composition

```java
@Service
public class ComposedResilience {

    private final CircuitBreakerRegistry circuitBreakerRegistry;
    private final RetryRegistry retryRegistry;
    private final BulkheadRegistry bulkheadRegistry;

    public <T> T executeWithResilience(String name, Supplier<T> supplier) {
        CircuitBreaker circuitBreaker = circuitBreakerRegistry.circuitBreaker(name);
        Retry retry = retryRegistry.retry(name);
        Bulkhead bulkhead = bulkheadRegistry.bulkhead(name);

        Supplier<T> decorated = Decorators.ofSupplier(supplier)
                .withCircuitBreaker(circuitBreaker)
                .withRetry(retry)
                .withBulkhead(bulkhead)
                .decorate();

        return Try.ofSupplier(decorated)
                .recover(throwable -> null)
                .get();
    }
}
```

## RestClient / WebClient Integration

### Resilient RestClient

```java
@Service
public class ResilientApiClient {

    private final RestClient restClient;

    @CircuitBreaker(name = "paymentService", fallbackMethod = "fallback")
    @Retry(name = "paymentService")
    public PaymentResponse charge(PaymentRequest request) {
        return restClient.post()
                .uri("/api/payments/charge")
                .body(request)
                .retrieve()
                .body(PaymentResponse.class);
    }

    private PaymentResponse fallback(PaymentRequest request, Throwable throwable) {
        return PaymentResponse.queued("Service temporarily unavailable");
    }
}
```

### Resilient WebClient (Reactive)

```java
@Service
public class ResilientReactiveClient {

    private final WebClient webClient;
    private final CircuitBreakerRegistry circuitBreakerRegistry;

    public Mono<PaymentResponse> charge(PaymentRequest request) {
        CircuitBreaker circuitBreaker = circuitBreakerRegistry.circuitBreaker("paymentService");

        return webClient.post()
                .uri("/api/payments/charge")
                .bodyValue(request)
                .retrieve()
                .bodyToMono(PaymentResponse.class)
                .transformDeferred(CircuitBreakerOperator.of(circuitBreaker))
                .onErrorResume(CallNotPermittedException.class,
                        e -> Mono.just(PaymentResponse.queued("Circuit breaker open")))
                .timeout(Duration.ofSeconds(3))
                .onErrorResume(TimeoutException.class,
                        e -> Mono.just(PaymentResponse.queued("Request timed out")));
    }
}
```

## Custom Circuit Breaker with Health Indicator

```java
@Component
public class CircuitBreakerHealthIndicator implements HealthIndicator {

    private final CircuitBreakerRegistry registry;

    public CircuitBreakerHealthIndicator(CircuitBreakerRegistry registry) {
        this.registry = registry;
    }

    @Override
    public Health health() {
        Map<String, Object> details = new LinkedHashMap<>();
        boolean anyOpen = false;

        for (CircuitBreaker cb : registry.getAllCircuitBreakers()) {
            CircuitBreaker.Metrics metrics = cb.getMetrics();
            Map<String, Object> cbDetails = Map.of(
                    "state", cb.getState().name(),
                    "failureRate", metrics.getFailureRate() + "%",
                    "bufferedCalls", metrics.getNumberOfBufferedCalls(),
                    "failedCalls", metrics.getNumberOfFailedCalls(),
                    "successCalls", metrics.getNumberOfSuccessfulCalls()
            );
            details.put(cb.getName(), cbDetails);

            if (cb.getState() == CircuitBreaker.State.OPEN) {
                anyOpen = true;
            }
        }

        return anyOpen
                ? Health.down().withDetails(details).build()
                : Health.up().withDetails(details).build();
    }
}
```

## Testing

```java
@SpringBootTest
class ResilienceTest {

    @Autowired
    private PaymentService paymentService;

    @MockitoBean
    private PaymentGatewayClient paymentGateway;

    @Autowired
    private CircuitBreakerRegistry circuitBreakerRegistry;

    @Test
    void shouldOpenCircuitBreakerAfterFailures() {
        // Simulate failures
        when(paymentGateway.charge(any()))
                .thenThrow(new RuntimeException("Connection refused"));

        CircuitBreaker cb = circuitBreakerRegistry.circuitBreaker("paymentService");
        assertThat(cb.getState()).isEqualTo(CircuitBreaker.State.CLOSED);

        // Make enough failing calls to trip the breaker
        for (int i = 0; i < 10; i++) {
            paymentService.processPayment(new PaymentRequest());
        }

        assertThat(cb.getState()).isEqualTo(CircuitBreaker.State.OPEN);
    }

    @Test
    void shouldReturnFallbackWhenCircuitIsOpen() {
        CircuitBreaker cb = circuitBreakerRegistry.circuitBreaker("paymentService");
        cb.transitionToOpenState();

        PaymentResponse response = paymentService.processPayment(new PaymentRequest());

        assertThat(response.getStatus()).isEqualTo(PaymentStatus.QUEUED);
    }

    @Test
    void shouldRetryOnTransientFailure() {
        when(paymentGateway.charge(any()))
                .thenThrow(new IOException("Timeout"))
                .thenThrow(new IOException("Timeout"))
                .thenReturn(PaymentResponse.success());

        PaymentResponse response = paymentService.processPayment(new PaymentRequest());

        assertThat(response.getStatus()).isEqualTo(PaymentStatus.SUCCESS);
        verify(paymentGateway, times(3)).charge(any());
    }
}
```

## Best Practices

1. **Configure per downstream service** - Each dependency may need different thresholds
2. **Always provide fallbacks** - Gracefully degrade instead of propagating errors
3. **Monitor circuit breaker states** - Expose metrics via Actuator and alert on OPEN state
4. **Use exponential backoff for retries** - Avoid hammering a struggling service
5. **Combine patterns thoughtfully** - Retry inside circuit breaker, not outside
6. **Test failure scenarios** - Simulate downstream failures in integration tests
7. **Don't retry non-idempotent operations** - Retrying a payment charge can cause double charges
8. **Set realistic timeouts** - Time limiter should match your SLA expectations

## Common Pitfalls

1. Retrying non-idempotent operations (duplicate side effects)
2. Setting the circuit breaker window too small, causing premature tripping
3. Not configuring `ignoreExceptions` for business exceptions (e.g., validation errors)
4. Stacking patterns in the wrong order, leading to unexpected behavior
5. Using bulkhead without understanding thread pool vs. semaphore tradeoffs
6. Forgetting that `@CircuitBreaker` fallback method must have the same return type plus a `Throwable` parameter

## Resources

- [Resilience4j Documentation](https://resilience4j.readme.io/docs)
- [Resilience4j Spring Boot Integration](https://resilience4j.readme.io/docs/getting-started-3)
- [Microsoft - Circuit Breaker Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker)
- [Release It! by Michael Nygard](https://pragprog.com/titles/mnee2/release-it-second-edition/)
