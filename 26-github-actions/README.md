# GitHub Actions

## Overview

GitHub Actions is a CI/CD platform that allows you to automate your build, test, and deployment pipeline. This guide covers comprehensive GitHub Actions workflows for Spring Boot microservices, including incremental builds, artifact management, and automated deployments.

## Table of Contents

- [What is GitHub Actions?](#what-is-github-actions)
- [Workflow Basics](#workflow-basics)
- [Building Spring Boot Applications](#building-spring-boot-applications)
- [Incremental Builds with Git Diff](#incremental-builds-with-git-diff)
- [Artifact Management with S3/MinIO](#artifact-management-with-s3minio)
- [Multi-Environment Deployments](#multi-environment-deployments)
- [GraalVM Native Images](#graalvm-native-images)
- [Docker Build and Push](#docker-build-and-push)
- [Self-Hosted Runners](#self-hosted-runners)
- [POC Implementation](#poc-implementation)

## What is GitHub Actions?

GitHub Actions enables you to:

- Automate software workflows
- Build, test, and deploy code
- Run CI/CD pipelines
- Respond to repository events
- Integrate with external services
- Manage releases and deployments

### Key Components

- **Workflow**: Automated process defined in YAML
- **Job**: Set of steps executed on the same runner
- **Step**: Individual task within a job
- **Action**: Reusable unit of code
- **Runner**: Server that executes workflows
- **Event**: Activity that triggers a workflow

## Workflow Basics

### Simple Workflow Structure

```yaml
name: Spring Boot CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        
      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: maven
          
      - name: Build with Maven
        run: mvn clean install
        
      - name: Run tests
        run: mvn test
```

### Workflow Triggers

```yaml
on:
  # Push to specific branches
  push:
    branches:
      - main
      - develop
      - 'release/**'
    paths:
      - 'src/**'
      - 'pom.xml'
      
  # Pull requests
  pull_request:
    branches:
      - main
    types:
      - opened
      - synchronize
      
  # Manual trigger
  workflow_dispatch:
    inputs:
      environment:
        description: 'Deployment environment'
        required: true
        default: 'staging'
        type: choice
        options:
          - dev
          - staging
          - production
          
  # Scheduled runs
  schedule:
    - cron: '0 2 * * *'  # Daily at 2 AM
    
  # Release events
  release:
    types: [published]
```

## Building Spring Boot Applications

### Maven Build Workflow

```yaml
name: Maven Build

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        
      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'maven'
          
      - name: Cache Maven packages
        uses: actions/cache@v4
        with:
          path: ~/.m2/repository
          key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
          restore-keys: |
            ${{ runner.os }}-maven-
            
      - name: Build with Maven
        run: mvn clean package -DskipTests
        
      - name: Run unit tests
        run: mvn test
        
      - name: Run integration tests
        run: mvn verify -P integration-tests
        
      - name: Generate test coverage report
        run: mvn jacoco:report
        
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: target/surefire-reports/
          
      - name: Upload coverage report
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: target/site/jacoco/
          
      - name: Upload JAR artifact
        uses: actions/upload-artifact@v4
        with:
          name: app-jar
          path: target/*.jar
```

### Gradle Build Workflow

```yaml
name: Gradle Build

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        
      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'gradle'
          
      - name: Grant execute permission for gradlew
        run: chmod +x gradlew
        
      - name: Cache Gradle packages
        uses: actions/cache@v4
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
          restore-keys: |
            ${{ runner.os }}-gradle-
            
      - name: Build with Gradle
        run: ./gradlew clean build -x test
        
      - name: Run tests
        run: ./gradlew test
        
      - name: Generate coverage report
        run: ./gradlew jacocoTestReport
        
      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: gradle-build
          path: build/libs/*.jar
```

## Incremental Builds with Git Diff

### Detect Changed Modules Script

```bash
#!/bin/bash
# detect-changes.sh

set -euo pipefail

BASE_REF="${1:-HEAD~1}"
HEAD_REF="${2:-HEAD}"

echo "🔍 Detecting changes between $BASE_REF and $HEAD_REF"

# Get changed files
CHANGED_FILES=$(git diff --name-only "$BASE_REF" "$HEAD_REF")

# Extract unique module names
CHANGED_MODULES=$(echo "$CHANGED_FILES" | \
  grep "^services/" | \
  cut -d'/' -f2 | \
  sort -u)

# Exclude common/shared modules
EXCLUDED_MODULES="common|shared|utils|test"
CHANGED_MODULES=$(echo "$CHANGED_MODULES" | \
  grep -vE "^($EXCLUDED_MODULES)$" || true)

if [ -z "$CHANGED_MODULES" ]; then
  echo "ℹ️ No modules changed"
  echo "[]" > changed-modules.json
  exit 0
fi

echo "📦 Changed modules:"
echo "$CHANGED_MODULES"

# Convert to JSON array
MODULES_JSON=$(echo "$CHANGED_MODULES" | jq -R . | jq -s .)
echo "$MODULES_JSON" > changed-modules.json
echo "modules=$MODULES_JSON" >> $GITHUB_OUTPUT
```

### Gradle Build Script for Changed Modules

```kotlin
// build.gradle.kts - Incremental build logic

val buildOrder = listOf(
    // Core services
    "auth-service",
    "user-service",
    
    // Domain services
    "order-service",
    "payment-service",
    "inventory-service",
    
    // API Gateway
    "api-gateway"
)

val excluded = setOf(
    "common",
    "shared",
    "utils",
    "test-utils"
)

fun getChangedModules(): Set<String> {
    if (project.hasProperty("all")) {
        println("🛠 Building all modules")
        return rootProject.subprojects
            .map { it.name }
            .filterNot { it in excluded }
            .toSet()
    }
    
    val base = project.findProperty("gitBase")?.toString() ?: "HEAD~1"
    val head = project.findProperty("gitHead")?.toString() ?: "HEAD"
    
    val output = ByteArrayOutputStream()
    exec {
        commandLine("git", "diff", "--name-only", "$base..$head")
        standardOutput = output
    }
    
    return output.toString()
        .lines()
        .filter { it.startsWith("services/") }
        .mapNotNull { it.split("/").getOrNull(1) }
        .filterNot { it in excluded }
        .toSet()
}

tasks.register("buildChangedModules") {
    group = "build"
    description = "Build only changed modules"
    
    val changed = getChangedModules()
    val ordered = buildOrder.filter { it in changed } +
                  changed.filterNot { it in buildOrder }.sorted()
    
    println("🔨 Building modules: $ordered")
    
    val bootJarTasks = ordered.mapNotNull { moduleName ->
        subprojects.find { it.name == moduleName }
            ?.tasks?.findByName("bootJar")
    }
    
    dependsOn(bootJarTasks)
}

tasks.register("exportChangedJars") {
    group = "distribution"
    description = "Export JARs for changed modules"
    dependsOn("buildChangedModules")
    
    val environment = project.findProperty("env")?.toString() ?: "dev"
    val outputDir = layout.buildDirectory.dir("artifacts/$environment")
    
    doLast {
        val changed = getChangedModules()
        val exportDir = outputDir.get().asFile
        exportDir.mkdirs()
        
        val manifestFile = exportDir.resolve("manifest.txt")
        manifestFile.writeText("# Changed modules\n")
        
        changed.forEach { moduleName ->
            val project = rootProject.subprojects.find { it.name == moduleName }
                ?: return@forEach
            
            val jarFile = project.buildDir
                .resolve("libs")
                .listFiles()
                ?.firstOrNull { it.name.endsWith(".jar") && !it.name.contains("plain") }
            
            if (jarFile?.exists() == true) {
                val targetDir = exportDir.resolve(moduleName)
                targetDir.mkdirs()
                
                jarFile.copyTo(targetDir.resolve("app.jar"), overwrite = true)
                manifestFile.appendText("$moduleName\n")
                
                println("✅ Exported: $moduleName")
            }
        }
    }
}
```

### GitHub Actions Workflow with Incremental Build

```yaml
name: Incremental Build and Deploy

on:
  push:
    branches:
      - main
      - develop
  workflow_dispatch:
    inputs:
      build_all:
        description: 'Build all modules'
        required: false
        type: boolean
        default: false

jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      changed_modules: ${{ steps.detect.outputs.modules }}
      has_changes: ${{ steps.detect.outputs.has_changes }}
      
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
          
      - name: Detect changed modules
        id: detect
        run: |
          chmod +x ./scripts/detect-changes.sh
          ./scripts/detect-changes.sh "${{ github.event.before }}" "${{ github.sha }}"
          
          MODULES=$(cat changed-modules.json)
          echo "modules=$MODULES" >> $GITHUB_OUTPUT
          
          if [ "$MODULES" = "[]" ]; then
            echo "has_changes=false" >> $GITHUB_OUTPUT
          else
            echo "has_changes=true" >> $GITHUB_OUTPUT
          fi

  build:
    needs: detect-changes
    if: needs.detect-changes.outputs.has_changes == 'true' || github.event.inputs.build_all == 'true'
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
          
      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'gradle'
          
      - name: Cache Gradle packages
        uses: actions/cache@v4
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*') }}
          restore-keys: |
            ${{ runner.os }}-gradle-
            
      - name: Build changed modules
        run: |
          if [ "${{ github.event.inputs.build_all }}" = "true" ]; then
            ./gradlew exportChangedJars -Pall -Penv=prod
          else
            ./gradlew exportChangedJars -Penv=prod \
              -PgitBase=${{ github.event.before }} \
              -PgitHead=${{ github.sha }}
          fi
          
      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: service-jars
          path: build/artifacts/prod/
          retention-days: 7
```

## Artifact Management with S3/MinIO

### Upload to S3 Script

```bash
#!/bin/bash
# upload-to-s3.sh

set -euo pipefail

ENVIRONMENT="${1:-dev}"
AWS_REGION="${AWS_REGION:-us-east-1}"
S3_BUCKET="${S3_BUCKET:-my-artifacts}"
ARTIFACTS_DIR="build/artifacts/$ENVIRONMENT"

echo "☁️ Uploading artifacts to S3..."
echo "Environment: $ENVIRONMENT"
echo "Bucket: s3://$S3_BUCKET/$ENVIRONMENT"

if [ ! -d "$ARTIFACTS_DIR" ]; then
  echo "❌ Artifacts directory not found: $ARTIFACTS_DIR"
  exit 1
fi

# Upload each module
for module_dir in "$ARTIFACTS_DIR"/*; do
  if [ -d "$module_dir" ]; then
    module_name=$(basename "$module_dir")
    
    echo "📤 Uploading $module_name..."
    
    # Delete old versions
    aws s3 rm "s3://$S3_BUCKET/$ENVIRONMENT/$module_name/" \
      --recursive --region "$AWS_REGION" || true
    
    # Upload new version
    aws s3 cp "$module_dir/" "s3://$S3_BUCKET/$ENVIRONMENT/$module_name/" \
      --recursive --region "$AWS_REGION"
    
    echo "✅ Uploaded $module_name"
  fi
done

# Upload manifest
if [ -f "$ARTIFACTS_DIR/manifest.txt" ]; then
  aws s3 cp "$ARTIFACTS_DIR/manifest.txt" \
    "s3://$S3_BUCKET/$ENVIRONMENT/manifest.txt" \
    --region "$AWS_REGION"
fi

echo "✅ Upload complete"
```

### Upload to MinIO Script

```bash
#!/bin/bash
# upload-to-minio.sh

set -euo pipefail

ENVIRONMENT="${1:-dev}"
ALIAS_NAME="${MINIO_ALIAS:-minio}"
ENDPOINT="${MINIO_ENDPOINT:-https://minio.example.com}"
ACCESS_KEY="${MINIO_ACCESS_KEY}"
SECRET_KEY="${MINIO_SECRET_KEY}"
BUCKET_NAME="${MINIO_BUCKET:-artifacts}"

ARTIFACTS_DIR="build/artifacts/$ENVIRONMENT"
MANIFEST_FILE="$ARTIFACTS_DIR/manifest.txt"

echo "🔗 Setting up MinIO connection..."
mc alias set "$ALIAS_NAME" "$ENDPOINT" "$ACCESS_KEY" "$SECRET_KEY" --insecure

if [ ! -f "$MANIFEST_FILE" ]; then
  echo "❌ Manifest file not found: $MANIFEST_FILE"
  exit 1
fi

echo "📦 Modules to upload:"
cat "$MANIFEST_FILE" | grep -v "^#"

echo "☁️ Uploading artifacts to MinIO..."

while IFS= read -r module; do
  [ -z "$module" ] || [ "${module:0:1}" = "#" ] && continue
  
  module_dir="$ARTIFACTS_DIR/$module"
  
  if [ ! -d "$module_dir" ]; then
    echo "⚠️ Skipping missing module: $module"
    continue
  fi
  
  echo "🗑️ Cleaning old artifacts for: $module"
  mc rm --recursive --force "$ALIAS_NAME/$BUCKET_NAME/$ENVIRONMENT/$module/" || true
  
  echo "📤 Uploading: $module"
  mc cp --recursive "$module_dir/" "$ALIAS_NAME/$BUCKET_NAME/$ENVIRONMENT/$module/"
  
  echo "✅ Uploaded: $module"
done < "$MANIFEST_FILE"

# Upload manifest
echo "📄 Uploading manifest..."
mc cp "$MANIFEST_FILE" "$ALIAS_NAME/$BUCKET_NAME/$ENVIRONMENT/manifest.txt"

echo "✅ Upload complete"
```

### GitHub Actions with S3/MinIO

```yaml
name: Build and Upload to MinIO

on:
  push:
    branches:
      - main
      - staging
      - production

jobs:
  build-and-upload:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
          
      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'gradle'
          
      - name: Determine environment
        id: env
        run: |
          if [[ "${{ github.ref }}" == "refs/heads/production" ]]; then
            echo "environment=production" >> $GITHUB_OUTPUT
          elif [[ "${{ github.ref }}" == "refs/heads/staging" ]]; then
            echo "environment=staging" >> $GITHUB_OUTPUT
          else
            echo "environment=dev" >> $GITHUB_OUTPUT
          fi
          
      - name: Build changed modules
        run: |
          ./gradlew exportChangedJars \
            -Penv=${{ steps.env.outputs.environment }} \
            -PgitBase=${{ github.event.before }} \
            -PgitHead=${{ github.sha }}
            
      - name: Install MinIO Client
        run: |
          wget https://dl.min.io/client/mc/release/linux-amd64/mc -O /usr/local/bin/mc
          chmod +x /usr/local/bin/mc
          mc --version
          
      - name: Upload to MinIO
        run: |
          chmod +x ./scripts/upload-to-minio.sh
          ./scripts/upload-to-minio.sh ${{ steps.env.outputs.environment }}
        env:
          MINIO_ALIAS: artifacts
          MINIO_ENDPOINT: ${{ secrets.MINIO_ENDPOINT }}
          MINIO_ACCESS_KEY: ${{ secrets.MINIO_ACCESS_KEY }}
          MINIO_SECRET_KEY: ${{ secrets.MINIO_SECRET_KEY }}
          MINIO_BUCKET: ${{ secrets.MINIO_BUCKET }}
          
      - name: Notify deployment
        if: success()
        run: |
          echo "✅ Artifacts uploaded to MinIO"
          echo "Environment: ${{ steps.env.outputs.environment }}"
```

## Multi-Environment Deployments

### Deployment Workflow

```yaml
name: Multi-Environment Deployment

on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        type: choice
        options:
          - dev
          - staging
          - production

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ github.event.inputs.environment }}
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        
      - name: Download artifacts from MinIO
        run: |
          mc alias set minio ${{ secrets.MINIO_ENDPOINT }} \
            ${{ secrets.MINIO_ACCESS_KEY }} ${{ secrets.MINIO_SECRET_KEY }}
          
          mc mirror minio/${{ secrets.MINIO_BUCKET }}/${{ github.event.inputs.environment }} \
            ./artifacts/
            
      - name: Deploy to VPS
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USERNAME }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          port: ${{ secrets.SSH_PORT }}
          script: |
            set -e
            cd ~/apps/${{ github.event.inputs.environment }}
            ./download-artifacts.sh
            ./deploy.sh
            
      - name: Health check
        run: |
          sleep 30
          curl -f https://${{ github.event.inputs.environment }}.example.com/actuator/health || exit 1
          
      - name: Notify team
        if: always()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "Deployment to ${{ github.event.inputs.environment }}: ${{ job.status }}",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Deployment Status*\nEnvironment: ${{ github.event.inputs.environment }}\nStatus: ${{ job.status }}\nCommit: ${{ github.sha }}"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### Server-Side Deployment Script

```bash
#!/bin/bash
# deploy.sh

set -euo pipefail

ENVIRONMENT="${1:-dev}"
MANIFEST_FILE="./manifest.txt"

echo "🚀 Starting deployment for $ENVIRONMENT"

if [ ! -f "$MANIFEST_FILE" ]; then
  echo "❌ Manifest file not found"
  exit 1
fi

# Read modules from manifest
while IFS= read -r module; do
  [ -z "$module" ] || [ "${module:0:1}" = "#" ] && continue
  
  echo "📦 Deploying $module..."
  
  cd "$module" || continue
  
  # Stop existing container
  docker-compose down || true
  
  # Build and start new container
  COMPOSE_PROJECT_NAME="app-$module-$ENVIRONMENT" \
    docker-compose up --build -d
  
  # Wait for health check
  echo "⏳ Waiting for $module to be healthy..."
  timeout 60 bash -c "
    until docker-compose ps | grep -q 'healthy\|Up'; do
      sleep 2
    done
  " || echo "⚠️ Health check timeout for $module"
  
  cd ..
  
  echo "✅ Deployed $module"
done < "$MANIFEST_FILE"

# Clean up old images
docker image prune -f

echo "✅ Deployment complete"
```

## GraalVM Native Images

### GraalVM Build Workflow

```yaml
name: GraalVM Native Image Build

on:
  push:
    branches: [ main ]

jobs:
  build-native:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        
      - name: Set up GraalVM
        uses: graalvm/setup-graalvm@v1
        with:
          java-version: '21'
          distribution: 'graalvm-community'
          github-token: ${{ secrets.GITHUB_TOKEN }}
          native-image-job-reports: 'true'
          
      - name: Cache Maven packages
        uses: actions/cache@v4
        with:
          path: ~/.m2/repository
          key: ${{ runner.os }}-maven-graalvm-${{ hashFiles('**/pom.xml') }}
          restore-keys: |
            ${{ runner.os }}-maven-graalvm-
            
      - name: Build native image
        run: |
          mvn -Pnative native:compile -DskipTests
          
      - name: Test native image
        run: |
          ./target/myapp
          
      - name: Build Docker image with native binary
        run: |
          docker build -f Dockerfile.native -t myapp:native .
          
      - name: Upload native binary
        uses: actions/upload-artifact@v4
        with:
          name: native-executable
          path: target/myapp
```

### Native Image Dockerfile

```dockerfile
# Dockerfile.native - Multi-stage for GraalVM
FROM ghcr.io/graalvm/graalvm-community:21 AS build

WORKDIR /app

# Install native-image
RUN gu install native-image

# Copy source
COPY pom.xml ./
COPY src ./src

# Build native image
RUN mvn -Pnative native:compile -DskipTests

# Runtime stage
FROM debian:bookworm-slim

WORKDIR /app

# Install runtime dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    ca-certificates && \
    rm -rf /var/lib/apt/lists/*

# Copy native executable
COPY --from=build /app/target/myapp ./myapp

# Create non-root user
RUN useradd -r -u 1000 -g root appuser
USER appuser

EXPOSE 8080

ENTRYPOINT ["./myapp"]
```

### GraalVM Configuration

```xml
<!-- pom.xml -->
<profiles>
    <profile>
        <id>native</id>
        <build>
            <plugins>
                <plugin>
                    <groupId>org.graalvm.buildtools</groupId>
                    <artifactId>native-maven-plugin</artifactId>
                    <version>0.10.1</version>
                    <configuration>
                        <imageName>myapp</imageName>
                        <mainClass>com.example.Application</mainClass>
                        <buildArgs>
                            <buildArg>--no-fallback</buildArg>
                            <buildArg>-H:+ReportExceptionStackTraces</buildArg>
                            <buildArg>--enable-preview</buildArg>
                        </buildArgs>
                    </configuration>
                    <executions>
                        <execution>
                            <id>build-native</id>
                            <goals>
                                <goal>compile-no-fork</goal>
                            </goals>
                            <phase>package</phase>
                        </execution>
                    </executions>
                </plugin>
            </plugins>
        </build>
    </profile>
</profiles>
```

## Docker Build and Push

### Docker Build and Push Workflow

```yaml
name: Docker Build and Push

on:
  push:
    branches:
      - main
    tags:
      - 'v*'

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
        
      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
          
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix={{branch}}-
            
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          platforms: linux/amd64,linux/arm64
```

### Multi-Service Docker Build

```yaml
name: Build Multiple Services

on:
  push:
    branches: [ main ]

jobs:
  build-matrix:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service:
          - auth-service
          - user-service
          - order-service
          - payment-service
          
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
        
      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
          
      - name: Build and push ${{ matrix.service }}
        uses: docker/build-push-action@v5
        with:
          context: ./services/${{ matrix.service }}
          push: true
          tags: |
            myorg/${{ matrix.service }}:latest
            myorg/${{ matrix.service }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

## Self-Hosted Runners

### Setup Self-Hosted Runner

```bash
#!/bin/bash
# setup-runner.sh

# Download runner
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64-2.311.0.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.311.0/actions-runner-linux-x64-2.311.0.tar.gz

# Extract
tar xzf ./actions-runner-linux-x64-2.311.0.tar.gz

# Configure
./config.sh --url https://github.com/myorg/myrepo \
  --token YOUR_TOKEN \
  --name my-runner \
  --labels linux,x64,docker

# Install as service
sudo ./svc.sh install
sudo ./svc.sh start
```

### Self-Hosted Runner Workflow

```yaml
name: Build on Self-Hosted Runner

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: self-hosted
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
          
      - name: Set up JDK (already installed on runner)
        run: java -version
        
      - name: Build with Gradle
        run: ./gradlew clean build
        
      - name: Build Docker images
        run: |
          docker build -t myapp:${{ github.sha }} .
          
      - name: Deploy locally
        run: |
          docker-compose up -d
```

## POC Implementation

### Project Structure

```
github-actions-poc/
├── .github/
│   └── workflows/
│       ├── build.yml
│       ├── deploy-dev.yml
│       ├── deploy-staging.yml
│       ├── deploy-production.yml
│       ├── native-image.yml
│       └── pr-check.yml
├── scripts/
│   ├── detect-changes.sh
│   ├── upload-to-minio.sh
│   ├── upload-to-s3.sh
│   └── deploy.sh
├── services/
│   ├── auth-service/
│   ├── user-service/
│   └── order-service/
├── build.gradle.kts
├── settings.gradle.kts
└── README.md
```

### Complete CI/CD Workflow

```yaml
name: Complete CI/CD Pipeline

on:
  push:
    branches:
      - main
      - develop
      - 'release/**'
  pull_request:
    branches:
      - main
  workflow_dispatch:
    inputs:
      environment:
        description: 'Deployment environment'
        required: true
        type: choice
        options:
          - dev
          - staging
          - production

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      changed_modules: ${{ steps.detect.outputs.modules }}
      has_changes: ${{ steps.detect.outputs.has_changes }}
      
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
          
      - name: Detect changes
        id: detect
        run: |
          chmod +x ./scripts/detect-changes.sh
          ./scripts/detect-changes.sh "${{ github.event.before }}" "${{ github.sha }}"
          
          MODULES=$(cat changed-modules.json)
          echo "modules=$MODULES" >> $GITHUB_OUTPUT
          
          if [ "$MODULES" = "[]" ]; then
            echo "has_changes=false" >> $GITHUB_OUTPUT
          else
            echo "has_changes=true" >> $GITHUB_OUTPUT
          fi

  test:
    needs: detect-changes
    if: needs.detect-changes.outputs.has_changes == 'true'
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        
      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'gradle'
          
      - name: Run tests
        run: ./gradlew test
        
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: '**/build/test-results/**/*.xml'

  build:
    needs: [detect-changes, test]
    if: needs.detect-changes.outputs.has_changes == 'true'
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
          
      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'gradle'
          
      - name: Cache Gradle packages
        uses: actions/cache@v4
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*') }}
          restore-keys: |
            ${{ runner.os }}-gradle-
            
      - name: Determine environment
        id: env
        run: |
          if [[ "${{ github.ref }}" == "refs/heads/main" ]]; then
            echo "environment=production" >> $GITHUB_OUTPUT
          elif [[ "${{ github.ref }}" == "refs/heads/develop" ]]; then
            echo "environment=staging" >> $GITHUB_OUTPUT
          else
            echo "environment=dev" >> $GITHUB_OUTPUT
          fi
          
      - name: Build changed modules
        run: |
          ./gradlew exportChangedJars \
            -Penv=${{ steps.env.outputs.environment }} \
            -PgitBase=${{ github.event.before }} \
            -PgitHead=${{ github.sha }}
            
      - name: Build Docker images
        run: |
          cd build/artifacts/${{ steps.env.outputs.environment }}
          for module in */; do
            module_name=${module%/}
            echo "Building Docker image for $module_name"
            cd "$module_name"
            docker build -t myorg/$module_name:${{ github.sha }} .
            cd ..
          done
          
      - name: Install MinIO Client
        run: |
          wget https://dl.min.io/client/mc/release/linux-amd64/mc -O /usr/local/bin/mc
          chmod +x /usr/local/bin/mc
          
      - name: Upload to MinIO
        run: |
          chmod +x ./scripts/upload-to-minio.sh
          ./scripts/upload-to-minio.sh ${{ steps.env.outputs.environment }}
        env:
          MINIO_ALIAS: artifacts
          MINIO_ENDPOINT: ${{ secrets.MINIO_ENDPOINT }}
          MINIO_ACCESS_KEY: ${{ secrets.MINIO_ACCESS_KEY }}
          MINIO_SECRET_KEY: ${{ secrets.MINIO_SECRET_KEY }}
          MINIO_BUCKET: ${{ secrets.MINIO_BUCKET }}

  deploy:
    needs: build
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    environment: ${{ github.ref == 'refs/heads/main' && 'production' || 'staging' }}
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        
      - name: Deploy to server
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USERNAME }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          port: ${{ secrets.SSH_PORT }}
          script: |
            set -e
            cd ~/apps
            ./download-artifacts.sh
            ./deploy.sh
            
      - name: Health check
        run: |
          sleep 30
          curl -f https://api.example.com/actuator/health || exit 1
          
      - name: Notify deployment
        if: always()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "Deployment ${{ job.status }}",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Deployment Complete*\nEnvironment: ${{ github.ref == 'refs/heads/main' && 'Production' || 'Staging' }}\nStatus: ${{ job.status }}\nCommit: ${{ github.sha }}"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### Pull Request Check Workflow

```yaml
name: PR Checks

on:
  pull_request:
    branches:
      - main
      - develop

jobs:
  validate:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
          
      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'gradle'
          
      - name: Run tests
        run: ./gradlew test
        
      - name: Check code style
        run: ./gradlew checkstyleMain checkstyleTest
        
      - name: Run static analysis
        run: ./gradlew spotbugsMain
        
      - name: Generate coverage report
        run: ./gradlew jacocoTestReport
        
      - name: Comment PR with coverage
        uses: madrapps/jacoco-report@v1.6
        with:
          paths: ${{ github.workspace }}/**/build/reports/jacoco/test/jacocoTestReport.xml
          token: ${{ secrets.GITHUB_TOKEN }}
          min-coverage-overall: 80
          min-coverage-changed-files: 80
```

## Best Practices

1. **Workflow Organization**
   - Use reusable workflows
   - Separate concerns (build, test, deploy)
   - Use workflow templates
   - Version workflow files

2. **Security**
   - Use secrets for sensitive data
   - Minimize secret exposure
   - Use OIDC for cloud authentication
   - Scan for vulnerabilities

3. **Performance**
   - Cache dependencies
   - Use matrix builds for parallel execution
   - Optimize Docker layer caching
   - Use self-hosted runners for large projects

4. **Reliability**
   - Set appropriate timeouts
   - Use retry mechanisms
   - Implement proper error handling
   - Monitor workflow execution

5. **Incremental Builds**
   - Detect changed modules accurately
   - Build only what's necessary
   - Maintain build order dependencies
   - Use artifact caching

## Common Pitfalls

- Not using dependency caching
- Running full builds for small changes
- Exposing secrets in logs
- Not setting workflow timeouts
- Insufficient error handling
- Missing rollback mechanisms
- Not testing workflows before merging

## Additional Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [GitHub Actions Marketplace](https://github.com/marketplace?type=actions)
- [Workflow Syntax Reference](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)
- [GraalVM Native Image](https://www.graalvm.org/latest/reference-manual/native-image/)