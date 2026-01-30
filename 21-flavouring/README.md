# Flavouring (Feature Flags/Profiles)

## Overview

Flavouring is a technique for managing application variations through feature flags, configuration profiles, and build variants. It enables controlled feature rollouts, A/B testing, environment-specific configurations, and multi-tenant customization without code changes or redeployment.

## Key Concepts

### Feature Management Strategies

1. **Feature Flags** - Runtime toggles for features
2. **Spring Profiles** - Environment-based configuration
3. **Build Flavours** - Compile-time variants
4. **Configuration Properties** - External configuration management
5. **Strategy Pattern** - Pluggable implementations

### Use Cases

- Progressive feature rollouts
- A/B testing
- Tenant-specific features
- Environment differentiation
- Emergency kill switches
- Beta features

## Dependencies

### Maven

```xml
<dependencies>
    <!-- Spring Boot -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    
    <!-- Togglz Feature Flags -->
    <dependency>
        <groupId>org.togglz</groupId>
        <artifactId>togglz-spring-boot-starter</artifactId>
        <version>4.2.0</version>
    </dependency>
    
    <dependency>
        <groupId>org.togglz</groupId>
        <artifactId>togglz-console</artifactId>
        <version>4.2.0</version>
    </dependency>
    
    <!-- FF4J Alternative -->
    <dependency>
        <groupId>org.ff4j</groupId>
        <artifactId>ff4j-spring-boot-starter</artifactId>
        <version>1.9.0</version>
    </dependency>
    
    <!-- Configuration Management -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-configuration-processor</artifactId>
        <optional>true</optional>
    </dependency>
    
    <!-- Redis for distributed flags -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
</dependencies>
```

### Gradle

```gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.togglz:togglz-spring-boot-starter:4.2.0'
    implementation 'org.togglz:togglz-console:4.2.0'
    implementation 'org.ff4j:ff4j-spring-boot-starter:1.9.0'
    implementation 'org.springframework.boot:spring-boot-starter-data-redis'
    annotationProcessor 'org.springframework.boot:spring-boot-configuration-processor'
}
```

## Spring Profiles

### Profile Configuration

**application.yml**

```yaml
spring:
  application:
    name: flavoured-service
    
  profiles:
    active: ${ACTIVE_PROFILE:development}

---
# Development Profile
spring:
  config:
    activate:
      on-profile: development
      
  datasource:
    url: jdbc:postgresql://localhost:5432/dev_db
    username: dev_user
    password: dev_pass
    
  jpa:
    show-sql: true
    hibernate:
      ddl-auto: update

logging:
  level:
    root: DEBUG
    
features:
  new-ui: true
  analytics: false
  experimental-api: true

---
# Staging Profile
spring:
  config:
    activate:
      on-profile: staging
      
  datasource:
    url: jdbc:postgresql://staging-db:5432/staging_db
    username: staging_user
    password: ${STAGING_DB_PASSWORD}
    
  jpa:
    show-sql: false
    hibernate:
      ddl-auto: validate

logging:
  level:
    root: INFO

features:
  new-ui: true
  analytics: true
  experimental-api: true

---
# Production Profile
spring:
  config:
    activate:
      on-profile: production
      
  datasource:
    url: jdbc:postgresql://prod-db:5432/prod_db
    username: prod_user
    password: ${PROD_DB_PASSWORD}
    
  jpa:
    show-sql: false
    hibernate:
      ddl-auto: none

logging:
  level:
    root: WARN

features:
  new-ui: false
  analytics: true
  experimental-api: false
```

### Profile-Specific Beans

```java
package com.example.flavour.config;

import com.example.flavour.service.PaymentService;
import com.example.flavour.service.impl.MockPaymentService;
import com.example.flavour.service.impl.ProductionPaymentService;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;

@Configuration
public class PaymentConfiguration {

    @Bean
    @Profile("development")
    public PaymentService developmentPaymentService() {
        return new MockPaymentService();
    }

    @Bean
    @Profile({"staging", "production"})
    public PaymentService productionPaymentService() {
        return new ProductionPaymentService();
    }
}
```

### Profile-Aware Components

```java
package com.example.flavour.service;

import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Service;

@Service
@Profile("development")
public class DevelopmentDataSeeder {

    public void seedTestData() {
        // Seed development data
        System.out.println("Seeding development test data...");
    }
}
```

## Feature Flags with Togglz

### Feature Enum Definition

```java
package com.example.flavour.features;

import org.togglz.core.Feature;
import org.togglz.core.annotation.EnabledByDefault;
import org.togglz.core.annotation.Label;
import org.togglz.core.context.FeatureContext;

public enum ApplicationFeatures implements Feature {

    @Label("New Dashboard UI")
    @EnabledByDefault
    NEW_DASHBOARD_UI,

    @Label("Advanced Analytics")
    ADVANCED_ANALYTICS,

    @Label("Export to PDF")
    EXPORT_PDF,

    @Label("Real-time Notifications")
    REALTIME_NOTIFICATIONS,

    @Label("Beta API v2")
    BETA_API_V2,

    @Label("Machine Learning Recommendations")
    ML_RECOMMENDATIONS,

    @Label("Multi-factor Authentication")
    @EnabledByDefault
    MFA,

    @Label("Dark Mode")
    DARK_MODE;

    public boolean isActive() {
        return FeatureContext.getFeatureManager().isActive(this);
    }
}
```

### Togglz Configuration

```java
package com.example.flavour.config;

import com.example.flavour.features.ApplicationFeatures;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.togglz.core.manager.EnumBasedFeatureProvider;
import org.togglz.core.spi.FeatureProvider;
import org.togglz.core.user.FeatureUser;
import org.togglz.core.user.SimpleFeatureUser;
import org.togglz.core.user.UserProvider;

@Configuration
public class TogglzConfiguration {

    @Bean
    public FeatureProvider featureProvider() {
        return new EnumBasedFeatureProvider(ApplicationFeatures.class);
    }

    @Bean
    public UserProvider userProvider() {
        return () -> new SimpleFeatureUser("admin", true);
    }
}
```

**application.yml (Togglz)**

```yaml
togglz:
  enabled: true
  feature-enums: com.example.flavour.features.ApplicationFeatures
  console:
    enabled: true
    path: /togglz-console
    secured: false
  features:
    NEW_DASHBOARD_UI:
      enabled: true
    ADVANCED_ANALYTICS:
      enabled: false
    EXPORT_PDF:
      enabled: true
    REALTIME_NOTIFICATIONS:
      enabled: false
    BETA_API_V2:
      enabled: false
    ML_RECOMMENDATIONS:
      enabled: false
    MFA:
      enabled: true
    DARK_MODE:
      enabled: true
```

### Using Feature Flags in Code

```java
package com.example.flavour.service;

import com.example.flavour.features.ApplicationFeatures;
import org.springframework.stereotype.Service;
import org.togglz.core.manager.FeatureManager;

@Service
public class DashboardService {

    private final FeatureManager featureManager;

    public DashboardService(FeatureManager featureManager) {
        this.featureManager = featureManager;
    }

    public String getDashboard() {
        if (featureManager.isActive(ApplicationFeatures.NEW_DASHBOARD_UI)) {
            return renderNewDashboard();
        } else {
            return renderLegacyDashboard();
        }
    }

    private String renderNewDashboard() {
        return "New Dashboard with enhanced features";
    }

    private String renderLegacyDashboard() {
        return "Legacy Dashboard";
    }
}
```

### Feature Flag Annotation

```java
package com.example.flavour.controller;

import com.example.flavour.features.ApplicationFeatures;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.togglz.core.annotation.FeatureCheck;

@RestController
@RequestMapping("/api/beta")
public class BetaApiController {

    @GetMapping("/v2/users")
    @FeatureCheck(ApplicationFeatures.BETA_API_V2)
    public String getBetaUsers() {
        return "Beta API V2 - Enhanced user list";
    }
}
```

## Custom Feature Flag Implementation

### Feature Flag Entity

```java
package com.example.flavour.entity;

import jakarta.persistence.*;
import java.time.LocalDateTime;

@Entity
@Table(name = "feature_flags")
public class FeatureFlag {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String featureName;

    @Column(nullable = false)
    private boolean enabled;

    private String description;

    @Column(name = "rollout_percentage")
    private Integer rolloutPercentage = 100;

    @Column(name = "target_users")
    private String targetUsers; // Comma-separated user IDs

    @Column(name = "target_tenants")
    private String targetTenants; // Comma-separated tenant IDs

    @Column(name = "start_date")
    private LocalDateTime startDate;

    @Column(name = "end_date")
    private LocalDateTime endDate;

    @Column(name = "created_at")
    private LocalDateTime createdAt;

    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    @PrePersist
    protected void onCreate() {
        createdAt = LocalDateTime.now();
        updatedAt = LocalDateTime.now();
    }

    @PreUpdate
    protected void onUpdate() {
        updatedAt = LocalDateTime.now();
    }

    // Getters and Setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }

    public String getFeatureName() { return featureName; }
    public void setFeatureName(String featureName) { this.featureName = featureName; }

    public boolean isEnabled() { return enabled; }
    public void setEnabled(boolean enabled) { this.enabled = enabled; }

    public String getDescription() { return description; }
    public void setDescription(String description) { this.description = description; }

    public Integer getRolloutPercentage() { return rolloutPercentage; }
    public void setRolloutPercentage(Integer rolloutPercentage) { this.rolloutPercentage = rolloutPercentage; }

    public String getTargetUsers() { return targetUsers; }
    public void setTargetUsers(String targetUsers) { this.targetUsers = targetUsers; }

    public String getTargetTenants() { return targetTenants; }
    public void setTargetTenants(String targetTenants) { this.targetTenants = targetTenants; }

    public LocalDateTime getStartDate() { return startDate; }
    public void setStartDate(LocalDateTime startDate) { this.startDate = startDate; }

    public LocalDateTime getEndDate() { return endDate; }
    public void setEndDate(LocalDateTime endDate) { this.endDate = endDate; }

    public LocalDateTime getCreatedAt() { return createdAt; }
    public LocalDateTime getUpdatedAt() { return updatedAt; }
}
```

### Feature Flag Repository

```java
package com.example.flavour.repository;

import com.example.flavour.entity.FeatureFlag;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.Optional;

@Repository
public interface FeatureFlagRepository extends JpaRepository<FeatureFlag, Long> {
    
    Optional<FeatureFlag> findByFeatureName(String featureName);
    
    boolean existsByFeatureName(String featureName);
}
```

### Feature Flag Service

```java
package com.example.flavour.service;

import com.example.flavour.entity.FeatureFlag;
import com.example.flavour.repository.FeatureFlagRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

import java.time.LocalDateTime;
import java.util.Arrays;
import java.util.List;
import java.util.Random;

@Service
public class FeatureFlagService {

    @Autowired
    private FeatureFlagRepository featureFlagRepository;

    private final Random random = new Random();

    @Cacheable(value = "featureFlags", key = "#featureName")
    public boolean isFeatureEnabled(String featureName) {
        return isFeatureEnabled(featureName, null, null);
    }

    public boolean isFeatureEnabled(String featureName, String userId, String tenantId) {
        FeatureFlag flag = featureFlagRepository.findByFeatureName(featureName)
                .orElse(null);

        if (flag == null || !flag.isEnabled()) {
            return false;
        }

        // Check date range
        LocalDateTime now = LocalDateTime.now();
        if (flag.getStartDate() != null && now.isBefore(flag.getStartDate())) {
            return false;
        }
        if (flag.getEndDate() != null && now.isAfter(flag.getEndDate())) {
            return false;
        }

        // Check tenant targeting
        if (tenantId != null && flag.getTargetTenants() != null) {
            List<String> targetTenants = Arrays.asList(flag.getTargetTenants().split(","));
            if (!targetTenants.contains(tenantId)) {
                return false;
            }
        }

        // Check user targeting
        if (userId != null && flag.getTargetUsers() != null) {
            List<String> targetUsers = Arrays.asList(flag.getTargetUsers().split(","));
            if (!targetUsers.contains(userId)) {
                return false;
            }
        }

        // Check rollout percentage
        if (flag.getRolloutPercentage() < 100) {
            int userHash = userId != null ? Math.abs(userId.hashCode()) : random.nextInt();
            return (userHash % 100) < flag.getRolloutPercentage();
        }

        return true;
    }

    public FeatureFlag createFeatureFlag(FeatureFlag flag) {
        return featureFlagRepository.save(flag);
    }

    public FeatureFlag updateFeatureFlag(String featureName, FeatureFlag updates) {
        FeatureFlag existing = featureFlagRepository.findByFeatureName(featureName)
                .orElseThrow(() -> new RuntimeException("Feature flag not found"));

        existing.setEnabled(updates.isEnabled());
        existing.setDescription(updates.getDescription());
        existing.setRolloutPercentage(updates.getRolloutPercentage());
        existing.setTargetUsers(updates.getTargetUsers());
        existing.setTargetTenants(updates.getTargetTenants());
        existing.setStartDate(updates.getStartDate());
        existing.setEndDate(updates.getEndDate());

        return featureFlagRepository.save(existing);
    }

    public void deleteFeatureFlag(String featureName) {
        FeatureFlag flag = featureFlagRepository.findByFeatureName(featureName)
                .orElseThrow(() -> new RuntimeException("Feature flag not found"));
        featureFlagRepository.delete(flag);
    }

    public List<FeatureFlag> getAllFeatureFlags() {
        return featureFlagRepository.findAll();
    }
}
```

### Feature Flag Controller

```java
package com.example.flavour.controller;

import com.example.flavour.entity.FeatureFlag;
import com.example.flavour.service.FeatureFlagService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.HashMap;
import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/api/feature-flags")
public class FeatureFlagController {

    @Autowired
    private FeatureFlagService featureFlagService;

    @GetMapping
    public ResponseEntity<List<FeatureFlag>> getAllFlags() {
        return ResponseEntity.ok(featureFlagService.getAllFeatureFlags());
    }

    @GetMapping("/{featureName}/check")
    public ResponseEntity<Map<String, Boolean>> checkFeature(
            @PathVariable String featureName,
            @RequestParam(required = false) String userId,
            @RequestParam(required = false) String tenantId) {

        boolean enabled = featureFlagService.isFeatureEnabled(featureName, userId, tenantId);
        
        Map<String, Boolean> response = new HashMap<>();
        response.put("enabled", enabled);
        
        return ResponseEntity.ok(response);
    }

    @PostMapping
    public ResponseEntity<FeatureFlag> createFeatureFlag(@RequestBody FeatureFlag flag) {
        FeatureFlag created = featureFlagService.createFeatureFlag(flag);
        return ResponseEntity.ok(created);
    }

    @PutMapping("/{featureName}")
    public ResponseEntity<FeatureFlag> updateFeatureFlag(
            @PathVariable String featureName,
            @RequestBody FeatureFlag flag) {
        
        FeatureFlag updated = featureFlagService.updateFeatureFlag(featureName, flag);
        return ResponseEntity.ok(updated);
    }

    @DeleteMapping("/{featureName}")
    public ResponseEntity<Void> deleteFeatureFlag(@PathVariable String featureName) {
        featureFlagService.deleteFeatureFlag(featureName);
        return ResponseEntity.noContent().build();
    }
}
```

## Configuration Properties Pattern

### Feature Configuration Properties

```java
package com.example.flavour.config;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

import java.util.HashMap;
import java.util.Map;

@Component
@ConfigurationProperties(prefix = "features")
public class FeatureProperties {

    private boolean newUi = false;
    private boolean analytics = false;
    private boolean experimentalApi = false;
    private Map<String, Boolean> flags = new HashMap<>();

    // Getters and Setters
    public boolean isNewUi() { return newUi; }
    public void setNewUi(boolean newUi) { this.newUi = newUi; }

    public boolean isAnalytics() { return analytics; }
    public void setAnalytics(boolean analytics) { this.analytics = analytics; }

    public boolean isExperimentalApi() { return experimentalApi; }
    public void setExperimentalApi(boolean experimentalApi) { this.experimentalApi = experimentalApi; }

    public Map<String, Boolean> getFlags() { return flags; }
    public void setFlags(Map<String, Boolean> flags) { this.flags = flags; }
}
```

### Using Configuration Properties

```java
package com.example.flavour.service;

import com.example.flavour.config.FeatureProperties;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class FeatureAwareService {

    @Autowired
    private FeatureProperties featureProperties;

    public String getHomePage() {
        if (featureProperties.isNewUi()) {
            return "new-home-page";
        }
        return "legacy-home-page";
    }

    public boolean shouldTrackAnalytics() {
        return featureProperties.isAnalytics();
    }
}
```

## Conditional Annotation

### Custom Conditional Annotation

```java
package com.example.flavour.annotation;

import org.springframework.context.annotation.Conditional;

import java.lang.annotation.*;

@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(FeatureCondition.class)
public @interface ConditionalOnFeature {
    String value();
}
```

### Feature Condition

```java
package com.example.flavour.annotation;

import com.example.flavour.service.FeatureFlagService;
import org.springframework.context.annotation.Condition;
import org.springframework.context.annotation.ConditionContext;
import org.springframework.core.type.AnnotatedTypeMetadata;

public class FeatureCondition implements Condition {

    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        String featureName = (String) metadata.getAnnotationAttributes(
                ConditionalOnFeature.class.getName()).get("value");

        FeatureFlagService featureService = context.getBeanFactory()
                .getBean(FeatureFlagService.class);

        return featureService.isFeatureEnabled(featureName);
    }
}
```

### Using Conditional Annotation

```java
package com.example.flavour.service;

import com.example.flavour.annotation.ConditionalOnFeature;
import org.springframework.stereotype.Service;

@Service
@ConditionalOnFeature("ML_RECOMMENDATIONS")
public class MachineLearningService {

    public String getRecommendations() {
        return "ML-powered recommendations";
    }
}
```

## Strategy Pattern for Flavours

### Payment Strategy Interface

```java
package com.example.flavour.strategy;

public interface PaymentStrategy {
    String processPayment(double amount);
    String getProviderName();
}
```

### Multiple Implementations

```java
package com.example.flavour.strategy.impl;

import com.example.flavour.strategy.PaymentStrategy;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.stereotype.Component;

@Component
@ConditionalOnProperty(name = "payment.provider", havingValue = "stripe")
public class StripePaymentStrategy implements PaymentStrategy {

    @Override
    public String processPayment(double amount) {
        return "Processing $" + amount + " via Stripe";
    }

    @Override
    public String getProviderName() {
        return "Stripe";
    }
}
```

```java
package com.example.flavour.strategy.impl;

import com.example.flavour.strategy.PaymentStrategy;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.stereotype.Component;

@Component
@ConditionalOnProperty(name = "payment.provider", havingValue = "paypal")
public class PayPalPaymentStrategy implements PaymentStrategy {

    @Override
    public String processPayment(double amount) {
        return "Processing $" + amount + " via PayPal";
    }

    @Override
    public String getProviderName() {
        return "PayPal";
    }
}
```

### Strategy Factory

```java
package com.example.flavour.factory;

import com.example.flavour.strategy.PaymentStrategy;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import java.util.HashMap;
import java.util.List;
import java.util.Map;

@Component
public class PaymentStrategyFactory {

    private final Map<String, PaymentStrategy> strategies = new HashMap<>();

    @Autowired
    public PaymentStrategyFactory(List<PaymentStrategy> strategyList) {
        for (PaymentStrategy strategy : strategyList) {
            strategies.put(strategy.getProviderName().toLowerCase(), strategy);
        }
    }

    public PaymentStrategy getStrategy(String provider) {
        PaymentStrategy strategy = strategies.get(provider.toLowerCase());
        if (strategy == null) {
            throw new IllegalArgumentException("Unknown payment provider: " + provider);
        }
        return strategy;
    }

    public Map<String, PaymentStrategy> getAllStrategies() {
        return strategies;
    }
}
```

## Redis-Based Distributed Flags

### Redis Feature Flag Configuration

```java
package com.example.flavour.config;

import org.springframework.cache.annotation.EnableCaching;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializationContext;

import java.time.Duration;

@Configuration
@EnableCaching
public class RedisCacheConfig {

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(10))
                .serializeValuesWith(
                        RedisSerializationContext.SerializationPair.fromSerializer(
                                new GenericJackson2JsonRedisSerializer()));

        return RedisCacheManager.builder(connectionFactory)
                .cacheDefaults(config)
                .build();
    }
}
```

### Distributed Feature Flag Service

```java
package com.example.flavour.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.util.concurrent.TimeUnit;

@Service
public class DistributedFeatureFlagService {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    private static final String FLAG_PREFIX = "feature:flag:";

    public void setFeatureFlag(String featureName, boolean enabled) {
        redisTemplate.opsForValue().set(FLAG_PREFIX + featureName, enabled);
    }

    public void setFeatureFlagWithExpiry(String featureName, boolean enabled, long ttl, TimeUnit unit) {
        redisTemplate.opsForValue().set(FLAG_PREFIX + featureName, enabled, ttl, unit);
    }

    public boolean isFeatureEnabled(String featureName) {
        Boolean enabled = (Boolean) redisTemplate.opsForValue().get(FLAG_PREFIX + featureName);
        return enabled != null && enabled;
    }

    public void removeFeatureFlag(String featureName) {
        redisTemplate.delete(FLAG_PREFIX + featureName);
    }
}
```

## A/B Testing Implementation

### A/B Test Configuration

```java
package com.example.flavour.ab;

import org.springframework.stereotype.Component;

import java.util.HashMap;
import java.util.Map;
import java.util.Random;

@Component
public class ABTestService {

    private final Random random = new Random();
    private final Map<String, ABTest> tests = new HashMap<>();

    public void createTest(String testName, int percentageVariantA) {
        tests.put(testName, new ABTest(testName, percentageVariantA));
    }

    public String getVariant(String testName, String userId) {
        ABTest test = tests.get(testName);
        if (test == null) {
            return "control";
        }

        int userHash = Math.abs(userId.hashCode() % 100);
        return userHash < test.percentageVariantA ? "variant-a" : "variant-b";
    }

    public void recordConversion(String testName, String userId, String variant) {
        // Record conversion metrics
        System.out.println("Conversion recorded: " + testName + " - " + variant + " - " + userId);
    }

    static class ABTest {
        String name;
        int percentageVariantA;

        ABTest(String name, int percentageVariantA) {
            this.name = name;
            this.percentageVariantA = percentageVariantA;
        }
    }
}
```

### A/B Test Controller

```java
package com.example.flavour.controller;

import com.example.flavour.ab.ABTestService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.Map;

@RestController
@RequestMapping("/api/ab-test")
public class ABTestController {

    @Autowired
    private ABTestService abTestService;

    @GetMapping("/{testName}/variant")
    public ResponseEntity<Map<String, String>> getVariant(
            @PathVariable String testName,
            @RequestParam String userId) {

        String variant = abTestService.getVariant(testName, userId);
        return ResponseEntity.ok(Map.of("variant", variant));
    }

    @PostMapping("/{testName}/conversion")
    public ResponseEntity<Void> recordConversion(
            @PathVariable String testName,
            @RequestParam String userId,
            @RequestParam String variant) {

        abTestService.recordConversion(testName, userId, variant);
        return ResponseEntity.ok().build();
    }
}
```

## Environment-Specific Beans

### Data Source Flavouring

```java
package com.example.flavour.config;

import com.zaxxer.hikari.HikariDataSource;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import javax.sql.DataSource;

@Configuration
public class DataSourceFlavourConfig {

    @Bean
    @ConditionalOnProperty(name = "datasource.type", havingValue = "h2")
    public DataSource h2DataSource() {
        HikariDataSource dataSource = new HikariDataSource();
        dataSource.setJdbcUrl("jdbc:h2:mem:testdb");
        dataSource.setDriverClassName("org.h2.Driver");
        return dataSource;
    }

    @Bean
    @ConditionalOnProperty(name = "datasource.type", havingValue = "postgresql")
    public DataSource postgresqlDataSource() {
        HikariDataSource dataSource = new HikariDataSource();
        dataSource.setJdbcUrl("jdbc:postgresql://localhost:5432/proddb");
        dataSource.setDriverClassName("org.postgresql.Driver");
        return dataSource;
    }
}
```

## Testing Feature Flags

### Feature Flag Test

```java
package com.example.flavour.service;

import com.example.flavour.entity.FeatureFlag;
import com.example.flavour.repository.FeatureFlagRepository;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.MockitoAnnotations;

import java.util.Optional;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class FeatureFlagServiceTest {

    @Mock
    private FeatureFlagRepository repository;

    @InjectMocks
    private FeatureFlagService service;

    @BeforeEach
    void setUp() {
        MockitoAnnotations.openMocks(this);
    }

    @Test
    void testFeatureEnabled() {
        FeatureFlag flag = new FeatureFlag();
        flag.setFeatureName("TEST_FEATURE");
        flag.setEnabled(true);

        when(repository.findByFeatureName("TEST_FEATURE")).thenReturn(Optional.of(flag));

        assertTrue(service.isFeatureEnabled("TEST_FEATURE"));
    }

    @Test
    void testFeatureDisabled() {
        FeatureFlag flag = new FeatureFlag();
        flag.setFeatureName("TEST_FEATURE");
        flag.setEnabled(false);

        when(repository.findByFeatureName("TEST_FEATURE")).thenReturn(Optional.of(flag));

        assertFalse(service.isFeatureEnabled("TEST_FEATURE"));
    }

    @Test
    void testRolloutPercentage() {
        FeatureFlag flag = new FeatureFlag();
        flag.setFeatureName("TEST_FEATURE");
        flag.setEnabled(true);
        flag.setRolloutPercentage(50);

        when(repository.findByFeatureName("TEST_FEATURE")).thenReturn(Optional.of(flag));

        // Test with specific user IDs
        boolean result1 = service.isFeatureEnabled("TEST_FEATURE", "user123", null);
        boolean result2 = service.isFeatureEnabled("TEST_FEATURE", "user456", null);

        // At least one should be different due to rollout
        assertNotEquals(result1 && result2, false);
    }
}
```

## Best Practices

### 1. Feature Flag Lifecycle

```java
// Start with disabled
// Enable for internal testing
// Gradual rollout (10% -> 50% -> 100%)
// Monitor metrics
// Full rollout or rollback
// Remove flag after stabilization
```

### 2. Naming Conventions

```java
// Use descriptive names
NEW_DASHBOARD_UI          // Good
FEAT_123                  // Bad

// Use consistent prefixes
BETA_API_V2              // Beta features
EXPERIMENT_ML_RECS       // Experiments
KILLSWITCH_PAYMENTS      // Emergency switches
```

### 3. Default to Safe State

```java
// Always default to false for new features
@EnabledByDefault(false)
```

### 4. Clean Up Old Flags

```java
// Remove flags after feature is stable
// Document flag lifecycle
// Set expiry dates
```

## Monitoring and Observability

### Feature Flag Metrics

```java
package com.example.flavour.metrics;

import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Tag;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.stereotype.Component;

import java.util.Arrays;

@Aspect
@Component
public class FeatureFlagMetricsAspect {

    private final MeterRegistry meterRegistry;

    public FeatureFlagMetricsAspect(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }

    @Around("execution(* com.example.flavour.service.FeatureFlagService.isFeatureEnabled(..))")
    public Object recordFeatureCheck(ProceedingJoinPoint joinPoint) throws Throwable {
        String featureName = (String) joinPoint.getArgs()[0];
        Object result = joinPoint.proceed();
        boolean enabled = (Boolean) result;

        meterRegistry.counter("feature.flag.check",
                Arrays.asList(
                        Tag.of("feature", featureName),
                        Tag.of("enabled", String.valueOf(enabled))
                )).increment();

        return result;
    }
}
```

## Additional Resources

- [Togglz Documentation](https://www.togglz.org/)
- [Spring Profiles](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.profiles)
- [Feature Flags Best Practices](https://martinfowler.com/articles/feature-toggles.html)
- [LaunchDarkly Feature Flags](https://launchdarkly.com/)

