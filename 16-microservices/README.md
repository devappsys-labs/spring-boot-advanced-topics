
# Microservices Architecture

## Overview

Microservices architecture is an approach to developing a single application as a suite of small, independently deployable services. Each service runs in its own process and communicates with lightweight mechanisms, often HTTP-based APIs.

## Key Concepts

### Core Principles

- **Single Responsibility** - Each service does one thing well
- **Independently Deployable** - Services can be deployed separately
- **Decentralized** - No central control or data store
- **Built Around Business Capabilities** - Aligned with business domains
- **Infrastructure Automation** - CI/CD pipelines
- **Design for Failure** - Resilience patterns

### Service Communication

- **Synchronous** - REST, gRPC
- **Asynchronous** - Message queues, event streams
- **Service Discovery** - Dynamic service location
- **Load Balancing** - Distribute requests
- **Circuit Breaker** - Prevent cascade failures

## Dependencies

```xml
<dependencies>
    <!-- Spring Boot Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    
    <!-- Spring Cloud -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    
    <!-- Load Balancer -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-loadbalancer</artifactId>
    </dependency>
    
    <!-- Circuit Breaker -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
    </dependency>
    
    <!-- OpenFeign -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
    
    <!-- API Gateway -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>
</dependencies>
```

## Service Discovery with Eureka

### Eureka Server

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

```yaml
# application.yml
server:
  port: 8761

eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false
    fetch-registry: false
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/

spring:
  application:
    name: eureka-server
```

### Eureka Client (Service)

```java
@SpringBootApplication
@EnableDiscoveryClient
public class UserServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(UserServiceApplication.class, args);
    }
}
```

```yaml
# application.yml
server:
  port: 8081

spring:
  application:
    name: user-service

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
    fetch-registry: true
    register-with-eureka: true
  instance:
    prefer-ip-address: true
    lease-renewal-interval-in-seconds: 30
    lease-expiration-duration-in-seconds: 90
```

## Inter-Service Communication

### RestTemplate with LoadBalancer

```java
@Configuration
public class RestTemplateConfig {
    
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}

@Service
public class UserService {
    
    @Autowired
    @LoadBalanced
    private RestTemplate restTemplate;
    
    public Order getUserOrder(String userId) {
        String url = "http://order-service/api/orders/user/" + userId;
        return restTemplate.getForObject(url, Order.class);
    }
}
```

### OpenFeign Client

```java
@EnableFeignClients
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

@FeignClient(name = "order-service", fallback = OrderClientFallback.class)
public interface OrderClient {
    
    @GetMapping("/api/orders/user/{userId}")
    List<Order> getOrdersByUser(@PathVariable("userId") String userId);
    
    @GetMapping("/api/orders/{orderId}")
    Order getOrder(@PathVariable("orderId") String orderId);
    
    @PostMapping("/api/orders")
    Order createOrder(@RequestBody OrderRequest request);
}

@Component
public class OrderClientFallback implements OrderClient {
    
    @Override
    public List<Order> getOrdersByUser(String userId) {
        return Collections.emptyList();
    }
    
    @Override
    public Order getOrder(String orderId) {
        return null;
    }
    
    @Override
    public Order createOrder(OrderRequest request) {
        throw new ServiceUnavailableException("Order service is unavailable");
    }
}
```

## API Gateway

### Spring Cloud Gateway

```java
@SpringBootApplication
public class ApiGatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(ApiGatewayApplication.class, args);
    }
}
```

```yaml
# application.yml
server:
  port: 8080

spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
          lower-case-service-id: true
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
          filters:
            - RewritePath=/api/users/(?<segment>.*), /${segment}
            - AddRequestHeader=X-Gateway, API-Gateway
            
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            - RewritePath=/api/orders/(?<segment>.*), /${segment}
            - CircuitBreaker=orderCircuitBreaker
            
        - id: product-service
          uri: lb://product-service
          predicates:
            - Path=/api/products/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

### Custom Gateway Filters

```java
@Component
public class AuthenticationFilter implements GlobalFilter, Ordered {
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        
        if (!request.getHeaders().containsKey("Authorization")) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
        
        String token = request.getHeaders().getFirst("Authorization");
        
        if (!isValidToken(token)) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
        
        return chain.filter(exchange);
    }
    
    @Override
    public int getOrder() {
        return -1;
    }
    
    private boolean isValidToken(String token) {
        // Validate JWT token
        return token != null && token.startsWith("Bearer ");
    }
}

@Component
public class LoggingFilter implements GlobalFilter, Ordered {
    
    private static final Logger logger = LoggerFactory.getLogger(LoggingFilter.class);
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        logger.info("Request: {} {}", 
                exchange.getRequest().getMethod(),
                exchange.getRequest().getURI());
        
        return chain.filter(exchange).then(Mono.fromRunnable(() -> {
            logger.info("Response: {}", exchange.getResponse().getStatusCode());
        }));
    }
    
    @Override
    public int getOrder() {
        return -2;
    }
}
```

## Circuit Breaker Pattern

### Resilience4j Configuration

```yaml
resilience4j:
  circuitbreaker:
    configs:
      default:
        sliding-window-size: 10
        minimum-number-of-calls: 5
        permitted-number-of-calls-in-half-open-state: 3
        automatic-transition-from-open-to-half-open-enabled: true
        wait-duration-in-open-state: 10s
        failure-rate-threshold: 50
        slow-call-duration-threshold: 2s
        slow-call-rate-threshold: 50
    instances:
      orderService:
        base-config: default
        
  retry:
    configs:
      default:
        max-attempts: 3
        wait-duration: 1s
        retry-exceptions:
          - org.springframework.web.client.HttpServerErrorException
    instances:
      orderService:
        base-config: default
        
  timelimiter:
    configs:
      default:
        timeout-duration: 3s
    instances:
      orderService:
        base-config: default
```

### Using Circuit Breaker

```java
@Service
public class OrderService {
    
    @Autowired
    private RestTemplate restTemplate;
    
    @CircuitBreaker(name = "orderService", fallbackMethod = "getOrderFallback")
    @Retry(name = "orderService")
    @TimeLimiter(name = "orderService")
    public CompletableFuture<Order> getOrder(String orderId) {
        return CompletableFuture.supplyAsync(() -> {
            String url = "http://order-service/api/orders/" + orderId;
            return restTemplate.getForObject(url, Order.class);
        });
    }
    
    private CompletableFuture<Order> getOrderFallback(String orderId, Exception ex) {
        logger.error("Fallback triggered for order: {}", orderId, ex);
        return CompletableFuture.completedFuture(createDefaultOrder(orderId));
    }
    
    private Order createDefaultOrder(String orderId) {
        Order order = new Order();
        order.setId(orderId);
        order.setStatus("UNAVAILABLE");
        return order;
    }
}
```

## Distributed Tracing

### Sleuth and Zipkin

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-sleuth-zipkin</artifactId>
</dependency>
```

```yaml
spring:
  sleuth:
    sampler:
      probability: 1.0
  zipkin:
    base-url: http://localhost:9411
    sender:
      type: web
```

## Service Mesh (Example: Istio)

### Service Deployment with Istio

```yaml
apiVersion: v1
kind: Service
metadata:
  name: user-service
  labels:
    app: user-service
spec:
  ports:
    - port: 8080
      name: http
  selector:
    app: user-service
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
        - name: user-service
          image: user-service:1.0.0
          ports:
            - containerPort: 8080
```

### Virtual Service

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: user-service
spec:
  hosts:
    - user-service
  http:
    - match:
        - headers:
            version:
              exact: v2
      route:
        - destination:
            host: user-service
            subset: v2
    - route:
        - destination:
            host: user-service
            subset: v1
          weight: 90
        - destination:
            host: user-service
            subset: v2
          weight: 10
```

## Saga Pattern (Distributed Transactions)

### Orchestration-based Saga

```java
@Service
public class OrderSagaOrchestrator {
    
    @Autowired
    private OrderService orderService;
    
    @Autowired
    private PaymentService paymentService;
    
    @Autowired
    private InventoryService inventoryService;
    
    @Autowired
    private ShippingService shippingService;
    
    public Order createOrder(OrderRequest request) {
        Order order = null;
        Payment payment = null;
        Reservation reservation = null;
        
        try {
            // Step 1: Create order
            order = orderService.createOrder(request);
            
            // Step 2: Reserve inventory
            reservation = inventoryService.reserveItems(order.getItems());
            
            // Step 3: Process payment
            payment = paymentService.processPayment(order.getId(), order.getAmount());
            
            // Step 4: Arrange shipping
            shippingService.createShipment(order);
            
            // All steps successful
            orderService.confirmOrder(order.getId());
            
            return order;
            
        } catch (Exception e) {
            // Compensate (rollback) in reverse order
            if (payment != null) {
                paymentService.refundPayment(payment.getId());
            }
            
            if (reservation != null) {
                inventoryService.releaseReservation(reservation.getId());
            }
            
            if (order != null) {
                orderService.cancelOrder(order.getId());
            }
            
            throw new OrderCreationException("Failed to create order", e);
        }
    }
}
```

### Choreography-based Saga (Event-driven)

```java
@Service
public class OrderEventHandler {
    
    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;
    
    @Transactional
    public void createOrder(OrderRequest request) {
        Order order = saveOrder(request);
        
        OrderEvent event = new OrderEvent(
                order.getId(),
                "ORDER_CREATED",
                order
        );
        
        kafkaTemplate.send("order-events", event);
    }
    
    @KafkaListener(topics = "payment-events")
    public void handlePaymentEvent(PaymentEvent event) {
        if (event.getType().equals("PAYMENT_COMPLETED")) {
            updateOrderStatus(event.getOrderId(), "PAID");
            
            // Trigger next step
            InventoryReservationEvent reservationEvent = new InventoryReservationEvent(
                    event.getOrderId(),
                    getOrderItems(event.getOrderId())
            );
            
            kafkaTemplate.send("inventory-events", reservationEvent);
        } else if (event.getType().equals("PAYMENT_FAILED")) {
            cancelOrder(event.getOrderId());
        }
    }
    
    @KafkaListener(topics = "inventory-events")
    public void handleInventoryEvent(InventoryEvent event) {
        if (event.getType().equals("RESERVATION_COMPLETED")) {
            updateOrderStatus(event.getOrderId(), "RESERVED");
            
            // Trigger shipping
            ShippingEvent shippingEvent = new ShippingEvent(
                    event.getOrderId(),
                    getShippingAddress(event.getOrderId())
            );
            
            kafkaTemplate.send("shipping-events", shippingEvent);
        } else if (event.getType().equals("RESERVATION_FAILED")) {
            // Compensate: refund payment
            kafkaTemplate.send("payment-events", new RefundEvent(event.getOrderId()));
        }
    }
}
```

## Database Per Service Pattern

### User Service Database

```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String username;
    private String email;
    // User-specific fields
}

@Configuration
@EnableJpaRepositories(
    basePackages = "com.example.userservice.repository",
    entityManagerFactoryRef = "userEntityManagerFactory",
    transactionManagerRef = "userTransactionManager"
)
public class UserDatabaseConfig {
    
    @Primary
    @Bean
    @ConfigurationProperties("spring.datasource.user")
    public DataSource userDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Primary
    @Bean
    public LocalContainerEntityManagerFactoryBean userEntityManagerFactory(
            EntityManagerFactoryBuilder builder) {
        return builder
                .dataSource(userDataSource())
                .packages("com.example.userservice.model")
                .persistenceUnit("user")
                .build();
    }
    
    @Primary
    @Bean
    public PlatformTransactionManager userTransactionManager(
            @Qualifier("userEntityManagerFactory") EntityManagerFactory entityManagerFactory) {
        return new JpaTransactionManager(entityManagerFactory);
    }
}
```

## API Versioning

### URL Versioning

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserControllerV1 {
    
    @GetMapping("/{id}")
    public UserV1 getUser(@PathVariable Long id) {
        return userService.getUserV1(id);
    }
}

@RestController
@RequestMapping("/api/v2/users")
public class UserControllerV2 {
    
    @GetMapping("/{id}")
    public UserV2 getUser(@PathVariable Long id) {
        return userService.getUserV2(id);
    }
}
```

### Header Versioning

```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @GetMapping(value = "/{id}", headers = "API-Version=1")
    public UserV1 getUserV1(@PathVariable Long id) {
        return userService.getUserV1(id);
    }
    
    @GetMapping(value = "/{id}", headers = "API-Version=2")
    public UserV2 getUserV2(@PathVariable Long id) {
        return userService.getUserV2(id);
    }
}
```

## Health Checks and Monitoring

### Custom Health Indicator

```java
@Component
public class CustomHealthIndicator implements HealthIndicator {
    
    @Autowired
    private OrderClient orderClient;
    
    @Override
    public Health health() {
        try {
            // Check if dependent service is healthy
            orderClient.healthCheck();
            
            return Health.up()
                    .withDetail("order-service", "Available")
                    .build();
        } catch (Exception e) {
            return Health.down()
                    .withDetail("order-service", "Unavailable")
                    .withException(e)
                    .build();
        }
    }
}
```

### Metrics

```java
@Configuration
public class MetricsConfig {
    
    @Bean
    public MeterRegistryCustomizer<MeterRegistry> metricsCommonTags() {
        return registry -> registry.config().commonTags(
                "application", "user-service",
                "environment", "production"
        );
    }
}

@Service
public class UserService {
    
    private final Counter userCreatedCounter;
    private final Timer userFetchTimer;
    
    public UserService(MeterRegistry registry) {
        this.userCreatedCounter = Counter.builder("users.created")
                .description("Number of users created")
                .register(registry);
        
        this.userFetchTimer = Timer.builder("users.fetch")
                .description("Time to fetch user")
                .register(registry);
    }
    
    public User createUser(UserRequest request) {
        User user = saveUser(request);
        userCreatedCounter.increment();
        return user;
    }
    
    public User getUser(Long id) {
        return userFetchTimer.record(() -> {
            return userRepository.findById(id)
                    .orElseThrow(() -> new UserNotFoundException(id));
        });
    }
}
```

## Docker Compose for Microservices

```yaml
version: '3.8'

services:
  eureka-server:
    build: ./eureka-server
    ports:
      - "8761:8761"
    environment:
      - SPRING_PROFILES_ACTIVE=docker

  api-gateway:
    build: ./api-gateway
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - EUREKA_CLIENT_SERVICEURL_DEFAULTZONE=http://eureka-server:8761/eureka/
    depends_on:
      - eureka-server

  user-service:
    build: ./user-service
    ports:
      - "8081:8081"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - EUREKA_CLIENT_SERVICEURL_DEFAULTZONE=http://eureka-server:8761/eureka/
      - SPRING_DATASOURCE_URL=jdbc:postgresql://user-db:5432/userdb
    depends_on:
      - eureka-server
      - user-db

  order-service:
    build: ./order-service
    ports:
      - "8082:8082"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - EUREKA_CLIENT_SERVICEURL_DEFAULTZONE=http://eureka-server:8761/eureka/
      - SPRING_DATASOURCE_URL=jdbc:postgresql://order-db:5432/orderdb
    depends_on:
      - eureka-server
      - order-db

  user-db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: userdb
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    volumes:
      - user-data:/var/lib/postgresql/data

  order-db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: orderdb
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    volumes:
      - order-data:/var/lib/postgresql/data

  zipkin:
    image: openzipkin/zipkin
    ports:
      - "9411:9411"

volumes:
  user-data:
  order-data:
```

## Best Practices

1. **Domain-Driven Design** - Align services with business domains
2. **API Gateway Pattern** - Single entry point for clients
3. **Service Discovery** - Dynamic service location
4. **Circuit Breaker** - Prevent cascade failures
5. **Distributed Tracing** - Track requests across services
6. **Centralized Logging** - Aggregate logs from all services
7. **API Versioning** - Support multiple API versions
8. **Health Checks** - Monitor service health
9. **Database Per Service** - Data independence
10. **Event-Driven Architecture** - Asynchronous communication

## Common Pitfalls

1. Distributed monolith
2. Lack of service boundaries
3. Tight coupling between services
4. Ignoring network failures
5. Inconsistent error handling
6. Not implementing circuit breakers
7. Inadequate monitoring and logging
8. Complex distributed transactions

## Resources

- [Spring Cloud Documentation](https://spring.io/projects/spring-cloud)
- [Microservices Patterns](https://microservices.io/patterns/index.html)
- [Martin Fowler - Microservices](https://martinfowler.com/articles/microservices.html)
- [Netflix OSS](https://netflix.github.io/)
