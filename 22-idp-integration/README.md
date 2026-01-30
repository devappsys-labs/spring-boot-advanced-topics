# IDP Integration and Runtime Configuration

## Overview

Identity Provider (IDP) integration enables centralized authentication and authorization in Spring Boot applications. This guide covers integrating with various IDPs (Keycloak, Okta, Auth0, Azure AD) and configuring them at runtime.

## Table of Contents

- [What is IDP Integration?](#what-is-idp-integration)
- [OAuth 2.0 and OpenID Connect](#oauth-20-and-openid-connect)
- [Spring Security OAuth2](#spring-security-oauth2)
- [Runtime Configuration](#runtime-configuration)
- [Multi-IDP Support](#multi-idp-support)
- [Token Validation](#token-validation)
- [POC Implementation](#poc-implementation)

## What is IDP Integration?

An Identity Provider (IDP) is a service that manages user identities and provides authentication services. Integrating with an IDP allows your application to:

- Delegate authentication to a trusted third party
- Enable Single Sign-On (SSO)
- Centralize user management
- Support social login (Google, Facebook, etc.)
- Implement enterprise identity standards

## OAuth 2.0 and OpenID Connect

### OAuth 2.0 Flow

```java
@Configuration
@EnableWebSecurity
public class OAuth2SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2Login(oauth2 -> oauth2
                .loginPage("/oauth2/authorization/my-idp")
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt.jwtAuthenticationConverter(jwtAuthConverter()))
            );
        return http.build();
    }
    
    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter grantedAuthoritiesConverter = 
            new JwtGrantedAuthoritiesConverter();
        grantedAuthoritiesConverter.setAuthoritiesClaimName("roles");
        grantedAuthoritiesConverter.setAuthorityPrefix("ROLE_");
        
        JwtAuthenticationConverter jwtAuthenticationConverter = 
            new JwtAuthenticationConverter();
        jwtAuthenticationConverter.setJwtGrantedAuthoritiesConverter(
            grantedAuthoritiesConverter
        );
        return jwtAuthenticationConverter;
    }
}
```

### OpenID Connect Configuration

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          keycloak:
            client-id: ${KEYCLOAK_CLIENT_ID}
            client-secret: ${KEYCLOAK_CLIENT_SECRET}
            scope:
              - openid
              - profile
              - email
            authorization-grant-type: authorization_code
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
        provider:
          keycloak:
            issuer-uri: ${KEYCLOAK_ISSUER_URI}
            user-name-attribute: preferred_username
```

## Spring Security OAuth2

### Dependencies

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-oauth2-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
</dependencies>
```

### Resource Server Configuration

```java
@Configuration
public class ResourceServerConfig {
    
    @Value("${spring.security.oauth2.resourceserver.jwt.issuer-uri}")
    private String issuerUri;
    
    @Bean
    public JwtDecoder jwtDecoder() {
        NimbusJwtDecoder jwtDecoder = JwtDecoders.fromIssuerLocation(issuerUri);
        
        OAuth2TokenValidator<Jwt> audienceValidator = 
            new AudienceValidator("my-api");
        OAuth2TokenValidator<Jwt> withIssuer = 
            JwtValidators.createDefaultWithIssuer(issuerUri);
        OAuth2TokenValidator<Jwt> withAudience = 
            new DelegatingOAuth2TokenValidator<>(withIssuer, audienceValidator);
        
        jwtDecoder.setJwtValidator(withAudience);
        return jwtDecoder;
    }
}

class AudienceValidator implements OAuth2TokenValidator<Jwt> {
    private final String audience;
    
    AudienceValidator(String audience) {
        this.audience = audience;
    }
    
    @Override
    public OAuth2TokenValidatorResult validate(Jwt jwt) {
        if (jwt.getAudience().contains(audience)) {
            return OAuth2TokenValidatorResult.success();
        }
        return OAuth2TokenValidatorResult.failure(
            new OAuth2Error("invalid_token", "Required audience not found", null)
        );
    }
}
```

## Runtime Configuration

### Dynamic IDP Configuration

```java
@Service
public class DynamicIdpService {
    
    private final ClientRegistrationRepository clientRegistrationRepository;
    private final Map<String, ClientRegistration> dynamicRegistrations = 
        new ConcurrentHashMap<>();
    
    public DynamicIdpService(ClientRegistrationRepository repository) {
        this.clientRegistrationRepository = repository;
    }
    
    public void registerIdp(IdpConfig config) {
        ClientRegistration registration = ClientRegistration
            .withRegistrationId(config.getRegistrationId())
            .clientId(config.getClientId())
            .clientSecret(config.getClientSecret())
            .scope(config.getScopes())
            .authorizationUri(config.getAuthorizationUri())
            .tokenUri(config.getTokenUri())
            .userInfoUri(config.getUserInfoUri())
            .userNameAttributeName(config.getUserNameAttribute())
            .jwkSetUri(config.getJwkSetUri())
            .clientName(config.getClientName())
            .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
            .redirectUri("{baseUrl}/login/oauth2/code/{registrationId}")
            .build();
        
        dynamicRegistrations.put(config.getRegistrationId(), registration);
    }
    
    public ClientRegistration findByRegistrationId(String registrationId) {
        return dynamicRegistrations.get(registrationId);
    }
}

@Data
@Builder
public class IdpConfig {
    private String registrationId;
    private String clientId;
    private String clientSecret;
    private List<String> scopes;
    private String authorizationUri;
    private String tokenUri;
    private String userInfoUri;
    private String jwkSetUri;
    private String userNameAttribute;
    private String clientName;
}
```

### Custom ClientRegistrationRepository

```java
@Component
public class DynamicClientRegistrationRepository 
    implements ClientRegistrationRepository, Iterable<ClientRegistration> {
    
    private final Map<String, ClientRegistration> registrations;
    private final DynamicIdpService dynamicIdpService;
    
    public DynamicClientRegistrationRepository(
            List<ClientRegistration> staticRegistrations,
            DynamicIdpService dynamicIdpService) {
        this.registrations = staticRegistrations.stream()
            .collect(Collectors.toMap(
                ClientRegistration::getRegistrationId, 
                Function.identity()
            ));
        this.dynamicIdpService = dynamicIdpService;
    }
    
    @Override
    public ClientRegistration findByRegistrationId(String registrationId) {
        ClientRegistration registration = registrations.get(registrationId);
        if (registration == null) {
            registration = dynamicIdpService.findByRegistrationId(registrationId);
        }
        return registration;
    }
    
    @Override
    public Iterator<ClientRegistration> iterator() {
        return registrations.values().iterator();
    }
}
```

### Runtime Configuration Endpoint

```java
@RestController
@RequestMapping("/api/admin/idp")
@PreAuthorize("hasRole('ADMIN')")
public class IdpConfigController {
    
    private final DynamicIdpService dynamicIdpService;
    
    @PostMapping("/register")
    public ResponseEntity<String> registerIdp(@RequestBody IdpConfigRequest request) {
        IdpConfig config = IdpConfig.builder()
            .registrationId(request.getRegistrationId())
            .clientId(request.getClientId())
            .clientSecret(request.getClientSecret())
            .scopes(request.getScopes())
            .authorizationUri(request.getAuthorizationUri())
            .tokenUri(request.getTokenUri())
            .userInfoUri(request.getUserInfoUri())
            .jwkSetUri(request.getJwkSetUri())
            .userNameAttribute(request.getUserNameAttribute())
            .clientName(request.getClientName())
            .build();
        
        dynamicIdpService.registerIdp(config);
        return ResponseEntity.ok("IDP registered successfully");
    }
}
```

## Multi-IDP Support

### IDP Selection Strategy

```java
@Service
public class IdpSelectionService {
    
    private final ClientRegistrationRepository clientRegistrationRepository;
    
    public List<IdpOption> getAvailableIdps() {
        List<IdpOption> options = new ArrayList<>();
        
        if (clientRegistrationRepository instanceof Iterable) {
            ((Iterable<ClientRegistration>) clientRegistrationRepository)
                .forEach(registration -> {
                    options.add(IdpOption.builder()
                        .id(registration.getRegistrationId())
                        .name(registration.getClientName())
                        .authorizationUri("/oauth2/authorization/" + 
                            registration.getRegistrationId())
                        .build());
                });
        }
        
        return options;
    }
}

@Data
@Builder
class IdpOption {
    private String id;
    private String name;
    private String authorizationUri;
}
```

### Custom Login Page

```java
@Controller
public class CustomLoginController {
    
    private final IdpSelectionService idpSelectionService;
    
    @GetMapping("/login")
    public String login(Model model) {
        List<IdpOption> idps = idpSelectionService.getAvailableIdps();
        model.addAttribute("idps", idps);
        return "login";
    }
}
```

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <title>Login</title>
</head>
<body>
    <h2>Select Identity Provider</h2>
    <div th:each="idp : ${idps}">
        <a th:href="@{${idp.authorizationUri}}" 
           th:text="${idp.name}">IDP Name</a>
    </div>
</body>
</html>
```

## Token Validation

### Custom JWT Validator

```java
@Component
public class CustomJwtValidator implements OAuth2TokenValidator<Jwt> {
    
    private final List<OAuth2TokenValidator<Jwt>> validators;
    
    public CustomJwtValidator(
            @Value("${spring.security.oauth2.resourceserver.jwt.issuer-uri}") 
            String issuerUri) {
        this.validators = List.of(
            new JwtTimestampValidator(),
            JwtValidators.createDefaultWithIssuer(issuerUri),
            new CustomClaimValidator()
        );
    }
    
    @Override
    public OAuth2TokenValidatorResult validate(Jwt jwt) {
        List<OAuth2Error> errors = new ArrayList<>();
        
        for (OAuth2TokenValidator<Jwt> validator : validators) {
            OAuth2TokenValidatorResult result = validator.validate(jwt);
            if (result.hasErrors()) {
                errors.addAll(result.getErrors());
            }
        }
        
        return errors.isEmpty() 
            ? OAuth2TokenValidatorResult.success() 
            : OAuth2TokenValidatorResult.failure(errors);
    }
}

class CustomClaimValidator implements OAuth2TokenValidator<Jwt> {
    
    @Override
    public OAuth2TokenValidatorResult validate(Jwt jwt) {
        Map<String, Object> claims = jwt.getClaims();
        
        // Validate custom claims
        if (!claims.containsKey("tenant_id")) {
            return OAuth2TokenValidatorResult.failure(
                new OAuth2Error("invalid_token", "Missing tenant_id claim", null)
            );
        }
        
        return OAuth2TokenValidatorResult.success();
    }
}
```

### Token Introspection

```java
@Service
public class TokenIntrospectionService {
    
    private final RestTemplate restTemplate;
    
    @Value("${idp.introspection.uri}")
    private String introspectionUri;
    
    @Value("${idp.client-id}")
    private String clientId;
    
    @Value("${idp.client-secret}")
    private String clientSecret;
    
    public TokenIntrospectionResponse introspect(String token) {
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_FORM_URLENCODED);
        headers.setBasicAuth(clientId, clientSecret);
        
        MultiValueMap<String, String> body = new LinkedMultiValueMap<>();
        body.add("token", token);
        body.add("token_type_hint", "access_token");
        
        HttpEntity<MultiValueMap<String, String>> request = 
            new HttpEntity<>(body, headers);
        
        ResponseEntity<TokenIntrospectionResponse> response = 
            restTemplate.postForEntity(
                introspectionUri, 
                request, 
                TokenIntrospectionResponse.class
            );
        
        return response.getBody();
    }
}

@Data
class TokenIntrospectionResponse {
    private boolean active;
    private String scope;
    private String clientId;
    private String username;
    private String tokenType;
    private Long exp;
    private Long iat;
    private String sub;
    private String aud;
    private String iss;
}
```

## POC Implementation

### Project Structure

```
idp-integration-poc/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/example/idp/
│   │   │       ├── config/
│   │   │       │   ├── SecurityConfig.java
│   │   │       │   ├── OAuth2Config.java
│   │   │       │   └── DynamicIdpConfig.java
│   │   │       ├── service/
│   │   │       │   ├── DynamicIdpService.java
│   │   │       │   ├── IdpSelectionService.java
│   │   │       │   └── TokenIntrospectionService.java
│   │   │       ├── controller/
│   │   │       │   ├── IdpConfigController.java
│   │   │       │   ├── LoginController.java
│   │   │       │   └── UserController.java
│   │   │       ├── model/
│   │   │       │   ├── IdpConfig.java
│   │   │       │   └── IdpOption.java
│   │   │       └── validator/
│   │   │           ├── CustomJwtValidator.java
│   │   │           └── AudienceValidator.java
│   │   └── resources/
│   │       ├── application.yml
│   │       └── templates/
│   │           └── login.html
│   └── test/
│       └── java/
│           └── com/example/idp/
│               └── IdpIntegrationTest.java
├── docker-compose.yml
└── pom.xml
```

### Complete Security Configuration

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {
    
    private final DynamicIdpService dynamicIdpService;
    private final CustomJwtValidator customJwtValidator;
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf
                .ignoringRequestMatchers("/api/**")
            )
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/login", "/oauth2/**", "/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2Login(oauth2 -> oauth2
                .loginPage("/login")
                .defaultSuccessUrl("/dashboard", true)
                .failureUrl("/login?error=true")
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .decoder(jwtDecoder())
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())
                )
            )
            .logout(logout -> logout
                .logoutSuccessUrl("/login?logout=true")
                .invalidateHttpSession(true)
                .clearAuthentication(true)
            );
        
        return http.build();
    }
    
    @Bean
    public JwtDecoder jwtDecoder() {
        NimbusJwtDecoder decoder = JwtDecoders.fromIssuerLocation(issuerUri);
        decoder.setJwtValidator(customJwtValidator);
        return decoder;
    }
    
    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter authoritiesConverter = 
            new JwtGrantedAuthoritiesConverter();
        authoritiesConverter.setAuthoritiesClaimName("roles");
        authoritiesConverter.setAuthorityPrefix("ROLE_");
        
        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(authoritiesConverter);
        converter.setPrincipalClaimName("preferred_username");
        
        return converter;
    }
}
```

### User Controller Example

```java
@RestController
@RequestMapping("/api/user")
public class UserController {
    
    @GetMapping("/profile")
    public ResponseEntity<UserProfile> getProfile(
            @AuthenticationPrincipal Jwt jwt) {
        UserProfile profile = UserProfile.builder()
            .username(jwt.getClaimAsString("preferred_username"))
            .email(jwt.getClaimAsString("email"))
            .firstName(jwt.getClaimAsString("given_name"))
            .lastName(jwt.getClaimAsString("family_name"))
            .roles(jwt.getClaimAsStringList("roles"))
            .tenantId(jwt.getClaimAsString("tenant_id"))
            .build();
        
        return ResponseEntity.ok(profile);
    }
    
    @GetMapping("/token-info")
    public ResponseEntity<Map<String, Object>> getTokenInfo(
            @AuthenticationPrincipal Jwt jwt) {
        Map<String, Object> tokenInfo = new HashMap<>();
        tokenInfo.put("subject", jwt.getSubject());
        tokenInfo.put("issuer", jwt.getIssuer());
        tokenInfo.put("issuedAt", jwt.getIssuedAt());
        tokenInfo.put("expiresAt", jwt.getExpiresAt());
        tokenInfo.put("claims", jwt.getClaims());
        
        return ResponseEntity.ok(tokenInfo);
    }
}

@Data
@Builder
class UserProfile {
    private String username;
    private String email;
    private String firstName;
    private String lastName;
    private List<String> roles;
    private String tenantId;
}
```

### Docker Compose for Keycloak

```yaml
version: '3.8'

services:
  keycloak:
    image: quay.io/keycloak/keycloak:23.0
    environment:
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin
    ports:
      - "8080:8080"
    command:
      - start-dev
    volumes:
      - keycloak_data:/opt/keycloak/data

  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: keycloak
      POSTGRES_USER: keycloak
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  keycloak_data:
  postgres_data:
```

### Integration Test

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
class IdpIntegrationTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Test
    @WithMockUser(roles = "ADMIN")
    void testRegisterIdp() throws Exception {
        IdpConfigRequest request = new IdpConfigRequest();
        request.setRegistrationId("custom-idp");
        request.setClientId("client-123");
        request.setClientSecret("secret-456");
        request.setScopes(List.of("openid", "profile", "email"));
        
        mockMvc.perform(post("/api/admin/idp/register")
                .contentType(MediaType.APPLICATION_JSON)
                .content(asJsonString(request)))
                .andExpect(status().isOk());
    }
    
    @Test
    void testProtectedEndpointWithoutAuth() throws Exception {
        mockMvc.perform(get("/api/user/profile"))
                .andExpect(status().isUnauthorized());
    }
    
    @Test
    void testProtectedEndpointWithJwt() throws Exception {
        String jwt = createMockJwt();
        
        mockMvc.perform(get("/api/user/profile")
                .header("Authorization", "Bearer " + jwt))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.username").exists());
    }
    
    private String createMockJwt() {
        // Create mock JWT for testing
        return "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...";
    }
    
    private String asJsonString(Object obj) throws JsonProcessingException {
        ObjectMapper mapper = new ObjectMapper();
        return mapper.writeValueAsString(obj);
    }
}
```

## Best Practices

1. **Token Security**
   - Always validate token signatures
   - Check expiration times
   - Validate issuer and audience claims
   - Use HTTPS in production

2. **Configuration Management**
   - Store sensitive data in environment variables or vault
   - Never commit client secrets to version control
   - Use different configurations per environment

3. **Error Handling**
   - Implement proper OAuth2 error responses
   - Log authentication failures for security monitoring
   - Provide meaningful error messages without exposing sensitive info

4. **Performance**
   - Cache JWK sets with appropriate TTL
   - Use connection pooling for token introspection
   - Implement token caching when appropriate

5. **Multi-Tenancy**
   - Validate tenant claims in tokens
   - Implement tenant isolation at application level
   - Support tenant-specific IDP configurations

## Common Pitfalls

- Not validating token audience
- Hardcoding IDP URLs
- Insufficient token validation
- Not handling token refresh properly
- Ignoring CORS configuration for SPA clients
- Not implementing proper logout handling

## Additional Resources

- [Spring Security OAuth2 Documentation](https://spring.io/projects/spring-security-oauth)
- [OAuth 2.0 RFC](https://tools.ietf.org/html/rfc6749)
- [OpenID Connect Specification](https://openid.net/connect/)
- [Keycloak Documentation](https://www.keycloak.org/documentation)