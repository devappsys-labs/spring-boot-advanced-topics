# gRPC Server

## Overview

gRPC (Google Remote Procedure Call) is a high-performance, open-source framework for building distributed systems. It uses Protocol Buffers (protobuf) for serialization and HTTP/2 for transport, offering superior performance compared to traditional REST APIs.

## Key Concepts

### Protocol Buffers
- Language-agnostic data serialization
- Strongly typed schema definition
- Automatic code generation
- Compact binary format

### gRPC Communication Patterns
1. **Unary RPC** - Single request, single response
2. **Server Streaming** - Single request, stream of responses
3. **Client Streaming** - Stream of requests, single response
4. **Bidirectional Streaming** - Stream of requests and responses

### Advantages
- High performance with HTTP/2
- Built-in code generation
- Bi-directional streaming
- Pluggable authentication
- Efficient binary serialization

## Spring Boot Integration

### Dependencies

```xml
<dependencies>
    <!-- gRPC Spring Boot Starter -->
    <dependency>
        <groupId>net.devh</groupId>
        <artifactId>grpc-spring-boot-starter</artifactId>
        <version>2.15.0.RELEASE</version>
    </dependency>
    
    <!-- Protobuf -->
    <dependency>
        <groupId>com.google.protobuf</groupId>
        <artifactId>protobuf-java</artifactId>
        <version>3.25.1</version>
    </dependency>
    
    <!-- gRPC Services -->
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-stub</artifactId>
        <version>1.60.0</version>
    </dependency>
    
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-protobuf</artifactId>
        <version>1.60.0</version>
    </dependency>
</dependencies>

<build>
    <extensions>
        <extension>
            <groupId>kr.motd.maven</groupId>
            <artifactId>os-maven-plugin</artifactId>
            <version>1.7.1</version>
        </extension>
    </extensions>
    
    <plugins>
        <plugin>
            <groupId>org.xolstice.maven.plugins</groupId>
            <artifactId>protobuf-maven-plugin</artifactId>
            <version>0.6.1</version>
            <configuration>
                <protocArtifact>
                    com.google.protobuf:protoc:3.25.1:exe:${os.detected.classifier}
                </protocArtifact>
                <pluginId>grpc-java</pluginId>
                <pluginArtifact>
                    io.grpc:protoc-gen-grpc-java:1.60.0:exe:${os.detected.classifier}
                </pluginArtifact>
            </configuration>
            <executions>
                <execution>
                    <goals>
                        <goal>compile</goal>
                        <goal>compile-custom</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

### Gradle Configuration

```gradle
plugins {
    id 'com.google.protobuf' version '0.9.4'
}

dependencies {
    implementation 'net.devh:grpc-spring-boot-starter:2.15.0.RELEASE'
    implementation 'io.grpc:grpc-stub:1.60.0'
    implementation 'io.grpc:grpc-protobuf:1.60.0'
    implementation 'com.google.protobuf:protobuf-java:3.25.1'
}

protobuf {
    protoc {
        artifact = 'com.google.protobuf:protoc:3.25.1'
    }
    plugins {
        grpc {
            artifact = 'io.grpc:protoc-gen-grpc-java:1.60.0'
        }
    }
    generateProtoTasks {
        all()*.plugins {
            grpc {}
        }
    }
}
```

## Protocol Buffer Definition

### Basic Service Definition

**src/main/proto/user_service.proto**

```protobuf
syntax = "proto3";

package com.example.grpc;

option java_multiple_files = true;
option java_package = "com.example.grpc.proto";
option java_outer_classname = "UserServiceProto";

// User message definition
message User {
  int64 id = 1;
  string username = 2;
  string email = 3;
  string first_name = 4;
  string last_name = 5;
  int64 created_at = 6;
}

// Request messages
message GetUserRequest {
  int64 user_id = 1;
}

message CreateUserRequest {
  string username = 1;
  string email = 2;
  string first_name = 3;
  string last_name = 4;
}

message UpdateUserRequest {
  int64 user_id = 1;
  string username = 2;
  string email = 3;
  string first_name = 4;
  string last_name = 5;
}

message DeleteUserRequest {
  int64 user_id = 1;
}

message ListUsersRequest {
  int32 page = 1;
  int32 page_size = 2;
}

// Response messages
message GetUserResponse {
  User user = 1;
  bool found = 2;
}

message CreateUserResponse {
  User user = 1;
  bool success = 2;
  string message = 3;
}

message UpdateUserResponse {
  User user = 1;
  bool success = 2;
}

message DeleteUserResponse {
  bool success = 1;
  string message = 2;
}

message ListUsersResponse {
  repeated User users = 1;
  int32 total_count = 2;
}

// Service definition
service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
  rpc UpdateUser(UpdateUserRequest) returns (UpdateUserResponse);
  rpc DeleteUser(DeleteUserRequest) returns (DeleteUserResponse);
  rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);
  rpc StreamUsers(ListUsersRequest) returns (stream User);
}
```

## Server Implementation

### gRPC Service Implementation

```java
package com.example.grpc.service;

import com.example.grpc.proto.*;
import io.grpc.stub.StreamObserver;
import net.devh.boot.grpc.server.service.GrpcService;
import org.springframework.beans.factory.annotation.Autowired;

import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;

@GrpcService
public class UserGrpcService extends UserServiceGrpc.UserServiceImplBase {

    private final Map<Long, User> userDatabase = new ConcurrentHashMap<>();
    private final AtomicLong idGenerator = new AtomicLong(1);

    @Override
    public void getUser(GetUserRequest request, StreamObserver<GetUserResponse> responseObserver) {
        User user = userDatabase.get(request.getUserId());
        
        GetUserResponse response = GetUserResponse.newBuilder()
                .setUser(user != null ? user : User.getDefaultInstance())
                .setFound(user != null)
                .build();

        responseObserver.onNext(response);
        responseObserver.onCompleted();
    }

    @Override
    public void createUser(CreateUserRequest request, StreamObserver<CreateUserResponse> responseObserver) {
        long userId = idGenerator.getAndIncrement();
        
        User user = User.newBuilder()
                .setId(userId)
                .setUsername(request.getUsername())
                .setEmail(request.getEmail())
                .setFirstName(request.getFirstName())
                .setLastName(request.getLastName())
                .setCreatedAt(Instant.now().toEpochMilli())
                .build();

        userDatabase.put(userId, user);

        CreateUserResponse response = CreateUserResponse.newBuilder()
                .setUser(user)
                .setSuccess(true)
                .setMessage("User created successfully")
                .build();

        responseObserver.onNext(response);
        responseObserver.onCompleted();
    }

    @Override
    public void updateUser(UpdateUserRequest request, StreamObserver<UpdateUserResponse> responseObserver) {
        User existingUser = userDatabase.get(request.getUserId());
        
        if (existingUser == null) {
            UpdateUserResponse response = UpdateUserResponse.newBuilder()
                    .setSuccess(false)
                    .build();
            responseObserver.onNext(response);
            responseObserver.onCompleted();
            return;
        }

        User updatedUser = User.newBuilder()
                .setId(request.getUserId())
                .setUsername(request.getUsername())
                .setEmail(request.getEmail())
                .setFirstName(request.getFirstName())
                .setLastName(request.getLastName())
                .setCreatedAt(existingUser.getCreatedAt())
                .build();

        userDatabase.put(request.getUserId(), updatedUser);

        UpdateUserResponse response = UpdateUserResponse.newBuilder()
                .setUser(updatedUser)
                .setSuccess(true)
                .build();

        responseObserver.onNext(response);
        responseObserver.onCompleted();
    }

    @Override
    public void deleteUser(DeleteUserRequest request, StreamObserver<DeleteUserResponse> responseObserver) {
        User removed = userDatabase.remove(request.getUserId());

        DeleteUserResponse response = DeleteUserResponse.newBuilder()
                .setSuccess(removed != null)
                .setMessage(removed != null ? "User deleted successfully" : "User not found")
                .build();

        responseObserver.onNext(response);
        responseObserver.onCompleted();
    }

    @Override
    public void listUsers(ListUsersRequest request, StreamObserver<ListUsersResponse> responseObserver) {
        List<User> allUsers = new ArrayList<>(userDatabase.values());
        
        int page = request.getPage();
        int pageSize = request.getPageSize() > 0 ? request.getPageSize() : 10;
        int start = page * pageSize;
        int end = Math.min(start + pageSize, allUsers.size());
        
        List<User> pageUsers = start < allUsers.size() 
                ? allUsers.subList(start, end) 
                : new ArrayList<>();

        ListUsersResponse response = ListUsersResponse.newBuilder()
                .addAllUsers(pageUsers)
                .setTotalCount(allUsers.size())
                .build();

        responseObserver.onNext(response);
        responseObserver.onCompleted();
    }

    @Override
    public void streamUsers(ListUsersRequest request, StreamObserver<User> responseObserver) {
        userDatabase.values().forEach(user -> {
            responseObserver.onNext(user);
            try {
                Thread.sleep(100); // Simulate streaming delay
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        responseObserver.onCompleted();
    }
}
```

### Configuration

**application.yml**

```yaml
grpc:
  server:
    port: 9090
    address: 0.0.0.0
    max-inbound-message-size: 4194304
    max-inbound-metadata-size: 8192
    
spring:
  application:
    name: grpc-server-demo

logging:
  level:
    net.devh.boot.grpc: DEBUG
    io.grpc: DEBUG
```

**application.properties**

```properties
grpc.server.port=9090
grpc.server.address=0.0.0.0
grpc.server.max-inbound-message-size=4MB
grpc.server.enable-reflection=true
```

## Advanced Features

### Server Interceptors

```java
package com.example.grpc.interceptor;

import io.grpc.*;
import net.devh.boot.grpc.server.interceptor.GrpcGlobalServerInterceptor;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@GrpcGlobalServerInterceptor
public class LoggingInterceptor implements ServerInterceptor {

    private static final Logger logger = LoggerFactory.getLogger(LoggingInterceptor.class);

    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
            ServerCall<ReqT, RespT> call,
            Metadata headers,
            ServerCallHandler<ReqT, RespT> next) {

        logger.info("gRPC method called: {}", call.getMethodDescriptor().getFullMethodName());
        logger.debug("Headers: {}", headers);

        return next.startCall(new ForwardingServerCall.SimpleForwardingServerCall<ReqT, RespT>(call) {
            @Override
            public void sendMessage(RespT message) {
                logger.debug("Sending response: {}", message);
                super.sendMessage(message);
            }
        }, headers);
    }
}
```

### Authentication Interceptor

```java
package com.example.grpc.interceptor;

import io.grpc.*;
import net.devh.boot.grpc.server.interceptor.GrpcGlobalServerInterceptor;
import org.springframework.core.annotation.Order;

@GrpcGlobalServerInterceptor
@Order(10)
public class AuthenticationInterceptor implements ServerInterceptor {

    private static final Metadata.Key<String> AUTH_TOKEN_KEY = 
            Metadata.Key.of("authorization", Metadata.ASCII_STRING_MARSHALLER);

    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
            ServerCall<ReqT, RespT> call,
            Metadata headers,
            ServerCallHandler<ReqT, RespT> next) {

        String token = headers.get(AUTH_TOKEN_KEY);

        if (token == null || !isValidToken(token)) {
            call.close(Status.UNAUTHENTICATED.withDescription("Invalid or missing token"), headers);
            return new ServerCall.Listener<ReqT>() {};
        }

        Context context = Context.current().withValue(AuthContext.TOKEN_KEY, token);
        return Contexts.interceptCall(context, call, headers, next);
    }

    private boolean isValidToken(String token) {
        // Implement token validation logic
        return token != null && token.startsWith("Bearer ");
    }
}
```

### Context Propagation

```java
package com.example.grpc.context;

import io.grpc.Context;

public class AuthContext {
    public static final Context.Key<String> TOKEN_KEY = Context.key("auth-token");
    public static final Context.Key<String> USER_ID_KEY = Context.key("user-id");
    public static final Context.Key<String> TENANT_ID_KEY = Context.key("tenant-id");

    public static String getToken() {
        return TOKEN_KEY.get();
    }

    public static String getUserId() {
        return USER_ID_KEY.get();
    }

    public static String getTenantId() {
        return TENANT_ID_KEY.get();
    }
}
```

### Error Handling

```java
package com.example.grpc.exception;

import io.grpc.Status;
import io.grpc.StatusRuntimeException;
import net.devh.boot.grpc.server.advice.GrpcAdvice;
import net.devh.boot.grpc.server.advice.GrpcExceptionHandler;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@GrpcAdvice
public class GrpcExceptionAdvice {

    private static final Logger logger = LoggerFactory.getLogger(GrpcExceptionAdvice.class);

    @GrpcExceptionHandler(IllegalArgumentException.class)
    public Status handleIllegalArgument(IllegalArgumentException e) {
        logger.error("Invalid argument: {}", e.getMessage());
        return Status.INVALID_ARGUMENT.withDescription(e.getMessage()).withCause(e);
    }

    @GrpcExceptionHandler(ResourceNotFoundException.class)
    public Status handleResourceNotFound(ResourceNotFoundException e) {
        logger.error("Resource not found: {}", e.getMessage());
        return Status.NOT_FOUND.withDescription(e.getMessage()).withCause(e);
    }

    @GrpcExceptionHandler(Exception.class)
    public Status handleGenericException(Exception e) {
        logger.error("Unexpected error", e);
        return Status.INTERNAL.withDescription("Internal server error").withCause(e);
    }
}
```

## Client Implementation

### gRPC Client Configuration

```java
package com.example.grpc.client.config;

import com.example.grpc.proto.UserServiceGrpc;
import io.grpc.ManagedChannel;
import io.grpc.ManagedChannelBuilder;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class GrpcClientConfig {

    @Value("${grpc.client.user-service.address:localhost:9090}")
    private String userServiceAddress;

    @Bean
    public ManagedChannel userServiceChannel() {
        return ManagedChannelBuilder
                .forTarget(userServiceAddress)
                .usePlaintext()
                .build();
    }

    @Bean
    public UserServiceGrpc.UserServiceBlockingStub userServiceBlockingStub(ManagedChannel channel) {
        return UserServiceGrpc.newBlockingStub(channel);
    }

    @Bean
    public UserServiceGrpc.UserServiceStub userServiceAsyncStub(ManagedChannel channel) {
        return UserServiceGrpc.newStub(channel);
    }
}
```

### Client Service

```java
package com.example.grpc.client.service;

import com.example.grpc.proto.*;
import io.grpc.stub.StreamObserver;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

@Service
public class UserClientService {

    @Autowired
    private UserServiceGrpc.UserServiceBlockingStub blockingStub;

    @Autowired
    private UserServiceGrpc.UserServiceStub asyncStub;

    public User getUser(long userId) {
        GetUserRequest request = GetUserRequest.newBuilder()
                .setUserId(userId)
                .build();

        GetUserResponse response = blockingStub.getUser(request);
        return response.getFound() ? response.getUser() : null;
    }

    public User createUser(String username, String email, String firstName, String lastName) {
        CreateUserRequest request = CreateUserRequest.newBuilder()
                .setUsername(username)
                .setEmail(email)
                .setFirstName(firstName)
                .setLastName(lastName)
                .build();

        CreateUserResponse response = blockingStub.createUser(request);
        return response.getSuccess() ? response.getUser() : null;
    }

    public List<User> streamUsers() throws InterruptedException {
        List<User> users = new ArrayList<>();
        CountDownLatch latch = new CountDownLatch(1);

        ListUsersRequest request = ListUsersRequest.newBuilder().build();

        asyncStub.streamUsers(request, new StreamObserver<User>() {
            @Override
            public void onNext(User user) {
                users.add(user);
            }

            @Override
            public void onError(Throwable t) {
                latch.countDown();
            }

            @Override
            public void onCompleted() {
                latch.countDown();
            }
        });

        latch.await(30, TimeUnit.SECONDS);
        return users;
    }
}
```

## Testing

### gRPC Server Test

```java
package com.example.grpc.service;

import com.example.grpc.proto.*;
import io.grpc.internal.testing.StreamRecorder;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.concurrent.TimeUnit;

import static org.junit.jupiter.api.Assertions.*;

class UserGrpcServiceTest {

    private UserGrpcService service;

    @BeforeEach
    void setUp() {
        service = new UserGrpcService();
    }

    @Test
    void testCreateUser() throws Exception {
        CreateUserRequest request = CreateUserRequest.newBuilder()
                .setUsername("testuser")
                .setEmail("test@example.com")
                .setFirstName("Test")
                .setLastName("User")
                .build();

        StreamRecorder<CreateUserResponse> responseObserver = StreamRecorder.create();
        service.createUser(request, responseObserver);

        if (!responseObserver.awaitCompletion(5, TimeUnit.SECONDS)) {
            fail("The call did not complete in time");
        }

        List<CreateUserResponse> responses = responseObserver.getValues();
        assertEquals(1, responses.size());
        
        CreateUserResponse response = responses.get(0);
        assertTrue(response.getSuccess());
        assertEquals("testuser", response.getUser().getUsername());
    }

    @Test
    void testGetUser() throws Exception {
        // First create a user
        CreateUserRequest createRequest = CreateUserRequest.newBuilder()
                .setUsername("testuser")
                .setEmail("test@example.com")
                .setFirstName("Test")
                .setLastName("User")
                .build();

        StreamRecorder<CreateUserResponse> createObserver = StreamRecorder.create();
        service.createUser(createRequest, createObserver);
        createObserver.awaitCompletion(5, TimeUnit.SECONDS);

        long userId = createObserver.getValues().get(0).getUser().getId();

        // Now get the user
        GetUserRequest getRequest = GetUserRequest.newBuilder()
                .setUserId(userId)
                .build();

        StreamRecorder<GetUserResponse> getObserver = StreamRecorder.create();
        service.getUser(getRequest, getObserver);

        if (!getObserver.awaitCompletion(5, TimeUnit.SECONDS)) {
            fail("The call did not complete in time");
        }

        List<GetUserResponse> responses = getObserver.getValues();
        assertEquals(1, responses.size());
        
        GetUserResponse response = responses.get(0);
        assertTrue(response.getFound());
        assertEquals("testuser", response.getUser().getUsername());
    }
}
```

### Integration Test with TestContainers

```java
package com.example.grpc.integration;

import com.example.grpc.proto.*;
import io.grpc.ManagedChannel;
import io.grpc.ManagedChannelBuilder;
import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;

import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
class GrpcIntegrationTest {

    private static ManagedChannel channel;
    private static UserServiceGrpc.UserServiceBlockingStub blockingStub;

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("grpc.server.port", () -> "9090");
    }

    @BeforeAll
    static void setup() {
        channel = ManagedChannelBuilder.forAddress("localhost", 9090)
                .usePlaintext()
                .build();
        blockingStub = UserServiceGrpc.newBlockingStub(channel);
    }

    @AfterAll
    static void cleanup() {
        if (channel != null) {
            channel.shutdown();
        }
    }

    @Test
    void testCreateAndGetUser() {
        // Create user
        CreateUserRequest createRequest = CreateUserRequest.newBuilder()
                .setUsername("integrationtest")
                .setEmail("integration@test.com")
                .setFirstName("Integration")
                .setLastName("Test")
                .build();

        CreateUserResponse createResponse = blockingStub.createUser(createRequest);
        assertTrue(createResponse.getSuccess());
        assertNotNull(createResponse.getUser());

        // Get user
        GetUserRequest getRequest = GetUserRequest.newBuilder()
                .setUserId(createResponse.getUser().getId())
                .build();

        GetUserResponse getResponse = blockingStub.getUser(getRequest);
        assertTrue(getResponse.getFound());
        assertEquals("integrationtest", getResponse.getUser().getUsername());
    }
}
```

## Best Practices

### 1. Use Streaming for Large Datasets

```java
@Override
public void exportUsers(ExportUsersRequest request, StreamObserver<User> responseObserver) {
    try {
        userRepository.findAll().forEach(userEntity -> {
            User user = convertToProto(userEntity);
            responseObserver.onNext(user);
        });
        responseObserver.onCompleted();
    } catch (Exception e) {
        responseObserver.onError(Status.INTERNAL
                .withDescription("Error exporting users")
                .withCause(e)
                .asRuntimeException());
    }
}
```

### 2. Implement Deadlines/Timeouts

```java
@Override
public void processWithDeadline(ProcessRequest request, StreamObserver<ProcessResponse> responseObserver) {
    Context ctx = Context.current();
    
    if (ctx.getDeadline() != null && ctx.getDeadline().isExpired()) {
        responseObserver.onError(Status.DEADLINE_EXCEEDED
                .withDescription("Request deadline exceeded")
                .asRuntimeException());
        return;
    }
    
    // Process request
}
```

### 3. Use Metadata for Cross-Cutting Concerns

```java
public static final Metadata.Key<String> CORRELATION_ID_KEY = 
        Metadata.Key.of("x-correlation-id", Metadata.ASCII_STRING_MARSHALLER);

@Override
public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
        ServerCall<ReqT, RespT> call,
        Metadata headers,
        ServerCallHandler<ReqT, RespT> next) {
    
    String correlationId = headers.get(CORRELATION_ID_KEY);
    if (correlationId == null) {
        correlationId = UUID.randomUUID().toString();
    }
    
    Context context = Context.current()
            .withValue(CORRELATION_ID_CONTEXT_KEY, correlationId);
    
    return Contexts.interceptCall(context, call, headers, next);
}
```

### 4. Structured Error Responses

```protobuf
message ErrorDetail {
  string code = 1;
  string message = 2;
  map<string, string> metadata = 3;
}

message CreateUserResponse {
  User user = 1;
  bool success = 2;
  ErrorDetail error = 3;
}
```

## Performance Optimization

### Connection Pooling

```java
@Bean
public ManagedChannel managedChannel() {
    return ManagedChannelBuilder
            .forAddress("localhost", 9090)
            .usePlaintext()
            .maxInboundMessageSize(10 * 1024 * 1024) // 10MB
            .keepAliveTime(30, TimeUnit.SECONDS)
            .keepAliveTimeout(10, TimeUnit.SECONDS)
            .keepAliveWithoutCalls(true)
            .build();
}
```

### Load Balancing

```java
@Bean
public ManagedChannel loadBalancedChannel() {
    return ManagedChannelBuilder
            .forTarget("dns:///user-service:9090")
            .defaultLoadBalancingPolicy("round_robin")
            .usePlaintext()
            .build();
}
```

## Common Use Cases

### 1. Microservice Communication
- Service-to-service calls
- Internal APIs
- High-performance data transfer

### 2. Real-time Features
- Live updates
- Streaming data
- Bi-directional communication

### 3. IoT and Mobile
- Efficient binary protocol
- Battery-friendly
- Network-efficient

## Monitoring and Observability

### gRPC Metrics

```java
@Component
public class GrpcMetricsInterceptor implements ServerInterceptor {

    private final MeterRegistry meterRegistry;

    public GrpcMetricsInterceptor(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }

    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
            ServerCall<ReqT, RespT> call,
            Metadata headers,
            ServerCallHandler<ReqT, RespT> next) {

        String methodName = call.getMethodDescriptor().getFullMethodName();
        Timer.Sample sample = Timer.start(meterRegistry);

        return new ForwardingServerCallListener.SimpleForwardingServerCallListener<ReqT>(
                next.startCall(call, headers)) {

            @Override
            public void onComplete() {
                sample.stop(Timer.builder("grpc.server.call.duration")
                        .tag("method", methodName)
                        .tag("status", "OK")
                        .register(meterRegistry));
                super.onComplete();
            }

            @Override
            public void onCancel() {
                sample.stop(Timer.builder("grpc.server.call.duration")
                        .tag("method", methodName)
                        .tag("status", "CANCELLED")
                        .register(meterRegistry));
                super.onCancel();
            }
        };
    }
}
```

## Additional Resources

- [gRPC Official Documentation](https://grpc.io/docs/)
- [gRPC Spring Boot Starter](https://github.com/grpc-ecosystem/grpc-spring)
- [Protocol Buffers Guide](https://protobuf.dev/)
- [gRPC Best Practices](https://grpc.io/docs/guides/performance/)
