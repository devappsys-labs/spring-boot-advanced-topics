# Flyway Database Migrations

## Overview

Flyway is a database migration tool that allows version control for database schemas. It tracks, manages, and applies database schema changes in a reliable and consistent manner across different environments.

## Key Concepts

### 1. Migration Files

Flyway uses SQL or Java-based migration files with a specific naming convention:

```
V{version}__{description}.sql
R{version}__{description}.sql  // Repeatable migrations
```

Example:
- `V1__create_users_table.sql`
- `V2__add_email_to_users.sql`
- `V3__create_orders_table.sql`

### 2. Basic Migration Example

```sql
-- V1__create_users_table.sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    full_name VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);
```

```sql
-- V2__create_orders_table.sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    amount DECIMAL(10, 2) NOT NULL,
    status VARCHAR(50) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);
```

### 3. Configuration

```yaml
# application.yml
spring:
  flyway:
    enabled: true
    baseline-on-migrate: true
    locations: classpath:db/migration
    schemas: public
    table: flyway_schema_history
    validate-on-migrate: true
    out-of-order: false
```

```properties
# application.properties
spring.flyway.enabled=true
spring.flyway.baseline-on-migrate=true
spring.flyway.locations=classpath:db/migration
spring.flyway.schemas=public
```

### 4. Directory Structure

```
src/main/resources/
└── db/
    └── migration/
        ├── V1__create_users_table.sql
        ├── V2__create_orders_table.sql
        ├── V3__add_indexes.sql
        └── R__refresh_materialized_views.sql
```

### 5. Versioned Migrations

```sql
-- V4__add_user_profile.sql
ALTER TABLE users ADD COLUMN phone_number VARCHAR(20);
ALTER TABLE users ADD COLUMN address TEXT;
ALTER TABLE users ADD COLUMN date_of_birth DATE;

CREATE TABLE user_preferences (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL UNIQUE,
    theme VARCHAR(20) DEFAULT 'light',
    language VARCHAR(10) DEFAULT 'en',
    notifications_enabled BOOLEAN DEFAULT true,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);
```

### 6. Repeatable Migrations

Repeatable migrations run every time their checksum changes:

```sql
-- R__create_views.sql
DROP VIEW IF EXISTS user_order_summary;

CREATE VIEW user_order_summary AS
SELECT 
    u.id as user_id,
    u.email,
    u.full_name,
    COUNT(o.id) as total_orders,
    COALESCE(SUM(o.amount), 0) as total_spent,
    MAX(o.created_at) as last_order_date
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.email, u.full_name;
```

### 7. Java-Based Migrations

For complex data transformations:

```java
package db.migration;

import org.flywaydb.core.api.migration.BaseJavaMigration;
import org.flywaydb.core.api.migration.Context;
import java.sql.Statement;

public class V5__migrate_user_data extends BaseJavaMigration {
    @Override
    public void migrate(Context context) throws Exception {
        try (Statement statement = context.getConnection().createStatement()) {
            // Complex data transformation
            statement.execute(
                "UPDATE users SET full_name = INITCAP(full_name) " +
                "WHERE full_name IS NOT NULL"
            );
            
            // Insert default preferences for existing users
            statement.execute(
                "INSERT INTO user_preferences (user_id, theme, language) " +
                "SELECT id, 'light', 'en' FROM users " +
                "WHERE id NOT IN (SELECT user_id FROM user_preferences)"
            );
        }
    }
}
```

### 8. Rollback Strategy

Flyway doesn't support automatic rollback, but you can create undo scripts:

```sql
-- V6__add_column.sql
ALTER TABLE users ADD COLUMN status VARCHAR(20) DEFAULT 'active';

-- U6__add_column.sql (manual undo script)
ALTER TABLE users DROP COLUMN status;
```

### 9. Environment-Specific Migrations

```yaml
# application-dev.yml
spring:
  flyway:
    locations: classpath:db/migration,classpath:db/migration/dev

# application-prod.yml
spring:
  flyway:
    locations: classpath:db/migration,classpath:db/migration/prod
```

### 10. Callbacks

Execute custom logic at specific points in the migration lifecycle:

```java
@Component
public class FlywayCallback implements Callback {
    
    @Override
    public boolean supports(Event event, Context context) {
        return event == Event.AFTER_MIGRATE;
    }
    
    @Override
    public boolean canHandleInTransaction(Event event, Context context) {
        return true;
    }
    
    @Override
    public void handle(Event event, Context context) {
        System.out.println("Migration completed successfully!");
        // Send notification, update cache, etc.
    }
    
    @Override
    public String getCallbackName() {
        return FlywayCallback.class.getSimpleName();
    }
}
```

## POC Requirements

Build a migration strategy that includes:

1. **Initial Schema**: Create base tables with proper constraints
2. **Incremental Changes**: Add columns, indexes, and relationships
3. **Data Migrations**: Include Java-based migration for data transformation
4. **Repeatable Migrations**: Create views and functions
5. **Callbacks**: Implement post-migration validation

## Best Practices

1. **Never Modify Committed Migrations**: Create new migration files instead
2. **Use Descriptive Names**: Make migration purpose clear from filename
3. **Test Migrations**: Always test in dev/staging before production
4. **Keep Migrations Small**: One logical change per migration
5. **Include Rollback Plan**: Document how to undo changes
6. **Use Transactions**: Wrap DDL in transactions where supported

## Common Commands

```bash
# Run migrations
mvn flyway:migrate

# Validate migrations
mvn flyway:validate

# Get migration info
mvn flyway:info

# Clean database (DEV ONLY!)
mvn flyway:clean

# Repair migration history
mvn flyway:repair

# Baseline existing database
mvn flyway:baseline
```

## Common Pitfalls

1. **Modifying Applied Migrations**: Will cause checksum mismatch
2. **Missing Dependencies**: Ensure database driver is in classpath
3. **Wrong Execution Order**: Use proper version numbering (V1, V2, not V1, V10)
4. **Not Testing Rollback**: Always have a rollback plan

## Integration with CI/CD

```yaml
# Example GitHub Actions workflow
- name: Run Flyway Migrations
  run: |
    mvn flyway:migrate \
      -Dflyway.url=${{ secrets.DB_URL }} \
      -Dflyway.user=${{ secrets.DB_USER }} \
      -Dflyway.password=${{ secrets.DB_PASSWORD }}
```

## Monitoring

Track migration status:

```sql
-- Query flyway_schema_history table
SELECT 
    installed_rank,
    version,
    description,
    type,
    script,
    checksum,
    installed_on,
    execution_time,
    success
FROM flyway_schema_history
ORDER BY installed_rank DESC;
```

## Additional Resources

- [Flyway Documentation](https://documentation.red-gate.com/fd)
- [Migration Best Practices](https://documentation.red-gate.com/fd/best-practices-184127474.html)

## Next Steps

Proceed to [Logback Spring](../03-logback-spring/README.md) to learn about application logging configuration.