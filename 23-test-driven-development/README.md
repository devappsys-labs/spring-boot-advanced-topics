# Test-Driven Development (TDD)

## Overview

Test-Driven Development (TDD) is a software development approach where tests are written before the actual code. This methodology ensures better code quality, design, and maintainability in Spring Boot applications.

## Table of Contents

- [What is TDD?](#what-is-tdd)
- [TDD Cycle: Red-Green-Refactor](#tdd-cycle-red-green-refactor)
- [Unit Testing with JUnit 5](#unit-testing-with-junit-5)
- [Integration Testing](#integration-testing)
- [Mocking with Mockito](#mocking-with-mockito)
- [Testing Spring Boot Components](#testing-spring-boot-components)
- [Test Containers](#test-containers)
- [BDD with Cucumber](#bdd-with-cucumber)
- [POC Implementation](#poc-implementation)

## What is TDD?

Test-Driven Development is a development process where:

1. Write a failing test (Red)
2. Write minimal code to pass the test (Green)
3. Refactor the code while keeping tests green (Refactor)

### Benefits of TDD

- Better code design and architecture
- Higher test coverage
- Fewer bugs in production
- Living documentation
- Confidence in refactoring
- Faster debugging

## TDD Cycle: Red-Green-Refactor

### The Red Phase

Write a test that fails because the functionality doesn't exist yet.

```java
@Test
void shouldCreateUserWithValidData() {
    // Arrange
    UserRequest request = new UserRequest("john.doe@example.com", "John", "Doe");
    
    // Act
    User user = userService.createUser(request);
    
    // Assert
    assertNotNull(user.getId());
    assertEquals("john.doe@example.com", user.getEmail());
    assertEquals("John", user.getFirstName());
    assertEquals("Doe", user.getLastName());
}
```

### The Green Phase

Write minimal code to make the test pass.

```java
@Service
public class UserService {
    
    private final UserRepository userRepository;
    
    public User createUser(UserRequest request) {
        User user = new User();
        user.setEmail(request.getEmail());
        user.setFirstName(request.getFirstName());
        user.setLastName(request.getLastName());
        return userRepository.save(user);
    }
}
```

### The Refactor Phase

Improve the code while keeping tests green.

```java
@Service
@RequiredArgsConstructor
public class UserService {
    
    private final UserRepository userRepository;
    private final UserValidator userValidator;
    
    public User createUser(UserRequest request) {
        userValidator.validate(request);
        
        User user = User.builder()
            .email(request.getEmail())
            .firstName(request.getFirstName())
            .lastName(request.getLastName())
            .createdAt(LocalDateTime.now())
            .build();
            
        return userRepository.save(user);
    }
}
```

## Unit Testing with JUnit 5

### Dependencies

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### Basic Test Structure

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    
    @Mock
    private UserRepository userRepository;
    
    @Mock
    private UserValidator userValidator;
    
    @InjectMocks
    private UserService userService;
    
    @Test
    @DisplayName("Should create user successfully with valid data")
    void shouldCreateUserSuccessfully() {
        // Arrange
        UserRequest request = new UserRequest("test@example.com", "Test", "User");
        User expectedUser = User.builder()
            .id(1L)
            .email("test@example.com")
            .firstName("Test")
            .lastName("User")
            .build();
        
        when(userRepository.save(any(User.class))).thenReturn(expectedUser);
        
        // Act
        User actualUser = userService.createUser(request);
        
        // Assert
        assertNotNull(actualUser);
        assertEquals(expectedUser.getId(), actualUser.getId());
        assertEquals(expectedUser.getEmail(), actualUser.getEmail());
        verify(userValidator).validate(request);
        verify(userRepository).save(any(User.class));
    }
    
    @Test
    @DisplayName("Should throw exception when email is invalid")
    void shouldThrowExceptionForInvalidEmail() {
        // Arrange
        UserRequest request = new UserRequest("invalid-email", "Test", "User");
        doThrow(new ValidationException("Invalid email"))
            .when(userValidator).validate(request);
        
        // Act & Assert
        assertThrows(ValidationException.class, () -> {
            userService.createUser(request);
        });
        
        verify(userRepository, never()).save(any(User.class));
    }
}
```

### Parameterized Tests

```java
@ParameterizedTest
@ValueSource(strings = {"test@example.com", "user@domain.org", "admin@company.co.uk"})
void shouldAcceptValidEmails(String email) {
    assertTrue(EmailValidator.isValid(email));
}

@ParameterizedTest
@CsvSource({
    "test@example.com, Test, User",
    "admin@company.com, Admin, Administrator",
    "user@domain.org, John, Doe"
})
void shouldCreateUserWithDifferentInputs(String email, String firstName, String lastName) {
    UserRequest request = new UserRequest(email, firstName, lastName);
    User user = userService.createUser(request);
    
    assertEquals(email, user.getEmail());
    assertEquals(firstName, user.getFirstName());
    assertEquals(lastName, user.getLastName());
}

@ParameterizedTest
@MethodSource("invalidUserRequests")
void shouldRejectInvalidUserRequests(UserRequest request) {
    assertThrows(ValidationException.class, () -> {
        userService.createUser(request);
    });
}

private static Stream<UserRequest> invalidUserRequests() {
    return Stream.of(
        new UserRequest("", "Test", "User"),
        new UserRequest("invalid-email", "Test", "User"),
        new UserRequest("test@example.com", "", "User"),
        new UserRequest("test@example.com", "Test", "")
    );
}
```

### Nested Tests

```java
@Nested
@DisplayName("User Creation Tests")
class UserCreationTests {
    
    @Nested
    @DisplayName("When user data is valid")
    class ValidUserData {
        
        @Test
        void shouldCreateUserSuccessfully() {
            // Test implementation
        }
        
        @Test
        void shouldGenerateUniqueId() {
            // Test implementation
        }
        
        @Test
        void shouldSetCreationTimestamp() {
            // Test implementation
        }
    }
    
    @Nested
    @DisplayName("When user data is invalid")
    class InvalidUserData {
        
        @Test
        void shouldRejectEmptyEmail() {
            // Test implementation
        }
        
        @Test
        void shouldRejectInvalidEmailFormat() {
            // Test implementation
        }
        
        @Test
        void shouldRejectDuplicateEmail() {
            // Test implementation
        }
    }
}
```

## Integration Testing

### Spring Boot Test Configuration

```java
@SpringBootTest
@AutoConfigureMockMvc
@ActiveProfiles("test")
class UserControllerIntegrationTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    @Autowired
    private UserRepository userRepository;
    
    @BeforeEach
    void setUp() {
        userRepository.deleteAll();
    }
    
    @Test
    void shouldCreateUserViaRestApi() throws Exception {
        UserRequest request = new UserRequest("test@example.com", "Test", "User");
        
        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.id").exists())
                .andExpect(jsonPath("$.email").value("test@example.com"))
                .andExpect(jsonPath("$.firstName").value("Test"))
                .andExpect(jsonPath("$.lastName").value("User"));
    }
    
    @Test
    void shouldReturnBadRequestForInvalidData() throws Exception {
        UserRequest request = new UserRequest("invalid-email", "Test", "User");
        
        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.errors").isArray())
                .andExpect(jsonPath("$.errors[0].field").value("email"))
                .andExpect(jsonPath("$.errors[0].message").value("Invalid email format"));
    }
}
```

### WebTestClient for WebFlux

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
class UserControllerWebFluxTest {
    
    @Autowired
    private WebTestClient webTestClient;
    
    @Test
    void shouldCreateUserReactively() {
        UserRequest request = new UserRequest("test@example.com", "Test", "User");
        
        webTestClient.post()
            .uri("/api/users")
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue(request)
            .exchange()
            .expectStatus().isCreated()
            .expectBody()
            .jsonPath("$.id").exists()
            .jsonPath("$.email").isEqualTo("test@example.com");
    }
}
```

## Mocking with Mockito

### Mock Behavior Configuration

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    
    @Mock
    private OrderRepository orderRepository;
    
    @Mock
    private PaymentService paymentService;
    
    @Mock
    private NotificationService notificationService;
    
    @InjectMocks
    private OrderService orderService;
    
    @Test
    void shouldProcessOrderSuccessfully() {
        // Arrange
        Order order = createTestOrder();
        Payment payment = createTestPayment();
        
        when(orderRepository.findById(1L)).thenReturn(Optional.of(order));
        when(paymentService.processPayment(any())).thenReturn(payment);
        doNothing().when(notificationService).sendOrderConfirmation(any());
        
        // Act
        OrderResult result = orderService.processOrder(1L);
        
        // Assert
        assertTrue(result.isSuccess());
        assertEquals(OrderStatus.CONFIRMED, order.getStatus());
        
        verify(orderRepository).findById(1L);
        verify(paymentService).processPayment(any());
        verify(notificationService).sendOrderConfirmation(order);
    }
    
    @Test
    void shouldHandlePaymentFailure() {
        // Arrange
        Order order = createTestOrder();
        
        when(orderRepository.findById(1L)).thenReturn(Optional.of(order));
        when(paymentService.processPayment(any()))
            .thenThrow(new PaymentException("Payment failed"));
        
        // Act
        OrderResult result = orderService.processOrder(1L);
        
        // Assert
        assertFalse(result.isSuccess());
        assertEquals(OrderStatus.PAYMENT_FAILED, order.getStatus());
        
        verify(notificationService, never()).sendOrderConfirmation(any());
    }
}
```

### Argument Captors

```java
@Test
void shouldSendEmailWithCorrectContent() {
    // Arrange
    ArgumentCaptor<EmailMessage> emailCaptor = ArgumentCaptor.forClass(EmailMessage.class);
    User user = createTestUser();
    
    // Act
    userService.registerUser(user);
    
    // Assert
    verify(emailService).sendEmail(emailCaptor.capture());
    
    EmailMessage capturedEmail = emailCaptor.getValue();
    assertEquals(user.getEmail(), capturedEmail.getRecipient());
    assertTrue(capturedEmail.getSubject().contains("Welcome"));
    assertTrue(capturedEmail.getBody().contains(user.getFirstName()));
}
```

### Spy Objects

```java
@Test
void shouldUseSpyForPartialMocking() {
    // Arrange
    List<String> list = new ArrayList<>();
    List<String> spyList = spy(list);
    
    // Configure specific behavior
    when(spyList.size()).thenReturn(100);
    
    // Act & Assert
    spyList.add("one");
    spyList.add("two");
    
    assertEquals(100, spyList.size()); // Mocked behavior
    assertTrue(spyList.contains("one")); // Real behavior
    verify(spyList).add("one");
}
```

## Testing Spring Boot Components

### Repository Tests

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class UserRepositoryTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private TestEntityManager entityManager;
    
    @Test
    void shouldFindUserByEmail() {
        // Arrange
        User user = User.builder()
            .email("test@example.com")
            .firstName("Test")
            .lastName("User")
            .build();
        entityManager.persist(user);
        entityManager.flush();
        
        // Act
        Optional<User> found = userRepository.findByEmail("test@example.com");
        
        // Assert
        assertTrue(found.isPresent());
        assertEquals("Test", found.get().getFirstName());
    }
    
    @Test
    void shouldReturnEmptyForNonExistentEmail() {
        Optional<User> found = userRepository.findByEmail("nonexistent@example.com");
        assertFalse(found.isPresent());
    }
}
```

### Custom Repository Tests

```java
@Test
void shouldFindActiveUsersByCreatedDateRange() {
    // Arrange
    LocalDateTime startDate = LocalDateTime.now().minusDays(7);
    LocalDateTime endDate = LocalDateTime.now();
    
    User activeUser1 = createUser("active1@example.com", true, startDate.plusDays(1));
    User activeUser2 = createUser("active2@example.com", true, startDate.plusDays(2));
    User inactiveUser = createUser("inactive@example.com", false, startDate.plusDays(3));
    User oldUser = createUser("old@example.com", true, startDate.minusDays(10));
    
    entityManager.persist(activeUser1);
    entityManager.persist(activeUser2);
    entityManager.persist(inactiveUser);
    entityManager.persist(oldUser);
    entityManager.flush();
    
    // Act
    List<User> users = userRepository.findActiveUsersByCreatedDateRange(
        startDate, endDate
    );
    
    // Assert
    assertEquals(2, users.size());
    assertTrue(users.stream().allMatch(User::isActive));
}
```

### Controller Tests

```java
@WebMvcTest(UserController.class)
class UserControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @MockBean
    private UserService userService;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    @Test
    void shouldReturnUserById() throws Exception {
        User user = User.builder()
            .id(1L)
            .email("test@example.com")
            .firstName("Test")
            .lastName("User")
            .build();
        
        when(userService.getUserById(1L)).thenReturn(user);
        
        mockMvc.perform(get("/api/users/1"))
            .andExpect(status().isOk())
            .andExpect(content().contentType(MediaType.APPLICATION_JSON))
            .andExpect(jsonPath("$.id").value(1))
            .andExpect(jsonPath("$.email").value("test@example.com"));
    }
    
    @Test
    void shouldReturn404WhenUserNotFound() throws Exception {
        when(userService.getUserById(999L))
            .thenThrow(new UserNotFoundException("User not found"));
        
        mockMvc.perform(get("/api/users/999"))
            .andExpect(status().isNotFound())
            .andExpect(jsonPath("$.message").value("User not found"));
    }
}
```

### Service Tests

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    
    @Mock
    private OrderRepository orderRepository;
    
    @Mock
    private InventoryService inventoryService;
    
    @Mock
    private PaymentService paymentService;
    
    @Captor
    private ArgumentCaptor<Order> orderCaptor;
    
    @InjectMocks
    private OrderService orderService;
    
    @Test
    void shouldCreateOrderAndReserveInventory() {
        // Arrange
        OrderRequest request = createOrderRequest();
        when(inventoryService.checkAvailability(any())).thenReturn(true);
        when(inventoryService.reserveItems(any())).thenReturn(true);
        when(orderRepository.save(any())).thenAnswer(i -> i.getArgument(0));
        
        // Act
        Order order = orderService.createOrder(request);
        
        // Assert
        assertNotNull(order);
        assertEquals(OrderStatus.PENDING, order.getStatus());
        
        verify(inventoryService).checkAvailability(request.getItems());
        verify(inventoryService).reserveItems(request.getItems());
        verify(orderRepository).save(orderCaptor.capture());
        
        Order savedOrder = orderCaptor.getValue();
        assertEquals(request.getItems().size(), savedOrder.getItems().size());
    }
}
```

## Test Containers

### Dependencies

```xml
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
```

### PostgreSQL Container Test

```java
@SpringBootTest
@Testcontainers
@ActiveProfiles("test")
class UserRepositoryIntegrationTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void shouldSaveAndRetrieveUser() {
        User user = User.builder()
            .email("test@example.com")
            .firstName("Test")
            .lastName("User")
            .build();
        
        User saved = userRepository.save(user);
        Optional<User> found = userRepository.findById(saved.getId());
        
        assertTrue(found.isPresent());
        assertEquals("test@example.com", found.get().getEmail());
    }
}
```

### Multiple Containers

```java
@SpringBootTest
@Testcontainers
class MultiContainerTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");
    
    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379);
    
    @Container
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.5.0")
    );
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        
        registry.add("spring.data.redis.host", redis::getHost);
        registry.add("spring.data.redis.port", redis::getFirstMappedPort);
        
        registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    }
    
    @Test
    void shouldWorkWithAllContainers() {
        // Test implementation using all containers
    }
}
```

### Custom Container Configuration

```java
@SpringBootTest
@Testcontainers
class CustomContainerTest {
    
    @Container
    static GenericContainer<?> customApp = new GenericContainer<>("my-app:latest")
        .withExposedPorts(8080)
        .withEnv("SPRING_PROFILES_ACTIVE", "test")
        .withNetwork(Network.SHARED)
        .waitingFor(Wait.forHttp("/actuator/health")
            .forStatusCode(200)
            .withStartupTimeout(Duration.ofMinutes(2)));
    
    @Test
    void shouldConnectToCustomContainer() {
        String baseUrl = "http://" + customApp.getHost() + ":" + 
            customApp.getMappedPort(8080);
        
        RestTemplate restTemplate = new RestTemplate();
        ResponseEntity<String> response = restTemplate.getForEntity(
            baseUrl + "/api/health", 
            String.class
        );
        
        assertEquals(HttpStatus.OK, response.getStatusCode());
    }
}
```

## BDD with Cucumber

### Dependencies

```xml
<dependency>
    <groupId>io.cucumber</groupId>
    <artifactId>cucumber-java</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>io.cucumber</groupId>
    <artifactId>cucumber-spring</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>io.cucumber</groupId>
    <artifactId>cucumber-junit-platform-engine</artifactId>
    <scope>test</scope>
</dependency>
```

### Feature File

```gherkin
# src/test/resources/features/user-registration.feature
Feature: User Registration
  As a new user
  I want to register an account
  So that I can access the system

  Scenario: Successful user registration
    Given I am on the registration page
    When I enter valid registration details:
      | email              | firstName | lastName |
      | test@example.com   | John      | Doe      |
    And I submit the registration form
    Then I should see a success message
    And a new user account should be created
    And a welcome email should be sent

  Scenario: Registration with existing email
    Given a user already exists with email "existing@example.com"
    When I try to register with email "existing@example.com"
    Then I should see an error message "Email already registered"
    And no new user account should be created

  Scenario Outline: Registration with invalid data
    When I try to register with <field> as "<value>"
    Then I should see validation error "<error>"

    Examples:
      | field     | value           | error                  |
      | email     | invalid-email   | Invalid email format   |
      | email     |                 | Email is required      |
      | firstName |                 | First name is required |
      | lastName  |                 | Last name is required  |
```

### Step Definitions

```java
@SpringBootTest
@CucumberContextConfiguration
public class UserRegistrationSteps {
    
    @Autowired
    private UserService userService;
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private EmailService emailService;
    
    private UserRequest registrationRequest;
    private User createdUser;
    private Exception thrownException;
    
    @Given("I am on the registration page")
    public void iAmOnTheRegistrationPage() {
        // Setup for registration
        registrationRequest = new UserRequest();
    }
    
    @Given("a user already exists with email {string}")
    public void aUserAlreadyExistsWithEmail(String email) {
        User existingUser = User.builder()
            .email(email)
            .firstName("Existing")
            .lastName("User")
            .build();
        userRepository.save(existingUser);
    }
    
    @When("I enter valid registration details:")
    public void iEnterValidRegistrationDetails(DataTable dataTable) {
        Map<String, String> data = dataTable.asMaps().get(0);
        registrationRequest = new UserRequest(
            data.get("email"),
            data.get("firstName"),
            data.get("lastName")
        );
    }
    
    @When("I submit the registration form")
    public void iSubmitTheRegistrationForm() {
        try {
            createdUser = userService.registerUser(registrationRequest);
        } catch (Exception e) {
            thrownException = e;
        }
    }
    
    @When("I try to register with email {string}")
    public void iTryToRegisterWithEmail(String email) {
        registrationRequest = new UserRequest(email, "Test", "User");
        try {
            userService.registerUser(registrationRequest);
        } catch (Exception e) {
            thrownException = e;
        }
    }
    
    @Then("I should see a success message")
    public void iShouldSeeASuccessMessage() {
        assertNotNull(createdUser);
    }
    
    @Then("a new user account should be created")
    public void aNewUserAccountShouldBeCreated() {
        assertNotNull(createdUser.getId());
        Optional<User> savedUser = userRepository.findById(createdUser.getId());
        assertTrue(savedUser.isPresent());
    }
    
    @Then("a welcome email should be sent")
    public void aWelcomeEmailShouldBeSent() {
        verify(emailService).sendWelcomeEmail(eq(createdUser.getEmail()));
    }
    
    @Then("I should see an error message {string}")
    public void iShouldSeeAnErrorMessage(String expectedMessage) {
        assertNotNull(thrownException);
        assertTrue(thrownException.getMessage().contains(expectedMessage));
    }
    
    @Then("no new user account should be created")
    public void noNewUserAccountShouldBeCreated() {
        assertNull(createdUser);
    }
}
```

### Test Runner

```java
@Suite
@IncludeEngines("cucumber")
@SelectClasspathResource("features")
@ConfigurationParameter(key = PLUGIN_PROPERTY_NAME, 
    value = "pretty, html:target/cucumber-reports/cucumber.html")
public class CucumberIntegrationTest {
}
```

## POC Implementation

### Project Structure

```
tdd-poc/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/example/tdd/
│   │   │       ├── controller/
│   │   │       │   └── UserController.java
│   │   │       ├── service/
│   │   │       │   ├── UserService.java
│   │   │       │   └── EmailService.java
│   │   │       ├── repository/
│   │   │       │   └── UserRepository.java
│   │   │       ├── model/
│   │   │       │   ├── User.java
│   │   │       │   └── UserRequest.java
│   │   │       ├── validator/
│   │   │       │   └── UserValidator.java
│   │   │       └── exception/
│   │   │           ├── UserNotFoundException.java
│   │   │           └── ValidationException.java
│   │   └── resources/
│   │       └── application.yml
│   └── test/
│       ├── java/
│       │   └── com/example/tdd/
│       │       ├── controller/
│       │       │   ├── UserControllerTest.java
│       │       │   └── UserControllerIntegrationTest.java
│       │       ├── service/
│       │       │   └── UserServiceTest.java
│       │       ├── repository/
│       │       │   └── UserRepositoryTest.java
│       │       └── cucumber/
│       │           ├── CucumberIntegrationTest.java
│       │           └── UserRegistrationSteps.java
│       └── resources/
│           ├── application-test.yml
│           └── features/
│               └── user-registration.feature
└── pom.xml
```

### Complete Test Suite Example

```java
@SpringBootTest
@AutoConfigureMockMvc
@Testcontainers
@ActiveProfiles("test")
class CompleteUserFlowTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
    
    @Autowired
    private MockMvc mockMvc;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    @Autowired
    private UserRepository userRepository;
    
    @BeforeEach
    void setUp() {
        userRepository.deleteAll();
    }
    
    @Test
    @DisplayName("Complete user lifecycle: Create, Read, Update, Delete")
    void completeUserLifecycle() throws Exception {
        // 1. Create User
        UserRequest createRequest = new UserRequest(
            "test@example.com", 
            "John", 
            "Doe"
        );
        
        MvcResult createResult = mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(createRequest)))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.id").exists())
                .andReturn();
        
        Long userId = JsonPath.read(createResult.getResponse().getContentAsString(), "$.id");
        
        // 2. Read User
        mockMvc.perform(get("/api/users/" + userId))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.email").value("test@example.com"))
                .andExpect(jsonPath("$.firstName").value("John"));
        
        // 3. Update User
        UserRequest updateRequest = new UserRequest(
            "test@example.com", 
            "Jane", 
            "Smith"
        );
        
        mockMvc.perform(put("/api/users/" + userId)
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(updateRequest)))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.firstName").value("Jane"))
                .andExpect(jsonPath("$.lastName").value("Smith"));
        
        // 4. Delete User
        mockMvc.perform(delete("/api/users/" + userId))
                .andExpect(status().isNoContent());
        
        // 5. Verify Deletion
        mockMvc.perform(get("/api/users/" + userId))
                .andExpect(status().isNotFound());
    }
}
```

### Test Configuration

```java
@TestConfiguration
public class TestConfig {
    
    @Bean
    @Primary
    public EmailService mockEmailService() {
        return mock(EmailService.class);
    }
    
    @Bean
    public Clock fixedClock() {
        return Clock.fixed(
            Instant.parse("2024-01-01T00:00:00Z"), 
            ZoneId.of("UTC")
        );
    }
    
    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

### Custom Test Annotations

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@SpringBootTest
@AutoConfigureMockMvc
@Testcontainers
@ActiveProfiles("test")
public @interface IntegrationTest {
}

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Test
@Transactional
@Rollback
public @interface TransactionalTest {
}
```

## Best Practices

1. **Test Naming**
   - Use descriptive test names that explain what is being tested
   - Follow pattern: `should[ExpectedBehavior]When[StateUnderTest]`
   - Use `@DisplayName` for better readability

2. **Test Structure (AAA Pattern)**
   - **Arrange**: Set up test data and preconditions
   - **Act**: Execute the code under test
   - **Assert**: Verify the expected outcome

3. **Test Independence**
   - Each test should be independent
   - Use `@BeforeEach` and `@AfterEach` for setup/cleanup
   - Avoid test interdependencies

4. **Test Coverage**
   - Aim for high code coverage but focus on meaningful tests
   - Test edge cases and error scenarios
   - Don't test framework code

5. **Mock Wisely**
   - Mock external dependencies
   - Don't mock value objects
   - Use real objects when possible

6. **Fast Tests**
   - Keep unit tests fast (< 100ms)
   - Use in-memory databases for integration tests
   - Parallelize test execution

## Common Pitfalls

- Writing tests after implementation
- Testing implementation details instead of behavior
- Over-mocking leading to brittle tests
- Not cleaning up test data
- Ignoring failing tests
- Poor test organization
- Slow test suites

## Testing Metrics

```java
// Test coverage with JaCoCo
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.11</version>
    <executions>
        <execution>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
        <execution>
            <id>jacoco-check</id>
            <goals>
                <goal>check</goal>
            </goals>
            <configuration>
                <rules>
                    <rule>
                        <element>PACKAGE</element>
                        <limits>
                            <limit>
                                <counter>LINE</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.80</minimum>
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

## Additional Resources

- [JUnit 5 User Guide](https://junit.org/junit5/docs/current/user-guide/)
- [Mockito Documentation](https://javadoc.io/doc/org.mockito/mockito-core/latest/org/mockito/Mockito.html)
- [Testcontainers Documentation](https://www.testcontainers.org/)
- [Spring Boot Testing Guide](https://spring.io/guides/gs/testing-web/)
