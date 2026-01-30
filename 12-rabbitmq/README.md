# RabbitMQ

## Overview

RabbitMQ is a robust message broker that implements the Advanced Message Queuing Protocol (AMQP). It enables asynchronous communication between services, providing reliable message delivery, routing, and queuing capabilities essential for building scalable microservices architectures.

## Key Concepts

### Core Components

- **Producer** - Application that sends messages
- **Consumer** - Application that receives messages
- **Queue** - Buffer that stores messages
- **Exchange** - Routes messages to queues
- **Binding** - Link between exchange and queue
- **Routing Key** - Message routing criteria
- **Virtual Host** - Logical grouping of resources

### Exchange Types

- **Direct Exchange** - Routes based on exact routing key match
- **Fanout Exchange** - Broadcasts to all bound queues
- **Topic Exchange** - Routes based on pattern matching
- **Headers Exchange** - Routes based on message headers

## Dependencies

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-amqp</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.amqp</groupId>
        <artifactId>spring-rabbit-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

## Configuration

### Application Properties

```properties
# RabbitMQ Configuration
spring.rabbitmq.host=localhost
spring.rabbitmq.port=5672
spring.rabbitmq.username=guest
spring.rabbitmq.password=guest
spring.rabbitmq.virtual-host=/

# Connection Pool
spring.rabbitmq.connection-timeout=30000
spring.rabbitmq.requested-heartbeat=60

# Consumer Configuration
spring.rabbitmq.listener.simple.concurrency=3
spring.rabbitmq.listener.simple.max-concurrency=10
spring.rabbitmq.listener.simple.prefetch=10
spring.rabbitmq.listener.simple.retry.enabled=true
spring.rabbitmq.listener.simple.retry.initial-interval=1000
spring.rabbitmq.listener.simple.retry.max-attempts=3
spring.rabbitmq.listener.simple.retry.multiplier=2.0

# Publisher Configuration
spring.rabbitmq.publisher-confirm-type=correlated
spring.rabbitmq.publisher-returns=true
spring.rabbitmq.template.mandatory=true
```

### RabbitMQ Configuration Class

```java
@Configuration
public class RabbitMQConfig {
    
    @Bean
    public Jackson2JsonMessageConverter messageConverter() {
        return new Jackson2JsonMessageConverter();
    }
    
    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
        RabbitTemplate template = new RabbitTemplate(connectionFactory);
        template.setMessageConverter(messageConverter());
        template.setMandatory(true);
        template.setReturnsCallback(returnedMessage -> {
            System.err.println("Message returned: " + returnedMessage.getMessage());
        });
        return template;
    }
}
```

## Basic Messaging Patterns

### Simple Queue

```java
@Configuration
public class SimpleQueueConfig {
    
    public static final String QUEUE_NAME = "simple.queue";
    
    @Bean
    public Queue simpleQueue() {
        return new Queue(QUEUE_NAME, true); // durable queue
    }
}

@Service
public class SimpleProducer {
    
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    public void sendMessage(String message) {
        rabbitTemplate.convertAndSend(SimpleQueueConfig.QUEUE_NAME, message);
        System.out.println("Sent: " + message);
    }
}

@Component
public class SimpleConsumer {
    
    @RabbitListener(queues = SimpleQueueConfig.QUEUE_NAME)
    public void receiveMessage(String message) {
        System.out.println("Received: " + message);
    }
}
```

### Direct Exchange Pattern

```java
@Configuration
public class DirectExchangeConfig {
    
    public static final String EXCHANGE_NAME = "direct.exchange";
    public static final String QUEUE_NAME = "direct.queue";
    public static final String ROUTING_KEY = "direct.routing.key";
    
    @Bean
    public DirectExchange directExchange() {
        return new DirectExchange(EXCHANGE_NAME);
    }
    
    @Bean
    public Queue directQueue() {
        return new Queue(QUEUE_NAME, true);
    }
    
    @Bean
    public Binding directBinding(Queue directQueue, DirectExchange directExchange) {
        return BindingBuilder.bind(directQueue)
                .to(directExchange)
                .with(ROUTING_KEY);
    }
}

@Service
public class DirectProducer {
    
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    public void sendMessage(String message) {
        rabbitTemplate.convertAndSend(
                DirectExchangeConfig.EXCHANGE_NAME,
                DirectExchangeConfig.ROUTING_KEY,
                message
        );
    }
}

@Component
public class DirectConsumer {
    
    @RabbitListener(queues = DirectExchangeConfig.QUEUE_NAME)
    public void receiveMessage(String message) {
        System.out.println("Direct Consumer received: " + message);
    }
}
```

### Fanout Exchange Pattern

```java
@Configuration
public class FanoutExchangeConfig {
    
    public static final String EXCHANGE_NAME = "fanout.exchange";
    public static final String QUEUE_1 = "fanout.queue.1";
    public static final String QUEUE_2 = "fanout.queue.2";
    public static final String QUEUE_3 = "fanout.queue.3";
    
    @Bean
    public FanoutExchange fanoutExchange() {
        return new FanoutExchange(EXCHANGE_NAME);
    }
    
    @Bean
    public Queue fanoutQueue1() {
        return new Queue(QUEUE_1);
    }
    
    @Bean
    public Queue fanoutQueue2() {
        return new Queue(QUEUE_2);
    }
    
    @Bean
    public Queue fanoutQueue3() {
        return new Queue(QUEUE_3);
    }
    
    @Bean
    public Binding fanoutBinding1(Queue fanoutQueue1, FanoutExchange fanoutExchange) {
        return BindingBuilder.bind(fanoutQueue1).to(fanoutExchange);
    }
    
    @Bean
    public Binding fanoutBinding2(Queue fanoutQueue2, FanoutExchange fanoutExchange) {
        return BindingBuilder.bind(fanoutQueue2).to(fanoutExchange);
    }
    
    @Bean
    public Binding fanoutBinding3(Queue fanoutQueue3, FanoutExchange fanoutExchange) {
        return BindingBuilder.bind(fanoutQueue3).to(fanoutExchange);
    }
}

@Service
public class FanoutProducer {
    
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    public void broadcast(String message) {
        rabbitTemplate.convertAndSend(FanoutExchangeConfig.EXCHANGE_NAME, "", message);
        System.out.println("Broadcast: " + message);
    }
}

@Component
public class FanoutConsumers {
    
    @RabbitListener(queues = FanoutExchangeConfig.QUEUE_1)
    public void consumer1(String message) {
        System.out.println("Consumer 1 received: " + message);
    }
    
    @RabbitListener(queues = FanoutExchangeConfig.QUEUE_2)
    public void consumer2(String message) {
        System.out.println("Consumer 2 received: " + message);
    }
    
    @RabbitListener(queues = FanoutExchangeConfig.QUEUE_3)
    public void consumer3(String message) {
        System.out.println("Consumer 3 received: " + message);
    }
}
```

### Topic Exchange Pattern

```java
@Configuration
public class TopicExchangeConfig {
    
    public static final String EXCHANGE_NAME = "topic.exchange";
    public static final String QUEUE_ORDERS = "topic.queue.orders";
    public static final String QUEUE_PAYMENTS = "topic.queue.payments";
    public static final String QUEUE_ALL = "topic.queue.all";
    
    @Bean
    public TopicExchange topicExchange() {
        return new TopicExchange(EXCHANGE_NAME);
    }
    
    @Bean
    public Queue ordersQueue() {
        return new Queue(QUEUE_ORDERS);
    }
    
    @Bean
    public Queue paymentsQueue() {
        return new Queue(QUEUE_PAYMENTS);
    }
    
    @Bean
    public Queue allEventsQueue() {
        return new Queue(QUEUE_ALL);
    }
    
    @Bean
    public Binding ordersBinding(Queue ordersQueue, TopicExchange topicExchange) {
        return BindingBuilder.bind(ordersQueue)
                .to(topicExchange)
                .with("order.*");
    }
    
    @Bean
    public Binding paymentsBinding(Queue paymentsQueue, TopicExchange topicExchange) {
        return BindingBuilder.bind(paymentsQueue)
                .to(topicExchange)
                .with("payment.*");
    }
    
    @Bean
    public Binding allEventsBinding(Queue allEventsQueue, TopicExchange topicExchange) {
        return BindingBuilder.bind(allEventsQueue)
                .to(topicExchange)
                .with("*.#");
    }
}

@Service
public class TopicProducer {
    
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    public void sendOrderEvent(String event) {
        rabbitTemplate.convertAndSend(
                TopicExchangeConfig.EXCHANGE_NAME,
                "order.created",
                event
        );
    }
    
    public void sendPaymentEvent(String event) {
        rabbitTemplate.convertAndSend(
                TopicExchangeConfig.EXCHANGE_NAME,
                "payment.processed",
                event
        );
    }
}

@Component
public class TopicConsumers {
    
    @RabbitListener(queues = TopicExchangeConfig.QUEUE_ORDERS)
    public void handleOrderEvents(String message) {
        System.out.println("Order event: " + message);
    }
    
    @RabbitListener(queues = TopicExchangeConfig.QUEUE_PAYMENTS)
    public void handlePaymentEvents(String message) {
        System.out.println("Payment event: " + message);
    }
    
    @RabbitListener(queues = TopicExchangeConfig.QUEUE_ALL)
    public void handleAllEvents(String message) {
        System.out.println("All events: " + message);
    }
}
```

## Advanced Patterns

### Dead Letter Queue (DLQ)

```java
@Configuration
public class DeadLetterQueueConfig {
    
    public static final String MAIN_QUEUE = "main.queue";
    public static final String DLQ_QUEUE = "dlq.queue";
    public static final String DLQ_EXCHANGE = "dlq.exchange";
    public static final String EXCHANGE_NAME = "main.exchange";
    
    @Bean
    public Queue mainQueue() {
        return QueueBuilder.durable(MAIN_QUEUE)
                .withArgument("x-dead-letter-exchange", DLQ_EXCHANGE)
                .withArgument("x-dead-letter-routing-key", "dlq")
                .withArgument("x-message-ttl", 60000) // 60 seconds TTL
                .build();
    }
    
    @Bean
    public Queue deadLetterQueue() {
        return new Queue(DLQ_QUEUE, true);
    }
    
    @Bean
    public DirectExchange mainExchange() {
        return new DirectExchange(EXCHANGE_NAME);
    }
    
    @Bean
    public DirectExchange deadLetterExchange() {
        return new DirectExchange(DLQ_EXCHANGE);
    }
    
    @Bean
    public Binding mainBinding(Queue mainQueue, DirectExchange mainExchange) {
        return BindingBuilder.bind(mainQueue)
                .to(mainExchange)
                .with("main");
    }
    
    @Bean
    public Binding dlqBinding(Queue deadLetterQueue, DirectExchange deadLetterExchange) {
        return BindingBuilder.bind(deadLetterQueue)
                .to(deadLetterExchange)
                .with("dlq");
    }
}

@Component
public class DLQConsumer {
    
    @RabbitListener(queues = DeadLetterQueueConfig.MAIN_QUEUE)
    public void processMessage(String message) {
        if (message.contains("error")) {
            throw new RuntimeException("Processing failed");
        }
        System.out.println("Processed: " + message);
    }
    
    @RabbitListener(queues = DeadLetterQueueConfig.DLQ_QUEUE)
    public void handleDeadLetter(String message) {
        System.err.println("Dead letter: " + message);
        // Log, alert, or retry logic
    }
}
```

### Message Priority Queue

```java
@Configuration
public class PriorityQueueConfig {
    
    public static final String PRIORITY_QUEUE = "priority.queue";
    
    @Bean
    public Queue priorityQueue() {
        return QueueBuilder.durable(PRIORITY_QUEUE)
                .maxPriority(10)
                .build();
    }
}

@Service
public class PriorityProducer {
    
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    public void sendWithPriority(String message, int priority) {
        rabbitTemplate.convertAndSend(
                PriorityQueueConfig.PRIORITY_QUEUE,
                message,
                msg -> {
                    msg.getMessageProperties().setPriority(priority);
                    return msg;
                }
        );
    }
}
```

### Request-Reply Pattern

```java
@Service
public class RpcService {
    
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    public String sendAndReceive(String request) {
        String response = (String) rabbitTemplate.convertSendAndReceive(
                "rpc.exchange",
                "rpc.request",
                request
        );
        return response;
    }
}

@Component
public class RpcServer {
    
    @RabbitListener(queues = "rpc.queue")
    public String handleRequest(String request) {
        System.out.println("RPC Request: " + request);
        return "Processed: " + request.toUpperCase();
    }
}
```

### Delayed Message Queue

```java
@Configuration
public class DelayedQueueConfig {
    
    public static final String DELAYED_QUEUE = "delayed.queue";
    public static final String DELAYED_EXCHANGE = "delayed.exchange";
    
    @Bean
    public CustomExchange delayedExchange() {
        Map<String, Object> args = new HashMap<>();
        args.put("x-delayed-type", "direct");
        return new CustomExchange(DELAYED_EXCHANGE, "x-delayed-message", true, false, args);
    }
    
    @Bean
    public Queue delayedQueue() {
        return new Queue(DELAYED_QUEUE, true);
    }
    
    @Bean
    public Binding delayedBinding(Queue delayedQueue, CustomExchange delayedExchange) {
        return BindingBuilder.bind(delayedQueue)
                .to(delayedExchange)
                .with("delayed")
                .noargs();
    }
}

@Service
public class DelayedProducer {
    
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    public void sendDelayedMessage(String message, int delayMs) {
        rabbitTemplate.convertAndSend(
                DelayedQueueConfig.DELAYED_EXCHANGE,
                "delayed",
                message,
                msg -> {
                    msg.getMessageProperties().setDelay(delayMs);
                    return msg;
                }
        );
    }
}
```

## Message Models

### DTOs and Serialization

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class OrderMessage {
    private Long orderId;
    private String customerId;
    private BigDecimal amount;
    private LocalDateTime timestamp;
    private String status;
}

@Service
public class OrderMessagingService {
    
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    public void publishOrder(OrderMessage order) {
        rabbitTemplate.convertAndSend(
                "orders.exchange",
                "order.created",
                order
        );
    }
}

@Component
public class OrderMessageConsumer {
    
    @RabbitListener(queues = "orders.queue")
    public void handleOrder(OrderMessage order) {
        System.out.println("Processing order: " + order.getOrderId());
        // Business logic
    }
}
```

### Custom Message Converter

```java
@Configuration
public class MessageConverterConfig {
    
    @Bean
    public MessageConverter messageConverter(ObjectMapper objectMapper) {
        Jackson2JsonMessageConverter converter = new Jackson2JsonMessageConverter(objectMapper);
        converter.setClassMapper(classMapper());
        return converter;
    }
    
    @Bean
    public DefaultClassMapper classMapper() {
        DefaultClassMapper classMapper = new DefaultClassMapper();
        Map<String, Class<?>> idClassMapping = new HashMap<>();
        idClassMapping.put("order", OrderMessage.class);
        idClassMapping.put("payment", PaymentMessage.class);
        classMapper.setIdClassMapping(idClassMapping);
        return classMapper;
    }
}
```

## Error Handling and Retry

### Custom Error Handler

```java
@Configuration
public class ErrorHandlingConfig {
    
    @Bean
    public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory(
            ConnectionFactory connectionFactory,
            MessageConverter messageConverter) {
        
        SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
        factory.setConnectionFactory(connectionFactory);
        factory.setMessageConverter(messageConverter);
        factory.setErrorHandler(customErrorHandler());
        factory.setAdviceChain(retryInterceptor());
        
        return factory;
    }
    
    @Bean
    public ErrorHandler customErrorHandler() {
        return new ConditionalRejectingErrorHandler(
                new ConditionalRejectingErrorHandler.DefaultExceptionStrategy() {
                    @Override
                    public boolean isFatal(Throwable t) {
                        return !(t.getCause() instanceof BusinessException);
                    }
                }
        );
    }
    
    @Bean
    public RetryOperationsInterceptor retryInterceptor() {
        return RetryInterceptorBuilder.stateless()
                .maxAttempts(3)
                .backOffOptions(1000, 2.0, 10000)
                .recoverer(new RejectAndDontRequeueRecoverer())
                .build();
    }
}
```

### Manual Acknowledgment

```java
@Component
public class ManualAckConsumer {
    
    @RabbitListener(queues = "manual.ack.queue", ackMode = "MANUAL")
    public void handleMessage(String message, Channel channel, 
                             @Header(AmqpHeaders.DELIVERY_TAG) long tag) {
        try {
            // Process message
            System.out.println("Processing: " + message);
            
            // Manual acknowledgment
            channel.basicAck(tag, false);
        } catch (Exception e) {
            try {
                // Reject and requeue
                channel.basicNack(tag, false, true);
            } catch (IOException ioException) {
                ioException.printStackTrace();
            }
        }
    }
}
```

## Monitoring and Management

### Admin Operations

```java
@Service
public class RabbitMQAdminService {
    
    @Autowired
    private AmqpAdmin amqpAdmin;
    
    public void createQueue(String queueName) {
        Queue queue = new Queue(queueName, true);
        amqpAdmin.declareQueue(queue);
    }
    
    public void deleteQueue(String queueName) {
        amqpAdmin.deleteQueue(queueName);
    }
    
    public void purgeQueue(String queueName) {
        amqpAdmin.purgeQueue(queueName);
    }
    
    public Properties getQueueProperties(String queueName) {
        return amqpAdmin.getQueueProperties(queueName);
    }
}
```

### Health Check

```java
@Component
public class RabbitMQHealthIndicator implements HealthIndicator {
    
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    @Override
    public Health health() {
        try {
            rabbitTemplate.execute(channel -> {
                channel.queueDeclarePassive("health.check.queue");
                return null;
            });
            return Health.up().withDetail("rabbitmq", "Available").build();
        } catch (Exception e) {
            return Health.down().withException(e).build();
        }
    }
}
```

## Testing

```java
@SpringBootTest
@TestPropertySource(properties = {
    "spring.rabbitmq.host=localhost",
    "spring.rabbitmq.port=5672"
})
class RabbitMQTest {
    
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    @Autowired
    private AmqpAdmin amqpAdmin;
    
    @Test
    void testMessageSendAndReceive() throws InterruptedException {
        String queueName = "test.queue";
        Queue queue = new Queue(queueName, false, false, true);
        amqpAdmin.declareQueue(queue);
        
        String message = "Test Message";
        rabbitTemplate.convertAndSend(queueName, message);
        
        Thread.sleep(1000);
        
        String received = (String) rabbitTemplate.receiveAndConvert(queueName);
        assertEquals(message, received);
    }
}
```

## Docker Compose Setup

```yaml
version: '3.8'
services:
  rabbitmq:
    image: rabbitmq:3-management-alpine
    container_name: rabbitmq
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASS: guest
    volumes:
      - rabbitmq-data:/var/lib/rabbitmq

volumes:
  rabbitmq-data:
```

## Best Practices

1. **Use appropriate exchange types** - Match pattern to use case
2. **Implement DLQ** - Handle failed messages
3. **Set message TTL** - Prevent queue bloat
4. **Use durable queues** - For important messages
5. **Implement retry logic** - With exponential backoff
6. **Monitor queue depth** - Prevent memory issues
7. **Use manual acknowledgment** - For critical processing
8. **Implement idempotency** - Handle duplicate messages

## Common Pitfalls

1. Not handling connection failures
2. Forgetting to acknowledge messages
3. Creating too many connections
4. Not setting message expiration
5. Ignoring dead letter queues
6. Not monitoring queue sizes
7. Blocking operations in consumers

## Resources

- [RabbitMQ Documentation](https://www.rabbitmq.com/documentation.html)
- [Spring AMQP Documentation](https://docs.spring.io/spring-amqp/reference/html/)
- [RabbitMQ Tutorials](https://www.rabbitmq.com/getstarted.html)
