# Docker and Docker Compose

## Overview

Docker is a platform for developing, shipping, and running applications in containers. This guide covers Docker fundamentals, Docker Compose for multi-container applications, networking, volumes, and best practices for Spring Boot applications.

## Table of Contents

- [What is Docker?](#what-is-docker)
- [Docker Architecture](#docker-architecture)
- [Dockerfile Best Practices](#dockerfile-best-practices)
- [Multi-Stage Builds](#multi-stage-builds)
- [Docker Networking](#docker-networking)
- [Docker Volumes](#docker-volumes)
- [Docker Compose](#docker-compose)
- [Container Orchestration](#container-orchestration)
- [POC Implementation](#poc-implementation)

## What is Docker?

Docker enables you to:

- Package applications with all dependencies
- Ensure consistency across environments
- Isolate applications from each other
- Scale applications easily
- Improve deployment speed
- Optimize resource utilization

### Key Concepts

- **Image**: Read-only template with application code and dependencies
- **Container**: Running instance of an image
- **Dockerfile**: Text file with instructions to build an image
- **Registry**: Storage for Docker images (Docker Hub, private registries)
- **Volume**: Persistent storage for container data
- **Network**: Communication channel between containers

## Docker Architecture

### Docker Components

```
┌─────────────────────────────────────────┐
│           Docker Client                 │
│  (docker commands: build, run, push)    │
└─────────────┬───────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────┐
│           Docker Daemon                 │
│  ┌──────────────────────────────────┐  │
│  │         Images                    │  │
│  ├──────────────────────────────────┤  │
│  │         Containers                │  │
│  ├──────────────────────────────────┤  │
│  │         Networks                  │  │
│  ├──────────────────────────────────┤  │
│  │         Volumes                   │  │
│  └──────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

### Container Lifecycle

```bash
# Build image
docker build -t myapp:1.0 .

# Run container
docker run -d --name myapp-container myapp:1.0

# Stop container
docker stop myapp-container

# Start container
docker start myapp-container

# Remove container
docker rm myapp-container

# Remove image
docker rmi myapp:1.0
```

## Dockerfile Best Practices

### Basic Dockerfile for Spring Boot

```dockerfile
FROM eclipse-temurin:21-jre-alpine

# Add metadata
LABEL maintainer="your-email@example.com"
LABEL version="1.0"
LABEL description="Spring Boot Application"

# Create app directory
WORKDIR /app

# Copy the jar file
COPY target/*.jar app.jar

# Expose port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=60s --retries=3 \
  CMD wget --quiet --tries=1 --spider http://localhost:8080/actuator/health || exit 1

# Run the application
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Optimized Dockerfile

```dockerfile
FROM eclipse-temurin:21-jre-alpine

# Install required packages
RUN apk add --no-cache curl

# Create non-root user
RUN addgroup -S spring && adduser -S spring -G spring

# Set working directory
WORKDIR /app

# Copy jar file
COPY --chown=spring:spring target/*.jar app.jar

# Switch to non-root user
USER spring:spring

# Expose port
EXPOSE 8080

# JVM optimization flags
ENV JAVA_OPTS="-XX:+UseContainerSupport \
               -XX:MaxRAMPercentage=75.0 \
               -XX:InitialRAMPercentage=50.0 \
               -XX:+UseG1GC \
               -XX:+UseStringDeduplication \
               -Djava.security.egd=file:/dev/./urandom"

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=60s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1

# Run application
ENTRYPOINT exec java $JAVA_OPTS -jar app.jar
```

### Dockerfile with Arguments

```dockerfile
FROM eclipse-temurin:21-jre-alpine

# Build arguments
ARG JAR_FILE=target/*.jar
ARG APP_VERSION=1.0
ARG SPRING_PROFILE=prod

# Environment variables
ENV APP_VERSION=${APP_VERSION}
ENV SPRING_PROFILES_ACTIVE=${SPRING_PROFILE}

WORKDIR /app

# Copy jar
COPY ${JAR_FILE} app.jar

EXPOSE 8080

ENTRYPOINT ["sh", "-c", "java ${JAVA_OPTS} -jar app.jar"]
```

### Build with Arguments

```bash
# Build with custom arguments
docker build \
  --build-arg JAR_FILE=target/myapp-1.0.jar \
  --build-arg APP_VERSION=1.0 \
  --build-arg SPRING_PROFILE=prod \
  -t myapp:1.0 .
```

## Multi-Stage Builds

### Maven Multi-Stage Build

```dockerfile
# Stage 1: Build
FROM maven:3.9-eclipse-temurin-21 AS build

WORKDIR /app

# Copy pom.xml and download dependencies (cached layer)
COPY pom.xml .
RUN mvn dependency:go-offline -B

# Copy source code
COPY src ./src

# Build application
RUN mvn clean package -DskipTests

# Stage 2: Runtime
FROM eclipse-temurin:21-jre-alpine

WORKDIR /app

# Copy jar from build stage
COPY --from=build /app/target/*.jar app.jar

# Create non-root user
RUN addgroup -S spring && adduser -S spring -G spring
USER spring:spring

EXPOSE 8080

ENV JAVA_OPTS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0"

HEALTHCHECK --interval=30s --timeout=3s --start-period=60s \
  CMD wget --quiet --tries=1 --spider http://localhost:8080/actuator/health || exit 1

ENTRYPOINT exec java $JAVA_OPTS -jar app.jar
```

### Gradle Multi-Stage Build

```dockerfile
# Stage 1: Build
FROM gradle:8.5-jdk21 AS build

WORKDIR /app

# Copy gradle files (cached layer)
COPY build.gradle settings.gradle ./
COPY gradle ./gradle

# Download dependencies
RUN gradle dependencies --no-daemon

# Copy source
COPY src ./src

# Build application
RUN gradle bootJar --no-daemon

# Stage 2: Runtime
FROM eclipse-temurin:21-jre-alpine

WORKDIR /app

COPY --from=build /app/build/libs/*.jar app.jar

RUN addgroup -S spring && adduser -S spring -G spring
USER spring:spring

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=60s \
  CMD wget -q --spider http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Spring Boot Layered Jar

```dockerfile
# Stage 1: Build
FROM eclipse-temurin:21-jdk AS build

WORKDIR /app

COPY target/*.jar app.jar

# Extract layers
RUN java -Djarmode=layertools -jar app.jar extract

# Stage 2: Runtime
FROM eclipse-temurin:21-jre-alpine

WORKDIR /app

# Copy layers in order of least to most frequently changing
COPY --from=build /app/dependencies/ ./
COPY --from=build /app/spring-boot-loader/ ./
COPY --from=build /app/snapshot-dependencies/ ./
COPY --from=build /app/application/ ./

RUN addgroup -S spring && adduser -S spring -G spring
USER spring:spring

EXPOSE 8080

ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

### Enable Layered Jar in pom.xml

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
                <layers>
                    <enabled>true</enabled>
                </layers>
            </configuration>
        </plugin>
    </plugins>
</build>
```

## Docker Networking

### Network Types

```bash
# Bridge network (default)
docker network create my-bridge-network

# Host network (uses host's network stack)
docker run --network host myapp:1.0

# None network (no networking)
docker run --network none myapp:1.0

# Custom bridge network
docker network create --driver bridge my-custom-network
```

### Network Commands

```bash
# List networks
docker network ls

# Inspect network
docker network inspect my-bridge-network

# Create network with subnet
docker network create \
  --driver=bridge \
  --subnet=172.28.0.0/16 \
  --ip-range=172.28.5.0/24 \
  --gateway=172.28.5.254 \
  my-network

# Connect container to network
docker network connect my-network myapp-container

# Disconnect container from network
docker network disconnect my-network myapp-container

# Remove network
docker network rm my-network
```

### Container Communication

```bash
# Create network
docker network create app-network

# Run database container
docker run -d \
  --name postgres-db \
  --network app-network \
  -e POSTGRES_PASSWORD=secret \
  postgres:15

# Run app container (can access postgres-db by name)
docker run -d \
  --name spring-app \
  --network app-network \
  -e SPRING_DATASOURCE_URL=jdbc:postgresql://postgres-db:5432/mydb \
  myapp:1.0
```

### Network Configuration Example

```dockerfile
# Dockerfile with network configuration
FROM eclipse-temurin:21-jre-alpine

WORKDIR /app

COPY target/*.jar app.jar

# Expose multiple ports
EXPOSE 8080 8081 9090

# Application will connect to other services by hostname
ENV DB_HOST=postgres-db
ENV REDIS_HOST=redis-cache
ENV KAFKA_BOOTSTRAP_SERVERS=kafka:9092

ENTRYPOINT ["java", "-jar", "app.jar"]
```

## Docker Volumes

### Volume Types

```bash
# Named volume
docker volume create app-data

# Anonymous volume (created automatically)
docker run -v /app/data myapp:1.0

# Bind mount (host directory)
docker run -v /host/path:/container/path myapp:1.0
```

### Volume Commands

```bash
# List volumes
docker volume ls

# Inspect volume
docker volume inspect app-data

# Create volume
docker volume create --driver local app-data

# Remove volume
docker volume rm app-data

# Remove all unused volumes
docker volume prune
```

### Volume Usage Examples

```bash
# Run with named volume
docker run -d \
  --name postgres-db \
  -v postgres-data:/var/lib/postgresql/data \
  postgres:15

# Run with bind mount
docker run -d \
  --name spring-app \
  -v $(pwd)/logs:/app/logs \
  -v $(pwd)/config:/app/config \
  myapp:1.0

# Run with multiple volumes
docker run -d \
  --name spring-app \
  -v app-data:/app/data \
  -v app-logs:/app/logs \
  -v app-config:/app/config:ro \
  myapp:1.0
```

### Volume Backup and Restore

```bash
# Backup volume
docker run --rm \
  -v postgres-data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/postgres-backup.tar.gz -C /data .

# Restore volume
docker run --rm \
  -v postgres-data:/data \
  -v $(pwd):/backup \
  alpine tar xzf /backup/postgres-backup.tar.gz -C /data
```

### tmpfs Mounts (In-Memory)

```bash
# Use tmpfs for temporary data
docker run -d \
  --name spring-app \
  --tmpfs /app/temp:rw,size=100m \
  myapp:1.0
```

## Docker Compose

### Basic docker-compose.yml

```yaml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=prod
    depends_on:
      - postgres
      - redis
    networks:
      - app-network

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - app-network

  redis:
    image: redis:7-alpine
    networks:
      - app-network

volumes:
  postgres-data:

networks:
  app-network:
    driver: bridge
```

### Complete Microservices Setup

```yaml
version: '3.8'

services:
  # PostgreSQL Database
  postgres:
    image: postgres:15-alpine
    container_name: postgres-db
    environment:
      POSTGRES_DB: ${DB_NAME:-mydb}
      POSTGRES_USER: ${DB_USER:-postgres}
      POSTGRES_PASSWORD: ${DB_PASSWORD:-secret}
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Redis Cache
  redis:
    image: redis:7-alpine
    container_name: redis-cache
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    networks:
      - backend
    command: redis-server --appendonly yes
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

  # Kafka
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    container_name: zookeeper
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    networks:
      - backend

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    container_name: kafka
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    networks:
      - backend

  # RabbitMQ
  rabbitmq:
    image: rabbitmq:3.12-management-alpine
    container_name: rabbitmq
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: ${RABBITMQ_USER:-admin}
      RABBITMQ_DEFAULT_PASS: ${RABBITMQ_PASS:-admin}
    volumes:
      - rabbitmq-data:/var/lib/rabbitmq
    networks:
      - backend

  # Spring Boot Application
  app:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        JAR_FILE: target/*.jar
    container_name: spring-boot-app
    ports:
      - "8080:8080"
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/mydb
      SPRING_DATASOURCE_USERNAME: ${DB_USER:-postgres}
      SPRING_DATASOURCE_PASSWORD: ${DB_PASSWORD:-secret}
      SPRING_REDIS_HOST: redis
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:29092
      SPRING_RABBITMQ_HOST: rabbitmq
      JAVA_OPTS: -Xmx512m -Xms256m
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
      kafka:
        condition: service_started
    networks:
      - backend
      - frontend
    volumes:
      - ./logs:/app/logs
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

  # Nginx Reverse Proxy
  nginx:
    image: nginx:alpine
    container_name: nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    depends_on:
      - app
    networks:
      - frontend
    restart: unless-stopped

volumes:
  postgres-data:
    driver: local
  redis-data:
    driver: local
  rabbitmq-data:
    driver: local

networks:
  backend:
    driver: bridge
  frontend:
    driver: bridge
```

### Environment Variables (.env)

```bash
# .env file
DB_NAME=mydb
DB_USER=postgres
DB_PASSWORD=secret123
RABBITMQ_USER=admin
RABBITMQ_PASS=admin123
ENVIRONMENT=production
```

### Docker Compose Commands

```bash
# Start all services
docker-compose up -d

# Start specific service
docker-compose up -d app

# View logs
docker-compose logs -f app

# Stop all services
docker-compose down

# Stop and remove volumes
docker-compose down -v

# Rebuild and start
docker-compose up -d --build

# Scale services
docker-compose up -d --scale app=3

# Execute command in container
docker-compose exec app sh

# View running services
docker-compose ps

# Restart service
docker-compose restart app
```

### Override Files

```yaml
# docker-compose.override.yml (automatically loaded)
version: '3.8'

services:
  app:
    environment:
      - DEBUG=true
      - LOG_LEVEL=DEBUG
    volumes:
      - ./src:/app/src
```

```yaml
# docker-compose.prod.yml (use with -f flag)
version: '3.8'

services:
  app:
    image: myregistry/myapp:1.0
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '2'
          memory: 2G
        reservations:
          cpus: '1'
          memory: 1G
```

```bash
# Use production override
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

## Container Orchestration

### Resource Limits

```yaml
services:
  app:
    image: myapp:1.0
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2G
        reservations:
          cpus: '1.0'
          memory: 1G
    mem_limit: 2g
    mem_reservation: 1g
    cpus: 2.0
```

### Health Checks

```yaml
services:
  app:
    image: myapp:1.0
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

### Restart Policies

```yaml
services:
  app:
    image: myapp:1.0
    restart: unless-stopped  # no, always, on-failure, unless-stopped
```

### Logging Configuration

```yaml
services:
  app:
    image: myapp:1.0
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

## POC Implementation

### Project Structure

```
docker-poc/
├── src/
│   └── main/
│       ├── java/
│       └── resources/
│           └── application.yml
├── docker/
│   ├── app/
│   │   └── Dockerfile
│   ├── nginx/
│   │   ├── Dockerfile
│   │   └── nginx.conf
│   └── init-scripts/
│       └── init.sql
├── Dockerfile
├── docker-compose.yml
├── docker-compose.prod.yml
├── .env.example
├── .dockerignore
└── README.md
```

### .dockerignore

```
# Version control
.git
.gitignore

# Build artifacts
target/
build/
*.class

# IDE
.idea/
.vscode/
*.iml
.project
.classpath

# OS
.DS_Store
Thumbs.db

# Logs
*.log
logs/

# Documentation
README.md
docs/

# Test files
src/test/

# Docker
Dockerfile*
docker-compose*
```

### Makefile for Common Commands

```makefile
.PHONY: build up down logs clean restart

build:
	docker-compose build

up:
	docker-compose up -d

down:
	docker-compose down

logs:
	docker-compose logs -f

clean:
	docker-compose down -v
	docker system prune -f

restart:
	docker-compose restart

rebuild:
	docker-compose down
	docker-compose build --no-cache
	docker-compose up -d

prod-up:
	docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d

prod-down:
	docker-compose -f docker-compose.yml -f docker-compose.prod.yml down
```

### Nginx Configuration

```nginx
# nginx.conf
events {
    worker_connections 1024;
}

http {
    upstream spring-app {
        least_conn;
        server app:8080 max_fails=3 fail_timeout=30s;
    }

    server {
        listen 80;
        server_name localhost;

        client_max_body_size 10M;

        location / {
            proxy_pass http://spring-app;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            # Timeouts
            proxy_connect_timeout 60s;
            proxy_send_timeout 60s;
            proxy_read_timeout 60s;
        }

        location /actuator/health {
            proxy_pass http://spring-app/actuator/health;
            access_log off;
        }
    }
}
```

### Init SQL Script

```sql
-- init.sql
CREATE DATABASE IF NOT EXISTS mydb;

\c mydb;

CREATE TABLE IF NOT EXISTS users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(255) NOT NULL UNIQUE,
    email VARCHAR(255) NOT NULL UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);

INSERT INTO users (username, email) VALUES
    ('admin', 'admin@example.com'),
    ('user1', 'user1@example.com');
```

### Application Configuration for Docker

```yaml
# application-docker.yml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST:postgres}:5432/${DB_NAME:mydb}
    username: ${DB_USER:postgres}
    password: ${DB_PASSWORD:secret}
    hikari:
      maximum-pool-size: 10
      minimum-idle: 5
      connection-timeout: 30000

  redis:
    host: ${REDIS_HOST:redis}
    port: 6379

  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:kafka:29092}

  rabbitmq:
    host: ${RABBITMQ_HOST:rabbitmq}
    port: 5672
    username: ${RABBITMQ_USER:admin}
    password: ${RABBITMQ_PASS:admin}

logging:
  level:
    root: INFO
    com.example: DEBUG
  file:
    name: /app/logs/application.log

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
```

## Best Practices

1. **Image Optimization**
   - Use multi-stage builds
   - Use specific base image versions
   - Minimize layer count
   - Remove unnecessary files
   - Use .dockerignore

2. **Security**
   - Don't run as root
   - Scan images for vulnerabilities
   - Use official base images
   - Keep images updated
   - Don't store secrets in images

3. **Networking**
   - Use custom bridge networks
   - Isolate services appropriately
   - Use container names for DNS
   - Limit exposed ports

4. **Volumes**
   - Use named volumes for persistence
   - Use bind mounts for development
   - Back up important data
   - Clean up unused volumes

5. **Compose Files**
   - Use environment variables
   - Separate concerns (dev/prod)
   - Version your compose files
   - Document configurations

6. **Performance**
   - Set resource limits
   - Use health checks
   - Configure restart policies
   - Optimize JVM settings for containers

## Common Commands

```bash
# Image management
docker images
docker pull image:tag
docker push image:tag
docker tag source target
docker rmi image:tag

# Container management
docker ps
docker ps -a
docker stop container
docker start container
docker restart container
docker rm container
docker exec -it container sh

# System management
docker system df
docker system prune -a
docker volume prune
docker network prune

# Logs and debugging
docker logs container
docker logs -f container
docker inspect container
docker stats
docker top container
```

## Troubleshooting

```bash
# Check container logs
docker-compose logs app

# Check container status
docker-compose ps

# Inspect container
docker inspect spring-boot-app

# Execute shell in container
docker-compose exec app sh

# Check network connectivity
docker-compose exec app ping postgres

# View resource usage
docker stats

# Check port bindings
docker port spring-boot-app
```

## Additional Resources

- [Docker Documentation](https://docs.docker.com/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [Docker Hub](https://hub.docker.com/)
- [Spring Boot Docker Guide](https://spring.io/guides/topicals/spring-boot-docker/)
