# Spring Data Batch Processing

## Overview

Spring Batch is a comprehensive framework for developing robust batch processing applications. It provides reusable functions essential for processing large volumes of data, including logging/tracing, transaction management, job processing statistics, job restart, skip, and resource management.

## Key Concepts

### Core Components

1. **Job** - The batch process to be executed
2. **Step** - A sequential phase of a job
3. **JobRepository** - Persistence mechanism for job metadata
4. **JobLauncher** - Interface for launching jobs
5. **ItemReader** - Strategy for reading input data
6. **ItemProcessor** - Business logic for transforming data
7. **ItemWriter** - Strategy for writing output data

### Job Execution Flow

```
Job → Step(s) → ItemReader → ItemProcessor → ItemWriter
```

## Spring Boot Integration

### Dependencies

```xml
<dependencies>
    <!-- Spring Batch -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-batch</artifactId>
    </dependency>
    
    <!-- Spring Data JPA -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    
    <!-- Database -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
    </dependency>
    
    <!-- CSV Processing -->
    <dependency>
        <groupId>com.opencsv</groupId>
        <artifactId>opencsv</artifactId>
        <version>5.9</version>
    </dependency>
    
    <!-- Test -->
    <dependency>
        <groupId>org.springframework.batch</groupId>
        <artifactId>spring-batch-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### Gradle Configuration

```gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-batch'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.postgresql:postgresql'
    implementation 'com.opencsv:opencsv:5.9'
    testImplementation 'org.springframework.batch:spring-batch-test'
}
```

## Basic Configuration

### Enable Batch Processing

```java
package com.example.batch.config;

import org.springframework.batch.core.configuration.annotation.EnableBatchProcessing;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
@EnableBatchProcessing
public class BatchApplication {
    public static void main(String[] args) {
        SpringApplication.run(BatchApplication.class, args);
    }
}
```

### Application Configuration

**application.yml**

```yaml
spring:
  application:
    name: spring-batch-demo
    
  datasource:
    url: jdbc:postgresql://localhost:5432/batchdb
    username: postgres
    password: postgres
    driver-class-name: org.postgresql.Driver
    
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        
  batch:
    jdbc:
      initialize-schema: always
    job:
      enabled: false # Prevent automatic job execution on startup

logging:
  level:
    org.springframework.batch: DEBUG
```

## Simple Batch Job Example

### Domain Model

```java
package com.example.batch.model;

import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity
@Table(name = "transactions")
public class Transaction {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String accountNumber;
    private String transactionType;
    private BigDecimal amount;
    private LocalDateTime transactionDate;
    private String status;
    private String description;
    
    // Constructors
    public Transaction() {}
    
    public Transaction(String accountNumber, String transactionType, 
                      BigDecimal amount, LocalDateTime transactionDate) {
        this.accountNumber = accountNumber;
        this.transactionType = transactionType;
        this.amount = amount;
        this.transactionDate = transactionDate;
        this.status = "PENDING";
    }
    
    // Getters and Setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public String getAccountNumber() { return accountNumber; }
    public void setAccountNumber(String accountNumber) { this.accountNumber = accountNumber; }
    
    public String getTransactionType() { return transactionType; }
    public void setTransactionType(String transactionType) { this.transactionType = transactionType; }
    
    public BigDecimal getAmount() { return amount; }
    public void setAmount(BigDecimal amount) { this.amount = amount; }
    
    public LocalDateTime getTransactionDate() { return transactionDate; }
    public void setTransactionDate(LocalDateTime transactionDate) { this.transactionDate = transactionDate; }
    
    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }
    
    public String getDescription() { return description; }
    public void setDescription(String description) { this.description = description; }
}
```

### CSV to Database Job

```java
package com.example.batch.config;

import com.example.batch.model.Transaction;
import com.example.batch.processor.TransactionItemProcessor;
import com.example.batch.listener.JobCompletionListener;
import org.springframework.batch.core.Job;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.job.builder.JobBuilder;
import org.springframework.batch.core.repository.JobRepository;
import org.springframework.batch.core.step.builder.StepBuilder;
import org.springframework.batch.item.database.JpaItemWriter;
import org.springframework.batch.item.file.FlatFileItemReader;
import org.springframework.batch.item.file.mapping.BeanWrapperFieldSetMapper;
import org.springframework.batch.item.file.mapping.DefaultLineMapper;
import org.springframework.batch.item.file.transform.DelimitedLineTokenizer;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.ClassPathResource;
import org.springframework.transaction.PlatformTransactionManager;

import jakarta.persistence.EntityManagerFactory;

@Configuration
public class TransactionBatchConfig {

    @Autowired
    private EntityManagerFactory entityManagerFactory;

    @Bean
    public FlatFileItemReader<Transaction> transactionReader() {
        FlatFileItemReader<Transaction> reader = new FlatFileItemReader<>();
        reader.setResource(new ClassPathResource("transactions.csv"));
        reader.setLinesToSkip(1); // Skip header
        
        DefaultLineMapper<Transaction> lineMapper = new DefaultLineMapper<>();
        
        DelimitedLineTokenizer tokenizer = new DelimitedLineTokenizer();
        tokenizer.setNames("accountNumber", "transactionType", "amount", "transactionDate");
        tokenizer.setDelimiter(",");
        
        BeanWrapperFieldSetMapper<Transaction> fieldSetMapper = new BeanWrapperFieldSetMapper<>();
        fieldSetMapper.setTargetType(Transaction.class);
        
        lineMapper.setLineTokenizer(tokenizer);
        lineMapper.setFieldSetMapper(fieldSetMapper);
        
        reader.setLineMapper(lineMapper);
        return reader;
    }

    @Bean
    public TransactionItemProcessor transactionProcessor() {
        return new TransactionItemProcessor();
    }

    @Bean
    public JpaItemWriter<Transaction> transactionWriter() {
        JpaItemWriter<Transaction> writer = new JpaItemWriter<>();
        writer.setEntityManagerFactory(entityManagerFactory);
        return writer;
    }

    @Bean
    public Step transactionStep(JobRepository jobRepository, 
                                PlatformTransactionManager transactionManager) {
        return new StepBuilder("transactionStep", jobRepository)
                .<Transaction, Transaction>chunk(100, transactionManager)
                .reader(transactionReader())
                .processor(transactionProcessor())
                .writer(transactionWriter())
                .build();
    }

    @Bean
    public Job importTransactionJob(JobRepository jobRepository,
                                    Step transactionStep,
                                    JobCompletionListener listener) {
        return new JobBuilder("importTransactionJob", jobRepository)
                .listener(listener)
                .start(transactionStep)
                .build();
    }
}
```

### Item Processor

```java
package com.example.batch.processor;

import com.example.batch.model.Transaction;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.batch.item.ItemProcessor;

import java.math.BigDecimal;

public class TransactionItemProcessor implements ItemProcessor<Transaction, Transaction> {

    private static final Logger logger = LoggerFactory.getLogger(TransactionItemProcessor.class);

    @Override
    public Transaction process(Transaction transaction) throws Exception {
        // Validate transaction
        if (transaction.getAmount().compareTo(BigDecimal.ZERO) <= 0) {
            logger.warn("Invalid transaction amount: {}", transaction.getAmount());
            return null; // Skip this item
        }

        // Apply business rules
        if (transaction.getAmount().compareTo(new BigDecimal("10000")) > 0) {
            transaction.setStatus("REVIEW");
            transaction.setDescription("High-value transaction requires review");
        } else {
            transaction.setStatus("APPROVED");
            transaction.setDescription("Transaction processed successfully");
        }

        logger.debug("Processing transaction: {}", transaction.getAccountNumber());
        return transaction;
    }
}
```

### Job Completion Listener

```java
package com.example.batch.listener;

import com.example.batch.model.Transaction;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.batch.core.BatchStatus;
import org.springframework.batch.core.JobExecution;
import org.springframework.batch.core.JobExecutionListener;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Component;

@Component
public class JobCompletionListener implements JobExecutionListener {

    private static final Logger logger = LoggerFactory.getLogger(JobCompletionListener.class);

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Override
    public void beforeJob(JobExecution jobExecution) {
        logger.info("Job starting: {}", jobExecution.getJobInstance().getJobName());
    }

    @Override
    public void afterJob(JobExecution jobExecution) {
        if (jobExecution.getStatus() == BatchStatus.COMPLETED) {
            logger.info("Job completed successfully!");
            
            Long count = jdbcTemplate.queryForObject(
                "SELECT COUNT(*) FROM transactions", Long.class);
            logger.info("Total transactions processed: {}", count);
        } else if (jobExecution.getStatus() == BatchStatus.FAILED) {
            logger.error("Job failed with status: {}", jobExecution.getStatus());
        }
    }
}
```

## Advanced Features

### Multi-Step Job

```java
package com.example.batch.config;

import org.springframework.batch.core.Job;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.job.builder.JobBuilder;
import org.springframework.batch.core.repository.JobRepository;
import org.springframework.batch.core.step.builder.StepBuilder;
import org.springframework.batch.repeat.RepeatStatus;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.transaction.PlatformTransactionManager;

@Configuration
public class MultiStepJobConfig {

    @Bean
    public Step validationStep(JobRepository jobRepository, 
                              PlatformTransactionManager transactionManager) {
        return new StepBuilder("validationStep", jobRepository)
                .tasklet((contribution, chunkContext) -> {
                    System.out.println("Validating data...");
                    // Validation logic
                    return RepeatStatus.FINISHED;
                }, transactionManager)
                .build();
    }

    @Bean
    public Step processingStep(JobRepository jobRepository,
                              PlatformTransactionManager transactionManager) {
        return new StepBuilder("processingStep", jobRepository)
                .tasklet((contribution, chunkContext) -> {
                    System.out.println("Processing data...");
                    // Processing logic
                    return RepeatStatus.FINISHED;
                }, transactionManager)
                .build();
    }

    @Bean
    public Step reportingStep(JobRepository jobRepository,
                             PlatformTransactionManager transactionManager) {
        return new StepBuilder("reportingStep", jobRepository)
                .tasklet((contribution, chunkContext) -> {
                    System.out.println("Generating report...");
                    // Reporting logic
                    return RepeatStatus.FINISHED;
                }, transactionManager)
                .build();
    }

    @Bean
    public Job multiStepJob(JobRepository jobRepository,
                           Step validationStep,
                           Step processingStep,
                           Step reportingStep) {
        return new JobBuilder("multiStepJob", jobRepository)
                .start(validationStep)
                .next(processingStep)
                .next(reportingStep)
                .build();
    }
}
```

### Conditional Flow

```java
@Bean
public Job conditionalJob(JobRepository jobRepository,
                         Step validationStep,
                         Step successStep,
                         Step failureStep) {
    return new JobBuilder("conditionalJob", jobRepository)
            .start(validationStep)
            .on("COMPLETED").to(successStep)
            .from(validationStep).on("FAILED").to(failureStep)
            .end()
            .build();
}
```

### Partitioning for Parallel Processing

```java
package com.example.batch.config;

import org.springframework.batch.core.Step;
import org.springframework.batch.core.configuration.annotation.StepScope;
import org.springframework.batch.core.partition.support.Partitioner;
import org.springframework.batch.core.repository.JobRepository;
import org.springframework.batch.core.step.builder.StepBuilder;
import org.springframework.batch.item.ExecutionContext;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.task.SimpleAsyncTaskExecutor;
import org.springframework.transaction.PlatformTransactionManager;

import java.util.HashMap;
import java.util.Map;

@Configuration
public class PartitioningConfig {

    @Bean
    public Partitioner rangePartitioner() {
        return gridSize -> {
            Map<String, ExecutionContext> result = new HashMap<>();
            
            int range = 1000;
            int fromId = 1;
            int toId = range;
            
            for (int i = 1; i <= gridSize; i++) {
                ExecutionContext context = new ExecutionContext();
                context.putInt("fromId", fromId);
                context.putInt("toId", toId);
                
                result.put("partition" + i, context);
                
                fromId = toId + 1;
                toId += range;
            }
            
            return result;
        };
    }

    @Bean
    public Step masterStep(JobRepository jobRepository,
                          Step slaveStep) {
        return new StepBuilder("masterStep", jobRepository)
                .partitioner("slaveStep", rangePartitioner())
                .step(slaveStep)
                .gridSize(4)
                .taskExecutor(new SimpleAsyncTaskExecutor())
                .build();
    }

    @Bean
    @StepScope
    public Step slaveStep(JobRepository jobRepository,
                         PlatformTransactionManager transactionManager,
                         @Value("#{stepExecutionContext['fromId']}") Integer fromId,
                         @Value("#{stepExecutionContext['toId']}") Integer toId) {
        return new StepBuilder("slaveStep", jobRepository)
                .tasklet((contribution, chunkContext) -> {
                    System.out.println("Processing partition from " + fromId + " to " + toId);
                    // Process partition
                    return RepeatStatus.FINISHED;
                }, transactionManager)
                .build();
    }
}
```

### Skip and Retry Logic

```java
@Bean
public Step resilientStep(JobRepository jobRepository,
                         PlatformTransactionManager transactionManager) {
    return new StepBuilder("resilientStep", jobRepository)
            .<Transaction, Transaction>chunk(100, transactionManager)
            .reader(transactionReader())
            .processor(transactionProcessor())
            .writer(transactionWriter())
            .faultTolerant()
            .skip(Exception.class)
            .skipLimit(10)
            .retry(Exception.class)
            .retryLimit(3)
            .build();
}
```

### Custom Skip Policy

```java
package com.example.batch.policy;

import org.springframework.batch.core.step.skip.SkipLimitExceededException;
import org.springframework.batch.core.step.skip.SkipPolicy;

public class CustomSkipPolicy implements SkipPolicy {

    @Override
    public boolean shouldSkip(Throwable t, long skipCount) throws SkipLimitExceededException {
        if (t instanceof ValidationException && skipCount < 5) {
            return true;
        }
        return false;
    }
}
```

## Database to Database Job

### JPA Item Reader

```java
@Bean
@StepScope
public JpaPagingItemReader<Transaction> databaseReader(EntityManagerFactory entityManagerFactory) {
    JpaPagingItemReader<Transaction> reader = new JpaPagingItemReader<>();
    reader.setEntityManagerFactory(entityManagerFactory);
    reader.setQueryString("SELECT t FROM Transaction t WHERE t.status = 'PENDING'");
    reader.setPageSize(100);
    return reader;
}
```

### JDBC Batch Item Writer

```java
@Bean
public JdbcBatchItemWriter<Transaction> jdbcWriter(DataSource dataSource) {
    JdbcBatchItemWriter<Transaction> writer = new JdbcBatchItemWriter<>();
    writer.setDataSource(dataSource);
    writer.setSql("INSERT INTO processed_transactions (account_number, amount, status) VALUES (?, ?, ?)");
    writer.setItemPreparedStatementSetter((item, ps) -> {
        ps.setString(1, item.getAccountNumber());
        ps.setBigDecimal(2, item.getAmount());
        ps.setString(3, item.getStatus());
    });
    return writer;
}
```

## Job Execution Control

### Job Launcher Service

```java
package com.example.batch.service;

import org.springframework.batch.core.Job;
import org.springframework.batch.core.JobParameters;
import org.springframework.batch.core.JobParametersBuilder;
import org.springframework.batch.core.launch.JobLauncher;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class BatchJobService {

    @Autowired
    private JobLauncher jobLauncher;

    @Autowired
    private Job importTransactionJob;

    public void runTransactionImportJob() throws Exception {
        JobParameters jobParameters = new JobParametersBuilder()
                .addLong("startTime", System.currentTimeMillis())
                .addString("fileName", "transactions.csv")
                .toJobParameters();

        jobLauncher.run(importTransactionJob, jobParameters);
    }
}
```

### REST Controller for Job Execution

```java
package com.example.batch.controller;

import com.example.batch.service.BatchJobService;
import org.springframework.batch.core.Job;
import org.springframework.batch.core.JobExecution;
import org.springframework.batch.core.JobParameters;
import org.springframework.batch.core.JobParametersBuilder;
import org.springframework.batch.core.explore.JobExplorer;
import org.springframework.batch.core.launch.JobLauncher;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.Set;

@RestController
@RequestMapping("/api/batch")
public class BatchJobController {

    @Autowired
    private JobLauncher jobLauncher;

    @Autowired
    private Job importTransactionJob;

    @Autowired
    private JobExplorer jobExplorer;

    @PostMapping("/jobs/import-transactions")
    public ResponseEntity<String> runImportJob(@RequestParam(required = false) String fileName) {
        try {
            JobParameters params = new JobParametersBuilder()
                    .addLong("time", System.currentTimeMillis())
                    .addString("fileName", fileName != null ? fileName : "transactions.csv")
                    .toJobParameters();

            JobExecution execution = jobLauncher.run(importTransactionJob, params);
            
            return ResponseEntity.ok("Job started with execution id: " + execution.getId());
        } catch (Exception e) {
            return ResponseEntity.internalServerError()
                    .body("Job failed to start: " + e.getMessage());
        }
    }

    @GetMapping("/jobs/{jobName}/executions")
    public ResponseEntity<?> getJobExecutions(@PathVariable String jobName) {
        Set<JobExecution> executions = jobExplorer.findRunningJobExecutions(jobName);
        return ResponseEntity.ok(executions);
    }

    @GetMapping("/jobs/executions/{executionId}")
    public ResponseEntity<?> getJobExecution(@PathVariable Long executionId) {
        JobExecution execution = jobExplorer.getJobExecution(executionId);
        return ResponseEntity.ok(execution);
    }
}
```

## Scheduling Batch Jobs

### Scheduled Job Execution

```java
package com.example.batch.scheduler;

import org.springframework.batch.core.Job;
import org.springframework.batch.core.JobParameters;
import org.springframework.batch.core.JobParametersBuilder;
import org.springframework.batch.core.launch.JobLauncher;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

@Component
public class BatchJobScheduler {

    @Autowired
    private JobLauncher jobLauncher;

    @Autowired
    private Job importTransactionJob;

    @Scheduled(cron = "0 0 2 * * ?") // Run at 2 AM daily
    public void runDailyImport() {
        try {
            JobParameters params = new JobParametersBuilder()
                    .addLong("time", System.currentTimeMillis())
                    .toJobParameters();

            jobLauncher.run(importTransactionJob, params);
        } catch (Exception e) {
            // Log error
        }
    }
}
```

## Custom Item Reader

```java
package com.example.batch.reader;

import com.example.batch.model.Transaction;
import org.springframework.batch.item.ItemReader;

import java.util.List;
import java.util.Iterator;

public class CustomTransactionReader implements ItemReader<Transaction> {

    private Iterator<Transaction> transactionIterator;

    public CustomTransactionReader(List<Transaction> transactions) {
        this.transactionIterator = transactions.iterator();
    }

    @Override
    public Transaction read() {
        if (transactionIterator.hasNext()) {
            return transactionIterator.next();
        }
        return null; // Signals end of data
    }
}
```

## Custom Item Writer

```java
package com.example.batch.writer;

import com.example.batch.model.Transaction;
import org.springframework.batch.item.Chunk;
import org.springframework.batch.item.ItemWriter;

public class CustomTransactionWriter implements ItemWriter<Transaction> {

    @Override
    public void write(Chunk<? extends Transaction> chunk) throws Exception {
        for (Transaction transaction : chunk) {
            // Custom write logic
            System.out.println("Writing transaction: " + transaction.getAccountNumber());
            // Could write to file, call API, etc.
        }
    }
}
```

## Testing

### Job Test Configuration

```java
package com.example.batch.test;

import com.example.batch.config.TransactionBatchConfig;
import org.junit.jupiter.api.Test;
import org.springframework.batch.core.JobExecution;
import org.springframework.batch.core.JobParameters;
import org.springframework.batch.core.JobParametersBuilder;
import org.springframework.batch.test.JobLauncherTestUtils;
import org.springframework.batch.test.context.SpringBatchTest;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit.jupiter.SpringJUnitConfig;

import static org.junit.jupiter.api.Assertions.assertEquals;

@SpringBatchTest
@SpringBootTest
@SpringJUnitConfig(TransactionBatchConfig.class)
class TransactionJobTest {

    @Autowired
    private JobLauncherTestUtils jobLauncherTestUtils;

    @Test
    void testTransactionJob() throws Exception {
        JobParameters jobParameters = new JobParametersBuilder()
                .addLong("time", System.currentTimeMillis())
                .toJobParameters();

        JobExecution jobExecution = jobLauncherTestUtils.launchJob(jobParameters);

        assertEquals("COMPLETED", jobExecution.getExitStatus().getExitCode());
    }

    @Test
    void testTransactionStep() {
        JobExecution jobExecution = jobLauncherTestUtils.launchStep("transactionStep");
        assertEquals("COMPLETED", jobExecution.getExitStatus().getExitCode());
    }
}
```

## Best Practices

### 1. Use Appropriate Chunk Sizes

```java
// For small items
.chunk(1000, transactionManager)

// For large items
.chunk(10, transactionManager)
```

### 2. Implement Proper Error Handling

```java
@Bean
public Step errorHandlingStep(JobRepository jobRepository,
                             PlatformTransactionManager transactionManager) {
    return new StepBuilder("errorHandlingStep", jobRepository)
            .<Transaction, Transaction>chunk(100, transactionManager)
            .reader(reader())
            .processor(processor())
            .writer(writer())
            .faultTolerant()
            .skip(ValidationException.class)
            .skipLimit(100)
            .retry(DatabaseException.class)
            .retryLimit(3)
            .listener(new CustomSkipListener())
            .build();
}
```

### 3. Use Job Parameters for Flexibility

```java
@Bean
@StepScope
public FlatFileItemReader<Transaction> reader(
        @Value("#{jobParameters['inputFile']}") String inputFile) {
    FlatFileItemReader<Transaction> reader = new FlatFileItemReader<>();
    reader.setResource(new FileSystemResource(inputFile));
    return reader;
}
```

### 4. Implement Restart Logic

```java
@Bean
public Job restartableJob(JobRepository jobRepository, Step step) {
    return new JobBuilder("restartableJob", jobRepository)
            .start(step)
            .incrementer(new RunIdIncrementer())
            .preventRestart() // or allow restart
            .build();
}
```

## Performance Optimization

### 1. Use Async Processing

```java
@Bean
public TaskExecutor taskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(4);
    executor.setMaxPoolSize(8);
    executor.setQueueCapacity(100);
    executor.setThreadNamePrefix("batch-");
    executor.initialize();
    return executor;
}
```

### 2. Enable Batch Inserts

```java
spring.jpa.properties.hibernate.jdbc.batch_size=100
spring.jpa.properties.hibernate.order_inserts=true
spring.jpa.properties.hibernate.order_updates=true
```

### 3. Use Pagination for Large Datasets

```java
@Bean
public JpaPagingItemReader<Transaction> pagingReader(EntityManagerFactory emf) {
    JpaPagingItemReader<Transaction> reader = new JpaPagingItemReader<>();
    reader.setEntityManagerFactory(emf);
    reader.setPageSize(1000);
    reader.setQueryString("SELECT t FROM Transaction t");
    return reader;
}
```

## Monitoring and Observability

### Custom Metrics

```java
package com.example.batch.listener;

import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.batch.core.ItemProcessListener;
import org.springframework.stereotype.Component;

@Component
public class MetricsItemProcessListener<T, S> implements ItemProcessListener<T, S> {

    private final MeterRegistry meterRegistry;

    public MetricsItemProcessListener(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }

    @Override
    public void beforeProcess(T item) {
        meterRegistry.counter("batch.item.process.started").increment();
    }

    @Override
    public void afterProcess(T item, S result) {
        meterRegistry.counter("batch.item.process.completed").increment();
    }

    @Override
    public void onProcessError(T item, Exception e) {
        meterRegistry.counter("batch.item.process.error").increment();
    }
}
```

## Common Use Cases

1. **ETL Operations** - Extract, Transform, Load
2. **Data Migration** - Moving data between systems
3. **Report Generation** - Periodic report creation
4. **Data Cleanup** - Archiving old data
5. **Batch Notifications** - Sending bulk emails/SMS

## Additional Resources

- [Spring Batch Documentation](https://docs.spring.io/spring-batch/docs/current/reference/html/)
- [Spring Batch Samples](https://github.com/spring-projects/spring-batch/tree/main/spring-batch-samples)
- [Batch Processing Patterns](https://www.enterpriseintegrationpatterns.com/patterns/messaging/BatchProcessing.html)
