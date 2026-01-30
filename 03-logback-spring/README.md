# Logback Spring

## Overview

Logback is the native logging framework for Spring Boot, providing powerful and flexible logging capabilities. It's the successor to Log4j and offers better performance, more features, and seamless integration with Spring Boot.

## Key Concepts

### 1. Basic Configuration

```xml
<!-- src/main/resources/logback-spring.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    
    <!-- Console Appender -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>
    
    <!-- File Appender -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/application.log</file>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/application-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy 
                class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>10MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
    </appender>
    
    <!-- Root Logger -->
    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="FILE"/>
    </root>
    
</configuration>
```

### 2. Spring Profiles

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    
    <!-- Development Profile -->
    <springProfile name="dev">
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>%d{HH:mm:ss.SSS} %highlight(%-5level) %cyan(%logger{36}) - %msg%n</pattern>
            </encoder>
        </appender>
        
        <root level="DEBUG">
            <appender-ref ref="CONSOLE"/>
        </root>
    </springProfile>
    
    <!-- Production Profile -->
    <springProfile name="prod">
        <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
            <file>/var/log/myapp/application.log</file>
            <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
            <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
                <fileNamePattern>/var/log/myapp/application-%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
                <timeBasedFileNamingAndTriggeringPolicy 
                    class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                    <maxFileSize>100MB</maxFileSize>
                </timeBasedFileNamingAndTriggeringPolicy>
                <maxHistory>60</maxHistory>
            </rollingPolicy>
        </appender>
        
        <root level="INFO">
            <appender-ref ref="FILE"/>
        </root>
    </springProfile>
    
</configuration>
```

### 3. JSON Logging (Logstash Format)

```xml
<!-- Add dependency -->
<!-- 
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>7.4</version>
</dependency>
-->

<configuration>
    <appender name="JSON_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/application.json</file>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <customFields>{"app":"myapp","env":"${spring.profiles.active}"}</customFields>
        </encoder>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/application-%d{yyyy-MM-dd}.json.gz</fileNamePattern>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
    </appender>
    
    <root level="INFO">
        <appender-ref ref="JSON_FILE"/>
    </root>
</configuration>
```

### 4. Async Logging

```xml
<configuration>
    
    <!-- Regular appender -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/application.log</file>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/application-%d{yyyy-MM-dd}.log</fileNamePattern>
        </rollingPolicy>
    </appender>
    
    <!-- Async wrapper -->
    <appender name="ASYNC_FILE" class="ch.qos.logback.classic.AsyncAppender">
        <queueSize>512</queueSize>
        <discardingThreshold>0</discardingThreshold>
        <appender-ref ref="FILE"/>
    </appender>
    
    <root level="INFO">
        <appender-ref ref="ASYNC_FILE"/>
    </root>
    
</configuration>
```

### 5. Package-Level Logging

```xml
<configuration>
    
    <!-- Set specific package log levels -->
    <logger name="com.example.myapp.service" level="DEBUG"/>
    <logger name="com.example.myapp.repository" level="TRACE"/>
    <logger name="org.springframework.web" level="INFO"/>
    <logger name="org.hibernate.SQL" level="DEBUG"/>
    <logger name="org.hibernate.type.descriptor.sql.BasicBinder" level="TRACE"/>
    
    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
    </root>
    
</configuration>
```

### 6. Custom Patterns

```xml
<configuration>
    
    <!-- Define custom pattern -->
    <property name="CUSTOM_PATTERN" 
        value="%d{yyyy-MM-dd HH:mm:ss.SSS} | %X{requestId} | %X{userId} | [%thread] | %-5level | %logger{36} | %msg%n"/>
    
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>${CUSTOM_PATTERN}</pattern>
        </encoder>
    </appender>
    
    <!-- Color-coded console for development -->
    <property name="COLOR_PATTERN" 
        value="%d{HH:mm:ss.SSS} | %highlight(%-5level) | %cyan([%thread]) | %yellow(%logger{36}) | %msg%n"/>
    
</configuration>
```

### 7. MDC (Mapped Diagnostic Context)

Add contextual information to logs:

```java
import org.slf4j.MDC;

@Component
public class LoggingFilter extends OncePerRequestFilter {
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, 
                                    HttpServletResponse response, 
                                    FilterChain filterChain) throws ServletException, IOException {
        try {
            // Add request ID to MDC
            MDC.put("requestId", UUID.randomUUID().toString());
            MDC.put("userId", extractUserId(request));
            MDC.put("ipAddress", request.getRemoteAddr());
            
            filterChain.doFilter(request, response);
        } finally {
            // Always clear MDC
            MDC.clear();
        }
    }
}

// Usage in service
@Service
public class UserService {
    private static final Logger log = LoggerFactory.getLogger(UserService.class);
    
    public void processUser(String userId) {
        MDC.put("userId", userId);
        log.info("Processing user"); // Will include userId in log
    }
}
```

### 8. Structured Logging

```java
import net.logstash.logback.argument.StructuredArguments;
import static net.logstash.logback.argument.StructuredArguments.*;

@Service
public class OrderService {
    private static final Logger log = LoggerFactory.getLogger(OrderService.class);
    
    public void createOrder(Order order) {
        log.info("Order created", 
            keyValue("orderId", order.getId()),
            keyValue("userId", order.getUserId()),
            keyValue("amount", order.getAmount()),
            keyValue("status", order.getStatus())
        );
    }
}
```

### 9. Conditional Logging

```xml
<configuration>
    
    <!-- Turbo Filter for conditional logging -->
    <turboFilter class="ch.qos.logback.classic.turbo.MDCFilter">
        <MDCKey>debug-enabled</MDCKey>
        <Value>true</Value>
        <OnMatch>ACCEPT</OnMatch>
    </turboFilter>
    
</configuration>
```

### 10. Error Logging with Stack Traces

```java
@Service
public class PaymentService {
    private static final Logger log = LoggerFactory.getLogger(PaymentService.class);
    
    public void processPayment(Payment payment) {
        try {
            // Process payment
        } catch (PaymentException e) {
            log.error("Payment processing failed for orderId: {}", 
                payment.getOrderId(), e); // Stack trace included
        }
    }
}
```

## POC Requirements

Create a logging configuration that demonstrates:

1. **Environment-Specific Configs**: Different logging for dev, staging, and prod
2. **Multiple Appenders**: Console, file, and JSON logging
3. **MDC Usage**: Request tracking across the application
4. **Structured Logging**: JSON format with custom fields
5. **Async Logging**: For high-throughput scenarios
6. **Log Rotation**: Size and time-based rotation policies

## Application Properties

```yaml
# application.yml
logging:
  level:
    root: INFO
    com.example.myapp: DEBUG
    org.springframework.web: INFO
    org.hibernate: INFO
  pattern:
    console: "%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n"
    file: "%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n"
  file:
    name: logs/application.log
    max-size: 10MB
    max-history: 30
```

## Best Practices

1. **Use Appropriate Log Levels**:
   - ERROR: System errors, exceptions
   - WARN: Potential issues
   - INFO: Important business events
   - DEBUG: Detailed flow information
   - TRACE: Very detailed information

2. **Avoid Logging Sensitive Data**: Never log passwords, tokens, PII
3. **Use Parameterized Logging**: `log.info("User {}", userId)` not string concatenation
4. **Structured Logging**: Use JSON format for production
5. **Log Rotation**: Implement proper rotation to manage disk space
6. **Async Logging**: Use for high-throughput applications

## Common Patterns

```xml
<!-- Pattern elements -->
%d{pattern}           - Date/time
%thread               - Thread name
%-5level              - Log level (left-justified, 5 chars)
%logger{length}       - Logger name (shortened to length)
%msg                  - Log message
%n                    - Newline
%X{key}               - MDC value
%highlight()          - Color coding (console only)
%cyan()              - Color text cyan
```

## Integration with Observability

```xml
<!-- For integration with Loki -->
<appender name="LOKI" class="com.github.loki4j.logback.Loki4jAppender">
    <http>
        <url>http://localhost:3100/loki/api/v1/push</url>
    </http>
    <format>
        <label>
            <pattern>app=myapp,host=${HOSTNAME},level=%level</pattern>
        </label>
        <message>
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </message>
    </format>
</appender>
```

## Additional Resources

- [Logback Documentation](https://logback.qos.ch/documentation.html)
- [Spring Boot Logging](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.logging)

## Next Steps

Continue to [Swagger/OpenAPI](../04-swagger/README.md) to learn about API documentation.