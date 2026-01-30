
# SSE/WebSocket with WebFlux

## Overview

Server-Sent Events (SSE) and WebSockets enable real-time, bidirectional communication between clients and servers. Spring WebFlux provides reactive support for both protocols, allowing you to build scalable, event-driven applications with backpressure handling.

## Key Concepts

### Server-Sent Events (SSE)

- **Unidirectional** - Server to client only
- **HTTP-based** - Uses standard HTTP protocol
- **Automatic reconnection** - Built into browsers
- **Text-based** - UTF-8 encoded data
- **Event streaming** - Continuous data flow

### WebSocket

- **Bidirectional** - Full-duplex communication
- **Persistent connection** - Long-lived connection
- **Binary/Text support** - Flexible data formats
- **Low latency** - Minimal overhead
- **Custom protocols** - STOMP, WAMP, etc.

### WebFlux Reactive Programming

- **Non-blocking** - Asynchronous processing
- **Backpressure** - Flow control
- **Mono** - 0 or 1 element
- **Flux** - 0 to N elements

## Dependencies

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webflux</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-websocket</artifactId>
    </dependency>
    <dependency>
        <groupId>io.projectreactor</groupId>
        <artifactId>reactor-core</artifactId>
    </dependency>
</dependencies>
```

## Server-Sent Events (SSE)

### Basic SSE Controller

```java
@RestController
@RequestMapping("/sse")
public class SSEController {
    
    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> streamEvents() {
        return Flux.interval(Duration.ofSeconds(1))
                .map(sequence -> "Event " + sequence + " at " + LocalDateTime.now());
    }
    
    @GetMapping(value = "/numbers", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<Integer> streamNumbers() {
        return Flux.range(1, 100)
                .delayElements(Duration.ofMillis(500));
    }
}
```

### SSE with ServerSentEvent

```java
@RestController
@RequestMapping("/sse")
public class AdvancedSSEController {
    
    @GetMapping(value = "/events", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<String>> streamServerSentEvents() {
        return Flux.interval(Duration.ofSeconds(1))
                .map(sequence -> ServerSentEvent.<String>builder()
                        .id(String.valueOf(sequence))
                        .event("periodic-event")
                        .data("Event data " + sequence)
                        .comment("This is event " + sequence)
                        .build());
    }
    
    @GetMapping(value = "/stock-prices", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<StockPrice>> streamStockPrices() {
        return Flux.interval(Duration.ofSeconds(2))
                .map(seq -> {
                    StockPrice price = new StockPrice(
                            "AAPL",
                            150.0 + Math.random() * 10,
                            LocalDateTime.now()
                    );
                    
                    return ServerSentEvent.<StockPrice>builder()
                            .id(String.valueOf(seq))
                            .event("stock-update")
                            .data(price)
                            .build();
                });
    }
}

@Data
@AllArgsConstructor
class StockPrice {
    private String symbol;
    private Double price;
    private LocalDateTime timestamp;
}
```

### SSE Service with Sink

```java
@Service
public class NotificationService {
    
    private final Sinks.Many<String> sink;
    private final Flux<String> notificationFlux;
    
    public NotificationService() {
        this.sink = Sinks.many().multicast().onBackpressureBuffer();
        this.notificationFlux = sink.asFlux();
    }
    
    public void sendNotification(String message) {
        sink.tryEmitNext(message);
    }
    
    public Flux<String> getNotificationStream() {
        return notificationFlux;
    }
}

@RestController
@RequestMapping("/notifications")
public class NotificationController {
    
    @Autowired
    private NotificationService notificationService;
    
    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<String>> streamNotifications() {
        return notificationService.getNotificationStream()
                .map(notification -> ServerSentEvent.<String>builder()
                        .data(notification)
                        .build());
    }
    
    @PostMapping("/send")
    public Mono<String> sendNotification(@RequestBody String message) {
        notificationService.sendNotification(message);
        return Mono.just("Notification sent");
    }
}
```

### SSE with Error Handling

```java
@RestController
@RequestMapping("/sse")
public class RobustSSEController {
    
    @GetMapping(value = "/resilient", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<String>> resilientStream() {
        return Flux.interval(Duration.ofSeconds(1))
                .map(seq -> {
                    if (seq % 10 == 0 && seq != 0) {
                        throw new RuntimeException("Simulated error at sequence " + seq);
                    }
                    return "Event " + seq;
                })
                .onErrorResume(error -> {
                    System.err.println("Error occurred: " + error.getMessage());
                    return Flux.just("Error: " + error.getMessage());
                })
                .map(data -> ServerSentEvent.<String>builder()
                        .data(data)
                        .build())
                .doOnCancel(() -> System.out.println("Client disconnected"))
                .doFinally(signalType -> System.out.println("Stream ended: " + signalType));
    }
}
```

## WebSocket

### Basic WebSocket Handler

```java
@Component
public class SimpleWebSocketHandler extends TextWebSocketHandler {
    
    private final List<WebSocketSession> sessions = new CopyOnWriteArrayList<>();
    
    @Override
    public void afterConnectionEstablished(WebSocketSession session) throws Exception {
        sessions.add(session);
        System.out.println("New connection: " + session.getId());
    }
    
    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) throws Exception {
        String payload = message.getPayload();
        System.out.println("Received: " + payload);
        
        // Echo back to sender
        session.sendMessage(new TextMessage("Echo: " + payload));
        
        // Broadcast to all
        for (WebSocketSession s : sessions) {
            if (s.isOpen()) {
                s.sendMessage(new TextMessage("Broadcast: " + payload));
            }
        }
    }
    
    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) throws Exception {
        sessions.remove(session);
        System.out.println("Connection closed: " + session.getId());
    }
    
    @Override
    public void handleTransportError(WebSocketSession session, Throwable exception) throws Exception {
        System.err.println("Transport error: " + exception.getMessage());
        session.close();
    }
}

@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {
    
    @Autowired
    private SimpleWebSocketHandler simpleWebSocketHandler;
    
    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(simpleWebSocketHandler, "/ws/simple")
                .setAllowedOrigins("*");
    }
}
```

### Reactive WebSocket Handler

```java
@Component
public class ReactiveWebSocketHandler implements WebSocketHandler {
    
    @Override
    public Mono<Void> handle(WebSocketSession session) {
        Flux<WebSocketMessage> output = Flux.interval(Duration.ofSeconds(1))
                .map(seq -> "Server time: " + LocalDateTime.now())
                .map(session::textMessage);
        
        Flux<String> input = session.receive()
                .map(WebSocketMessage::getPayloadAsText)
                .doOnNext(message -> System.out.println("Received: " + message));
        
        return session.send(output)
                .and(input);
    }
}

@Configuration
public class ReactiveWebSocketConfig {
    
    @Bean
    public HandlerMapping webSocketHandlerMapping(ReactiveWebSocketHandler handler) {
        Map<String, WebSocketHandler> map = new HashMap<>();
        map.put("/ws/reactive", handler);
        
        SimpleUrlHandlerMapping handlerMapping = new SimpleUrlHandlerMapping();
        handlerMapping.setOrder(1);
        handlerMapping.setUrlMap(map);
        
        return handlerMapping;
    }
    
    @Bean
    public WebSocketHandlerAdapter handlerAdapter() {
        return new WebSocketHandlerAdapter();
    }
}
```

### Chat Application with WebSocket

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
class ChatMessage {
    private String username;
    private String content;
    private LocalDateTime timestamp;
}

@Component
public class ChatWebSocketHandler implements WebSocketHandler {
    
    private final Map<String, WebSocketSession> sessions = new ConcurrentHashMap<>();
    private final ObjectMapper objectMapper = new ObjectMapper();
    
    @Override
    public Mono<Void> handle(WebSocketSession session) {
        String sessionId = session.getId();
        sessions.put(sessionId, session);
        
        Flux<WebSocketMessage> output = session.receive()
                .map(WebSocketMessage::getPayloadAsText)
                .flatMap(payload -> {
                    try {
                        ChatMessage message = objectMapper.readValue(payload, ChatMessage.class);
                        message.setTimestamp(LocalDateTime.now());
                        
                        String broadcastMessage = objectMapper.writeValueAsString(message);
                        
                        return Flux.fromIterable(sessions.values())
                                .filter(s -> s.isOpen())
                                .map(s -> s.textMessage(broadcastMessage));
                    } catch (Exception e) {
                        return Flux.empty();
                    }
                })
                .doFinally(signalType -> sessions.remove(sessionId));
        
        return session.send(output);
    }
}

@Configuration
public class ChatWebSocketConfig {
    
    @Bean
    public HandlerMapping chatWebSocketHandlerMapping(ChatWebSocketHandler handler) {
        Map<String, WebSocketHandler> map = new HashMap<>();
        map.put("/ws/chat", handler);
        
        SimpleUrlHandlerMapping handlerMapping = new SimpleUrlHandlerMapping();
        handlerMapping.setOrder(1);
        handlerMapping.setUrlMap(map);
        
        return handlerMapping;
    }
}
```

### STOMP Over WebSocket

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketMessageBrokerConfig implements WebSocketMessageBrokerConfigurer {
    
    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.enableSimpleBroker("/topic", "/queue");
        config.setApplicationDestinationPrefixes("/app");
    }
    
    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws/stomp")
                .setAllowedOrigins("*")
                .withSockJS();
    }
}

@Controller
public class StompMessageController {
    
    @MessageMapping("/chat.send")
    @SendTo("/topic/messages")
    public ChatMessage sendMessage(ChatMessage message) {
        message.setTimestamp(LocalDateTime.now());
        return message;
    }
    
    @MessageMapping("/chat.private")
    public void sendPrivateMessage(@Payload ChatMessage message,
                                   @DestinationVariable String username) {
        messagingTemplate.convertAndSendToUser(
                username,
                "/queue/private",
                message
        );
    }
    
    @Autowired
    private SimpMessagingTemplate messagingTemplate;
    
    @MessageMapping("/chat.join")
    @SendTo("/topic/join")
    public String userJoin(@Payload String username) {
        return username + " joined the chat";
    }
}
```

## Advanced Patterns

### Real-time Data Feed

```java
@Service
public class RealTimeDataService {
    
    public Flux<MarketData> getMarketDataStream() {
        return Flux.interval(Duration.ofMillis(100))
                .map(seq -> new MarketData(
                        "SYMBOL-" + (seq % 10),
                        100.0 + Math.random() * 50,
                        (long) (Math.random() * 1000),
                        LocalDateTime.now()
                ));
    }
}

@Data
@AllArgsConstructor
class MarketData {
    private String symbol;
    private Double price;
    private Long volume;
    private LocalDateTime timestamp;
}

@RestController
@RequestMapping("/market")
public class MarketDataController {
    
    @Autowired
    private RealTimeDataService dataService;
    
    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<MarketData>> streamMarketData() {
        return dataService.getMarketDataStream()
                .map(data -> ServerSentEvent.<MarketData>builder()
                        .id(String.valueOf(data.getTimestamp().toEpochSecond(ZoneOffset.UTC)))
                        .event("market-update")
                        .data(data)
                        .build());
    }
}
```

### Backpressure Handling

```java
@RestController
@RequestMapping("/backpressure")
public class BackpressureController {
    
    @GetMapping(value = "/buffered", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<Integer>> bufferedStream() {
        return Flux.range(1, 1000)
                .delayElements(Duration.ofMillis(10))
                .onBackpressureBuffer(100, i -> System.out.println("Dropped: " + i))
                .map(i -> ServerSentEvent.<Integer>builder()
                        .data(i)
                        .build());
    }
    
    @GetMapping(value = "/dropped", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<Integer>> droppedStream() {
        return Flux.range(1, 1000)
                .delayElements(Duration.ofMillis(10))
                .onBackpressureDrop(i -> System.out.println("Dropped: " + i))
                .map(i -> ServerSentEvent.<Integer>builder()
                        .data(i)
                        .build());
    }
    
    @GetMapping(value = "/latest", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<Integer>> latestStream() {
        return Flux.range(1, 1000)
                .delayElements(Duration.ofMillis(10))
                .onBackpressureLatest()
                .map(i -> ServerSentEvent.<Integer>builder()
                        .data(i)
                        .build());
    }
}
```

### WebSocket with Authentication

```java
@Component
public class AuthenticatedWebSocketHandler implements WebSocketHandler {
    
    @Override
    public Mono<Void> handle(WebSocketSession session) {
        String token = extractToken(session);
        
        if (!isValidToken(token)) {
            return session.close(CloseStatus.NOT_ACCEPTABLE.withReason("Invalid token"));
        }
        
        String userId = getUserIdFromToken(token);
        session.getAttributes().put("userId", userId);
        
        // Handle authenticated session
        return handleAuthenticatedSession(session, userId);
    }
    
    private String extractToken(WebSocketSession session) {
        URI uri = session.getHandshakeInfo().getUri();
        String query = uri.getQuery();
        // Extract token from query parameters
        return query != null && query.contains("token=") 
                ? query.substring(query.indexOf("token=") + 6) 
                : null;
    }
    
    private boolean isValidToken(String token) {
        // Validate JWT or session token
        return token != null && token.length() > 0;
    }
    
    private String getUserIdFromToken(String token) {
        // Extract user ID from token
        return "user-" + token.hashCode();
    }
    
    private Mono<Void> handleAuthenticatedSession(WebSocketSession session, String userId) {
        Flux<WebSocketMessage> output = Flux.interval(Duration.ofSeconds(1))
                .map(seq -> String.format("User %s: Message %d", userId, seq))
                .map(session::textMessage);
        
        return session.send(output);
    }
}
```

### Multiplexing Multiple Streams

```java
@Component
public class MultiplexedWebSocketHandler implements WebSocketHandler {
    
    @Override
    public Mono<Void> handle(WebSocketSession session) {
        Flux<WebSocketMessage> stockPrices = Flux.interval(Duration.ofSeconds(1))
                .map(seq -> createMessage("stocks", "AAPL", 150.0 + seq))
                .map(session::textMessage);
        
        Flux<WebSocketMessage> news = Flux.interval(Duration.ofSeconds(3))
                .map(seq -> createMessage("news", "headline", "Breaking news " + seq))
                .map(session::textMessage);
        
        Flux<WebSocketMessage> multiplexed = Flux.merge(stockPrices, news);
        
        return session.send(multiplexed);
    }
    
    private Map<String, Object> createMessage(String channel, String key, Object value) {
        Map<String, Object> message = new HashMap<>();
        message.put("channel", channel);
        message.put("key", key);
        message.put("value", value);
        message.put("timestamp", System.currentTimeMillis());
        return message;
    }
}
```

## Client Examples

### JavaScript SSE Client

```javascript
const eventSource = new EventSource('http://localhost:8080/sse/stream');

eventSource.onmessage = (event) => {
    console.log('Received:', event.data);
};

eventSource.addEventListener('stock-update', (event) => {
    const data = JSON.parse(event.data);
    console.log('Stock update:', data);
});

eventSource.onerror = (error) => {
    console.error('SSE error:', error);
    eventSource.close();
};

// Close connection
eventSource.close();
```

### JavaScript WebSocket Client

```javascript
const socket = new WebSocket('ws://localhost:8080/ws/chat');

socket.onopen = () => {
    console.log('Connected');
    socket.send(JSON.stringify({
        username: 'John',
        content: 'Hello World'
    }));
};

socket.onmessage = (event) => {
    const message = JSON.parse(event.data);
    console.log('Received:', message);
};

socket.onerror = (error) => {
    console.error('WebSocket error:', error);
};

socket.onclose = () => {
    console.log('Disconnected');
};

// Close connection
socket.close();
```

### STOMP Client

```javascript
const socket = new SockJS('http://localhost:8080/ws/stomp');
const stompClient = Stomp.over(socket);

stompClient.connect({}, (frame) => {
    console.log('Connected:', frame);
    
    // Subscribe to topic
    stompClient.subscribe('/topic/messages', (message) => {
        const data = JSON.parse(message.body);
        console.log('Received:', data);
    });
    
    // Send message
    stompClient.send('/app/chat.send', {}, JSON.stringify({
        username: 'John',
        content: 'Hello'
    }));
});

// Disconnect
stompClient.disconnect();
```

## Testing

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class SSEWebSocketTest {
    
    @LocalServerPort
    private int port;
    
    @Test
    void testSSE() {
        WebTestClient client = WebTestClient
                .bindToServer()
                .baseUrl("http://localhost:" + port)
                .build();
        
        FluxExchangeResult<String> result = client.get()
                .uri("/sse/stream")
                .accept(MediaType.TEXT_EVENT_STREAM)
                .exchange()
                .expectStatus().isOk()
                .returnResult(String.class);
        
        StepVerifier.create(result.getResponseBody().take(3))
                .expectNextCount(3)
                .verifyComplete();
    }
    
    @Test
    void testWebSocket() throws Exception {
        WebSocketClient client = new StandardWebSocketClient();
        
        WebSocketSession session = client.execute(
                new TextWebSocketHandler() {
                    @Override
                    protected void handleTextMessage(WebSocketSession session, TextMessage message) {
                        System.out.println("Received: " + message.getPayload());
                    }
                },
                "ws://localhost:" + port + "/ws/simple"
        ).get();
        
        session.sendMessage(new TextMessage("Test message"));
        Thread.sleep(1000);
        session.close();
    }
}
```

## Best Practices

1. **Handle client disconnections** - Clean up resources properly
2. **Implement backpressure** - Prevent memory issues
3. **Use appropriate buffer sizes** - Balance memory and throughput
4. **Implement heartbeat/ping** - Detect dead connections
5. **Secure WebSocket connections** - Use WSS and authentication
6. **Monitor connection count** - Prevent resource exhaustion
7. **Implement reconnection logic** - Handle network failures
8. **Use appropriate message formats** - JSON for structured data

## Common Pitfalls

1. Not handling backpressure
2. Memory leaks from unclosed streams
3. Blocking operations in reactive handlers
4. Not implementing error handling
5. Ignoring connection limits
6. Not cleaning up resources
7. Improper message serialization

## Resources

- [Spring WebFlux Documentation](https://docs.spring.io/spring-framework/reference/web/webflux.html)
- [Server-Sent Events Specification](https://html.spec.whatwg.org/multipage/server-sent-events.html)
- [WebSocket API](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API)
- [Project Reactor Documentation](https://projectreactor.io/docs)
