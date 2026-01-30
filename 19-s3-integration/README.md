# AWS S3 Integration

## Overview

Amazon S3 (Simple Storage Service) is a scalable object storage service designed for storing and retrieving any amount of data. Spring Boot applications commonly integrate with S3 for file storage, backup, static content hosting, and data archiving.

## Key Concepts

### S3 Core Components

1. **Bucket** - Container for storing objects
2. **Object** - Files stored in buckets (up to 5TB each)
3. **Key** - Unique identifier for objects within a bucket
4. **Region** - Geographic location where bucket is hosted
5. **Access Control** - IAM policies, bucket policies, ACLs

### Common Operations

- Upload/Download files
- List objects
- Delete objects
- Generate presigned URLs
- Multipart uploads
- Server-side encryption

## Spring Boot Integration

### Dependencies

```xml
<dependencies>
    <!-- AWS SDK v2 for S3 -->
    <dependency>
        <groupId>software.amazon.awssdk</groupId>
        <artifactId>s3</artifactId>
        <version>2.20.26</version>
    </dependency>
    
    <!-- Spring Boot Starter -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    
    <!-- For async operations -->
    <dependency>
        <groupId>software.amazon.awssdk</groupId>
        <artifactId>netty-nio-client</artifactId>
        <version>2.20.26</version>
    </dependency>
    
    <!-- Apache Commons IO -->
    <dependency>
        <groupId>commons-io</groupId>
        <artifactId>commons-io</artifactId>
        <version>2.11.0</version>
    </dependency>
    
    <!-- For LocalStack testing -->
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>localstack</artifactId>
        <version>1.19.3</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### Gradle Configuration

```gradle
dependencies {
    implementation platform('software.amazon.awssdk:bom:2.20.26')
    implementation 'software.amazon.awssdk:s3'
    implementation 'software.amazon.awssdk:netty-nio-client'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'commons-io:commons-io:2.11.0'
    testImplementation 'org.testcontainers:localstack:1.19.3'
}
```

## Configuration

### Application Properties

**application.yml**

```yaml
aws:
  s3:
    bucket-name: my-application-bucket
    region: us-east-1
    access-key: ${AWS_ACCESS_KEY_ID}
    secret-key: ${AWS_SECRET_ACCESS_KEY}
    endpoint: ${AWS_ENDPOINT:} # For LocalStack testing
    
spring:
  servlet:
    multipart:
      max-file-size: 100MB
      max-request-size: 100MB
      enabled: true

logging:
  level:
    software.amazon.awssdk: DEBUG
```

**application.properties**

```properties
aws.s3.bucket-name=my-application-bucket
aws.s3.region=us-east-1
aws.s3.access-key=${AWS_ACCESS_KEY_ID}
aws.s3.secret-key=${AWS_SECRET_ACCESS_KEY}

spring.servlet.multipart.max-file-size=100MB
spring.servlet.multipart.max-request-size=100MB
```

### S3 Configuration Class

```java
package com.example.s3.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import software.amazon.awssdk.auth.credentials.AwsBasicCredentials;
import software.amazon.awssdk.auth.credentials.StaticCredentialsProvider;
import software.amazon.awssdk.regions.Region;
import software.amazon.awssdk.services.s3.S3AsyncClient;
import software.amazon.awssdk.services.s3.S3Client;
import software.amazon.awssdk.services.s3.presigner.S3Presigner;

import java.net.URI;

@Configuration
public class S3Config {

    @Value("${aws.s3.region}")
    private String region;

    @Value("${aws.s3.access-key}")
    private String accessKey;

    @Value("${aws.s3.secret-key}")
    private String secretKey;

    @Value("${aws.s3.endpoint:}")
    private String endpoint;

    @Bean
    public S3Client s3Client() {
        var builder = S3Client.builder()
                .region(Region.of(region))
                .credentialsProvider(StaticCredentialsProvider.create(
                        AwsBasicCredentials.create(accessKey, secretKey)));

        // For LocalStack or MinIO
        if (endpoint != null && !endpoint.isEmpty()) {
            builder.endpointOverride(URI.create(endpoint));
        }

        return builder.build();
    }

    @Bean
    public S3AsyncClient s3AsyncClient() {
        var builder = S3AsyncClient.builder()
                .region(Region.of(region))
                .credentialsProvider(StaticCredentialsProvider.create(
                        AwsBasicCredentials.create(accessKey, secretKey)));

        if (endpoint != null && !endpoint.isEmpty()) {
            builder.endpointOverride(URI.create(endpoint));
        }

        return builder.build();
    }

    @Bean
    public S3Presigner s3Presigner() {
        var builder = S3Presigner.builder()
                .region(Region.of(region))
                .credentialsProvider(StaticCredentialsProvider.create(
                        AwsBasicCredentials.create(accessKey, secretKey)));

        if (endpoint != null && !endpoint.isEmpty()) {
            builder.endpointOverride(URI.create(endpoint));
        }

        return builder.build();
    }
}
```

## Service Implementation

### S3 Service

```java
package com.example.s3.service;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;
import software.amazon.awssdk.core.ResponseInputStream;
import software.amazon.awssdk.core.sync.RequestBody;
import software.amazon.awssdk.services.s3.S3Client;
import software.amazon.awssdk.services.s3.model.*;
import software.amazon.awssdk.services.s3.presigner.S3Presigner;
import software.amazon.awssdk.services.s3.presigner.model.GetObjectPresignRequest;
import software.amazon.awssdk.services.s3.presigner.model.PresignedGetObjectRequest;
import software.amazon.awssdk.services.s3.presigner.model.PresignedPutObjectRequest;
import software.amazon.awssdk.services.s3.presigner.model.PutObjectPresignRequest;

import java.io.IOException;
import java.io.InputStream;
import java.time.Duration;
import java.util.ArrayList;
import java.util.List;
import java.util.UUID;
import java.util.stream.Collectors;

@Service
public class S3Service {

    private static final Logger logger = LoggerFactory.getLogger(S3Service.class);

    @Autowired
    private S3Client s3Client;

    @Autowired
    private S3Presigner s3Presigner;

    @Value("${aws.s3.bucket-name}")
    private String bucketName;

    /**
     * Upload file to S3
     */
    public String uploadFile(MultipartFile file) throws IOException {
        String key = generateFileName(file.getOriginalFilename());
        
        PutObjectRequest putObjectRequest = PutObjectRequest.builder()
                .bucket(bucketName)
                .key(key)
                .contentType(file.getContentType())
                .contentLength(file.getSize())
                .build();

        s3Client.putObject(putObjectRequest, RequestBody.fromInputStream(
                file.getInputStream(), file.getSize()));

        logger.info("File uploaded successfully: {}", key);
        return key;
    }

    /**
     * Upload file with custom key
     */
    public String uploadFile(String key, InputStream inputStream, long contentLength, String contentType) {
        PutObjectRequest putObjectRequest = PutObjectRequest.builder()
                .bucket(bucketName)
                .key(key)
                .contentType(contentType)
                .contentLength(contentLength)
                .build();

        s3Client.putObject(putObjectRequest, RequestBody.fromInputStream(
                inputStream, contentLength));

        logger.info("File uploaded successfully: {}", key);
        return key;
    }

    /**
     * Download file from S3
     */
    public byte[] downloadFile(String key) {
        GetObjectRequest getObjectRequest = GetObjectRequest.builder()
                .bucket(bucketName)
                .key(key)
                .build();

        ResponseInputStream<GetObjectResponse> response = s3Client.getObject(getObjectRequest);

        try {
            return response.readAllBytes();
        } catch (IOException e) {
            logger.error("Error downloading file: {}", key, e);
            throw new RuntimeException("Error downloading file", e);
        }
    }

    /**
     * Get file as InputStream
     */
    public InputStream getFileAsStream(String key) {
        GetObjectRequest getObjectRequest = GetObjectRequest.builder()
                .bucket(bucketName)
                .key(key)
                .build();

        return s3Client.getObject(getObjectRequest);
    }

    /**
     * Delete file from S3
     */
    public void deleteFile(String key) {
        DeleteObjectRequest deleteObjectRequest = DeleteObjectRequest.builder()
                .bucket(bucketName)
                .key(key)
                .build();

        s3Client.deleteObject(deleteObjectRequest);
        logger.info("File deleted successfully: {}", key);
    }

    /**
     * Delete multiple files
     */
    public void deleteFiles(List<String> keys) {
        List<ObjectIdentifier> objectIdentifiers = keys.stream()
                .map(key -> ObjectIdentifier.builder().key(key).build())
                .collect(Collectors.toList());

        Delete delete = Delete.builder()
                .objects(objectIdentifiers)
                .build();

        DeleteObjectsRequest deleteObjectsRequest = DeleteObjectsRequest.builder()
                .bucket(bucketName)
                .delete(delete)
                .build();

        s3Client.deleteObjects(deleteObjectsRequest);
        logger.info("Deleted {} files", keys.size());
    }

    /**
     * List all files in bucket
     */
    public List<String> listFiles() {
        ListObjectsV2Request listRequest = ListObjectsV2Request.builder()
                .bucket(bucketName)
                .build();

        ListObjectsV2Response listResponse = s3Client.listObjectsV2(listRequest);

        return listResponse.contents().stream()
                .map(S3Object::key)
                .collect(Collectors.toList());
    }

    /**
     * List files with prefix
     */
    public List<String> listFilesWithPrefix(String prefix) {
        ListObjectsV2Request listRequest = ListObjectsV2Request.builder()
                .bucket(bucketName)
                .prefix(prefix)
                .build();

        ListObjectsV2Response listResponse = s3Client.listObjectsV2(listRequest);

        return listResponse.contents().stream()
                .map(S3Object::key)
                .collect(Collectors.toList());
    }

    /**
     * Check if file exists
     */
    public boolean fileExists(String key) {
        try {
            HeadObjectRequest headObjectRequest = HeadObjectRequest.builder()
                    .bucket(bucketName)
                    .key(key)
                    .build();

            s3Client.headObject(headObjectRequest);
            return true;
        } catch (NoSuchKeyException e) {
            return false;
        }
    }

    /**
     * Get file metadata
     */
    public S3ObjectMetadata getFileMetadata(String key) {
        HeadObjectRequest headObjectRequest = HeadObjectRequest.builder()
                .bucket(bucketName)
                .key(key)
                .build();

        HeadObjectResponse response = s3Client.headObject(headObjectRequest);

        return new S3ObjectMetadata(
                key,
                response.contentLength(),
                response.contentType(),
                response.lastModified()
        );
    }

    /**
     * Generate presigned URL for download (valid for 1 hour)
     */
    public String generatePresignedUrl(String key) {
        return generatePresignedUrl(key, Duration.ofHours(1));
    }

    /**
     * Generate presigned URL with custom expiration
     */
    public String generatePresignedUrl(String key, Duration expiration) {
        GetObjectRequest getObjectRequest = GetObjectRequest.builder()
                .bucket(bucketName)
                .key(key)
                .build();

        GetObjectPresignRequest presignRequest = GetObjectPresignRequest.builder()
                .signatureDuration(expiration)
                .getObjectRequest(getObjectRequest)
                .build();

        PresignedGetObjectRequest presignedRequest = s3Presigner.presignGetObject(presignRequest);

        return presignedRequest.url().toString();
    }

    /**
     * Generate presigned URL for upload
     */
    public String generatePresignedUploadUrl(String key, Duration expiration) {
        PutObjectRequest putObjectRequest = PutObjectRequest.builder()
                .bucket(bucketName)
                .key(key)
                .build();

        PutObjectPresignRequest presignRequest = PutObjectPresignRequest.builder()
                .signatureDuration(expiration)
                .putObjectRequest(putObjectRequest)
                .build();

        PresignedPutObjectRequest presignedRequest = s3Presigner.presignPutObject(presignRequest);

        return presignedRequest.url().toString();
    }

    /**
     * Copy file within S3
     */
    public void copyFile(String sourceKey, String destinationKey) {
        CopyObjectRequest copyObjectRequest = CopyObjectRequest.builder()
                .sourceBucket(bucketName)
                .sourceKey(sourceKey)
                .destinationBucket(bucketName)
                .destinationKey(destinationKey)
                .build();

        s3Client.copyObject(copyObjectRequest);
        logger.info("File copied from {} to {}", sourceKey, destinationKey);
    }

    /**
     * Move file within S3
     */
    public void moveFile(String sourceKey, String destinationKey) {
        copyFile(sourceKey, destinationKey);
        deleteFile(sourceKey);
        logger.info("File moved from {} to {}", sourceKey, destinationKey);
    }

    private String generateFileName(String originalFileName) {
        return UUID.randomUUID().toString() + "-" + originalFileName;
    }
}
```

### Metadata Model

```java
package com.example.s3.service;

import java.time.Instant;

public record S3ObjectMetadata(
        String key,
        Long size,
        String contentType,
        Instant lastModified
) {}
```

## REST Controller

```java
package com.example.s3.controller;

import com.example.s3.service.S3ObjectMetadata;
import com.example.s3.service.S3Service;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.io.InputStreamResource;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import java.io.InputStream;
import java.time.Duration;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/api/s3")
public class S3Controller {

    @Autowired
    private S3Service s3Service;

    @PostMapping("/upload")
    public ResponseEntity<Map<String, String>> uploadFile(@RequestParam("file") MultipartFile file) {
        try {
            String key = s3Service.uploadFile(file);
            
            Map<String, String> response = new HashMap<>();
            response.put("key", key);
            response.put("message", "File uploaded successfully");
            
            return ResponseEntity.ok(response);
        } catch (Exception e) {
            Map<String, String> error = new HashMap<>();
            error.put("error", e.getMessage());
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
        }
    }

    @GetMapping("/download/{key}")
    public ResponseEntity<byte[]> downloadFile(@PathVariable String key) {
        try {
            byte[] data = s3Service.downloadFile(key);
            S3ObjectMetadata metadata = s3Service.getFileMetadata(key);

            return ResponseEntity.ok()
                    .contentType(MediaType.parseMediaType(metadata.contentType()))
                    .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"" + key + "\"")
                    .body(data);
        } catch (Exception e) {
            return ResponseEntity.notFound().build();
        }
    }

    @GetMapping("/stream/{key}")
    public ResponseEntity<InputStreamResource> streamFile(@PathVariable String key) {
        try {
            InputStream inputStream = s3Service.getFileAsStream(key);
            S3ObjectMetadata metadata = s3Service.getFileMetadata(key);

            return ResponseEntity.ok()
                    .contentType(MediaType.parseMediaType(metadata.contentType()))
                    .contentLength(metadata.size())
                    .body(new InputStreamResource(inputStream));
        } catch (Exception e) {
            return ResponseEntity.notFound().build();
        }
    }

    @DeleteMapping("/{key}")
    public ResponseEntity<Map<String, String>> deleteFile(@PathVariable String key) {
        try {
            s3Service.deleteFile(key);
            
            Map<String, String> response = new HashMap<>();
            response.put("message", "File deleted successfully");
            
            return ResponseEntity.ok(response);
        } catch (Exception e) {
            Map<String, String> error = new HashMap<>();
            error.put("error", e.getMessage());
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
        }
    }

    @GetMapping("/list")
    public ResponseEntity<List<String>> listFiles(@RequestParam(required = false) String prefix) {
        List<String> files = prefix != null 
                ? s3Service.listFilesWithPrefix(prefix) 
                : s3Service.listFiles();
        return ResponseEntity.ok(files);
    }

    @GetMapping("/exists/{key}")
    public ResponseEntity<Map<String, Boolean>> fileExists(@PathVariable String key) {
        boolean exists = s3Service.fileExists(key);
        
        Map<String, Boolean> response = new HashMap<>();
        response.put("exists", exists);
        
        return ResponseEntity.ok(response);
    }

    @GetMapping("/metadata/{key}")
    public ResponseEntity<S3ObjectMetadata> getMetadata(@PathVariable String key) {
        try {
            S3ObjectMetadata metadata = s3Service.getFileMetadata(key);
            return ResponseEntity.ok(metadata);
        } catch (Exception e) {
            return ResponseEntity.notFound().build();
        }
    }

    @GetMapping("/presigned-url/{key}")
    public ResponseEntity<Map<String, String>> getPresignedUrl(
            @PathVariable String key,
            @RequestParam(defaultValue = "3600") long expirationSeconds) {
        
        try {
            String url = s3Service.generatePresignedUrl(key, Duration.ofSeconds(expirationSeconds));
            
            Map<String, String> response = new HashMap<>();
            response.put("url", url);
            response.put("expiresIn", expirationSeconds + " seconds");
            
            return ResponseEntity.ok(response);
        } catch (Exception e) {
            Map<String, String> error = new HashMap<>();
            error.put("error", e.getMessage());
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
        }
    }

    @GetMapping("/presigned-upload-url")
    public ResponseEntity<Map<String, String>> getPresignedUploadUrl(
            @RequestParam String fileName,
            @RequestParam(defaultValue = "3600") long expirationSeconds) {
        
        try {
            String url = s3Service.generatePresignedUploadUrl(fileName, Duration.ofSeconds(expirationSeconds));
            
            Map<String, String> response = new HashMap<>();
            response.put("url", url);
            response.put("expiresIn", expirationSeconds + " seconds");
            
            return ResponseEntity.ok(response);
        } catch (Exception e) {
            Map<String, String> error = new HashMap<>();
            error.put("error", e.getMessage());
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
        }
    }

    @PostMapping("/copy")
    public ResponseEntity<Map<String, String>> copyFile(
            @RequestParam String sourceKey,
            @RequestParam String destinationKey) {
        
        try {
            s3Service.copyFile(sourceKey, destinationKey);
            
            Map<String, String> response = new HashMap<>();
            response.put("message", "File copied successfully");
            
            return ResponseEntity.ok(response);
        } catch (Exception e) {
            Map<String, String> error = new HashMap<>();
            error.put("error", e.getMessage());
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
        }
    }

    @PostMapping("/move")
    public ResponseEntity<Map<String, String>> moveFile(
            @RequestParam String sourceKey,
            @RequestParam String destinationKey) {
        
        try {
            s3Service.moveFile(sourceKey, destinationKey);
            
            Map<String, String> response = new HashMap<>();
            response.put("message", "File moved successfully");
            
            return ResponseEntity.ok(response);
        } catch (Exception e) {
            Map<String, String> error = new HashMap<>();
            error.put("error", e.getMessage());
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
        }
    }
}
```

## Advanced Features

### Multipart Upload for Large Files

```java
package com.example.s3.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import software.amazon.awssdk.core.sync.RequestBody;
import software.amazon.awssdk.services.s3.S3Client;
import software.amazon.awssdk.services.s3.model.*;

import java.io.File;
import java.io.IOException;
import java.io.RandomAccessFile;
import java.util.ArrayList;
import java.util.List;

@Service
public class S3MultipartUploadService {

    @Autowired
    private S3Client s3Client;

    @Value("${aws.s3.bucket-name}")
    private String bucketName;

    private static final long PART_SIZE = 5 * 1024 * 1024; // 5MB

    public String uploadLargeFile(File file, String key) throws IOException {
        // Initiate multipart upload
        CreateMultipartUploadRequest createRequest = CreateMultipartUploadRequest.builder()
                .bucket(bucketName)
                .key(key)
                .build();

        CreateMultipartUploadResponse createResponse = s3Client.createMultipartUpload(createRequest);
        String uploadId = createResponse.uploadId();

        List<CompletedPart> completedParts = new ArrayList<>();

        try (RandomAccessFile randomAccessFile = new RandomAccessFile(file, "r")) {
            long fileSize = file.length();
            long position = 0;
            int partNumber = 1;

            while (position < fileSize) {
                long partSize = Math.min(PART_SIZE, fileSize - position);
                byte[] buffer = new byte[(int) partSize];
                randomAccessFile.seek(position);
                randomAccessFile.read(buffer);

                UploadPartRequest uploadPartRequest = UploadPartRequest.builder()
                        .bucket(bucketName)
                        .key(key)
                        .uploadId(uploadId)
                        .partNumber(partNumber)
                        .build();

                UploadPartResponse uploadPartResponse = s3Client.uploadPart(
                        uploadPartRequest,
                        RequestBody.fromBytes(buffer));

                completedParts.add(CompletedPart.builder()
                        .partNumber(partNumber)
                        .eTag(uploadPartResponse.eTag())
                        .build());

                position += partSize;
                partNumber++;
            }

            // Complete multipart upload
            CompletedMultipartUpload completedUpload = CompletedMultipartUpload.builder()
                    .parts(completedParts)
                    .build();

            CompleteMultipartUploadRequest completeRequest = CompleteMultipartUploadRequest.builder()
                    .bucket(bucketName)
                    .key(key)
                    .uploadId(uploadId)
                    .multipartUpload(completedUpload)
                    .build();

            s3Client.completeMultipartUpload(completeRequest);

            return key;
        } catch (Exception e) {
            // Abort upload on error
            AbortMultipartUploadRequest abortRequest = AbortMultipartUploadRequest.builder()
                    .bucket(bucketName)
                    .key(key)
                    .uploadId(uploadId)
                    .build();

            s3Client.abortMultipartUpload(abortRequest);
            throw new RuntimeException("Multipart upload failed", e);
        }
    }
}
```

### Async Upload Service

```java
package com.example.s3.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import software.amazon.awssdk.core.async.AsyncRequestBody;
import software.amazon.awssdk.services.s3.S3AsyncClient;
import software.amazon.awssdk.services.s3.model.PutObjectRequest;
import software.amazon.awssdk.services.s3.model.PutObjectResponse;

import java.nio.file.Path;
import java.util.concurrent.CompletableFuture;

@Service
public class S3AsyncService {

    @Autowired
    private S3AsyncClient s3AsyncClient;

    @Value("${aws.s3.bucket-name}")
    private String bucketName;

    public CompletableFuture<PutObjectResponse> uploadFileAsync(String key, Path filePath) {
        PutObjectRequest putObjectRequest = PutObjectRequest.builder()
                .bucket(bucketName)
                .key(key)
                .build();

        return s3AsyncClient.putObject(putObjectRequest, AsyncRequestBody.fromFile(filePath));
    }

    public CompletableFuture<PutObjectResponse> uploadBytesAsync(String key, byte[] data) {
        PutObjectRequest putObjectRequest = PutObjectRequest.builder()
                .bucket(bucketName)
                .key(key)
                .build();

        return s3AsyncClient.putObject(putObjectRequest, AsyncRequestBody.fromBytes(data));
    }
}
```

### S3 Event Listener

```java
package com.example.s3.listener;

import com.example.s3.event.S3UploadEvent;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.context.event.EventListener;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Component;

@Component
public class S3EventListener {

    private static final Logger logger = LoggerFactory.getLogger(S3EventListener.class);

    @Async
    @EventListener
    public void handleS3UploadEvent(S3UploadEvent event) {
        logger.info("File uploaded to S3: {}", event.getKey());
        
        // Perform post-upload processing
        // - Generate thumbnail
        // - Extract metadata
        // - Send notification
        // - Update database
    }
}
```

### S3 Upload Event

```java
package com.example.s3.event;

import org.springframework.context.ApplicationEvent;

public class S3UploadEvent extends ApplicationEvent {

    private final String key;
    private final long size;
    private final String contentType;

    public S3UploadEvent(Object source, String key, long size, String contentType) {
        super(source);
        this.key = key;
        this.size = size;
        this.contentType = contentType;
    }

    public String getKey() {
        return key;
    }

    public long getSize() {
        return size;
    }

    public String getContentType() {
        return contentType;
    }
}
```

## Bucket Operations

### Bucket Management Service

```java
package com.example.s3.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import software.amazon.awssdk.services.s3.S3Client;
import software.amazon.awssdk.services.s3.model.*;

import java.util.List;
import java.util.stream.Collectors;

@Service
public class S3BucketService {

    @Autowired
    private S3Client s3Client;

    public void createBucket(String bucketName) {
        CreateBucketRequest createBucketRequest = CreateBucketRequest.builder()
                .bucket(bucketName)
                .build();

        s3Client.createBucket(createBucketRequest);
    }

    public void deleteBucket(String bucketName) {
        DeleteBucketRequest deleteBucketRequest = DeleteBucketRequest.builder()
                .bucket(bucketName)
                .build();

        s3Client.deleteBucket(deleteBucketRequest);
    }

    public boolean bucketExists(String bucketName) {
        try {
            HeadBucketRequest headBucketRequest = HeadBucketRequest.builder()
                    .bucket(bucketName)
                    .build();

            s3Client.headBucket(headBucketRequest);
            return true;
        } catch (NoSuchBucketException e) {
            return false;
        }
    }

    public List<String> listBuckets() {
        ListBucketsResponse listBucketsResponse = s3Client.listBuckets();
        
        return listBucketsResponse.buckets().stream()
                .map(Bucket::name)
                .collect(Collectors.toList());
    }

    public void setBucketPolicy(String bucketName, String policy) {
        PutBucketPolicyRequest policyRequest = PutBucketPolicyRequest.builder()
                .bucket(bucketName)
                .policy(policy)
                .build();

        s3Client.putBucketPolicy(policyRequest);
    }

    public void enableVersioning(String bucketName) {
        VersioningConfiguration versioningConfig = VersioningConfiguration.builder()
                .status(BucketVersioningStatus.ENABLED)
                .build();

        PutBucketVersioningRequest versioningRequest = PutBucketVersioningRequest.builder()
                .bucket(bucketName)
                .versioningConfiguration(versioningConfig)
                .build();

        s3Client.putBucketVersioning(versioningRequest);
    }
}
```

## Security Best Practices

### Encrypted Upload

```java
public String uploadEncryptedFile(MultipartFile file) throws IOException {
    String key = generateFileName(file.getOriginalFilename());
    
    PutObjectRequest putObjectRequest = PutObjectRequest.builder()
            .bucket(bucketName)
            .key(key)
            .contentType(file.getContentType())
            .serverSideEncryption(ServerSideEncryption.AES256)
            .build();

    s3Client.putObject(putObjectRequest, RequestBody.fromInputStream(
            file.getInputStream(), file.getSize()));

    return key;
}
```

### Access Control

```java
public void setObjectAcl(String key, String granteeEmail) {
    Grantee grantee = Grantee.builder()
            .emailAddress(granteeEmail)
            .type(Type.AMAZON_CUSTOMER_BY_EMAIL)
            .build();

    Grant grant = Grant.builder()
            .grantee(grantee)
            .permission(Permission.READ)
            .build();

    AccessControlPolicy acp = AccessControlPolicy.builder()
            .grants(grant)
            .build();

    PutObjectAclRequest aclRequest = PutObjectAclRequest.builder()
            .bucket(bucketName)
            .key(key)
            .accessControlPolicy(acp)
            .build();

    s3Client.putObjectAcl(aclRequest);
}
```

## Testing

### LocalStack Integration Test

```java
package com.example.s3.integration;

import com.example.s3.service.S3Service;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.mock.web.MockMultipartFile;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.localstack.LocalStackContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.utility.DockerImageName;

import static org.junit.jupiter.api.Assertions.*;
import static org.testcontainers.containers.localstack.LocalStackContainer.Service.S3;

@SpringBootTest
@Testcontainers
class S3ServiceIntegrationTest {

    @Container
    static LocalStackContainer localStack = new LocalStackContainer(
            DockerImageName.parse("localstack/localstack:latest"))
            .withServices(S3);

    @Autowired
    private S3Service s3Service;

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("aws.s3.endpoint", () -> localStack.getEndpointOverride(S3).toString());
        registry.add("aws.s3.region", () -> localStack.getRegion());
        registry.add("aws.s3.access-key", () -> localStack.getAccessKey());
        registry.add("aws.s3.secret-key", () -> localStack.getSecretKey());
    }

    @Test
    void testUploadAndDownload() throws Exception {
        // Create test file
        MockMultipartFile file = new MockMultipartFile(
                "test.txt",
                "test.txt",
                "text/plain",
                "Hello S3!".getBytes()
        );

        // Upload
        String key = s3Service.uploadFile(file);
        assertNotNull(key);

        // Download
        byte[] downloaded = s3Service.downloadFile(key);
        assertEquals("Hello S3!", new String(downloaded));

        // Clean up
        s3Service.deleteFile(key);
    }

    @Test
    void testFileExists() throws Exception {
        MockMultipartFile file = new MockMultipartFile(
                "exists.txt",
                "exists.txt",
                "text/plain",
                "Test".getBytes()
        );

        String key = s3Service.uploadFile(file);
        assertTrue(s3Service.fileExists(key));

        s3Service.deleteFile(key);
        assertFalse(s3Service.fileExists(key));
    }
}
```

## Performance Optimization

### Connection Pooling

```java
@Bean
public S3Client s3Client() {
    return S3Client.builder()
            .region(Region.of(region))
            .credentialsProvider(credentialsProvider())
            .httpClientBuilder(ApacheHttpClient.builder()
                    .maxConnections(100)
                    .connectionTimeout(Duration.ofSeconds(30))
                    .socketTimeout(Duration.ofSeconds(30)))
            .build();
}
```

### Transfer Manager for Large Files

```xml
<dependency>
    <groupId>software.amazon.awssdk</groupId>
    <artifactId>s3-transfer-manager</artifactId>
    <version>2.20.26</version>
</dependency>
```

```java
package com.example.s3.service;

import org.springframework.stereotype.Service;
import software.amazon.awssdk.transfer.s3.S3TransferManager;
import software.amazon.awssdk.transfer.s3.model.*;

import java.nio.file.Path;
import java.util.concurrent.CompletableFuture;

@Service
public class S3TransferService {

    private final S3TransferManager transferManager;

    public S3TransferService(S3TransferManager transferManager) {
        this.transferManager = transferManager;
    }

    public CompletableFuture<CompletedFileUpload> uploadFile(String bucket, String key, Path filePath) {
        UploadFileRequest uploadRequest = UploadFileRequest.builder()
                .putObjectRequest(req -> req.bucket(bucket).key(key))
                .source(filePath)
                .build();

        FileUpload upload = transferManager.uploadFile(uploadRequest);
        return upload.completionFuture();
    }

    public CompletableFuture<CompletedFileDownload> downloadFile(String bucket, String key, Path destination) {
        DownloadFileRequest downloadRequest = DownloadFileRequest.builder()
                .getObjectRequest(req -> req.bucket(bucket).key(key))
                .destination(destination)
                .build();

        FileDownload download = transferManager.downloadFile(downloadRequest);
        return download.completionFuture();
    }
}
```

## Common Use Cases

1. **Static Asset Storage** - Images, CSS, JavaScript
2. **File Uploads** - User documents, media files
3. **Backup and Archive** - Database backups, log files
4. **Data Lakes** - Large-scale data storage
5. **Content Distribution** - CDN source files

## Additional Resources

- [AWS S3 Documentation](https://docs.aws.amazon.com/s3/)
- [AWS SDK for Java v2](https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/)
- [S3 Best Practices](https://docs.aws.amazon.com/AmazonS3/latest/userguide/best-practices.html)
- [LocalStack for Testing](https://docs.localstack.cloud/user-guide/aws/s3/)
