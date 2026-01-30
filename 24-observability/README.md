# Prometheus, Loki, and Tempo

## Overview

Observability is the ability to understand the internal state of a system by examining its outputs. This guide covers implementing comprehensive observability in Spring Boot applications using Prometheus (metrics), Loki (logs), and Tempo (traces).

## Table of Contents

- [What is Observability?](#what-is-observability)
- [The Three Pillars of Observability](#the-three-pillars-of-observability)
- [Prometheus - Metrics](#prometheus---metrics)
- [Loki - Logs](#loki---logs)
- [Tempo - Traces](#tempo---traces)
- [Grafana - Visualization](#grafana---visualization)
- [Spring Boot Actuator](#spring-boot-actuator)
- [Custom Metrics](#custom-metrics)
- [Distributed Tracing](#distributed-tracing)
- [POC Implementation](#poc-implementation)

## What is Observability?

Observability enables you to:

- Monitor application health and performance
- Debug production issues quickly
- Understand system behavior
- Track SLA compliance
- Detect anomalies and patterns
- Optimize resource usage

## The Three Pillars of Observability

### 1. Metrics (Prometheus)
Numerical measurements over time (CPU, memory, request rate)

### 2. Logs (Loki)
Detailed event records with context

### 3. Traces (Tempo)
Request flow through distributed systems

## Prometheus - Metrics

### Dependencies

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-tracing-bridge-brave</artifactId>
    </dependency>
</dependencies>
```

### Application Configuration

```yaml
spring:
  application:
    name: observability-demo

management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics
      base-path: /actuator
  endpoint:
    health:
      show-details: always
    prometheus:
      enabled: true
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      application: ${spring.application.name}
      environment: ${ENVIRONMENT:dev}
    distribution:
      percentiles-histogram:
        http.server.requests: true
  tracing:
    sampling:
      probability: 1.0
```

### Custom Metrics with Micrometer

```java
@Service
@RequiredArgsConstructor
public class OrderService {
    
    private final OrderRepository orderRepository;
    private final MeterRegistry meterRegistry;
    
    private Counter orderCreatedCounter;
    private Timer orderProcessingTimer;
    private Gauge activeOrdersGauge;
    
    @PostConstruct
    public void initMetrics() {
        // Counter - tracks total number of orders
        orderCreatedCounter = Counter.builder("orders.created")
            .description("Total number of orders created")
            .tags("service", "order")
            .register(meterRegistry);
        
        // Timer - tracks order processing time
        orderProcessingTimer = Timer.builder("orders.processing.time")
            .description("Time taken to process orders")
            .tags("service", "order")
            .register(meterRegistry);
        
        // Gauge - tracks current active orders
        activeOrdersGauge = Gauge.builder("orders.active", 
            orderRepository, OrderRepository::countActiveOrders)
            .description("Number of active orders")
            .tags("service", "order")
            .register(meterRegistry);
    }
    
    public Order createOrder(OrderRequest request) {
        return orderProcessingTimer.record(() -> {
            Order order = processOrder(request);
            orderCreatedCounter.increment();
            
            // Add custom tags
            meterRegistry.counter("orders.created.by.type", 
                "type", request.getType()).increment();
            
            return order;
        });
    }
    
    public void recordOrderValue(BigDecimal amount) {
        DistributionSummary.builder("orders.value")
            .description("Distribution of order values")
            .baseUnit("currency")
            .tags("service", "order")
            .register(meterRegistry)
            .record(amount.doubleValue());
    }
}
```

### Metric Types

```java
@Component
public class CustomMetrics {
    
    private final MeterRegistry registry;
    
    public CustomMetrics(MeterRegistry registry) {
        this.registry = registry;
    }
    
    // Counter - monotonically increasing value
    public void incrementRequestCounter(String endpoint) {
        Counter.builder("api.requests")
            .tag("endpoint", endpoint)
            .description("Total API requests")
            .register(registry)
            .increment();
    }
    
    // Gauge - current value that can go up or down
    public void registerQueueSizeGauge(Queue<?> queue) {
        Gauge.builder("queue.size", queue, Queue::size)
            .tag("queue", "processing")
            .description("Current queue size")
            .register(registry);
    }
    
    // Timer - measures duration
    public void recordExecutionTime(String operation, Runnable task) {
        Timer.builder("operation.duration")
            .tag("operation", operation)
            .description("Operation execution time")
            .register(registry)
            .record(task);
    }
    
    // Distribution Summary - statistical distribution
    public void recordPayloadSize(long size) {
        DistributionSummary.builder("payload.size")
            .description("Payload size distribution")
            .baseUnit("bytes")
            .register(registry)
            .record(size);
    }
    
    // Long Task Timer - tracks long-running operations
    public void trackLongRunningTask(String taskName) {
        LongTaskTimer.builder("long.task")
            .tag("task", taskName)
            .description("Long running task duration")
            .register(registry);
    }
}
```

### Timed Annotation

```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @GetMapping("/{id}")
    @Timed(value = "user.get", description = "Get user by ID")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        // Method automatically timed
        return ResponseEntity.ok(userService.getUser(id));
    }
    
    @PostMapping
    @Timed(value = "user.create", 
           description = "Create new user",
           extraTags = {"operation", "create"})
    public ResponseEntity<User> createUser(@RequestBody UserRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(userService.createUser(request));
    }
}
```

### Prometheus Configuration

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'spring-boot-app'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['localhost:8080']
        labels:
          application: 'observability-demo'
          environment: 'dev'
    
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
```

### Common Prometheus Queries

```promql
# Request rate (requests per second)
rate(http_server_requests_seconds_count[5m])

# 95th percentile response time
histogram_quantile(0.95, 
  sum(rate(http_server_requests_seconds_bucket[5m])) by (le, uri)
)

# Error rate
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m])) 
/ 
sum(rate(http_server_requests_seconds_count[5m]))

# JVM memory usage
jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"}

# Active orders
orders_active

# Orders created per minute
rate(orders_created_total[1m]) * 60
```

## Loki - Logs

### Dependencies

```xml
<dependency>
    <groupId>com.github.loki4j</groupId>
    <artifactId>loki-logback-appender</artifactId>
    <version>1.4.2</version>
</dependency>
```

### Logback Configuration

```xml
<!-- logback-spring.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>
    
    <springProperty scope="context" name="appName" source="spring.application.name"/>
    <springProperty scope="context" name="environment" source="ENVIRONMENT" defaultValue="dev"/>
    
    <!-- Console Appender -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} [%X{traceId},%X{spanId}] - %msg%n</pattern>
        </encoder>
    </appender>
    
    <!-- Loki Appender -->
    <appender name="LOKI" class="com.github.loki4j.logback.Loki4jAppender">
        <http>
            <url>http://localhost:3100/loki/api/v1/push</url>
        </http>
        <format>
            <label>
                <pattern>application=${appName},environment=${environment},level=%level</pattern>
            </label>
            <message>
                <pattern>
                {
                  "timestamp": "%d{yyyy-MM-dd'T'HH:mm:ss.SSS}",
                  "level": "%level",
                  "thread": "%thread",
                  "logger": "%logger{36}",
                  "message": "%message",
                  "traceId": "%X{traceId}",
                  "spanId": "%X{spanId}",
                  "exception": "%ex{full}"
                }
                </pattern>
            </message>
        </format>
    </appender>
    
    <!-- JSON Appender for structured logging -->
    <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <includeMdcKeyName>traceId</includeMdcKeyName>
            <includeMdcKeyName>spanId</includeMdcKeyName>
            <includeMdcKeyName>userId</includeMdcKeyName>
            <includeMdcKeyName>tenantId</includeMdcKeyName>
            <customFields>{"application":"${appName}","environment":"${environment}"}</customFields>
        </encoder>
    </appender>
    
    <!-- File Appender with Rolling -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/application.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/application-%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
            <maxHistory>30</maxHistory>
            <timeBasedFileNamingAndTriggeringPolicy 
                class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>10MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
        </rollingPolicy>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>
    
    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="LOKI"/>
        <appender-ref ref="FILE"/>
    </root>
    
    <logger name="com.example" level="DEBUG"/>
    <logger name="org.springframework.web" level="DEBUG"/>
</configuration>
```

### Structured Logging

```java
@Slf4j
@Service
public class OrderService {
    
    public Order createOrder(OrderRequest request) {
        // Add context to MDC
        MDC.put("userId", request.getUserId());
        MDC.put("orderId", UUID.randomUUID().toString());
        
        try {
            log.info("Creating order for user: {}", request.getUserId());
            
            Order order = processOrder(request);
            
            // Structured logging with markers
            log.info("Order created successfully: orderId={}, amount={}, items={}", 
                order.getId(), 
                order.getTotalAmount(), 
                order.getItems().size());
            
            return order;
            
        } catch (InsufficientInventoryException e) {
            log.warn("Insufficient inventory for order: {}", e.getMessage());
            throw e;
        } catch (Exception e) {
            log.error("Failed to create order", e);
            throw new OrderCreationException("Order creation failed", e);
        } finally {
            MDC.clear();
        }
    }
    
    public void processPayment(Order order, PaymentDetails payment) {
        // Use structured arguments
        log.info("Processing payment: orderId={}, amount={}, method={}", 
            order.getId(), 
            payment.getAmount(), 
            payment.getMethod());
        
        Map<String, Object> logData = Map.of(
            "orderId", order.getId(),
            "amount", payment.getAmount(),
            "currency", payment.getCurrency(),
            "method", payment.getMethod()
        );
        
        log.info("Payment details: {}", logData);
    }
}
```

### Custom Log Appender

```java
@Component
public class CustomLogAppender extends AppenderBase<ILoggingEvent> {
    
    private final MetricService metricService;
    
    @Override
    protected void append(ILoggingEvent event) {
        if (event.getLevel().isGreaterOrEqual(Level.ERROR)) {
            metricService.incrementErrorCounter(event.getLoggerName());
        }
        
        if (event.getLevel().isGreaterOrEqual(Level.WARN)) {
            metricService.incrementWarningCounter(event.getLoggerName());
        }
    }
}
```

### LogQL Queries (Loki)

```logql
# All logs for application
{application="observability-demo"}

# Error logs only
{application="observability-demo"} |= "level=ERROR"

# Logs for specific user
{application="observability-demo"} | json | userId="12345"

# Rate of errors
rate({application="observability-demo"} |= "ERROR" [5m])

# Count by log level
sum by (level) (count_over_time({application="observability-demo"}[5m]))

# Logs with trace ID
{application="observability-demo"} | json | traceId="abc123"

# Parse and filter JSON logs
{application="observability-demo"} 
  | json 
  | line_format "{{.message}}" 
  | level="ERROR"
```

## Tempo - Traces

### Dependencies

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-reporter-brave</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
```

### Tracing Configuration

```yaml
management:
  tracing:
    sampling:
      probability: 1.0  # Sample 100% in dev, reduce in prod
  otlp:
    tracing:
      endpoint: http://localhost:4318/v1/traces
      
  zipkin:
    tracing:
      endpoint: http://localhost:9411/api/v2/spans
```

### Custom Spans

```java
@Service
@RequiredArgsConstructor
public class OrderService {
    
    private final Tracer tracer;
    private final OrderRepository orderRepository;
    private final PaymentService paymentService;
    private final InventoryService inventoryService;
    
    public Order createOrder(OrderRequest request) {
        Span span = tracer.nextSpan().name("createOrder").start();
        
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            span.tag("order.type", request.getType());
            span.tag("user.id", request.getUserId());
            span.tag("items.count", String.valueOf(request.getItems().size()));
            
            // Child span for inventory check
            Span inventorySpan = tracer.nextSpan()
                .name("checkInventory")
                .start();
            
            try (Tracer.SpanInScope inventoryScope = tracer.withSpan(inventorySpan)) {
                boolean available = inventoryService.checkAvailability(request.getItems());
                inventorySpan.tag("inventory.available", String.valueOf(available));
                
                if (!available) {
                    span.tag("error", "insufficient_inventory");
                    throw new InsufficientInventoryException();
                }
            } finally {
                inventorySpan.end();
            }
            
            // Process payment
            Span paymentSpan = tracer.nextSpan()
                .name("processPayment")
                .start();
            
            try (Tracer.SpanInScope paymentScope = tracer.withSpan(paymentSpan)) {
                Payment payment = paymentService.process(request.getPayment());
                paymentSpan.tag("payment.status", payment.getStatus());
                paymentSpan.tag("payment.amount", payment.getAmount().toString());
            } finally {
                paymentSpan.end();
            }
            
            // Save order
            Order order = orderRepository.save(buildOrder(request));
            span.tag("order.id", order.getId().toString());
            span.event("order.created");
            
            return order;
            
        } catch (Exception e) {
            span.error(e);
            span.tag("error.type", e.getClass().getSimpleName());
            throw e;
        } finally {
            span.end();
        }
    }
}
```

### Aspect for Automatic Tracing

```java
@Aspect
@Component
@RequiredArgsConstructor
public class TracingAspect {
    
    private final Tracer tracer;
    
    @Around("@annotation(traced)")
    public Object traceMethod(ProceedingJoinPoint joinPoint, Traced traced) 
            throws Throwable {
        
        String spanName = traced.value().isEmpty() 
            ? joinPoint.getSignature().getName() 
            : traced.value();
        
        Span span = tracer.nextSpan().name(spanName).start();
        
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            // Add method parameters as tags
            Object[] args = joinPoint.getArgs();
            for (int i = 0; i < args.length; i++) {
                if (args[i] != null) {
                    span.tag("arg." + i, args[i].toString());
                }
            }
            
            Object result = joinPoint.proceed();
            
            if (result != null) {
                span.tag("result.type", result.getClass().getSimpleName());
            }
            
            return result;
            
        } catch (Exception e) {
            span.error(e);
            throw e;
        } finally {
            span.end();
        }
    }
}

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Traced {
    String value() default "";
}
```

### Usage of Traced Annotation

```java
@Service
public class UserService {
    
    @Traced("user.create")
    public User createUser(UserRequest request) {
        // Automatically traced
        return userRepository.save(buildUser(request));
    }
    
    @Traced("user.validate")
    public void validateUser(User user) {
        // Validation logic
    }
}
```

### Distributed Tracing

```java
@Service
@RequiredArgsConstructor
public class DistributedOrderService {
    
    private final RestTemplate restTemplate;
    private final Tracer tracer;
    
    public Order processOrder(OrderRequest request) {
        Span span = tracer.currentSpan();
        
        // Trace ID is automatically propagated via HTTP headers
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        
        // Call inventory service
        String inventoryUrl = "http://inventory-service/api/check";
        ResponseEntity<InventoryResponse> inventoryResponse = 
            restTemplate.exchange(
                inventoryUrl,
                HttpMethod.POST,
                new HttpEntity<>(request.getItems(), headers),
                InventoryResponse.class
            );
        
        // Call payment service
        String paymentUrl = "http://payment-service/api/process";
        ResponseEntity<PaymentResponse> paymentResponse = 
            restTemplate.exchange(
                paymentUrl,
                HttpMethod.POST,
                new HttpEntity<>(request.getPayment(), headers),
                PaymentResponse.class
            );
        
        span.event("services.called");
        
        return buildOrder(request, inventoryResponse.getBody(), 
            paymentResponse.getBody());
    }
}
```

## Grafana - Visualization

### Grafana Configuration

```yaml
# grafana-datasources.yml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: true
    
  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    editable: true
    jsonData:
      derivedFields:
        - datasourceUid: tempo
          matcherRegex: "traceId=(\\w+)"
          name: TraceID
          url: '$${__value.raw}'
    
  - name: Tempo
    type: tempo
    access: proxy
    url: http://tempo:3200
    editable: true
    jsonData:
      tracesToLogs:
        datasourceUid: loki
        tags: ['traceId']
      tracesToMetrics:
        datasourceUid: prometheus
        tags: [{ key: 'service.name', value: 'application' }]
      serviceMap:
        datasourceUid: prometheus
```

### Custom Dashboard JSON

```json
{
  "dashboard": {
    "title": "Application Observability",
    "panels": [
      {
        "title": "Request Rate",
        "targets": [
          {
            "expr": "rate(http_server_requests_seconds_count[5m])",
            "legendFormat": "{{uri}}"
          }
        ],
        "type": "graph"
      },
      {
        "title": "Error Rate",
        "targets": [
          {
            "expr": "sum(rate(http_server_requests_seconds_count{status=~\"5..\"}[5m])) / sum(rate(http_server_requests_seconds_count[5m]))",
            "legendFormat": "Error Rate"
          }
        ],
        "type": "graph"
      },
      {
        "title": "Response Time (p95)",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, sum(rate(http_server_requests_seconds_bucket[5m])) by (le, uri))",
            "legendFormat": "{{uri}}"
          }
        ],
        "type": "graph"
      }
    ]
  }
}
```

## Spring Boot Actuator

### Health Indicators

```java
@Component
public class CustomHealthIndicator implements HealthIndicator {
    
    private final ExternalService externalService;
    
    @Override
    public Health health() {
        try {
            externalService.ping();
            return Health.up()
                .withDetail("service", "external-api")
                .withDetail("status", "reachable")
                .withDetail("timestamp", LocalDateTime.now())
                .build();
        } catch (Exception e) {
            return Health.down()
                .withDetail("service", "external-api")
                .withDetail("error", e.getMessage())
                .withException(e)
                .build();
        }
    }
}
```

### Custom Info Contributor

```java
@Component
public class CustomInfoContributor implements InfoContributor {
    
    @Override
    public void contribute(Info.Builder builder) {
        Map<String, Object> details = new HashMap<>();
        details.put("version", "1.0.0");
        details.put("build-time", LocalDateTime.now());
        details.put("active-users", getUserCount());
        
        builder.withDetail("application", details);
    }
    
    private int getUserCount() {
        // Logic to count active users
        return 100;
    }
}
```

### Custom Actuator Endpoint

```java
@Component
@Endpoint(id = "custom-metrics")
public class CustomMetricsEndpoint {
    
    private final OrderService orderService;
    
    @ReadOperation
    public Map<String, Object> getMetrics() {
        Map<String, Object> metrics = new HashMap<>();
        metrics.put("activeOrders", orderService.getActiveOrderCount());
        metrics.put("pendingOrders", orderService.getPendingOrderCount());
        metrics.put("completedToday", orderService.getCompletedTodayCount());
        return metrics;
    }
    
    @ReadOperation
    public Map<String, Object> getMetricsByType(@Selector String type) {
        Map<String, Object> metrics = new HashMap<>();
        metrics.put("count", orderService.getCountByType(type));
        metrics.put("revenue", orderService.getRevenueByType(type));
        return metrics;
    }
}
```

## Custom Metrics

### Business Metrics

```java
@Component
@RequiredArgsConstructor
public class BusinessMetrics {
    
    private final MeterRegistry registry;
    
    public void recordOrderValue(BigDecimal amount, String currency) {
        registry.counter("business.orders.value",
            "currency", currency)
            .increment(amount.doubleValue());
    }
    
    public void recordUserSignup(String source) {
        registry.counter("business.users.signup",
            "source", source)
            .increment();
    }
    
    public void recordConversion(String funnel, String step) {
        registry.counter("business.conversion",
            "funnel", funnel,
            "step", step)
            .increment();
    }
    
    public void recordRevenue(BigDecimal amount, String product) {
        DistributionSummary.builder("business.revenue")
            .tag("product", product)
            .baseUnit("currency")
            .register(registry)
            .record(amount.doubleValue());
    }
}
```

### Performance Metrics

```java
@Aspect
@Component
@RequiredArgsConstructor
public class PerformanceMetricsAspect {
    
    private final MeterRegistry registry;
    
    @Around("@annotation(org.springframework.web.bind.annotation.RequestMapping)")
    public Object measurePerformance(ProceedingJoinPoint joinPoint) throws Throwable {
        Timer.Sample sample = Timer.start(registry);
        
        try {
            Object result = joinPoint.proceed();
            
            sample.stop(Timer.builder("method.execution.time")
                .tag("class", joinPoint.getTarget().getClass().getSimpleName())
                .tag("method", joinPoint.getSignature().getName())
                .tag("status", "success")
                .register(registry));
            
            return result;
            
        } catch (Exception e) {
            sample.stop(Timer.builder("method.execution.time")
                .tag("class", joinPoint.getTarget().getClass().getSimpleName())
                .tag("method", joinPoint.getSignature().getName())
                .tag("status", "error")
                .register(registry));
            
            throw e;
        }
    }
}
```

## Distributed Tracing

### Baggage Propagation

```java
@Service
@RequiredArgsConstructor
public class BaggageService {
    
    private final Tracer tracer;
    
    public void addUserContext(String userId, String tenantId) {
        try (BaggageInScope userIdScope = 
                tracer.createBaggage("userId").set(userId).makeCurrent()) {
            try (BaggageInScope tenantIdScope = 
                    tracer.createBaggage("tenantId").set(tenantId).makeCurrent()) {
                
                // Baggage is automatically propagated to child spans
                // and across service boundaries
                processRequest();
            }
        }
    }
    
    public String getUserFromBaggage() {
        Baggage baggage = tracer.getBaggage("userId");
        return baggage != null ? baggage.get() : null;
    }
}
```

## POC Implementation

### Project Structure

```
observability-poc/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/example/observability/
│   │   │       ├── config/
│   │   │       │   ├── ObservabilityConfig.java
│   │   │       │   ├── MetricsConfig.java
│   │   │       │   └── TracingConfig.java
│   │   │       ├── controller/
│   │   │       │   └── OrderController.java
│   │   │       ├── service/
│   │   │       │   ├── OrderService.java
│   │   │       │   └── MetricsService.java
│   │   │       ├── aspect/
│   │   │       │   ├── TracingAspect.java
│   │   │       │   └── MetricsAspect.java
│   │   │       ├── health/
│   │   │       │   └── CustomHealthIndicator.java
│   │   │       └── metrics/
│   │   │           └── BusinessMetrics.java
│   │   └── resources/
│   │       ├── application.yml
│   │       ├── logback-spring.xml
│   │       └── grafana/
│   │           └── dashboards/
│   │               └── application-dashboard.json
│   └── test/
├── docker/
│   ├── prometheus/
│   │   └── prometheus.yml
│   ├── loki/
│   │   └── loki-config.yml
│   ├── tempo/
│   │   └── tempo-config.yml
│   └── grafana/
│       ├── provisioning/
│       │   ├── datasources/
│       │   │   └── datasources.yml
│       │   └── dashboards/
│       │       └── dashboards.yml
│       └── dashboards/
└── docker-compose.yml
```

### Docker Compose Setup

```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./docker/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    networks:
      - observability

  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"
    volumes:
      - ./docker/loki/loki-config.yml:/etc/loki/local-config.yaml
      - loki-data:/loki
    command: -config.file=/etc/loki/local-config.yaml
    networks:
      - observability

  tempo:
    image: grafana/tempo:latest
    ports:
      - "3200:3200"   # Tempo HTTP
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
      - "9411:9411"   # Zipkin
    volumes:
      - ./docker/tempo/tempo-config.yml:/etc/tempo.yaml
      - tempo-data:/var/tempo
    command: [ "-config.file=/etc/tempo.yaml" ]
    networks:
      - observability

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - ./docker/grafana/provisioning:/etc/grafana/provisioning
      - grafana-data:/var/lib/grafana
    depends_on:
      - prometheus
      - loki
      - tempo
    networks:
      - observability

  spring-boot-app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - MANAGEMENT_OTLP_TRACING_ENDPOINT=http://tempo:4318/v1/traces
    depends_on:
      - prometheus
      - loki
      - tempo
    networks:
      - observability

volumes:
  prometheus-data:
  loki-data:
  tempo-data:
  grafana-data:

networks:
  observability:
    driver: bridge
```

### Complete Configuration Example

```java
@Configuration
public class ObservabilityConfig {
    
    @Bean
    public MeterRegistryCustomizer<MeterRegistry> metricsCommonTags(
            @Value("${spring.application.name}") String applicationName) {
        return registry -> registry.config()
            .commonTags(
                "application", applicationName,
                "environment", System.getenv().getOrDefault("ENVIRONMENT", "dev")
            );
    }
    
    @Bean
    public TimedAspect timedAspect(MeterRegistry registry) {
        return new TimedAspect(registry);
    }
    
    @Bean
    @ConditionalOnMissingBean
    public RestTemplate restTemplate(RestTemplateBuilder builder) {
        return builder
            .setConnectTimeout(Duration.ofSeconds(5))
            .setReadTimeout(Duration.ofSeconds(5))
            .build();
    }
}
```

### Loki Configuration

```yaml
# loki-config.yml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2024-01-01
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

limits_config:
  reject_old_samples: true
  reject_old_samples_max_age: 168h
  ingestion_rate_mb: 10
  ingestion_burst_size_mb: 20
```

### Tempo Configuration

```yaml
# tempo-config.yml
server:
  http_listen_port: 3200

distributor:
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317
        http:
          endpoint: 0.0.0.0:4318
    zipkin:
      endpoint: 0.0.0.0:9411

storage:
  trace:
    backend: local
    local:
      path: /var/tempo/traces

compactor:
  compaction:
    block_retention: 48h
```

## Best Practices

1. **Metrics**
   - Use appropriate metric types
   - Add meaningful tags
   - Keep cardinality low
   - Set up alerts on key metrics

2. **Logging**
   - Use structured logging
   - Include correlation IDs
   - Log at appropriate levels
   - Avoid logging sensitive data

3. **Tracing**
   - Sample appropriately in production
   - Add meaningful span names and tags
   - Trace across service boundaries
   - Use baggage for context propagation

4. **Performance**
   - Minimize overhead
   - Use sampling for high-volume traces
   - Buffer logs and metrics
   - Monitor observability infrastructure

5. **Security**
   - Secure observability endpoints
   - Sanitize sensitive data
   - Control access to dashboards
   - Encrypt data in transit

## Common Pitfalls

- High cardinality metrics
- Excessive logging volume
- Not sampling traces in production
- Missing correlation between pillars
- Poor metric naming conventions
- Logging sensitive information
- Not monitoring the monitoring system

## Additional Resources

- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Loki Documentation](https://grafana.com/docs/loki/)
- [Grafana Tempo Documentation](https://grafana.com/docs/tempo/)
- [Micrometer Documentation](https://micrometer.io/docs)
- [Spring Boot Actuator](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html)
