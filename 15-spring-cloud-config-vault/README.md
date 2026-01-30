# Spring Cloud Config with HashiCorp Vault

## Overview

Spring Cloud Config provides server-side and client-side support for externalized configuration in distributed systems. HashiCorp Vault adds secure secret management, providing a unified interface to access secrets, encryption keys, and sensitive data with fine-grained access control.

## Key Concepts

### Spring Cloud Config

- **Config Server** - Centralized configuration management
- **Config Client** - Applications consuming configuration
- **Git Backend** - Version-controlled configuration
- **Environment-specific** - Profile-based configuration
- **Encryption/Decryption** - Sensitive data protection

### HashiCorp Vault

- **Secret Storage** - Secure secret management
- **Dynamic Secrets** - On-demand credential generation
- **Encryption as a Service** - Data encryption/decryption
- **Lease and Renewal** - Time-limited credentials
- **Audit Logging** - Complete access history

## Dependencies

```xml
<dependencies>
    <!-- Config Server -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-config-server</artifactId>
    </dependency>
    
    <!-- Config Client -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-config</artifactId>
    </dependency>
    
    <!-- Vault Integration -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-vault-config</artifactId>
    </dependency>
    
    <!-- Spring Cloud Bootstrap -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-bootstrap</artifactId>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>2023.0.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

## Spring Cloud Config Server

### Config Server Setup

```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

### Config Server Properties

```yaml
# application.yml
server:
  port: 8888

spring:
  application:
    name: config-server
  cloud:
    config:
      server:
        git:
          uri: https://github.com/your-org/config-repo
          default-label: main
          search-paths: '{application}'
          clone-on-start: true
          force-pull: true
        # Native profile for local file system
        native:
          search-locations: classpath:/config
  security:
    user:
      name: configuser
      password: ${CONFIG_PASSWORD:configpass}

management:
  endpoints:
    web:
      exposure:
        include: '*'
```

### Multiple Backend Support

```yaml
spring:
  profiles:
    active: composite
  cloud:
    config:
      server:
        composite:
          - type: git
            uri: https://github.com/your-org/config-repo
            search-paths: common
          - type: vault
            host: localhost
            port: 8200
            scheme: http
            backend: secret
          - type: native
            search-locations: file:///config
```

### Security Configuration

```java
@Configuration
@EnableWebSecurity
public class ConfigServerSecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/actuator/health").permitAll()
                .anyRequest().authenticated()
            )
            .httpBasic(withDefaults())
            .csrf(csrf -> csrf.disable());
        
        return http.build();
    }
}
```

## Spring Cloud Config Client

### Client Configuration

```yaml
# bootstrap.yml
spring:
  application:
    name: my-service
  cloud:
    config:
      uri: http://localhost:8888
      username: configuser
      password: configpass
      fail-fast: true
      retry:
        initial-interval: 1000
        max-attempts: 6
        multiplier: 1.1
  config:
    import: optional:configserver:http://localhost:8888
```

### Using Configuration Properties

```java
@Configuration
@ConfigurationProperties(prefix = "app")
@Data
public class AppConfig {
    private String name;
    private String version;
    private Database database;
    private Security security;
    
    @Data
    public static class Database {
        private String url;
        private String username;
        private String password;
        private int maxPoolSize;
    }
    
    @Data
    public static class Security {
        private String jwtSecret;
        private long jwtExpiration;
    }
}
```

### Refreshable Configuration

```java
@RestController
@RefreshScope
public class ConfigController {
    
    @Value("${app.message:default}")
    private String message;
    
    @Autowired
    private AppConfig appConfig;
    
    @GetMapping("/config")
    public Map<String, Object> getConfig() {
        Map<String, Object> config = new HashMap<>();
        config.put("message", message);
        config.put("appName", appConfig.getName());
        config.put("version", appConfig.getVersion());
        return config;
    }
}
```

### Manual Refresh

```java
@RestController
@RequestMapping("/admin")
public class AdminController {
    
    @Autowired
    private RefreshEndpoint refreshEndpoint;
    
    @PostMapping("/refresh")
    public Collection<String> refresh() {
        return refreshEndpoint.refresh();
    }
}
```

## HashiCorp Vault Integration

### Vault Setup

```bash
# Start Vault in dev mode
vault server -dev -dev-root-token-id="root"

# Set environment variable
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='root'

# Enable KV secrets engine
vault secrets enable -path=secret kv-v2

# Write secrets
vault kv put secret/my-service \
    database.password=dbpassword \
    api.key=apikey123 \
    jwt.secret=jwtsecret456

# Write application-specific secrets
vault kv put secret/my-service/dev \
    database.url=jdbc:postgresql://localhost:5432/dev_db

vault kv put secret/my-service/prod \
    database.url=jdbc:postgresql://prod-db:5432/prod_db
```

### Vault Configuration

```yaml
# bootstrap.yml
spring:
  application:
    name: my-service
  profiles:
    active: dev
  cloud:
    vault:
      enabled: true
      uri: http://localhost:8200
      token: root
      scheme: http
      authentication: TOKEN
      kv:
        enabled: true
        backend: secret
        default-context: my-service
        profile-separator: '/'
      fail-fast: true
```

### Vault with AppRole Authentication

```yaml
spring:
  cloud:
    vault:
      authentication: APPROLE
      app-role:
        role-id: ${VAULT_ROLE_ID}
        secret-id: ${VAULT_SECRET_ID}
        role: my-service-role
        app-role-path: approle
```

```bash
# Enable AppRole auth
vault auth enable approle

# Create policy
vault policy write my-service-policy - <<EOF
path "secret/data/my-service/*" {
  capabilities = ["read"]
}
EOF

# Create role
vault write auth/approle/role/my-service-role \
    token_policies="my-service-policy" \
    token_ttl=1h \
    token_max_ttl=4h

# Get Role ID
vault read auth/approle/role/my-service-role/role-id

# Generate Secret ID
vault write -f auth/approle/role/my-service-role/secret-id
```

### Vault with Kubernetes Authentication

```yaml
spring:
  cloud:
    vault:
      authentication: KUBERNETES
      kubernetes:
        role: my-service-role
        kubernetes-path: kubernetes
        service-account-token-file: /var/run/secrets/kubernetes.io/serviceaccount/token
```

## Advanced Vault Features

### Dynamic Database Credentials

```yaml
spring:
  cloud:
    vault:
      database:
        enabled: true
        role: my-service-role
        backend: database
```

```bash
# Enable database secrets engine
vault secrets enable database

# Configure PostgreSQL connection
vault write database/config/postgresql \
    plugin_name=postgresql-database-plugin \
    allowed_roles="my-service-role" \
    connection_url="postgresql://{{username}}:{{password}}@localhost:5432/mydb" \
    username="vaultadmin" \
    password="vaultpass"

# Create role
vault write database/roles/my-service-role \
    db_name=postgresql \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
        GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
    default_ttl="1h" \
    max_ttl="24h"
```

```java
@Configuration
public class VaultDatabaseConfig {
    
    @Bean
    @VaultPropertySource(value = "database/creds/my-service-role", renewal = VaultPropertySource.Renewal.ROTATE)
    public VaultConfigurer vaultConfigurer() {
        return new VaultConfigurer() {
            @Override
            public void addSecretBackends(SecretBackendConfigurer configurer) {
                configurer.add("database");
            }
        };
    }
}
```

### Transit Encryption Engine

```java
@Service
public class EncryptionService {
    
    @Autowired
    private VaultOperations vaultOperations;
    
    private static final String TRANSIT_PATH = "transit/encrypt/my-key";
    private static final String DECRYPT_PATH = "transit/decrypt/my-key";
    
    public String encrypt(String plaintext) {
        Map<String, String> request = Map.of("plaintext", 
                Base64.getEncoder().encodeToString(plaintext.getBytes()));
        
        VaultResponse response = vaultOperations.write(TRANSIT_PATH, request);
        return response.getData().get("ciphertext").toString();
    }
    
    public String decrypt(String ciphertext) {
        Map<String, String> request = Map.of("ciphertext", ciphertext);
        
        VaultResponse response = vaultOperations.write(DECRYPT_PATH, request);
        String base64Plaintext = response.getData().get("plaintext").toString();
        return new String(Base64.getDecoder().decode(base64Plaintext));
    }
}
```

```bash
# Enable transit engine
vault secrets enable transit

# Create encryption key
vault write -f transit/keys/my-key
```

### PKI Secrets Engine

```bash
# Enable PKI engine
vault secrets enable pki

# Configure max lease TTL
vault secrets tune -max-lease-ttl=87600h pki

# Generate root certificate
vault write -field=certificate pki/root/generate/internal \
    common_name="Example Inc." \
    ttl=87600h > CA_cert.crt

# Configure CA and CRL URLs
vault write pki/config/urls \
    issuing_certificates="http://127.0.0.1:8200/v1/pki/ca" \
    crl_distribution_points="http://127.0.0.1:8200/v1/pki/crl"

# Create role
vault write pki/roles/my-service \
    allowed_domains="example.com" \
    allow_subdomains=true \
    max_ttl="720h"
```

```java
@Configuration
public class VaultPKIConfig {
    
    @Bean
    public RestTemplate restTemplate(VaultOperations vaultOperations) throws Exception {
        // Request certificate from Vault
        Map<String, String> request = Map.of(
                "common_name", "my-service.example.com",
                "ttl", "24h"
        );
        
        VaultResponse response = vaultOperations.write("pki/issue/my-service", request);
        
        String certificate = response.getData().get("certificate").toString();
        String privateKey = response.getData().get("private_key").toString();
        
        // Configure SSL context with the certificate
        SSLContext sslContext = createSSLContext(certificate, privateKey);
        
        HttpClient httpClient = HttpClients.custom()
                .setSSLContext(sslContext)
                .build();
        
        HttpComponentsClientHttpRequestFactory factory = 
                new HttpComponentsClientHttpRequestFactory(httpClient);
        
        return new RestTemplate(factory);
    }
    
    private SSLContext createSSLContext(String certificate, String privateKey) {
        // Implementation to create SSL context from certificate and private key
        return null;
    }
}
```

## Configuration Repository Structure

### Git Repository Layout

```
config-repo/
├── application.yml                 # Default configuration for all services
├── application-dev.yml             # Dev environment defaults
├── application-prod.yml            # Prod environment defaults
├── my-service/
│   ├── my-service.yml              # Service-specific defaults
│   ├── my-service-dev.yml          # Service dev configuration
│   └── my-service-prod.yml         # Service prod configuration
├── user-service/
│   ├── user-service.yml
│   ├── user-service-dev.yml
│   └── user-service-prod.yml
└── shared/
    ├── database.yml                # Shared database config
    └── messaging.yml               # Shared messaging config
```

### application.yml (Default)

```yaml
# Common configuration for all services
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      show-details: when-authorized

logging:
  level:
    root: INFO
    org.springframework: INFO

app:
  common:
    timeout: 30000
    retry:
      max-attempts: 3
      backoff: 1000
```

### my-service.yml

```yaml
app:
  name: My Service
  version: 1.0.0
  features:
    feature-a: true
    feature-b: false

server:
  port: 8080
  compression:
    enabled: true

spring:
  datasource:
    hikari:
      maximum-pool-size: 10
      minimum-idle: 5
      connection-timeout: 30000
```

### my-service-dev.yml

```yaml
app:
  debug: true

spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/dev_db
    username: dev_user
    # Password from Vault

logging:
  level:
    root: DEBUG
    com.example: TRACE
```

### my-service-prod.yml

```yaml
app:
  debug: false

spring:
  datasource:
    url: jdbc:postgresql://prod-db.example.com:5432/prod_db
    username: prod_user
    # Password from Vault
    hikari:
      maximum-pool-size: 50

logging:
  level:
    root: WARN
    com.example: INFO
```

## Encryption in Config Server

### Symmetric Encryption

```yaml
# Config Server application.yml
encrypt:
  key: mySecretKey123456789012345678901234567890
```

```bash
# Encrypt value
curl http://localhost:8888/encrypt -d "mysecretpassword"
# Returns: {cipher}AQA...encrypted...value

# Decrypt value
curl http://localhost:8888/decrypt -d "{cipher}AQA...encrypted...value"
```

### Using Encrypted Values

```yaml
# my-service.yml
spring:
  datasource:
    password: '{cipher}AQA...encrypted...value'
    
app:
  api:
    key: '{cipher}AQB...another...encrypted...value'
```

### Asymmetric Encryption (RSA)

```bash
# Generate key pair
keytool -genkeypair -alias configserver -keyalg RSA \
    -dname "CN=Config Server,OU=Unit,O=Org,L=City,S=State,C=US" \
    -keypass password -keystore configserver.jks \
    -storepass password
```

```yaml
# Config Server application.yml
encrypt:
  key-store:
    location: classpath:/configserver.jks
    password: password
    alias: configserver
    secret: password
```

## Monitoring and Health Checks

### Config Server Actuator

```java
@RestController
@RequestMapping("/admin")
public class ConfigServerMonitoringController {
    
    @Autowired
    private Environment environment;
    
    @GetMapping("/properties")
    public Map<String, Object> getAllProperties() {
        Map<String, Object> properties = new HashMap<>();
        
        for (PropertySource<?> propertySource : 
                ((AbstractEnvironment) environment).getPropertySources()) {
            if (propertySource instanceof EnumerablePropertySource) {
                for (String key : ((EnumerablePropertySource<?>) propertySource).getPropertyNames()) {
                    properties.put(key, environment.getProperty(key));
                }
            }
        }
        
        return properties;
    }
}
```

### Vault Health Indicator

```java
@Component
public class VaultHealthIndicator implements HealthIndicator {
    
    @Autowired
    private VaultOperations vaultOperations;
    
    @Override
    public Health health() {
        try {
            VaultHealth vaultHealth = vaultOperations.opsForSys().health();
            
            if (vaultHealth.isInitialized() && !vaultHealth.isSealed()) {
                return Health.up()
                        .withDetail("initialized", vaultHealth.isInitialized())
                        .withDetail("sealed", vaultHealth.isSealed())
                        .withDetail("version", vaultHealth.getVersion())
                        .build();
            } else {
                return Health.down()
                        .withDetail("initialized", vaultHealth.isInitialized())
                        .withDetail("sealed", vaultHealth.isSealed())
                        .build();
            }
        } catch (Exception e) {
            return Health.down()
                    .withException(e)
                    .build();
        }
    }
}
```

## Testing

### Config Server Test

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@TestPropertySource(properties = {
    "spring.cloud.config.server.git.uri=classpath:/test-config"
})
class ConfigServerTest {
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Test
    void testConfigRetrieval() {
        ResponseEntity<Map> response = restTemplate
                .withBasicAuth("configuser", "configpass")
                .getForEntity("/my-service/dev", Map.class);
        
        assertEquals(HttpStatus.OK, response.getStatusCode());
        assertNotNull(response.getBody());
    }
}
```

### Config Client Test

```java
@SpringBootTest
@TestPropertySource(properties = {
    "spring.cloud.config.enabled=false"
})
class ConfigClientTest {
    
    @Autowired
    private AppConfig appConfig;
    
    @Test
    void testConfigurationProperties() {
        assertNotNull(appConfig.getName());
        assertNotNull(appConfig.getDatabase());
        assertNotNull(appConfig.getDatabase().getUrl());
    }
}
```

## Docker Compose Setup

```yaml
version: '3.8'

services:
  vault:
    image: hashicorp/vault:latest
    container_name: vault
    ports:
      - "8200:8200"
    environment:
      VAULT_DEV_ROOT_TOKEN_ID: root
      VAULT_DEV_LISTEN_ADDRESS: 0.0.0.0:8200
    cap_add:
      - IPC_LOCK
    command: server -dev

  config-server:
    build: ./config-server
    container_name: config-server
    ports:
      - "8888:8888"
    environment:
      SPRING_PROFILES_ACTIVE: native,vault
      SPRING_CLOUD_VAULT_URI: http://vault:8200
      SPRING_CLOUD_VAULT_TOKEN: root
    depends_on:
      - vault

  my-service:
    build: ./my-service
    container_name: my-service
    ports:
      - "8080:8080"
    environment:
      SPRING_PROFILES_ACTIVE: dev
      SPRING_CLOUD_CONFIG_URI: http://config-server:8888
      SPRING_CLOUD_VAULT_URI: http://vault:8200
      SPRING_CLOUD_VAULT_TOKEN: root
    depends_on:
      - config-server
      - vault
```

## Best Practices

1. **Use Git for configuration** - Version control and audit trail
2. **Encrypt sensitive data** - Use encryption or Vault
3. **Environment-specific profiles** - Separate dev/prod configs
4. **Implement health checks** - Monitor config and Vault health
5. **Use AppRole authentication** - For production Vault access
6. **Rotate secrets regularly** - Use dynamic secrets when possible
7. **Implement retry logic** - Handle config server failures
8. **Cache configurations** - Reduce load on config server
9. **Use refresh scope** - For dynamic configuration updates
10. **Audit access** - Enable Vault audit logging

## Common Pitfalls

1. Hardcoding secrets in configuration files
2. Not using encryption for sensitive data
3. Sharing Vault tokens across environments
4. Not implementing proper error handling
5. Ignoring certificate validation in production
6. Not rotating secrets regularly
7. Exposing config endpoints without authentication
8. Not monitoring config and Vault health

## Resources

- [Spring Cloud Config Documentation](https://docs.spring.io/spring-cloud-config/docs/current/reference/html/)
- [HashiCorp Vault Documentation](https://developer.hashicorp.com/vault/docs)
- [Spring Cloud Vault Documentation](https://docs.spring.io/spring-cloud-vault/docs/current/reference/html/)
- [Vault Best Practices](https://developer.hashicorp.com/vault/tutorials/operations/production-hardening)
