# GraphQL with Spring Boot

## Overview

GraphQL is a query language for APIs that gives clients the power to ask for exactly the data they need. Unlike REST, where the server defines fixed endpoints and response shapes, GraphQL exposes a single endpoint and lets the client specify the structure of the response. Spring for GraphQL (formerly Spring Boot GraphQL Starter) provides first-class integration with Spring Boot, leveraging annotations, schema-first development, and the full Spring ecosystem.

## Key Concepts

### GraphQL Fundamentals

- **Schema-first** - Define your API contract in `.graphqls` schema files
- **Single endpoint** - All operations go through one URL (`/graphql`)
- **Client-driven queries** - Clients request exactly the fields they need
- **Strongly typed** - Every field and argument has a defined type
- **Introspectable** - Clients can query the schema itself

### Operation Types

- **Query** - Read operations (analogous to GET)
- **Mutation** - Write/update operations (analogous to POST/PUT/DELETE)
- **Subscription** - Real-time data via server push (WebSocket-based)

### Core Building Blocks

- **Types** - Object definitions with fields (like DTOs)
- **Resolvers** - Functions that return data for each field
- **DataLoader** - Batching mechanism to solve the N+1 problem
- **Scalars** - Primitive types (String, Int, Float, Boolean, ID) and custom ones

## Dependencies

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-graphql</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <!-- For subscriptions (WebSocket) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-websocket</artifactId>
    </dependency>
    <!-- For testing -->
    <dependency>
        <groupId>org.springframework.graphql</groupId>
        <artifactId>spring-graphql-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

## Application Configuration

```yaml
spring:
  graphql:
    graphiql:
      enabled: true          # Enable GraphiQL UI at /graphiql
    schema:
      printer:
        enabled: true        # Enable schema endpoint at /graphql/schema
      locations: classpath:graphql/   # Schema file location
    websocket:
      path: /graphql         # WebSocket endpoint for subscriptions
```

## Schema Definition

### schema.graphqls

Place schema files under `src/main/resources/graphql/`.

```graphql
type Query {
    bookById(id: ID!): Book
    allBooks(page: Int = 0, size: Int = 10): BookConnection!
    booksByAuthor(authorId: ID!): [Book!]!
    searchBooks(keyword: String!): [Book!]!
}

type Mutation {
    createBook(input: CreateBookInput!): Book!
    updateBook(id: ID!, input: UpdateBookInput!): Book!
    deleteBook(id: ID!): Boolean!
    addReview(bookId: ID!, input: CreateReviewInput!): Review!
}

type Subscription {
    bookCreated: Book!
    reviewAdded(bookId: ID!): Review!
}

type Book {
    id: ID!
    title: String!
    isbn: String
    publishedDate: String
    rating: Float
    author: Author!
    reviews: [Review!]!
}

type Author {
    id: ID!
    name: String!
    email: String
    books: [Book!]!
}

type Review {
    id: ID!
    content: String!
    rating: Int!
    reviewer: String!
    createdAt: String!
}

input CreateBookInput {
    title: String!
    isbn: String
    publishedDate: String
    authorId: ID!
}

input UpdateBookInput {
    title: String
    isbn: String
    publishedDate: String
}

input CreateReviewInput {
    content: String!
    rating: Int!
    reviewer: String!
}

type BookConnection {
    content: [Book!]!
    totalElements: Int!
    totalPages: Int!
    currentPage: Int!
}
```

## Entity/Model Setup

### Book Entity

```java
@Entity
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Book {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String title;

    private String isbn;

    private LocalDate publishedDate;

    private Double rating;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "author_id", nullable = false)
    private Author author;

    @OneToMany(mappedBy = "book", cascade = CascadeType.ALL)
    private List<Review> reviews = new ArrayList<>();
}
```

### Author Entity

```java
@Entity
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Author {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    private String email;

    @OneToMany(mappedBy = "author", cascade = CascadeType.ALL)
    private List<Book> books = new ArrayList<>();
}
```

### Review Entity

```java
@Entity
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Review {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String content;

    @Column(nullable = false)
    private Integer rating;

    @Column(nullable = false)
    private String reviewer;

    private LocalDateTime createdAt;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "book_id", nullable = false)
    private Book book;
}
```

## Query Resolvers

### BookController (Query)

```java
@Controller
public class BookController {

    private final BookRepository bookRepository;
    private final AuthorRepository authorRepository;

    public BookController(BookRepository bookRepository, AuthorRepository authorRepository) {
        this.bookRepository = bookRepository;
        this.authorRepository = authorRepository;
    }

    @QueryMapping
    public Book bookById(@Argument Long id) {
        return bookRepository.findById(id)
                .orElseThrow(() -> new BookNotFoundException("Book not found: " + id));
    }

    @QueryMapping
    public BookConnection allBooks(@Argument int page, @Argument int size) {
        Page<Book> bookPage = bookRepository.findAll(PageRequest.of(page, size));
        return new BookConnection(
                bookPage.getContent(),
                bookPage.getTotalElements(),
                bookPage.getTotalPages(),
                bookPage.getNumber()
        );
    }

    @QueryMapping
    public List<Book> booksByAuthor(@Argument Long authorId) {
        return bookRepository.findByAuthorId(authorId);
    }

    @QueryMapping
    public List<Book> searchBooks(@Argument String keyword) {
        return bookRepository.findByTitleContainingIgnoreCase(keyword);
    }
}
```

### BookConnection DTO

```java
@Data
@AllArgsConstructor
public class BookConnection {
    private List<Book> content;
    private long totalElements;
    private int totalPages;
    private int currentPage;
}
```

## Nested Field Resolvers

When a field requires separate data fetching (e.g., loading an `Author` for a `Book`), use `@SchemaMapping`.

```java
@Controller
public class BookFieldResolver {

    private final AuthorRepository authorRepository;
    private final ReviewRepository reviewRepository;

    public BookFieldResolver(AuthorRepository authorRepository, ReviewRepository reviewRepository) {
        this.authorRepository = authorRepository;
        this.reviewRepository = reviewRepository;
    }

    @SchemaMapping(typeName = "Book", field = "author")
    public Author getAuthor(Book book) {
        return authorRepository.findById(book.getAuthor().getId())
                .orElse(null);
    }

    @SchemaMapping(typeName = "Book", field = "reviews")
    public List<Review> getReviews(Book book) {
        return reviewRepository.findByBookId(book.getId());
    }

    @SchemaMapping(typeName = "Author", field = "books")
    public List<Book> getBooksByAuthor(Author author) {
        return bookRepository.findByAuthorId(author.getId());
    }
}
```

## Mutation Resolvers

```java
@Controller
public class BookMutationController {

    private final BookRepository bookRepository;
    private final AuthorRepository authorRepository;
    private final ReviewRepository reviewRepository;
    private final BookEventPublisher bookEventPublisher;

    public BookMutationController(BookRepository bookRepository,
                                   AuthorRepository authorRepository,
                                   ReviewRepository reviewRepository,
                                   BookEventPublisher bookEventPublisher) {
        this.bookRepository = bookRepository;
        this.authorRepository = authorRepository;
        this.reviewRepository = reviewRepository;
        this.bookEventPublisher = bookEventPublisher;
    }

    @MutationMapping
    public Book createBook(@Argument CreateBookInput input) {
        Author author = authorRepository.findById(input.getAuthorId())
                .orElseThrow(() -> new AuthorNotFoundException("Author not found: " + input.getAuthorId()));

        Book book = Book.builder()
                .title(input.getTitle())
                .isbn(input.getIsbn())
                .publishedDate(input.getPublishedDate())
                .author(author)
                .build();

        Book saved = bookRepository.save(book);
        bookEventPublisher.publishBookCreated(saved);
        return saved;
    }

    @MutationMapping
    public Book updateBook(@Argument Long id, @Argument UpdateBookInput input) {
        Book book = bookRepository.findById(id)
                .orElseThrow(() -> new BookNotFoundException("Book not found: " + id));

        if (input.getTitle() != null) book.setTitle(input.getTitle());
        if (input.getIsbn() != null) book.setIsbn(input.getIsbn());
        if (input.getPublishedDate() != null) book.setPublishedDate(input.getPublishedDate());

        return bookRepository.save(book);
    }

    @MutationMapping
    public boolean deleteBook(@Argument Long id) {
        if (!bookRepository.existsById(id)) {
            throw new BookNotFoundException("Book not found: " + id);
        }
        bookRepository.deleteById(id);
        return true;
    }

    @MutationMapping
    public Review addReview(@Argument Long bookId, @Argument CreateReviewInput input) {
        Book book = bookRepository.findById(bookId)
                .orElseThrow(() -> new BookNotFoundException("Book not found: " + bookId));

        Review review = Review.builder()
                .content(input.getContent())
                .rating(input.getRating())
                .reviewer(input.getReviewer())
                .createdAt(LocalDateTime.now())
                .book(book)
                .build();

        Review saved = reviewRepository.save(review);
        bookEventPublisher.publishReviewAdded(bookId, saved);
        return saved;
    }
}
```

### Input DTOs

```java
@Data
public class CreateBookInput {
    private String title;
    private String isbn;
    private LocalDate publishedDate;
    private Long authorId;
}

@Data
public class UpdateBookInput {
    private String title;
    private String isbn;
    private LocalDate publishedDate;
}

@Data
public class CreateReviewInput {
    private String content;
    private Integer rating;
    private String reviewer;
}
```

## Subscriptions

### Event Publisher

```java
@Component
public class BookEventPublisher {

    private final Sinks.Many<Book> bookCreatedSink = Sinks.many().multicast().onBackpressureBuffer();
    private final Sinks.Many<Review> reviewAddedSink = Sinks.many().multicast().onBackpressureBuffer();

    public void publishBookCreated(Book book) {
        bookCreatedSink.tryEmitNext(book);
    }

    public void publishReviewAdded(Long bookId, Review review) {
        reviewAddedSink.tryEmitNext(review);
    }

    public Flux<Book> getBookCreatedStream() {
        return bookCreatedSink.asFlux();
    }

    public Flux<Review> getReviewAddedStream(Long bookId) {
        return reviewAddedSink.asFlux()
                .filter(review -> review.getBook().getId().equals(bookId));
    }
}
```

### Subscription Controller

```java
@Controller
public class BookSubscriptionController {

    private final BookEventPublisher bookEventPublisher;

    public BookSubscriptionController(BookEventPublisher bookEventPublisher) {
        this.bookEventPublisher = bookEventPublisher;
    }

    @SubscriptionMapping
    public Flux<Book> bookCreated() {
        return bookEventPublisher.getBookCreatedStream();
    }

    @SubscriptionMapping
    public Flux<Review> reviewAdded(@Argument Long bookId) {
        return bookEventPublisher.getReviewAddedStream(bookId);
    }
}
```

### WebSocket Configuration

```java
@Configuration
public class GraphQLWebSocketConfig {

    @Bean
    public ServerWebSocketHandler graphQlWebSocketHandler(
            WebGraphQlHandler webGraphQlHandler) {
        return new GraphQlWebSocketHandler(webGraphQlHandler,
                Jackson2ObjectMapperBuilder.json().build(),
                Duration.ofSeconds(60));
    }
}
```

## DataLoader - Solving the N+1 Problem

Without DataLoader, fetching 10 books with their authors triggers 1 query for books + 10 individual queries for each author. DataLoader batches these into 2 queries total.

### Batch Loader Registration

```java
@Configuration
public class DataLoaderConfig {

    @Bean
    public BatchLoaderRegistry batchLoaderRegistry(AuthorRepository authorRepository,
                                                    ReviewRepository reviewRepository) {
        return new DefaultBatchLoaderRegistry();
    }
}
```

### Registering DataLoaders with RuntimeWiringConfigurer

```java
@Component
public class GraphQLDataLoaders implements RuntimeWiringConfigurer {

    private final AuthorRepository authorRepository;
    private final ReviewRepository reviewRepository;

    public GraphQLDataLoaders(AuthorRepository authorRepository,
                               ReviewRepository reviewRepository) {
        this.authorRepository = authorRepository;
        this.reviewRepository = reviewRepository;
    }

    @Override
    public void configure(RuntimeWiring.Builder builder) {
        // No-op: using @BatchMapping instead
    }
}
```

### Using @BatchMapping (Recommended Approach)

```java
@Controller
public class BookBatchController {

    private final AuthorRepository authorRepository;
    private final ReviewRepository reviewRepository;

    public BookBatchController(AuthorRepository authorRepository,
                                ReviewRepository reviewRepository) {
        this.authorRepository = authorRepository;
        this.reviewRepository = reviewRepository;
    }

    @BatchMapping(typeName = "Book", field = "author")
    public Map<Book, Author> authors(List<Book> books) {
        List<Long> authorIds = books.stream()
                .map(book -> book.getAuthor().getId())
                .distinct()
                .toList();

        Map<Long, Author> authorMap = authorRepository.findAllById(authorIds)
                .stream()
                .collect(Collectors.toMap(Author::getId, Function.identity()));

        return books.stream()
                .collect(Collectors.toMap(
                        Function.identity(),
                        book -> authorMap.get(book.getAuthor().getId())
                ));
    }

    @BatchMapping(typeName = "Book", field = "reviews")
    public Map<Book, List<Review>> reviews(List<Book> books) {
        List<Long> bookIds = books.stream()
                .map(Book::getId)
                .toList();

        Map<Long, List<Review>> reviewMap = reviewRepository.findByBookIdIn(bookIds)
                .stream()
                .collect(Collectors.groupingBy(review -> review.getBook().getId()));

        return books.stream()
                .collect(Collectors.toMap(
                        Function.identity(),
                        book -> reviewMap.getOrDefault(book.getId(), Collections.emptyList())
                ));
    }
}
```

## Custom Scalars

### Registering a Date Scalar

```java
@Configuration
public class GraphQLScalarConfig {

    @Bean
    public RuntimeWiringConfigurer runtimeWiringConfigurer() {
        return wiringBuilder -> wiringBuilder
                .scalar(dateScalar())
                .scalar(dateTimeScalar());
    }

    private GraphQLScalarType dateScalar() {
        return GraphQLScalarType.newScalar()
                .name("Date")
                .description("Java LocalDate as scalar")
                .coercing(new Coercing<LocalDate, String>() {
                    @Override
                    public String serialize(Object dataFetcherResult) {
                        if (dataFetcherResult instanceof LocalDate date) {
                            return date.format(DateTimeFormatter.ISO_LOCAL_DATE);
                        }
                        throw new CoercingSerializeException("Expected a LocalDate object");
                    }

                    @Override
                    public LocalDate parseValue(Object input) {
                        if (input instanceof String str) {
                            return LocalDate.parse(str, DateTimeFormatter.ISO_LOCAL_DATE);
                        }
                        throw new CoercingParseValueException("Expected a String");
                    }

                    @Override
                    public LocalDate parseLiteral(Object input) {
                        if (input instanceof StringValue stringValue) {
                            return LocalDate.parse(stringValue.getValue(),
                                    DateTimeFormatter.ISO_LOCAL_DATE);
                        }
                        throw new CoercingParseLiteralException("Expected a StringValue");
                    }
                })
                .build();
    }

    private GraphQLScalarType dateTimeScalar() {
        return GraphQLScalarType.newScalar()
                .name("DateTime")
                .description("Java LocalDateTime as scalar")
                .coercing(new Coercing<LocalDateTime, String>() {
                    @Override
                    public String serialize(Object dataFetcherResult) {
                        if (dataFetcherResult instanceof LocalDateTime dateTime) {
                            return dateTime.format(DateTimeFormatter.ISO_LOCAL_DATE_TIME);
                        }
                        throw new CoercingSerializeException("Expected a LocalDateTime object");
                    }

                    @Override
                    public LocalDateTime parseValue(Object input) {
                        if (input instanceof String str) {
                            return LocalDateTime.parse(str,
                                    DateTimeFormatter.ISO_LOCAL_DATE_TIME);
                        }
                        throw new CoercingParseValueException("Expected a String");
                    }

                    @Override
                    public LocalDateTime parseLiteral(Object input) {
                        if (input instanceof StringValue stringValue) {
                            return LocalDateTime.parse(stringValue.getValue(),
                                    DateTimeFormatter.ISO_LOCAL_DATE_TIME);
                        }
                        throw new CoercingParseLiteralException("Expected a StringValue");
                    }
                })
                .build();
    }
}
```

### Updated Schema with Custom Scalars

```graphql
scalar Date
scalar DateTime

type Book {
    id: ID!
    title: String!
    isbn: String
    publishedDate: Date
    rating: Float
    author: Author!
    reviews: [Review!]!
}

type Review {
    id: ID!
    content: String!
    rating: Int!
    reviewer: String!
    createdAt: DateTime!
}
```

## Error Handling

### Custom Exception

```java
public class BookNotFoundException extends RuntimeException {
    public BookNotFoundException(String message) {
        super(message);
    }
}

public class AuthorNotFoundException extends RuntimeException {
    public AuthorNotFoundException(String message) {
        super(message);
    }
}
```

### GraphQL Exception Resolver

```java
@Component
public class GraphQLExceptionResolver extends DataFetcherExceptionResolverAdapter {

    @Override
    protected GraphQLError resolveToSingleError(Throwable ex, DataFetchingEnvironment env) {
        if (ex instanceof BookNotFoundException) {
            return GraphqlErrorBuilder.newError(env)
                    .message(ex.getMessage())
                    .errorType(ErrorType.NOT_FOUND)
                    .build();
        }
        if (ex instanceof AuthorNotFoundException) {
            return GraphqlErrorBuilder.newError(env)
                    .message(ex.getMessage())
                    .errorType(ErrorType.NOT_FOUND)
                    .build();
        }
        if (ex instanceof ConstraintViolationException) {
            return GraphqlErrorBuilder.newError(env)
                    .message("Validation failed: " + ex.getMessage())
                    .errorType(ErrorType.BAD_REQUEST)
                    .build();
        }

        return GraphqlErrorBuilder.newError(env)
                .message("Internal server error")
                .errorType(ErrorType.INTERNAL_ERROR)
                .build();
    }
}
```

### Custom Error Type Enum

```java
public enum ErrorType implements graphql.ErrorClassification {
    NOT_FOUND,
    BAD_REQUEST,
    FORBIDDEN,
    INTERNAL_ERROR
}
```

## Input Validation

```java
@Controller
public class ValidatedBookMutationController {

    private final Validator validator;
    private final BookRepository bookRepository;
    private final AuthorRepository authorRepository;

    public ValidatedBookMutationController(Validator validator,
                                            BookRepository bookRepository,
                                            AuthorRepository authorRepository) {
        this.validator = validator;
        this.bookRepository = bookRepository;
        this.authorRepository = authorRepository;
    }

    @MutationMapping
    public Book createBook(@Argument @Valid CreateBookInput input) {
        Set<ConstraintViolation<CreateBookInput>> violations = validator.validate(input);
        if (!violations.isEmpty()) {
            String message = violations.stream()
                    .map(ConstraintViolation::getMessage)
                    .collect(Collectors.joining(", "));
            throw new ValidationException(message);
        }

        Author author = authorRepository.findById(input.getAuthorId())
                .orElseThrow(() -> new AuthorNotFoundException(
                        "Author not found: " + input.getAuthorId()));

        Book book = Book.builder()
                .title(input.getTitle())
                .isbn(input.getIsbn())
                .publishedDate(input.getPublishedDate())
                .author(author)
                .build();

        return bookRepository.save(book);
    }
}
```

### Validated Input DTO

```java
@Data
public class CreateBookInput {

    @NotBlank(message = "Title is required")
    @Size(max = 255, message = "Title must be at most 255 characters")
    private String title;

    @Pattern(regexp = "^(97[89])-\\d{1,5}-\\d{1,7}-\\d{1,7}-\\d$",
             message = "Invalid ISBN format")
    private String isbn;

    @PastOrPresent(message = "Published date cannot be in the future")
    private LocalDate publishedDate;

    @NotNull(message = "Author ID is required")
    private Long authorId;
}
```

## Security and Authorization

### Securing with @PreAuthorize

```java
@Controller
public class SecuredBookController {

    private final BookRepository bookRepository;

    public SecuredBookController(BookRepository bookRepository) {
        this.bookRepository = bookRepository;
    }

    @QueryMapping
    @PreAuthorize("hasRole('USER')")
    public List<Book> allBooks() {
        return bookRepository.findAll();
    }

    @MutationMapping
    @PreAuthorize("hasRole('ADMIN')")
    public Book createBook(@Argument CreateBookInput input) {
        // Only admins can create books
        // ...
        return null;
    }

    @MutationMapping
    @PreAuthorize("hasRole('ADMIN')")
    public boolean deleteBook(@Argument Long id) {
        bookRepository.deleteById(id);
        return true;
    }
}
```

### GraphQL Security Configuration

```java
@Configuration
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/graphiql/**").permitAll()
                .requestMatchers("/graphql").authenticated()
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()));

        return http.build();
    }
}
```

## Pagination - Cursor-Based

### Schema

```graphql
type Query {
    booksConnection(first: Int, after: String): BookEdgeConnection!
}

type BookEdgeConnection {
    edges: [BookEdge!]!
    pageInfo: PageInfo!
    totalCount: Int!
}

type BookEdge {
    node: Book!
    cursor: String!
}

type PageInfo {
    hasNextPage: Boolean!
    hasPreviousPage: Boolean!
    startCursor: String
    endCursor: String
}
```

### Cursor Pagination Controller

```java
@Controller
public class BookPaginationController {

    private final BookRepository bookRepository;

    public BookPaginationController(BookRepository bookRepository) {
        this.bookRepository = bookRepository;
    }

    @QueryMapping
    public BookEdgeConnection booksConnection(@Argument Integer first, @Argument String after) {
        int limit = (first != null) ? first : 10;
        Long afterId = (after != null) ? decodeCursor(after) : 0L;

        List<Book> books = bookRepository.findByIdGreaterThanOrderByIdAsc(afterId,
                PageRequest.of(0, limit + 1));

        boolean hasNextPage = books.size() > limit;
        List<Book> resultBooks = hasNextPage ? books.subList(0, limit) : books;

        List<BookEdge> edges = resultBooks.stream()
                .map(book -> new BookEdge(book, encodeCursor(book.getId())))
                .toList();

        PageInfo pageInfo = new PageInfo(
                hasNextPage,
                afterId > 0,
                edges.isEmpty() ? null : edges.get(0).getCursor(),
                edges.isEmpty() ? null : edges.get(edges.size() - 1).getCursor()
        );

        long totalCount = bookRepository.count();

        return new BookEdgeConnection(edges, pageInfo, totalCount);
    }

    private String encodeCursor(Long id) {
        return Base64.getEncoder().encodeToString(("cursor:" + id).getBytes());
    }

    private Long decodeCursor(String cursor) {
        String decoded = new String(Base64.getDecoder().decode(cursor));
        return Long.parseLong(decoded.replace("cursor:", ""));
    }
}

@Data
@AllArgsConstructor
public class BookEdge {
    private Book node;
    private String cursor;
}

@Data
@AllArgsConstructor
public class BookEdgeConnection {
    private List<BookEdge> edges;
    private PageInfo pageInfo;
    private long totalCount;
}

@Data
@AllArgsConstructor
public class PageInfo {
    private boolean hasNextPage;
    private boolean hasPreviousPage;
    private String startCursor;
    private String endCursor;
}
```

## Interceptors

### Request Interceptor

```java
@Component
public class GraphQLRequestInterceptor implements WebGraphQlInterceptor {

    private static final Logger log = LoggerFactory.getLogger(GraphQLRequestInterceptor.class);

    @Override
    public Mono<WebGraphQlResponse> intercept(WebGraphQlRequest request, Chain chain) {
        log.info("GraphQL operation: {} | name: {}",
                request.getDocument(),
                request.getOperationName());

        long startTime = System.currentTimeMillis();

        return chain.next(request)
                .doOnNext(response -> {
                    long duration = System.currentTimeMillis() - startTime;
                    log.info("GraphQL response completed in {}ms", duration);
                });
    }
}
```

### Query Complexity Limiter

```java
@Component
public class QueryComplexityInterceptor implements WebGraphQlInterceptor {

    private static final int MAX_DEPTH = 5;

    @Override
    public Mono<WebGraphQlResponse> intercept(WebGraphQlRequest request, Chain chain) {
        int depth = calculateDepth(request.getDocument());
        if (depth > MAX_DEPTH) {
            throw new RuntimeException(
                    "Query depth " + depth + " exceeds maximum allowed depth of " + MAX_DEPTH);
        }
        return chain.next(request);
    }

    private int calculateDepth(String document) {
        int depth = 0;
        int maxDepth = 0;
        for (char c : document.toCharArray()) {
            if (c == '{') {
                depth++;
                maxDepth = Math.max(maxDepth, depth);
            } else if (c == '}') {
                depth--;
            }
        }
        return maxDepth;
    }
}
```

## File Upload

### Schema

```graphql
scalar Upload

type Mutation {
    uploadBookCover(bookId: ID!, file: Upload!): String!
}
```

### Upload Controller

```java
@Controller
public class FileUploadController {

    private final BookRepository bookRepository;
    private final StorageService storageService;

    public FileUploadController(BookRepository bookRepository,
                                 StorageService storageService) {
        this.bookRepository = bookRepository;
        this.storageService = storageService;
    }

    @MutationMapping
    public String uploadBookCover(@Argument Long bookId,
                                   @Argument MultipartFile file) throws IOException {
        Book book = bookRepository.findById(bookId)
                .orElseThrow(() -> new BookNotFoundException("Book not found: " + bookId));

        String url = storageService.store(file, "covers/" + bookId);
        return url;
    }
}
```

## Testing

### GraphQlTester - Schema Test

```java
@GraphQlTest(BookController.class)
class BookControllerTest {

    @Autowired
    private GraphQlTester graphQlTester;

    @MockitoBean
    private BookRepository bookRepository;

    @MockitoBean
    private AuthorRepository authorRepository;

    @Test
    void shouldReturnBookById() {
        Author author = Author.builder().id(1L).name("Author 1").build();
        Book book = Book.builder()
                .id(1L)
                .title("Spring in Action")
                .isbn("978-1617294945")
                .author(author)
                .build();

        when(bookRepository.findById(1L)).thenReturn(Optional.of(book));

        graphQlTester.document("""
                    query {
                        bookById(id: 1) {
                            id
                            title
                            isbn
                        }
                    }
                """)
                .execute()
                .path("bookById.id").entity(String.class).isEqualTo("1")
                .path("bookById.title").entity(String.class).isEqualTo("Spring in Action")
                .path("bookById.isbn").entity(String.class).isEqualTo("978-1617294945");
    }

    @Test
    void shouldReturnErrorForMissingBook() {
        when(bookRepository.findById(999L)).thenReturn(Optional.empty());

        graphQlTester.document("""
                    query {
                        bookById(id: 999) {
                            id
                            title
                        }
                    }
                """)
                .execute()
                .errors()
                .satisfy(errors -> {
                    assertThat(errors).hasSize(1);
                    assertThat(errors.get(0).getMessage()).contains("Book not found");
                });
    }
}
```

### Mutation Test

```java
@GraphQlTest(BookMutationController.class)
class BookMutationControllerTest {

    @Autowired
    private GraphQlTester graphQlTester;

    @MockitoBean
    private BookRepository bookRepository;

    @MockitoBean
    private AuthorRepository authorRepository;

    @MockitoBean
    private ReviewRepository reviewRepository;

    @MockitoBean
    private BookEventPublisher bookEventPublisher;

    @Test
    void shouldCreateBook() {
        Author author = Author.builder().id(1L).name("Author 1").build();
        Book savedBook = Book.builder()
                .id(1L)
                .title("New Book")
                .isbn("978-0000000000")
                .author(author)
                .build();

        when(authorRepository.findById(1L)).thenReturn(Optional.of(author));
        when(bookRepository.save(any(Book.class))).thenReturn(savedBook);

        graphQlTester.document("""
                    mutation {
                        createBook(input: {
                            title: "New Book",
                            isbn: "978-0000000000",
                            authorId: 1
                        }) {
                            id
                            title
                            isbn
                        }
                    }
                """)
                .execute()
                .path("createBook.title").entity(String.class).isEqualTo("New Book")
                .path("createBook.isbn").entity(String.class).isEqualTo("978-0000000000");
    }

    @Test
    void shouldDeleteBook() {
        when(bookRepository.existsById(1L)).thenReturn(true);

        graphQlTester.document("""
                    mutation {
                        deleteBook(id: 1)
                    }
                """)
                .execute()
                .path("deleteBook").entity(Boolean.class).isEqualTo(true);

        verify(bookRepository).deleteById(1L);
    }
}
```

### Integration Test with HttpGraphQlTester

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureHttpGraphQlTester
class BookIntegrationTest {

    @Autowired
    private HttpGraphQlTester graphQlTester;

    @Test
    void shouldFetchAllBooksEndToEnd() {
        graphQlTester.document("""
                    query {
                        allBooks(page: 0, size: 5) {
                            content {
                                id
                                title
                                author {
                                    name
                                }
                            }
                            totalElements
                            totalPages
                        }
                    }
                """)
                .execute()
                .path("allBooks.content").entityList(Book.class).hasSizeGreaterThan(0)
                .path("allBooks.totalElements").entity(Integer.class).satisfies(total -> {
                    assertThat(total).isGreaterThan(0);
                });
    }
}
```

### Subscription Test

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureWebTestClient
class BookSubscriptionTest {

    @Autowired
    private WebSocketGraphQlTester graphQlTester;

    @Autowired
    private BookEventPublisher bookEventPublisher;

    @Test
    void shouldReceiveBookCreatedEvent() {
        Flux<Book> result = graphQlTester.document("""
                    subscription {
                        bookCreated {
                            id
                            title
                        }
                    }
                """)
                .executeSubscription()
                .toFlux("bookCreated", Book.class);

        // Publish an event
        Book book = Book.builder().id(1L).title("New Book").build();
        bookEventPublisher.publishBookCreated(book);

        StepVerifier.create(result.take(1))
                .assertNext(received -> {
                    assertThat(received.getTitle()).isEqualTo("New Book");
                })
                .verifyComplete();
    }
}
```

## Client Query Examples

### Query

```graphql
# Fetch a book with author and reviews
query {
    bookById(id: 1) {
        id
        title
        isbn
        publishedDate
        rating
        author {
            name
            email
        }
        reviews {
            content
            rating
            reviewer
            createdAt
        }
    }
}

# Search books
query {
    searchBooks(keyword: "Spring") {
        id
        title
        author {
            name
        }
    }
}

# Paginated books
query {
    allBooks(page: 0, size: 5) {
        content {
            id
            title
        }
        totalElements
        totalPages
        currentPage
    }
}
```

### Mutation

```graphql
# Create a book
mutation {
    createBook(input: {
        title: "Spring Boot in Practice"
        isbn: "978-1617298813"
        publishedDate: "2022-08-16"
        authorId: 1
    }) {
        id
        title
    }
}

# Update a book
mutation {
    updateBook(id: 1, input: {
        title: "Spring Boot in Practice - 2nd Edition"
    }) {
        id
        title
    }
}

# Add a review
mutation {
    addReview(bookId: 1, input: {
        content: "Excellent book for Spring Boot developers"
        rating: 5
        reviewer: "John Doe"
    }) {
        id
        content
        rating
        createdAt
    }
}
```

### Subscription

```graphql
# Subscribe to new books
subscription {
    bookCreated {
        id
        title
        author {
            name
        }
    }
}

# Subscribe to reviews on a specific book
subscription {
    reviewAdded(bookId: 1) {
        content
        rating
        reviewer
    }
}
```

## Best Practices

1. **Schema-first approach** - Design the schema before writing resolvers; the schema is your API contract
2. **Use @BatchMapping for relations** - Avoid N+1 queries by batching related entity fetches
3. **Validate inputs** - Use Bean Validation on input DTOs to catch bad data early
4. **Limit query depth and complexity** - Prevent abusive queries from overloading the server
5. **Use cursor-based pagination** - More reliable than offset-based for large, changing datasets
6. **Handle errors gracefully** - Map domain exceptions to meaningful GraphQL errors with proper error types
7. **Secure at the resolver level** - Use `@PreAuthorize` to enforce authorization per operation
8. **Keep resolvers thin** - Delegate business logic to service classes
9. **Use custom scalars** - Map Java types (LocalDate, LocalDateTime, BigDecimal) to GraphQL scalars
10. **Enable GraphiQL in dev only** - Disable the interactive UI in production

## Common Pitfalls

1. N+1 queries from nested resolvers without DataLoader or `@BatchMapping`
2. Over-fetching from the database even when the client requests few fields
3. Exposing internal entity structure directly instead of using a dedicated GraphQL schema
4. Missing error handling causing raw stack traces in API responses
5. Not limiting query depth, allowing deeply nested queries to exhaust resources
6. Blocking calls inside reactive subscription handlers
7. Forgetting to configure WebSocket for subscriptions
8. Using offset pagination for real-time data where cursor pagination is more appropriate

## Resources

- [Spring for GraphQL Documentation](https://docs.spring.io/spring-graphql/reference/)
- [GraphQL Specification](https://spec.graphql.org/)
- [GraphQL Java Library](https://www.graphql-java.com/)
- [GraphiQL IDE](https://github.com/graphql/graphiql)
- [Relay Cursor Connection Specification](https://relay.dev/graphql/connections.htm)
