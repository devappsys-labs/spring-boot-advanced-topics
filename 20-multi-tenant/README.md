# Multi-Tenant Database Switching

## Overview

Multi-tenancy is an architecture where a single instance of software serves multiple tenants (customers/organizations). Database switching allows each tenant to have isolated data storage while sharing the same application infrastructure. This implementation focuses on runtime database switching using gRPC context and REST context.

## Key Concepts

### Multi-Tenancy Strategies

1. **Database per Tenant** - Separate database for each tenant (highest isolation)
2. **Schema per Tenant** - Separate schema within same database (moderate isolation)
3. **Shared Database, Shared Schema** - Tenant ID column discriminator (lowest isolation)

### Context-Based Routing

- **gRPC Context** - Propagate tenant info through gRPC metadata
- **REST Context** - Extract tenant from HTTP headers/tokens
- **Thread-Local Storage** - Maintain tenant context per request

## Architecture Components

```
Request (with Tenant ID) 
    ↓
Interceptor/Filter (Extract Tenant)
    ↓
Tenant Context (ThreadLocal)
    ↓
DataSource Router
    ↓
Tenant-Specific Database
```

## Dependencies

### Maven

```xml
<dependencies>
    <!-- Spring Boot -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    
    <!-- gRPC -->
    <dependency>
        <groupId>net.devh</groupId>
        <artifactId>grpc-spring-boot-starter</artifactId>
        <version>2.15.0.RELEASE</version>
    </dependency>
    
    <!-- Database Drivers -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
    </dependency>
    
    <!-- HikariCP for connection pooling -->
    <dependency>
        <groupId>com.zaxxer</groupId>
        <artifactId>HikariCP</artifactId>
    </dependency>
    
    <!-- Flyway for migrations -->
    <dependency>
        <groupId>org.flywaydb</groupId>
        <artifactId>flyway-core</artifactId>
    </dependency>
</dependencies>
```

### Gradle

```gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'net.devh:grpc-spring-boot-starter:2.15.0.RELEASE'
    implementation 'org.postgresql:postgresql'
    implementation 'com.zaxxer:HikariCP'
    implementation 'org.flywaydb:flyway-core'
}
```

## Configuration

### Application Properties

**application.yml**

```yaml
spring:
  application:
    name: multi-tenant-service
    
  jpa:
    hibernate:
      ddl-auto: none
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        dialect: org.hibernate.dialect.PostgreSQLDialect

# Tenant configurations
tenants:
  default: tenant1
  datasources:
    tenant1:
      url: jdbc:postgresql://localhost:5432/tenant1_db
      username: tenant1_user
      password: tenant1_pass
      driver-class-name: org.postgresql.Driver
      hikari:
        maximum-pool-size: 10
        minimum-idle: 5
        connection-timeout: 30000
        
    tenant2:
      url: jdbc:postgresql://localhost:5432/tenant2_db
      username: tenant2_user
      password: tenant2_pass
      driver-class-name: org.postgresql.Driver
      hikari:
        maximum-pool-size: 10
        minimum-idle: 5
        connection-timeout: 30000
        
    tenant3:
      url: jdbc:postgresql://localhost:5432/tenant3_db
      username: tenant3_user
      password: tenant3_pass
      driver-class-name: org.postgresql.Driver
      hikari:
        maximum-pool-size: 10
        minimum-idle: 5
        connection-timeout: 30000

grpc:
  server:
    port: 9090

logging:
  level:
    com.example.multitenant: DEBUG
```

## Tenant Context Management

### Tenant Context Holder

```java
package com.example.multitenant.context;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class TenantContext {

    private static final Logger logger = LoggerFactory.getLogger(TenantContext.class);
    
    private static final ThreadLocal<String> CURRENT_TENANT = new ThreadLocal<>();
    private static final String DEFAULT_TENANT = "tenant1";

    public static void setTenantId(String tenantId) {
        logger.debug("Setting tenant context to: {}", tenantId);
        CURRENT_TENANT.set(tenantId);
    }

    public static String getTenantId() {
        String tenantId = CURRENT_TENANT.get();
        return tenantId != null ? tenantId : DEFAULT_TENANT;
    }

    public static void clear() {
        logger.debug("Clearing tenant context");
        CURRENT_TENANT.remove();
    }
    
    public static boolean hasTenant() {
        return CURRENT_TENANT.get() != null;
    }
}
```

### Tenant Identifier Resolver

```java
package com.example.multitenant.config;

import com.example.multitenant.context.TenantContext;
import org.hibernate.context.spi.CurrentTenantIdentifierResolver;
import org.springframework.stereotype.Component;

@Component
public class CurrentTenantIdentifierResolverImpl implements CurrentTenantIdentifierResolver {

    @Override
    public String resolveCurrentTenantIdentifier() {
        return TenantContext.getTenantId();
    }

    @Override
    public boolean validateExistingCurrentSessions() {
        return true;
    }
}
```

## DataSource Configuration

### Tenant DataSource Properties

```java
package com.example.multitenant.config;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

import java.util.HashMap;
import java.util.Map;

@Component
@ConfigurationProperties(prefix = "tenants")
public class TenantDataSourceProperties {

    private String defaultTenant;
    private Map<String, DataSourceProperties> datasources = new HashMap<>();

    public String getDefaultTenant() {
        return defaultTenant;
    }

    public void setDefaultTenant(String defaultTenant) {
        this.defaultTenant = defaultTenant;
    }

    public Map<String, DataSourceProperties> getDatasources() {
        return datasources;
    }

    public void setDatasources(Map<String, DataSourceProperties> datasources) {
        this.datasources = datasources;
    }

    public static class DataSourceProperties {
        private String url;
        private String username;
        private String password;
        private String driverClassName;
        private HikariProperties hikari;

        // Getters and setters
        public String getUrl() { return url; }
        public void setUrl(String url) { this.url = url; }

        public String getUsername() { return username; }
        public void setUsername(String username) { this.username = username; }

        public String getPassword() { return password; }
        public void setPassword(String password) { this.password = password; }

        public String getDriverClassName() { return driverClassName; }
        public void setDriverClassName(String driverClassName) { this.driverClassName = driverClassName; }

        public HikariProperties getHikari() { return hikari; }
        public void setHikari(HikariProperties hikari) { this.hikari = hikari; }
    }

    public static class HikariProperties {
        private int maximumPoolSize = 10;
        private int minimumIdle = 5;
        private long connectionTimeout = 30000;

        public int getMaximumPoolSize() { return maximumPoolSize; }
        public void setMaximumPoolSize(int maximumPoolSize) { this.maximumPoolSize = maximumPoolSize; }

        public int getMinimumIdle() { return minimumIdle; }
        public void setMinimumIdle(int minimumIdle) { this.minimumIdle = minimumIdle; }

        public long getConnectionTimeout() { return connectionTimeout; }
        public void setConnectionTimeout(long connectionTimeout) { this.connectionTimeout = connectionTimeout; }
    }
}
```

### Multi-Tenant DataSource Configuration

```java
package com.example.multitenant.config;

import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;
import org.hibernate.cfg.AvailableSettings;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.autoconfigure.orm.jpa.JpaProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.jpa.repository.config.EnableJpaRepositories;
import org.springframework.orm.jpa.JpaTransactionManager;
import org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean;
import org.springframework.orm.jpa.vendor.HibernateJpaVendorAdapter;
import org.springframework.transaction.PlatformTransactionManager;

import javax.sql.DataSource;
import java.util.HashMap;
import java.util.Map;

@Configuration
@EnableJpaRepositories(
    basePackages = "com.example.multitenant.repository",
    entityManagerFactoryRef = "entityManagerFactory",
    transactionManagerRef = "transactionManager"
)
public class MultiTenantDataSourceConfig {

    @Autowired
    private TenantDataSourceProperties tenantProperties;

    @Autowired
    private JpaProperties jpaProperties;

    @Bean
    public DataSource dataSource() {
        TenantRoutingDataSource routingDataSource = new TenantRoutingDataSource();
        
        Map<Object, Object> targetDataSources = new HashMap<>();
        
        tenantProperties.getDatasources().forEach((tenantId, props) -> {
            targetDataSources.put(tenantId, createDataSource(props));
        });
        
        routingDataSource.setTargetDataSources(targetDataSources);
        routingDataSource.setDefaultTargetDataSource(
            targetDataSources.get(tenantProperties.getDefaultTenant())
        );
        
        routingDataSource.afterPropertiesSet();
        return routingDataSource;
    }

    private DataSource createDataSource(TenantDataSourceProperties.DataSourceProperties props) {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(props.getUrl());
        config.setUsername(props.getUsername());
        config.setPassword(props.getPassword());
        config.setDriverClassName(props.getDriverClassName());
        
        if (props.getHikari() != null) {
            config.setMaximumPoolSize(props.getHikari().getMaximumPoolSize());
            config.setMinimumIdle(props.getHikari().getMinimumIdle());
            config.setConnectionTimeout(props.getHikari().getConnectionTimeout());
        }
        
        return new HikariDataSource(config);
    }

    @Bean
    public LocalContainerEntityManagerFactoryBean entityManagerFactory(DataSource dataSource) {
        LocalContainerEntityManagerFactoryBean em = new LocalContainerEntityManagerFactoryBean();
        em.setDataSource(dataSource);
        em.setPackagesToScan("com.example.multitenant.entity");
        
        HibernateJpaVendorAdapter vendorAdapter = new HibernateJpaVendorAdapter();
        em.setJpaVendorAdapter(vendorAdapter);
        
        Map<String, Object> properties = new HashMap<>(jpaProperties.getProperties());
        properties.put(AvailableSettings.MULTI_TENANT_IDENTIFIER_RESOLVER, 
                      currentTenantIdentifierResolver());
        
        em.setJpaPropertyMap(properties);
        return em;
    }

    @Bean
    public PlatformTransactionManager transactionManager(
            LocalContainerEntityManagerFactoryBean entityManagerFactory) {
        JpaTransactionManager transactionManager = new JpaTransactionManager();
        transactionManager.setEntityManagerFactory(entityManagerFactory.getObject());
        return transactionManager;
    }

    @Bean
    public CurrentTenantIdentifierResolverImpl currentTenantIdentifierResolver() {
        return new CurrentTenantIdentifierResolverImpl();
    }
}
```

### Tenant Routing DataSource

```java
package com.example.multitenant.config;

import com.example.multitenant.context.TenantContext;
import org.springframework.jdbc.datasource.lookup.AbstractRoutingDataSource;

public class TenantRoutingDataSource extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        return TenantContext.getTenantId();
    }
}
```

## REST Context Integration

### Tenant Interceptor for REST

```java
package com.example.multitenant.interceptor;

import com.example.multitenant.context.TenantContext;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerInterceptor;

@Component
public class TenantInterceptor implements HandlerInterceptor {

    private static final Logger logger = LoggerFactory.getLogger(TenantInterceptor.class);
    private static final String TENANT_HEADER = "X-Tenant-ID";

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        String tenantId = extractTenantId(request);
        
        if (tenantId != null) {
            logger.debug("Setting tenant from REST context: {}", tenantId);
            TenantContext.setTenantId(tenantId);
        } else {
            logger.warn("No tenant ID found in request");
        }
        
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, 
                               Object handler, Exception ex) {
        TenantContext.clear();
    }

    private String extractTenantId(HttpServletRequest request) {
        // Try header first
        String tenantId = request.getHeader(TENANT_HEADER);
        
        // Try path parameter
        if (tenantId == null) {
            tenantId = request.getParameter("tenantId");
        }
        
        // Try subdomain (e.g., tenant1.example.com)
        if (tenantId == null) {
            String host = request.getServerName();
            if (host.contains(".")) {
                tenantId = host.substring(0, host.indexOf("."));
            }
        }
        
        return tenantId;
    }
}
```

### Web MVC Configuration

```java
package com.example.multitenant.config;

import com.example.multitenant.interceptor.TenantInterceptor;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    @Autowired
    private TenantInterceptor tenantInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(tenantInterceptor)
                .addPathPatterns("/api/**")
                .excludePathPatterns("/api/public/**", "/actuator/**");
    }
}
```

### Tenant Filter (Alternative to Interceptor)

```java
package com.example.multitenant.filter;

import com.example.multitenant.context.TenantContext;
import jakarta.servlet.*;
import jakarta.servlet.http.HttpServletRequest;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

import java.io.IOException;

@Component
@Order(1)
public class TenantFilter implements Filter {

    private static final Logger logger = LoggerFactory.getLogger(TenantFilter.class);
    private static final String TENANT_HEADER = "X-Tenant-ID";

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        String tenantId = httpRequest.getHeader(TENANT_HEADER);

        try {
            if (tenantId != null) {
                logger.debug("Tenant filter - Setting tenant: {}", tenantId);
                TenantContext.setTenantId(tenantId);
            }
            chain.doFilter(request, response);
        } finally {
            TenantContext.clear();
        }
    }
}
```

## gRPC Context Integration

### gRPC Tenant Interceptor

```java
package com.example.multitenant.grpc.interceptor;

import com.example.multitenant.context.TenantContext;
import io.grpc.*;
import net.devh.boot.grpc.server.interceptor.GrpcGlobalServerInterceptor;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@GrpcGlobalServerInterceptor
public class GrpcTenantInterceptor implements ServerInterceptor {

    private static final Logger logger = LoggerFactory.getLogger(GrpcTenantInterceptor.class);
    
    private static final Metadata.Key<String> TENANT_KEY = 
            Metadata.Key.of("x-tenant-id", Metadata.ASCII_STRING_MARSHALLER);

    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
            ServerCall<ReqT, RespT> call,
            Metadata headers,
            ServerCallHandler<ReqT, RespT> next) {

        String tenantId = headers.get(TENANT_KEY);

        if (tenantId == null) {
            logger.warn("No tenant ID in gRPC metadata");
            call.close(Status.UNAUTHENTICATED.withDescription("Tenant ID required"), headers);
            return new ServerCall.Listener<ReqT>() {};
        }

        logger.debug("gRPC interceptor - Setting tenant: {}", tenantId);
        TenantContext.setTenantId(tenantId);

        Context context = Context.current().withValue(GrpcTenantContext.TENANT_KEY, tenantId);

        return Contexts.interceptCall(context, new ForwardingServerCall.SimpleForwardingServerCall<ReqT, RespT>(call) {
            @Override
            public void close(Status status, Metadata trailers) {
                try {
                    super.close(status, trailers);
                } finally {
                    TenantContext.clear();
                }
            }
        }, headers, next);
    }
}
```

### gRPC Tenant Context

```java
package com.example.multitenant.grpc.interceptor;

import io.grpc.Context;

public class GrpcTenantContext {
    
    public static final Context.Key<String> TENANT_KEY = Context.key("tenant-id");
    
    public static String getTenantId() {
        return TENANT_KEY.get();
    }
}
```

### gRPC Service with Tenant Context

```java
package com.example.multitenant.grpc.service;

import com.example.multitenant.context.TenantContext;
import com.example.multitenant.entity.User;
import com.example.multitenant.repository.UserRepository;
import com.example.proto.*;
import io.grpc.stub.StreamObserver;
import net.devh.boot.grpc.server.service.GrpcService;
import org.springframework.beans.factory.annotation.Autowired;

@GrpcService
public class UserGrpcService extends UserServiceGrpc.UserServiceImplBase {

    @Autowired
    private UserRepository userRepository;

    @Override
    public void getUser(GetUserRequest request, StreamObserver<GetUserResponse> responseObserver) {
        String tenantId = TenantContext.getTenantId();
        
        // Data is automatically isolated by tenant due to DataSource routing
        User user = userRepository.findById(request.getUserId()).orElse(null);
        
        GetUserResponse.Builder responseBuilder = GetUserResponse.newBuilder();
        
        if (user != null) {
            responseBuilder.setUser(convertToProto(user));
            responseBuilder.setFound(true);
        } else {
            responseBuilder.setFound(false);
        }
        
        responseObserver.onNext(responseBuilder.build());
        responseObserver.onCompleted();
    }

    private com.example.proto.User convertToProto(User user) {
        return com.example.proto.User.newBuilder()
                .setId(user.getId())
                .setUsername(user.getUsername())
                .setEmail(user.getEmail())
                .build();
    }
}
```

## Entity and Repository

### User Entity

```java
package com.example.multitenant.entity;

import jakarta.persistence.*;

import java.time.LocalDateTime;

@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String username;

    @Column(nullable = false)
    private String email;

    @Column(name = "created_at")
    private LocalDateTime createdAt;

    @PrePersist
    protected void onCreate() {
        createdAt = LocalDateTime.now();
    }

    // Getters and Setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }

    public String getUsername() { return username; }
    public void setUsername(String username) { this.username = username; }

    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }

    public LocalDateTime getCreatedAt() { return createdAt; }
    public void setCreatedAt(LocalDateTime createdAt) { this.createdAt = createdAt; }
}
```

### User Repository

```java
package com.example.multitenant.repository;

import com.example.multitenant.entity.User;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.Optional;

@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    Optional<User> findByUsername(String username);
    
    Optional<User> findByEmail(String email);
    
    boolean existsByUsername(String username);
}
```

## REST Controller

```java
package com.example.multitenant.controller;

import com.example.multitenant.context.TenantContext;
import com.example.multitenant.entity.User;
import com.example.multitenant.service.UserService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.HashMap;
import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/api/users")
public class UserController {

    @Autowired
    private UserService userService;

    @GetMapping
    public ResponseEntity<List<User>> getAllUsers() {
        return ResponseEntity.ok(userService.getAllUsers());
    }

    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        return userService.getUserById(id)
                .map(ResponseEntity::ok)
                .orElse(ResponseEntity.notFound().build());
    }

    @PostMapping
    public ResponseEntity<User> createUser(@RequestBody User user) {
        User created = userService.createUser(user);
        return ResponseEntity.ok(created);
    }

    @PutMapping("/{id}")
    public ResponseEntity<User> updateUser(@PathVariable Long id, @RequestBody User user) {
        return userService.updateUser(id, user)
                .map(ResponseEntity::ok)
                .orElse(ResponseEntity.notFound().build());
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        userService.deleteUser(id);
        return ResponseEntity.noContent().build();
    }

    @GetMapping("/tenant-info")
    public ResponseEntity<Map<String, String>> getTenantInfo() {
        Map<String, String> info = new HashMap<>();
        info.put("currentTenant", TenantContext.getTenantId());
        return ResponseEntity.ok(info);
    }
}
```

## Service Layer

```java
package com.example.multitenant.service;

import com.example.multitenant.entity.User;
import com.example.multitenant.repository.UserRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.Optional;

@Service
@Transactional
public class UserService {

    @Autowired
    private UserRepository userRepository;

    public List<User> getAllUsers() {
        return userRepository.findAll();
    }

    public Optional<User> getUserById(Long id) {
        return userRepository.findById(id);
    }

    public User createUser(User user) {
        return userRepository.save(user);
    }

    public Optional<User> updateUser(Long id, User userDetails) {
        return userRepository.findById(id)
                .map(user -> {
                    user.setUsername(userDetails.getUsername());
                    user.setEmail(userDetails.getEmail());
                    return userRepository.save(user);
                });
    }

    public void deleteUser(Long id) {
        userRepository.deleteById(id);
    }

    public Optional<User> findByUsername(String username) {
        return userRepository.findByUsername(username);
    }
}
```

## Dynamic Tenant Management

### Tenant Management Service

```java
package com.example.multitenant.service;

import com.example.multitenant.config.TenantDataSourceProperties;
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import javax.sql.DataSource;
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@Service
public class TenantManagementService {

    private final Map<String, DataSource> tenantDataSources = new ConcurrentHashMap<>();

    @Autowired
    private TenantDataSourceProperties tenantProperties;

    public void addTenant(String tenantId, String url, String username, String password) {
        if (tenantDataSources.containsKey(tenantId)) {
            throw new IllegalArgumentException("Tenant already exists: " + tenantId);
        }

        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(url);
        config.setUsername(username);
        config.setPassword(password);
        config.setDriverClassName("org.postgresql.Driver");
        config.setMaximumPoolSize(10);

        DataSource dataSource = new HikariDataSource(config);
        tenantDataSources.put(tenantId, dataSource);
    }

    public void removeTenant(String tenantId) {
        DataSource dataSource = tenantDataSources.remove(tenantId);
        if (dataSource instanceof HikariDataSource) {
            ((HikariDataSource) dataSource).close();
        }
    }

    public boolean tenantExists(String tenantId) {
        return tenantDataSources.containsKey(tenantId) || 
               tenantProperties.getDatasources().containsKey(tenantId);
    }

    public Map<String, String> getAllTenants() {
        Map<String, String> tenants = new HashMap<>();
        tenantProperties.getDatasources().forEach((id, props) -> {
            tenants.put(id, props.getUrl());
        });
        return tenants;
    }
}
```

### Tenant Management Controller

```java
package com.example.multitenant.controller;

import com.example.multitenant.service.TenantManagementService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.Map;

@RestController
@RequestMapping("/api/admin/tenants")
public class TenantManagementController {

    @Autowired
    private TenantManagementService tenantManagementService;

    @GetMapping
    public ResponseEntity<Map<String, String>> getAllTenants() {
        return ResponseEntity.ok(tenantManagementService.getAllTenants());
    }

    @PostMapping
    public ResponseEntity<Map<String, String>> addTenant(@RequestBody TenantRequest request) {
        try {
            tenantManagementService.addTenant(
                request.getTenantId(),
                request.getUrl(),
                request.getUsername(),
                request.getPassword()
            );
            return ResponseEntity.ok(Map.of("message", "Tenant added successfully"));
        } catch (Exception e) {
            return ResponseEntity.badRequest()
                    .body(Map.of("error", e.getMessage()));
        }
    }

    @DeleteMapping("/{tenantId}")
    public ResponseEntity<Map<String, String>> removeTenant(@PathVariable String tenantId) {
        tenantManagementService.removeTenant(tenantId);
        return ResponseEntity.ok(Map.of("message", "Tenant removed successfully"));
    }

    @GetMapping("/{tenantId}/exists")
    public ResponseEntity<Map<String, Boolean>> tenantExists(@PathVariable String tenantId) {
        boolean exists = tenantManagementService.tenantExists(tenantId);
        return ResponseEntity.ok(Map.of("exists", exists));
    }

    static class TenantRequest {
        private String tenantId;
        private String url;
        private String username;
        private String password;

        // Getters and setters
        public String getTenantId() { return tenantId; }
        public void setTenantId(String tenantId) { this.tenantId = tenantId; }

        public String getUrl() { return url; }
        public void setUrl(String url) { this.url = url; }

        public String getUsername() { return username; }
        public void setUsername(String username) { this.username = username; }

        public String getPassword() { return password; }
        public void setPassword(String password) { this.password = password; }
    }
}
```

## Database Migration per Tenant

### Flyway Tenant Migration

```java
package com.example.multitenant.migration;

import com.example.multitenant.config.TenantDataSourceProperties;
import org.flywaydb.core.Flyway;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Component;

import javax.sql.DataSource;

@Component
public class TenantFlywayMigration implements CommandLineRunner {

    @Autowired
    private TenantDataSourceProperties tenantProperties;

    @Override
    public void run(String... args) {
        tenantProperties.getDatasources().forEach((tenantId, props) -> {
            System.out.println("Running Flyway migration for tenant: " + tenantId);
            
            Flyway flyway = Flyway.configure()
                    .dataSource(props.getUrl(), props.getUsername(), props.getPassword())
                    .locations("classpath:db/migration")
                    .baselineOnMigrate(true)
                    .load();
            
            flyway.migrate();
        });
    }
}
```

## Testing

### Multi-Tenant Integration Test

```java
package com.example.multitenant;

import com.example.multitenant.context.TenantContext;
import com.example.multitenant.entity.User;
import com.example.multitenant.repository.UserRepository;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest
class MultiTenantTest {

    @Autowired
    private UserRepository userRepository;

    @AfterEach
    void cleanup() {
        TenantContext.clear();
    }

    @Test
    void testTenantIsolation() {
        // Create user for tenant1
        TenantContext.setTenantId("tenant1");
        User user1 = new User();
        user1.setUsername("user_tenant1");
        user1.setEmail("user@tenant1.com");
        userRepository.save(user1);
        
        long tenant1Count = userRepository.count();

        // Switch to tenant2
        TenantContext.clear();
        TenantContext.setTenantId("tenant2");
        
        // Verify tenant2 doesn't see tenant1's data
        long tenant2Count = userRepository.count();
        
        assertNotEquals(tenant1Count, tenant2Count);
    }

    @Test
    void testDataIsolationBetweenTenants() {
        // Tenant 1
        TenantContext.setTenantId("tenant1");
        User tenant1User = new User();
        tenant1User.setUsername("isolated_user");
        tenant1User.setEmail("user@tenant1.com");
        userRepository.save(tenant1User);

        // Tenant 2
        TenantContext.clear();
        TenantContext.setTenantId("tenant2");
        
        // Should not find tenant1's user
        assertFalse(userRepository.findByUsername("isolated_user").isPresent());
    }
}
```

### REST API Test

```java
package com.example.multitenant.controller;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@AutoConfigureMockMvc
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void testGetUsersWithTenantHeader() throws Exception {
        mockMvc.perform(get("/api/users")
                .header("X-Tenant-ID", "tenant1"))
                .andExpect(status().isOk())
                .andExpect(content().contentType(MediaType.APPLICATION_JSON));
    }

    @Test
    void testCreateUserWithTenant() throws Exception {
        String userJson = """
            {
                "username": "testuser",
                "email": "test@example.com"
            }
            """;

        mockMvc.perform(post("/api/users")
                .header("X-Tenant-ID", "tenant1")
                .contentType(MediaType.APPLICATION_JSON)
                .content(userJson))
                .andExpect(status().isOk());
    }
}
```

## Best Practices

### 1. Always Clear Tenant Context

```java
try {
    TenantContext.setTenantId(tenantId);
    // Perform operations
} finally {
    TenantContext.clear();
}
```

### 2. Validate Tenant Before Operations

```java
public void validateTenant(String tenantId) {
    if (!tenantManagementService.tenantExists(tenantId)) {
        throw new TenantNotFoundException("Tenant not found: " + tenantId);
    }
}
```

### 3. Connection Pool Management

```yaml
hikari:
  maximum-pool-size: 10
  minimum-idle: 5
  connection-timeout: 30000
  idle-timeout: 600000
  max-lifetime: 1800000
```

### 4. Tenant Security

```java
@PreAuthorize("hasPermission(#tenantId, 'TENANT', 'ACCESS')")
public void accessTenantData(String tenantId) {
    // Secured method
}
```

## Monitoring and Observability

### Tenant Metrics

```java
package com.example.multitenant.monitoring;

import com.example.multitenant.context.TenantContext;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Tag;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.stereotype.Component;

import java.util.Arrays;

@Aspect
@Component
public class TenantMetricsAspect {

    private final MeterRegistry meterRegistry;

    public TenantMetricsAspect(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }

    @Around("@within(org.springframework.stereotype.Repository)")
    public Object recordTenantMetrics(ProceedingJoinPoint joinPoint) throws Throwable {
        String tenantId = TenantContext.getTenantId();
        
        return meterRegistry.timer("tenant.database.operation",
                Arrays.asList(
                    Tag.of("tenant", tenantId),
                    Tag.of("method", joinPoint.getSignature().getName())
                ))
                .record(() -> {
                    try {
                        return joinPoint.proceed();
                    } catch (Throwable e) {
                        throw new RuntimeException(e);
                    }
                });
    }
}
```

## Common Pitfalls

1. **Forgetting to clear TenantContext** - Always use try-finally
2. **Thread pool contamination** - Context doesn't propagate to new threads automatically
3. **Missing tenant validation** - Always validate tenant exists
4. **Connection pool exhaustion** - Monitor and tune pool sizes per tenant

## Additional Resources

- [Spring Data JPA Multi-Tenancy](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#multi-tenancy)
- [Hibernate Multi-Tenancy](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#multitenacy)
- [gRPC Context Propagation](https://grpc.io/docs/guides/context/)
