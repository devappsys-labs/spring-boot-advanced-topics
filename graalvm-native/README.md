# GraalVM Native Image

## Overview

GraalVM Native Image compiles your Spring Boot application ahead-of-time (AOT) into a standalone native executable. Instead of running on the JVM, the resulting binary starts in milliseconds and uses a fraction of the memory. This makes it ideal for serverless functions, containerized microservices, and CLI tools where startup time and memory footprint matter. Spring Boot 3+ has first-class support for GraalVM native compilation.

## Key Concepts

### Why Native Image?

- **Near-instant startup** - Milliseconds instead of seconds
- **Low memory footprint** - Fraction of JVM memory usage
- **Smaller container images** - No JDK required in the image
- **Predictable performance** - No JIT warm-up, consistent from first request
- **Ideal for serverless** - Cold starts become negligible

### How It Works

```text
Traditional JVM:
  .java → .class (bytecode) → JVM interprets/JIT compiles at runtime

Native Image:
  .java → .class → Static Analysis → AOT Compilation → Native Binary
                        │
                        ├── Reachability analysis (finds all used classes)
                        ├── Closed-world assumption (no dynamic class loading)
                        └── Initializes static state at build time
```

### Tradeoffs

| Aspect | JVM | Native Image |
|--------|-----|--------------|
| Startup time | Seconds | Milliseconds |
| Memory usage | Higher | Much lower |
| Peak throughput | Higher (JIT optimized) | Slightly lower |
| Build time | Fast | Slow (minutes) |
| Reflection | Unrestricted | Requires configuration |
| Dynamic proxies | Unrestricted | Requires configuration |
| Debugging | Full support | Limited |

## Prerequisites

### Install GraalVM

```bash
# Using SDKMAN
sdk install java 21.0.2-graal
sdk use java 21.0.2-graal

# Verify installation
java -version
native-image --version
```

### Dependencies

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.0</version>
</parent>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.graalvm.buildtools</groupId>
            <artifactId>native-maven-plugin</artifactId>
        </plugin>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>

<!-- Native profile -->
<profiles>
    <profile>
        <id>native</id>
        <build>
            <plugins>
                <plugin>
                    <groupId>org.graalvm.buildtools</groupId>
                    <artifactId>native-maven-plugin</artifactId>
                    <configuration>
                        <buildArgs>
                            <buildArg>--no-fallback</buildArg>
                            <buildArg>-H:+ReportExceptionStackTraces</buildArg>
                        </buildArgs>
                    </configuration>
                </plugin>
            </plugins>
        </build>
    </profile>
</profiles>
```

### Gradle

```groovy
plugins {
    id 'org.springframework.boot' version '3.3.0'
    id 'io.spring.dependency-management' version '1.1.5'
    id 'org.graalvm.buildtools.native' version '0.10.2'
    id 'java'
}

graalvmNative {
    binaries {
        main {
            buildArgs.add('--no-fallback')
            buildArgs.add('-H:+ReportExceptionStackTraces')
        }
    }
}
```

## Building a Native Image

### Using Maven

```bash
# Build native image directly
./mvnw -Pnative native:compile

# Build a native container image (no local GraalVM needed)
./mvnw -Pnative spring-boot:build-image

# The binary will be at target/<app-name>
./target/my-app
```

### Using Gradle

```bash
# Build native image
./gradlew nativeCompile

# Build native container image
./gradlew bootBuildImage

# Run the binary
./build/native/nativeCompile/my-app
```

### Build Output Comparison

```text
JVM JAR:
  target/my-app.jar          → ~45 MB
  Startup time               → ~2.5 seconds
  Memory (RSS at idle)       → ~250 MB

Native Image:
  target/my-app              → ~80 MB (self-contained)
  Startup time               → ~0.05 seconds (50ms)
  Memory (RSS at idle)       → ~50 MB
```

## Spring AOT Processing

Spring Boot's AOT engine runs at build time to analyze your application and generate optimized code.

### What AOT Does

```text
Build time:
  ├── Evaluates @Conditional annotations
  ├── Generates bean definitions (no runtime reflection)
  ├── Pre-computes configuration property bindings
  ├── Generates reflection/proxy/resource hints
  └── Serializes bean factory initialization

Runtime:
  └── Loads pre-computed bean factory (no classpath scanning)
```

### AOT-Compatible Application

```java
@SpringBootApplication
public class NativeApplication {

    public static void main(String[] args) {
        SpringApplication.run(NativeApplication.class, args);
    }
}

@RestController
@RequestMapping("/api/products")
public class ProductController {

    private final ProductRepository productRepository;

    public ProductController(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    @GetMapping
    public List<Product> findAll() {
        return productRepository.findAll();
    }

    @GetMapping("/{id}")
    public ResponseEntity<Product> findById(@PathVariable Long id) {
        return productRepository.findById(id)
                .map(ResponseEntity::ok)
                .orElse(ResponseEntity.notFound().build());
    }

    @PostMapping
    public ResponseEntity<Product> create(@RequestBody @Valid Product product) {
        Product saved = productRepository.save(product);
        return ResponseEntity.status(HttpStatus.CREATED).body(saved);
    }
}

@Entity
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    private String description;

    @Column(nullable = false)
    private BigDecimal price;
}

public interface ProductRepository extends JpaRepository<Product, Long> {
}
```

## Handling Reflection

Native image uses closed-world analysis - it must know all reflectively accessed classes at build time.

### Automatic Hints (Spring Boot 3+)

Spring Boot automatically generates hints for:
- JPA entities
- Spring beans
- Configuration properties
- Jackson serialization

### Custom Runtime Hints

```java
@Configuration
@ImportRuntimeHints(AppRuntimeHints.class)
public class NativeConfig {
}

public class AppRuntimeHints implements RuntimeHintsRegistrar {

    @Override
    public void registerHints(RuntimeHints hints, ClassLoader classLoader) {
        // Register reflection for classes used via reflection
        hints.reflection()
                .registerType(CustomDto.class, MemberCategory.values())
                .registerType(ExternalLibraryClass.class,
                        MemberCategory.INVOKE_PUBLIC_CONSTRUCTORS,
                        MemberCategory.INVOKE_PUBLIC_METHODS);

        // Register resources
        hints.resources()
                .registerPattern("templates/*.html")
                .registerPattern("static/**");

        // Register JDK proxies
        hints.proxies()
                .registerJdkProxy(MyInterface.class);

        // Register serialization
        hints.serialization()
                .registerType(MySerializableClass.class);
    }
}
```

### @RegisterReflectionForBinding

```java
@RestController
@RequestMapping("/api/external")
public class ExternalApiController {

    // Automatically registers ExternalResponse for reflection
    @RegisterReflectionForBinding(ExternalResponse.class)
    @GetMapping("/data")
    public ExternalResponse getData() {
        return externalClient.fetchData();
    }
}

// This class needs reflection hints since it's from an external library
// or used only via Jackson deserialization
public record ExternalResponse(
        String id,
        String name,
        Map<String, Object> metadata
) {
}
```

## Handling Dynamic Proxies

```java
// JDK proxies (interfaces) - need explicit registration
public interface AuditService {
    void audit(String action);
}

// Register the proxy
public class ProxyHints implements RuntimeHintsRegistrar {
    @Override
    public void registerHints(RuntimeHints hints, ClassLoader classLoader) {
        hints.proxies().registerJdkProxy(AuditService.class);
    }
}
```

## Configuration Properties in Native

```java
// This works out of the box in native
@ConfigurationProperties(prefix = "app")
@Validated
public record AppProperties(
        @NotBlank String name,
        @NotNull Integer maxRetries,
        @NotNull Duration timeout,
        DatabaseProperties database
) {
    public record DatabaseProperties(
            String host,
            int port,
            String name
    ) {}
}
```

```yaml
app:
  name: my-native-app
  max-retries: 3
  timeout: 5s
  database:
    host: localhost
    port: 5432
    name: mydb
```

## Docker Native Image

### Buildpack (Recommended)

```bash
# Build native image using Cloud Native Buildpacks
./mvnw -Pnative spring-boot:build-image \
  -Dspring-boot.build-image.imageName=my-app:native

# Run
docker run --rm -p 8080:8080 my-app:native
```

### Multi-stage Dockerfile

```dockerfile
# Stage 1: Build native image
FROM ghcr.io/graalvm/graalvm-community:21 AS builder
WORKDIR /build
COPY . .
RUN ./mvnw -Pnative native:compile -DskipTests

# Stage 2: Minimal runtime image
FROM debian:bookworm-slim
WORKDIR /app
COPY --from=builder /build/target/my-app /app/my-app
EXPOSE 8080
ENTRYPOINT ["/app/my-app"]
```

```bash
docker build -t my-app:native .
docker run --rm -p 8080:8080 my-app:native
```

### Image Size Comparison

```text
JVM image (eclipse-temurin:21-jre):     ~350 MB
Native image (debian-slim):              ~90 MB
Native image (distroless):               ~85 MB
Native image (scratch/static):           ~75 MB
```

## Testing Native Applications

### Running Tests in Native Mode

```bash
# Run tests as native image (validates AOT/native compatibility)
./mvnw -Pnative native:test
```

### Test with AOT Processing

```java
@SpringBootTest
class ProductControllerTest {

    @Autowired
    private ProductRepository productRepository;

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void shouldCreateAndFetchProduct() {
        Product product = new Product(null, "Widget", "A widget", new BigDecimal("9.99"));

        ResponseEntity<Product> createResponse = restTemplate.postForEntity(
                "/api/products", product, Product.class);
        assertThat(createResponse.getStatusCode()).isEqualTo(HttpStatus.CREATED);

        Long id = createResponse.getBody().getId();
        ResponseEntity<Product> getResponse = restTemplate.getForEntity(
                "/api/products/" + id, Product.class);
        assertThat(getResponse.getBody().getName()).isEqualTo("Widget");
    }
}
```

### RuntimeHints Testing

```java
class AppRuntimeHintsTest {

    @Test
    void shouldRegisterAllHints() {
        RuntimeHints hints = new RuntimeHints();
        new AppRuntimeHints().registerHints(hints, getClass().getClassLoader());

        // Verify reflection hints
        assertThat(RuntimeHintsPredicates.reflection()
                .onType(CustomDto.class))
                .accepts(hints);

        // Verify resource hints
        assertThat(RuntimeHintsPredicates.resource()
                .forResource("templates/index.html"))
                .accepts(hints);

        // Verify proxy hints
        assertThat(RuntimeHintsPredicates.proxies()
                .forInterfaces(MyInterface.class))
                .accepts(hints);
    }
}
```

## Tracing Agent (For Discovering Hints)

When migrating existing apps, use the tracing agent to discover what needs hints.

```bash
# Run with the tracing agent to collect metadata
java -agentlib:native-image-agent=config-output-dir=src/main/resources/META-INF/native-image \
     -jar target/my-app.jar

# Exercise the app (call all endpoints, run all code paths)
# Then stop the app

# Generated files:
# src/main/resources/META-INF/native-image/
#   ├── reflect-config.json
#   ├── proxy-config.json
#   ├── resource-config.json
#   ├── serialization-config.json
#   └── jni-config.json
```

## Common Libraries Compatibility

### Libraries with Native Support

| Library | Status | Notes |
|---------|--------|-------|
| Spring Data JPA | Supported | Entities auto-detected |
| Spring Security | Supported | Full AOT support |
| Spring Cache | Supported | Caffeine recommended |
| Jackson | Supported | Records work best |
| Lombok | Supported | Compile-time only |
| MapStruct | Supported | Compile-time only |
| Flyway | Supported | With native hints |
| Hibernate | Supported | Via Spring Data |

### Libraries Requiring Extra Configuration

```java
// Example: Registering hints for a library that uses reflection
public class LibraryHints implements RuntimeHintsRegistrar {
    @Override
    public void registerHints(RuntimeHints hints, ClassLoader classLoader) {
        // Register all classes the library accesses reflectively
        hints.reflection()
                .registerType(SomeLibraryClass.class, MemberCategory.values());
    }
}
```

## Profile-Guided Optimization (PGO)

Improve native image performance by providing a usage profile.

```bash
# Step 1: Build an instrumented image
native-image --pgo-instrument -jar target/my-app.jar

# Step 2: Run the instrumented image under typical load
./my-app
# ... run load tests, exercise all code paths ...
# Stop → generates default.iprof

# Step 3: Build optimized image with the profile
native-image --pgo=default.iprof -jar target/my-app.jar
```

## Best Practices

1. **Use records for DTOs** - Records are natively supported without reflection issues
2. **Test in native mode early** - Don't wait until deployment; run `native:test` in CI
3. **Prefer constructor injection** - Avoids reflection-based field injection
4. **Use Buildpacks for Docker** - Simplest path to containerized native images
5. **Keep dependencies native-friendly** - Check library compatibility before adopting
6. **Use the tracing agent** - For migrating existing applications to native
7. **Profile before optimizing** - Native image peak throughput may be lower; benchmark first
8. **Minimize reflection** - Use `@RegisterReflectionForBinding` only where needed

## Common Pitfalls

1. Build failures due to missing reflection/resource hints for third-party libraries
2. Expecting JVM-level peak throughput - native images trade peak perf for startup/memory
3. Long build times (5-10+ minutes) - use CI for native builds, JVM for local dev
4. Dynamic class loading at runtime (not supported - everything must be known at build time)
5. Using CGLIB proxies (use interface-based proxies instead; `proxyBeanMethods = false`)
6. Not testing in native mode, discovering issues only at deployment
7. Large native binaries in CI pipelines without proper caching

## Resources

- [Spring Boot GraalVM Documentation](https://docs.spring.io/spring-boot/reference/native-image/index.html)
- [GraalVM Native Image Reference](https://www.graalvm.org/latest/reference-manual/native-image/)
- [Spring AOT Reference](https://docs.spring.io/spring-framework/reference/core/aot.html)
- [GraalVM Reachability Metadata Repository](https://github.com/oracle/graalvm-reachability-metadata)
