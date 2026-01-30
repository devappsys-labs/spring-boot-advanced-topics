# Spring Data JPA

## Overview

Spring Data JPA is a powerful abstraction over JPA (Java Persistence API) that simplifies database operations and reduces boilerplate code. It provides repository support for JPA, enabling developers to work with databases using simple interface-based programming.

## Key Concepts

### 1. Repository Interfaces

Spring Data JPA provides a hierarchy of repository interfaces:

```java
// JpaRepository extends PagingAndSortingRepository, which extends CrudRepository
public interface UserRepository extends JpaRepository<User, Long> {
    // Custom query methods
}
```

### 2. Entity Mapping

```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false, unique = true)
    private String email;
    
    @Column(name = "full_name")
    private String fullName;
    
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL)
    private List<Order> orders;
    
    // Getters and setters
}
```

### 3. Query Methods

#### Derived Query Methods

Spring Data JPA automatically generates queries based on method names:

```java
public interface UserRepository extends JpaRepository<User, Long> {
    // Find by single field
    List<User> findByEmail(String email);
    
    // Find with multiple conditions
    List<User> findByFullNameAndEmail(String fullName, String email);
    
    // Find with LIKE
    List<User> findByFullNameContaining(String namePart);
    
    // Find with ordering
    List<User> findByEmailOrderByFullNameAsc(String email);
    
    // Count queries
    long countByEmail(String email);
    
    // Existence check
    boolean existsByEmail(String email);
    
    // Delete queries
    void deleteByEmail(String email);
}
```

#### @Query Annotation (JPQL)

```java
public interface UserRepository extends JpaRepository<User, Long> {
    @Query("SELECT u FROM User u WHERE u.email = :email")
    Optional<User> findUserByEmail(@Param("email") String email);
    
    @Query("SELECT u FROM User u WHERE u.fullName LIKE %:name%")
    List<User> searchByName(@Param("name") String name);
    
    // Update query
    @Modifying
    @Query("UPDATE User u SET u.fullName = :name WHERE u.id = :id")
    int updateUserName(@Param("id") Long id, @Param("name") String name);
}
```

### 4. Native Queries

For complex queries or database-specific features:

```java
public interface UserRepository extends JpaRepository<User, Long> {
    @Query(value = "SELECT * FROM users WHERE email = ?1", nativeQuery = true)
    User findByEmailNative(String email);
    
    @Query(value = """
        SELECT u.*, COUNT(o.id) as order_count 
        FROM users u 
        LEFT JOIN orders o ON u.id = o.user_id 
        GROUP BY u.id 
        HAVING COUNT(o.id) > :minOrders
        """, nativeQuery = true)
    List<Object[]> findUsersWithMinimumOrders(@Param("minOrders") int minOrders);
}
```

### 5. Specifications (Dynamic Queries)

For building dynamic, type-safe queries:

```java
public class UserSpecifications {
    public static Specification<User> hasEmail(String email) {
        return (root, query, criteriaBuilder) -> 
            criteriaBuilder.equal(root.get("email"), email);
    }
    
    public static Specification<User> hasFullNameContaining(String namePart) {
        return (root, query, criteriaBuilder) -> 
            criteriaBuilder.like(root.get("fullName"), "%" + namePart + "%");
    }
    
    public static Specification<User> createdAfter(LocalDateTime date) {
        return (root, query, criteriaBuilder) -> 
            criteriaBuilder.greaterThan(root.get("createdAt"), date);
    }
}

// Repository interface
public interface UserRepository extends JpaRepository<User, Long>, 
                                        JpaSpecificationExecutor<User> {
}

// Usage
Specification<User> spec = Specification
    .where(UserSpecifications.hasEmail("test@example.com"))
    .and(UserSpecifications.hasFullNameContaining("John"));
    
List<User> users = userRepository.findAll(spec);
```

### 6. Materialized Views

Materialized views improve query performance for complex aggregations:

```java
// Create materialized view entity
@Entity
@Immutable
@Subselect("""
    SELECT 
        u.id as user_id,
        u.email,
        COUNT(o.id) as total_orders,
        SUM(o.amount) as total_spent
    FROM users u
    LEFT JOIN orders o ON u.id = o.user_id
    GROUP BY u.id, u.email
    """)
public class UserOrderSummary {
    @Id
    @Column(name = "user_id")
    private Long userId;
    
    private String email;
    
    @Column(name = "total_orders")
    private Long totalOrders;
    
    @Column(name = "total_spent")
    private BigDecimal totalSpent;
    
    // Getters only (no setters for immutable view)
}

// Repository for materialized view
public interface UserOrderSummaryRepository extends JpaRepository<UserOrderSummary, Long> {
    List<UserOrderSummary> findByTotalOrdersGreaterThan(Long minOrders);
}
```

### 7. Projections

Fetch only specific fields to optimize performance:

```java
// Interface-based projection
public interface UserSummary {
    String getEmail();
    String getFullName();
    
    @Value("#{target.fullName + ' <' + target.email + '>'}")
    String getFullInfo();
}

// Class-based projection (DTO)
public class UserDTO {
    private String email;
    private String fullName;
    
    public UserDTO(String email, String fullName) {
        this.email = email;
        this.fullName = fullName;
    }
    
    // Getters
}

// Repository
public interface UserRepository extends JpaRepository<User, Long> {
    List<UserSummary> findByFullNameContaining(String name);
    
    @Query("SELECT new com.example.dto.UserDTO(u.email, u.fullName) FROM User u")
    List<UserDTO> findAllUserDTOs();
}
```

### 8. Pagination and Sorting

```java
public interface UserRepository extends JpaRepository<User, Long> {
    Page<User> findByFullNameContaining(String name, Pageable pageable);
}

// Usage
Pageable pageable = PageRequest.of(0, 10, Sort.by("fullName").ascending());
Page<User> userPage = userRepository.findByFullNameContaining("John", pageable);

System.out.println("Total elements: " + userPage.getTotalElements());
System.out.println("Total pages: " + userPage.getTotalPages());
System.out.println("Current page: " + userPage.getNumber());
```

## POC Requirements

Build a complete application demonstrating:

1. **Entity Design**: Create at least 3 entities with relationships (One-to-Many, Many-to-Many)
2. **Repository Methods**: Implement derived queries, @Query, and native queries
3. **Specifications**: Create dynamic query builder using Specifications
4. **Materialized View**: Implement at least one materialized view for reporting
5. **Projections**: Use both interface and class-based projections
6. **Pagination**: Implement paginated endpoints with sorting

## Configuration

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: postgres
    password: password
    driver-class-name: org.postgresql.Driver
  
  jpa:
    hibernate:
      ddl-auto: validate  # Use Flyway for migrations
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        dialect: org.hibernate.dialect.PostgreSQLDialect
        default_schema: public
```

## Best Practices

1. **Use Specifications for Dynamic Queries**: Avoid string concatenation in WHERE clauses
2. **Optimize with Projections**: Don't fetch entire entities when you need only few fields
3. **Index Materialized Views**: Refresh them periodically for updated data
4. **Use @Transactional**: For write operations and multi-step processes
5. **Avoid N+1 Queries**: Use `@EntityGraph` or JOIN FETCH
6. **Batch Operations**: Use `saveAll()` instead of multiple `save()` calls

## Common Pitfalls

1. **Missing @Transactional on Updates**: `@Modifying` queries need transactional context
2. **Lazy Loading Outside Transaction**: Access lazy collections within transactional boundaries
3. **Overusing Native Queries**: Prefer JPQL for database portability
4. **Not Using Pagination**: Always paginate large result sets

## Additional Resources

- [Spring Data JPA Documentation](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/)
- [JPA Specification](https://jakarta.ee/specifications/persistence/3.0/)
- [Hibernate Documentation](https://hibernate.org/orm/documentation/)
