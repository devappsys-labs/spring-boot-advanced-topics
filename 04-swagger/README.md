# Swagger / OpenAPI

## Overview

Swagger (OpenAPI 3.0) provides interactive API documentation for REST APIs. It automatically generates documentation from code annotations and allows testing endpoints directly from the browser.

## Key Concepts

### 1. Dependencies

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.2.0</version>
</dependency>
```

### 2. Basic Configuration

```java
@Configuration
public class OpenAPIConfiguration {
    
    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
            .info(new Info()
                .title("My Application API")
                .version("1.0")
                .description("Comprehensive API documentation for My Application")
                .termsOfService("https://example.com/terms")
                .contact(new Contact()
                    .name("API Support")
                    .email("support@example.com")
                    .url("https://example.com/support"))
                .license(new License()
                    .name("Apache 2.0")
                    .url("https://www.apache.org/licenses/LICENSE-2.0.html")))
            .servers(List.of(
                new Server().url("http://localhost:8080").description("Development"),
                new Server().url("https://api.example.com").description("Production")
            ));
    }
}
```

### 3. Controller Documentation

```java
@RestController
@RequestMapping("/api/users")
@Tag(name = "User Management", description = "APIs for managing users")
public class UserController {
    
    @Operation(
        summary = "Get user by ID",
        description = "Retrieve a single user by their unique identifier"
    )
    @ApiResponses(value = {
        @ApiResponse(
            responseCode = "200",
            description = "User found",
            content = @Content(schema = @Schema(implementation = UserDTO.class))
        ),
        @ApiResponse(
            responseCode = "404",
            description = "User not found",
            content = @Content(schema = @Schema(implementation = ErrorResponse.class))
        )
    })
    @GetMapping("/{id}")
    public ResponseEntity<UserDTO> getUserById(
        @Parameter(description = "User ID", required = true, example = "123")
        @PathVariable Long id
    ) {
        return ResponseEntity.ok(userService.getUserById(id));
    }
    
    @Operation(summary = "Create new user")
    @ApiResponse(responseCode = "201", description = "User created successfully")
    @PostMapping
    public ResponseEntity<UserDTO> createUser(
        @io.swagger.v3.oas.annotations.parameters.RequestBody(
            description = "User details",
            required = true,
            content = @Content(schema = @Schema(implementation = CreateUserRequest.class))
        )
        @RequestBody @Valid CreateUserRequest request
    ) {
        UserDTO user = userService.createUser(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(user);
    }
}
```

### 4. Model Documentation

```java
@Schema(description = "User data transfer object")
public class UserDTO {
    
    @Schema(description = "Unique user identifier", example = "123", accessMode = Schema.AccessMode.READ_ONLY)
    private Long id;
    
    @Schema(description = "User's email address", example = "user@example.com", required = true)
    private String email;
    
    @Schema(description = "User's full name", example = "John Doe")
    private String fullName;
    
    @Schema(description = "User account status", example = "ACTIVE", allowableValues = {"ACTIVE", "INACTIVE", "SUSPENDED"})
    private String status;
    
    @Schema(description = "Account creation timestamp", example = "2024-01-15T10:30:00Z", accessMode = Schema.AccessMode.READ_ONLY)
    private LocalDateTime createdAt;
    
    // Getters and setters
}
```

### 5. Security Configuration

```java
@Configuration
public class OpenAPISecurityConfig {
    
    @Bean
    public OpenAPI secureOpenAPI() {
        return new OpenAPI()
            .components(new Components()
                .addSecuritySchemes("bearer-jwt", new SecurityScheme()
                    .type(SecurityScheme.Type.HTTP)
                    .scheme("bearer")
                    .bearerFormat("JWT")
                    .in(SecurityScheme.In.HEADER)
                    .name("Authorization"))
            )
            .addSecurityItem(new SecurityRequirement().addList("bearer-jwt"));
    }
}

// Apply to specific endpoint
@SecurityRequirement(name = "bearer-jwt")
@GetMapping("/protected")
public ResponseEntity<String> protectedEndpoint() {
    return ResponseEntity.ok("Protected data");
}
```

### 6. Grouped APIs

```java
@Configuration
public class OpenAPIGroupConfig {
    
    @Bean
    public GroupedOpenApi publicApi() {
        return GroupedOpenApi.builder()
            .group("public")
            .pathsToMatch("/api/public/**")
            .build();
    }
    
    @Bean
    public GroupedOpenApi adminApi() {
        return GroupedOpenApi.builder()
            .group("admin")
            .pathsToMatch("/api/admin/**")
            .build();
    }
}
```

### 7. Custom Examples

```java
@Schema(description = "Create user request")
public class CreateUserRequest {
    
    @Schema(
        description = "User's email",
        example = "john.doe@example.com",
        pattern = "^[A-Za-z0-9+_.-]+@(.+)$"
    )
    @NotBlank
    @Email
    private String email;
    
    @Schema(
        description = "User's password",
        example = "SecurePass123!",
        minLength = 8,
        maxLength = 50
    )
    @NotBlank
    @Size(min = 8, max = 50)
    private String password;
    
    @Schema(
        description = "User preferences",
        example = """
            {
                "theme": "dark",
                "language": "en",
                "notifications": true
            }
            """
    )
    private Map<String, Object> preferences;
}
```

### 8. Pagination Documentation

```java
@Operation(summary = "Get all users with pagination")
@GetMapping
public ResponseEntity<Page<UserDTO>> getAllUsers(
    @Parameter(description = "Page number (0-indexed)", example = "0")
    @RequestParam(defaultValue = "0") int page,
    
    @Parameter(description = "Page size", example = "20")
    @RequestParam(defaultValue = "20") int size,
    
    @Parameter(description = "Sort field", example = "createdAt")
    @RequestParam(defaultValue = "createdAt") String sortBy,
    
    @Parameter(description = "Sort direction", schema = @Schema(allowableValues = {"ASC", "DESC"}))
    @RequestParam(defaultValue = "DESC") String sortDirection
) {
    // Implementation
}
```

### 9. File Upload Documentation

```java
@Operation(summary = "Upload user avatar")
@PostMapping(value = "/{id}/avatar", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
public ResponseEntity<String> uploadAvatar(
    @PathVariable Long id,
    
    @Parameter(
        description = "Avatar image file",
        content = @Content(mediaType = MediaType.MULTIPART_FORM_DATA_VALUE)
    )
    @RequestParam("file") MultipartFile file
) {
    // Implementation
}
```

### 10. Custom Responses

```java
@Schema(description = "Standard error response")
public class ErrorResponse {
    
    @Schema(description = "Error code", example = "USER_NOT_FOUND")
    private String code;
    
    @Schema(description = "Human-readable error message", example = "User with ID 123 not found")
    private String message;
    
    @Schema(description = "Error timestamp", example = "2024-01-15T10:30:00Z")
    private LocalDateTime timestamp;
    
    @Schema(description = "Request path", example = "/api/users/123")
    private String path;
    
    @Schema(description = "Validation errors")
    private Map<String, String> errors;
}
```

## Application Configuration

```yaml
# application.yml
springdoc:
  api-docs:
    path: /v3/api-docs
    enabled: true
  swagger-ui:
    path: /swagger-ui.html
    enabled: true
    operationsSorter: method
    tagsSorter: alpha
    tryItOutEnabled: true
    filter: true
    syntaxHighlight:
      activated: true
  show-actuator: true
  default-consumes-media-type: application/json
  default-produces-media-type: application/json
```

## POC Requirements

Create comprehensive API documentation that includes:

1. **API Information**: Title, description, version, contact
2. **Security Schemes**: JWT authentication documentation
3. **Grouped APIs**: Public and admin endpoint groups
4. **Complete Annotations**: All endpoints properly documented
5. **Model Examples**: Realistic examples for all DTOs
6. **Error Responses**: Documented error scenarios

## Best Practices

1. **Complete Documentation**: Document all public endpoints
2. **Meaningful Examples**: Provide realistic example values
3. **Security Documentation**: Clearly document authentication requirements
4. **Error Scenarios**: Document all possible error responses
5. **Group Related APIs**: Use groups for better organization
6. **Keep It Updated**: Ensure documentation matches implementation

## Access URLs

- Swagger UI: `http://localhost:8080/swagger-ui.html`
- API Docs JSON: `http://localhost:8080/v3/api-docs`
- API Docs YAML: `http://localhost:8080/v3/api-docs.yaml`

## Additional Resources

- [SpringDoc OpenAPI Documentation](https://springdoc.org/)
- [OpenAPI Specification](https://swagger.io/specification/)

## Next Steps

Proceed to [Spring AOP](../05-spring-aop/README.md) to learn about aspect-oriented programming.