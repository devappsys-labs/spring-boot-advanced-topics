# Apache Kafka

## Overview

Apache Kafka is a distributed event streaming platform capable of handling trillions of events a day. It provides high-throughput, low-latency platform for handling real-time data feeds, making it ideal for building real-time streaming data pipelines and applications.

## Key Concepts

### Core Components

- **Producer** - Publishes messages to Kafka topics
- **Consumer** - Subscribes to topics and processes messages
- **Broker** - Kafka server that stores and serves data
- **Topic** - Category or feed name to which records are published
- **Partition** - Ordered, immutable sequence of records
- **Consumer Group** - Group of consumers working together
- **Offset** - Unique identifier of a record within a partition
- **Zookeeper/KRaft** - Cluster coordination (KRaft is newer)

### Key Features

- **High Throughput** - Millions of messages per second
- **Scalability** - Horizontal scaling through partitions
- **Durability** - Persistent storage with replication
- **Fault Tolerance** - Built-in redundancy
- **Real-time Processing** - Low latency message delivery

## Dependencies

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.kafka</groupId>
        <artifactId>spring-kafka</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.kafka</groupId>
        <artifactId>spring-kafka-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
    </dependency>
</dependencies>
```

## Configuration

### Application Properties

```properties
# Kafka Producer Configuration
spring.kafka.bootstrap-servers=localhost:9092
spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.value-serializer=org.springframework.kafka.support.serializer.JsonSerializer
spring.kafka.producer.acks=all
spring.kafka.producer.retries=3
spring.kafka.producer.properties.max.in.flight.requests.per.connection=5
spring.kafka.producer.properties.enable.idempotence=true
spring.kafka.producer.compression-type=snappy

# Kafka Consumer Configuration
spring.kafka.consumer.bootstrap-servers=localhost:9092
spring.kafka.consumer.group-id=default-group
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer=org.springframework.kafka.support.serializer.JsonDeserializer
spring.kafka.consumer.auto-offset-reset=earliest
spring.kafka.consumer.enable-auto-commit=false
spring.kafka.consumer.properties.spring.json.trusted.packages=*

# Kafka Listener Configuration
spring.kafka.listener.ack-mode=manual
spring.kafka.listener.concurrency=3
spring.kafka.listener.poll-timeout=3000
```

### Kafka Configuration Class

```java
@Configuration
@EnableKafka
public class KafkaConfig {
    
    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServers;
    
    // Producer Configuration
    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> configProps = new HashMap<>();
        configProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        configProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        configProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        configProps.put(ProducerConfig.ACKS_CONFIG, "all");
        configProps.put(ProducerConfig.RETRIES_CONFIG, 3);
        configProps.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        configProps.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy");
        return new DefaultKafkaProducerFactory<>(configProps);
    }
    
    @Bean
    public KafkaTemplate<String, Object> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
    
    // Consumer Configuration
    @Bean
    public ConsumerFactory<String, Object> consumerFactory() {
        Map<String, Object> configProps = new HashMap<>();
        configProps.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        configProps.put(ConsumerConfig.GROUP_ID_CONFIG, "default-group");
        configProps.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        configProps.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);
        configProps.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        configProps.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        configProps.put(JsonDeserializer.TRUSTED_PACKAGES, "*");
        return new DefaultKafkaConsumerFactory<>(configProps);
    }
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, Object> factory =
                new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        factory.setConcurrency(3);
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL);
        factory.getContainerProperties().setPollTimeout(3000);
        return factory;
    }
}
```

## Basic Producer

### Simple Producer

```java
@Service
public class KafkaProducerService {
    
    private static final Logger logger = LoggerFactory.getLogger(KafkaProducerService.class);
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    public void sendMessage(String topic, String message) {
        kafkaTemplate.send(topic, message);
        logger.info("Sent message: {} to topic: {}", message, topic);
    }
    
    public void sendMessageWithKey(String topic, String key, String message) {
        kafkaTemplate.send(topic, key, message);
        logger.info("Sent message with key: {} to topic: {}", key, topic);
    }
    
    public void sendMessageWithCallback(String topic, String message) {
        CompletableFuture<SendResult<String, Object>> future = kafkaTemplate.send(topic, message);
        
        future.whenComplete((result, ex) -> {
            if (ex == null) {
                logger.info("Sent message=[{}] with offset=[{}]",
                        message,
                        result.getRecordMetadata().offset());
            } else {
                logger.error("Unable to send message=[{}] due to : {}", message, ex.getMessage());
            }
        });
    }
}
```

### Typed Producer

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class OrderEvent {
    private Long orderId;
    private String customerId;
    private BigDecimal amount;
    private LocalDateTime timestamp;
    private String status;
}

@Service
public class OrderEventProducer {
    
    private static final String TOPIC = "order-events";
    
    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;
    
    public void publishOrderCreated(OrderEvent order) {
        kafkaTemplate.send(TOPIC, order.getOrderId().toString(), order);
    }
    
    public void publishOrderWithPartition(OrderEvent order, int partition) {
        kafkaTemplate.send(TOPIC, partition, order.getOrderId().toString(), order);
    }
}
```

### Producer with Headers

```java
@Service
public class KafkaProducerWithHeaders {
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    public void sendWithHeaders(String topic, String key, Object message, Map<String, String> headers) {
        ProducerRecord<String, Object> record = new ProducerRecord<>(topic, key, message);
        
        headers.forEach((headerKey, headerValue) -> {
            record.headers().add(headerKey, headerValue.getBytes());
        });
        
        kafkaTemplate.send(record);
    }
}
```

## Basic Consumer

### Simple Consumer

```java
@Component
public class KafkaConsumerService {
    
    private static final Logger logger = LoggerFactory.getLogger(KafkaConsumerService.class);
    
    @KafkaListener(topics = "simple-topic", groupId = "simple-group")
    public void consume(String message) {
        logger.info("Consumed message: {}", message);
    }
    
    @KafkaListener(topics = "simple-topic", groupId = "simple-group")
    public void consumeWithMetadata(
            @Payload String message,
            @Header(KafkaHeaders.RECEIVED_TOPIC) String topic,
            @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
            @Header(KafkaHeaders.OFFSET) long offset) {
        
        logger.info("Received message: {} from topic: {}, partition: {}, offset: {}",
                message, topic, partition, offset);
    }
}
```

### Typed Consumer

```java
@Component
public class OrderEventConsumer {
    
    private static final Logger logger = LoggerFactory.getLogger(OrderEventConsumer.class);
    
    @KafkaListener(topics = "order-events", groupId = "order-processing-group")
    public void consumeOrderEvent(OrderEvent order) {
        logger.info("Processing order: {}", order.getOrderId());
        // Business logic
    }
    
    @KafkaListener(
            topics = "order-events",
            groupId = "order-audit-group",
            containerFactory = "kafkaListenerContainerFactory"
    )
    public void auditOrderEvent(
            @Payload OrderEvent order,
            @Header(KafkaHeaders.RECEIVED_KEY) String key,
            Acknowledgment acknowledgment) {
        
        logger.info("Auditing order: {} with key: {}", order.getOrderId(), key);
        
        try {
            // Process message
            acknowledgment.acknowledge();
        } catch (Exception e) {
            logger.error("Error processing message", e);
            // Don't acknowledge - message will be reprocessed
        }
    }
}
```

### Multiple Topic Consumer

```java
@Component
public class MultiTopicConsumer {
    
    @KafkaListener(topics = {"topic1", "topic2", "topic3"}, groupId = "multi-topic-group")
    public void consumeFromMultipleTopics(
            @Payload String message,
            @Header(KafkaHeaders.RECEIVED_TOPIC) String topic) {
        
        switch (topic) {
            case "topic1":
                handleTopic1(message);
                break;
            case "topic2":
                handleTopic2(message);
                break;
            case "topic3":
                handleTopic3(message);
                break;
        }
    }
    
    private void handleTopic1(String message) {
        System.out.println("Processing Topic 1: " + message);
    }
    
    private void handleTopic2(String message) {
        System.out.println("Processing Topic 2: " + message);
    }
    
    private void handleTopic3(String message) {
        System.out.println("Processing Topic 3: " + message);
    }
}
```

## Advanced Patterns

### Custom Partitioner

```java
public class CustomPartitioner implements Partitioner {
    
    @Override
    public int partition(String topic, Object key, byte[] keyBytes, 
                        Object value, byte[] valueBytes, Cluster cluster) {
        
        List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
        int numPartitions = partitions.size();
        
        if (key == null) {
            return ThreadLocalRandom.current().nextInt(numPartitions);
        }
        
        // Custom partitioning logic based on key
        return Math.abs(key.hashCode()) % numPartitions;
    }
    
    @Override
    public void close() {
    }
    
    @Override
    public void configure(Map<String, ?> configs) {
    }
}

@Configuration
public class CustomPartitionerConfig {
    
    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> configProps = new HashMap<>();
        // ... other configs
        configProps.put(ProducerConfig.PARTITIONER_CLASS_CONFIG, CustomPartitioner.class);
        return new DefaultKafkaProducerFactory<>(configProps);
    }
}
```

### Batch Consumer

```java
@Component
public class BatchConsumer {
    
    private static final Logger logger = LoggerFactory.getLogger(BatchConsumer.class);
    
    @KafkaListener(
            topics = "batch-topic",
            groupId = "batch-group",
            containerFactory = "batchKafkaListenerContainerFactory"
    )
    public void consumeBatch(List<OrderEvent> orders) {
        logger.info("Received batch of {} orders", orders.size());
        
        orders.forEach(order -> {
            logger.info("Processing order: {}", order.getOrderId());
            // Batch processing logic
        });
    }
}

@Configuration
public class BatchConsumerConfig {
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, OrderEvent> batchKafkaListenerContainerFactory(
            ConsumerFactory<String, OrderEvent> consumerFactory) {
        
        ConcurrentKafkaListenerContainerFactory<String, OrderEvent> factory =
                new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);
        factory.setBatchListener(true);
        factory.setConcurrency(3);
        
        return factory;
    }
}
```

### Error Handling

```java
@Configuration
public class KafkaErrorHandlingConfig {
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> kafkaListenerContainerFactory(
            ConsumerFactory<String, Object> consumerFactory) {
        
        ConcurrentKafkaListenerContainerFactory<String, Object> factory =
                new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);
        factory.setCommonErrorHandler(errorHandler());
        
        return factory;
    }
    
    @Bean
    public DefaultErrorHandler errorHandler() {
        BackOff fixedBackOff = new FixedBackOff(1000L, 3L);
        
        DefaultErrorHandler errorHandler = new DefaultErrorHandler(
                (consumerRecord, exception) -> {
                    // Recovery logic - send to DLQ
                    System.err.println("Failed to process: " + consumerRecord.value());
                    System.err.println("Exception: " + exception.getMessage());
                },
                fixedBackOff
        );
        
        // Don't retry on these exceptions
        errorHandler.addNotRetryableExceptions(IllegalArgumentException.class);
        
        return errorHandler;
    }
}
```

### Dead Letter Topic (DLT)

```java
@Configuration
public class DeadLetterTopicConfig {
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> kafkaListenerContainerFactory(
            ConsumerFactory<String, Object> consumerFactory,
            KafkaTemplate<String, Object> kafkaTemplate) {
        
        ConcurrentKafkaListenerContainerFactory<String, Object> factory =
                new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);
        factory.setCommonErrorHandler(deadLetterErrorHandler(kafkaTemplate));
        
        return factory;
    }
    
    @Bean
    public DefaultErrorHandler deadLetterErrorHandler(KafkaTemplate<String, Object> kafkaTemplate) {
        DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(kafkaTemplate,
                (record, ex) -> {
                    // DLT topic name pattern
                    return new TopicPartition(record.topic() + ".DLT", record.partition());
                });
        
        return new DefaultErrorHandler(recoverer, new FixedBackOff(1000L, 3L));
    }
}

@Component
public class DeadLetterTopicConsumer {
    
    @KafkaListener(topics = "order-events.DLT", groupId = "dlt-group")
    public void consumeDeadLetterTopic(
            @Payload String message,
            @Header(KafkaHeaders.EXCEPTION_MESSAGE) String exceptionMessage) {
        
        System.err.println("DLT Message: " + message);
        System.err.println("Exception: " + exceptionMessage);
        
        // Log to monitoring system, send alerts, etc.
    }
}
```

### Kafka Streams

```java
@Configuration
@EnableKafkaStreams
public class KafkaStreamsConfig {
    
    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServers;
    
    @Bean(name = KafkaStreamsDefaultConfiguration.DEFAULT_STREAMS_CONFIG_BEAN_NAME)
    public KafkaStreamsConfiguration kStreamsConfig() {
        Map<String, Object> props = new HashMap<>();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "streams-app");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        
        return new KafkaStreamsConfiguration(props);
    }
}

@Component
public class KafkaStreamsProcessor {
    
    @Autowired
    public void processStream(StreamsBuilder streamsBuilder) {
        KStream<String, String> stream = streamsBuilder.stream("input-topic");
        
        stream
                .filter((key, value) -> value.length() > 5)
                .mapValues(value -> value.toUpperCase())
                .to("output-topic");
    }
    
    @Autowired
    public void aggregateStream(StreamsBuilder streamsBuilder) {
        KStream<String, OrderEvent> orderStream = streamsBuilder.stream("order-events");
        
        KTable<String, Long> orderCountByCustomer = orderStream
                .groupBy((key, order) -> order.getCustomerId())
                .count(Materialized.as("order-count-store"));
        
        orderCountByCustomer.toStream().to("order-count-topic");
    }
}
```

### Transactional Producer

```java
@Configuration
public class TransactionalKafkaConfig {
    
    @Bean
    public ProducerFactory<String, Object> transactionalProducerFactory() {
        Map<String, Object> configProps = new HashMap<>();
        configProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        configProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        configProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        configProps.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "tx-");
        configProps.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        
        DefaultKafkaProducerFactory<String, Object> factory = 
                new DefaultKafkaProducerFactory<>(configProps);
        factory.setTransactionIdPrefix("tx-");
        
        return factory;
    }
    
    @Bean
    public KafkaTemplate<String, Object> transactionalKafkaTemplate() {
        return new KafkaTemplate<>(transactionalProducerFactory());
    }
}

@Service
public class TransactionalService {
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    @Transactional
    public void sendTransactional(String topic, String key, Object message) {
        kafkaTemplate.executeInTransaction(operations -> {
            operations.send(topic, key, message);
            operations.send(topic + "-audit", key, message);
            return true;
        });
    }
}
```

## Topic Management

### Admin Client

```java
@Configuration
public class KafkaAdminConfig {
    
    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServers;
    
    @Bean
    public KafkaAdmin kafkaAdmin() {
        Map<String, Object> configs = new HashMap<>();
        configs.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        return new KafkaAdmin(configs);
    }
    
    @Bean
    public NewTopic orderEventsTopic() {
        return TopicBuilder.name("order-events")
                .partitions(3)
                .replicas(1)
                .config(TopicConfig.RETENTION_MS_CONFIG, "604800000") // 7 days
                .build();
    }
    
    @Bean
    public NewTopic compactedTopic() {
        return TopicBuilder.name("user-profiles")
                .partitions(3)
                .replicas(1)
                .config(TopicConfig.CLEANUP_POLICY_CONFIG, TopicConfig.CLEANUP_POLICY_COMPACT)
                .build();
    }
}

@Service
public class KafkaAdminService {
    
    @Autowired
    private AdminClient adminClient;
    
    public void createTopic(String topicName, int partitions, short replicationFactor) {
        NewTopic newTopic = new NewTopic(topicName, partitions, replicationFactor);
        
        try {
            adminClient.createTopics(Collections.singleton(newTopic)).all().get();
            System.out.println("Topic created: " + topicName);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    
    public void deleteTopic(String topicName) {
        try {
            adminClient.deleteTopics(Collections.singleton(topicName)).all().get();
            System.out.println("Topic deleted: " + topicName);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    
    public void listTopics() {
        try {
            Set<String> topics = adminClient.listTopics().names().get();
            topics.forEach(System.out::println);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

## Consumer Group Management

### Manual Offset Management

```java
@Component
public class ManualOffsetConsumer {
    
    private static final Logger logger = LoggerFactory.getLogger(ManualOffsetConsumer.class);
    
    @KafkaListener(
            topics = "manual-offset-topic",
            groupId = "manual-offset-group",
            containerFactory = "manualOffsetContainerFactory"
    )
    public void consume(
            ConsumerRecord<String, String> record,
            Acknowledgment acknowledgment) {
        
        try {
            logger.info("Processing: {}, offset: {}", record.value(), record.offset());
            
            // Process message
            
            // Manually commit offset
            acknowledgment.acknowledge();
            
        } catch (Exception e) {
            logger.error("Error processing message", e);
            // Don't acknowledge - message will be reprocessed
        }
    }
}

@Configuration
public class ManualOffsetConfig {
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String> manualOffsetContainerFactory(
            ConsumerFactory<String, String> consumerFactory) {
        
        ConcurrentKafkaListenerContainerFactory<String, String> factory =
                new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL);
        
        return factory;
    }
}
```

### Seeking to Specific Offset

```java
@Component
public class SeekingConsumer implements ConsumerSeekAware {
    
    private static final Logger logger = LoggerFactory.getLogger(SeekingConsumer.class);
    
    @Override
    public void onPartitionsAssigned(Map<TopicPartition, Long> assignments, ConsumerSeekCallback callback) {
        // Seek to beginning
        assignments.keySet().forEach(callback::seekToBeginning);
        
        // Or seek to specific offset
        assignments.forEach((tp, offset) -> callback.seek(tp.topic(), tp.partition(), 100));
    }
    
    @KafkaListener(topics = "seeking-topic", groupId = "seeking-group")
    public void consume(String message) {
        logger.info("Consumed: {}", message);
    }
}
```

## Monitoring and Metrics

### Custom Metrics

```java
@Component
public class KafkaMetrics {
    
    private final Counter messagesProcessed;
    private final Timer processingTime;
    
    public KafkaMetrics(MeterRegistry registry) {
        this.messagesProcessed = Counter.builder("kafka.messages.processed")
                .description("Total messages processed")
                .register(registry);
        
        this.processingTime = Timer.builder("kafka.processing.time")
                .description("Message processing time")
                .register(registry);
    }
    
    @KafkaListener(topics = "metrics-topic", groupId = "metrics-group")
    public void consume(String message) {
        processingTime.record(() -> {
            // Process message
            System.out.println("Processing: " + message);
            messagesProcessed.increment();
        });
    }
}
```

## Testing

```java
@SpringBootTest
@EmbeddedKafka(
        partitions = 1,
        brokerProperties = {
                "listeners=PLAINTEXT://localhost:9092",
                "port=9092"
        }
)
class KafkaIntegrationTest {
    
    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;
    
    @Autowired
    private EmbeddedKafkaBroker embeddedKafkaBroker;
    
    @Test
    void testSendAndReceive() throws Exception {
        String topic = "test-topic";
        String message = "Test Message";
        
        // Send message
        kafkaTemplate.send(topic, message).get();
        
        // Create consumer
        Map<String, Object> consumerProps = KafkaTestUtils.consumerProps(
                "test-group", "true", embeddedKafkaBroker);
        
        ConsumerFactory<String, String> cf = new DefaultKafkaConsumerFactory<>(consumerProps);
        Consumer<String, String> consumer = cf.createConsumer();
        
        embeddedKafkaBroker.consumeFromAnEmbeddedTopic(consumer, topic);
        
        // Receive message
        ConsumerRecords<String, String> records = KafkaTestUtils.getRecords(consumer);
        
        assertEquals(1, records.count());
        assertEquals(message, records.iterator().next().value());
        
        consumer.close();
    }
}
```

## Docker Compose Setup

```yaml
version: '3.8'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - "2181:2181"
  
  kafka:
    image: confluentinc/cp-kafka:latest
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
  
  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    depends_on:
      - kafka
    ports:
      - "8080:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
```

## Best Practices

1. **Use appropriate number of partitions** - Balance parallelism and overhead
2. **Set proper retention policies** - Based on use case
3. **Enable idempotence** - For exactly-once semantics
4. **Use consumer groups** - For scalability
5. **Implement proper error handling** - With DLT
6. **Monitor consumer lag** - Prevent backlog
7. **Use compression** - Reduce network and storage
8. **Implement graceful shutdown** - Properly close resources

## Common Pitfalls

1. Not handling rebalances properly
2. Ignoring consumer lag
3. Improper partition key selection
4. Not setting appropriate timeouts
5. Blocking operations in consumers
6. Not monitoring broker health
7. Inadequate error handling

## Resources

- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [Spring for Apache Kafka](https://spring.io/projects/spring-kafka)
- [Confluent Documentation](https://docs.confluent.io/)
