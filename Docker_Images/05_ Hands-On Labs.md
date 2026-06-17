# Hands-On Labs: Build, Ship, and Run Real Applications
# Python | Java | Node.js

---

## What We Are Building Today

Three complete end-to-end journeys. Each one starts from zero — no code, no image, nothing — and ends with a running container accessible from the public internet via your VM's IP address.

For each application you will:

1. Write the application code from scratch
2. Write a Dockerfile and build an image
3. Run it locally and test it
4. Write a multi-stage Dockerfile and compare image sizes
5. Push the optimized image to Docker Hub
6. Pull the image and run it
7. Access the running application from the internet using your VM's public IP

By the end of this session you will have done the complete Docker workflow three times, with three different languages, with your own images live on Docker Hub.

---

## Pre-Lab Setup — Do This Once Before Starting

### Step 1: Verify Docker is running

```bash
docker --version
docker ps
```

### Step 2: Log in to Docker Hub

If you do not have a Docker Hub account, create one at hub.docker.com now. It is free.

```bash
docker login
```

Enter your Docker Hub username and password. You will see:

```
Login Succeeded
```

Store your username in a variable so every command in this session works without editing:

```bash
export DOCKERHUB_USERNAME=yourDockerHubUsername
echo "Logged in as: $DOCKERHUB_USERNAME"
```

Replace yourDockerHubUsername with your actual Docker Hub username.

### Step 3: Create working directory

```bash
mkdir -p /home/docker-labs
cd /home/docker-labs
```

### Step 4: Find your VM public IP

```bash
curl -s ifconfig.me
```

Write this IP down. You will use it to access each application from a browser after running the container.

---

---

# LAB 1: Python Flask Application

---

## Lab 1 Overview

You will build a Python Flask web API that returns live system information — hostname, IP address, and the current time. This is a realistic application that demonstrates environment variables, JSON responses, and port mapping.

**What the application does:**
- Endpoint `/` returns a welcome message
- Endpoint `/info` returns hostname, IP, and current time as JSON
- Endpoint `/health` returns health status

---

## Lab 1 — Step 1: Create Project Structure

```bash
mkdir -p /home/docker-labs/lab1-python
cd /home/docker-labs/lab1-python
```

---

## Lab 1 — Step 2: Write the Application

```bash
cat > app.py << 'EOF'
from flask import Flask, jsonify
import socket
import datetime
import os

app = Flask(__name__)

APP_VERSION = os.environ.get("APP_VERSION", "1.0.0")
APP_ENV = os.environ.get("APP_ENV", "development")

@app.route("/")
def home():
    return jsonify({
        "message": "Welcome to Build Automate Architect",
        "app": "Python Flask API",
        "version": APP_VERSION,
        "environment": APP_ENV
    })

@app.route("/info")
def info():
    hostname = socket.gethostname()
    try:
        ip = socket.gethostbyname(hostname)
    except:
        ip = "unavailable"

    return jsonify({
        "hostname": hostname,
        "ip_address": ip,
        "timestamp": datetime.datetime.utcnow().isoformat(),
        "version": APP_VERSION,
        "environment": APP_ENV,
        "message": "Container is running successfully"
    })

@app.route("/health")
def health():
    return jsonify({
        "status": "healthy",
        "version": APP_VERSION
    })

if __name__ == "__main__":
    port = int(os.environ.get("PORT", 5000))
    print(f"Starting app on port {port}")
    app.run(host="0.0.0.0", port=port)
EOF
```

```bash
cat > requirements.txt << 'EOF'
flask==3.0.0
EOF
```

Verify the files are created:

```bash
ls -la
cat app.py
cat requirements.txt
```

---

## Lab 1 — Step 3: Write the Single-Stage Dockerfile

```bash
cat > Dockerfile.single << 'EOF'
# Single-stage Dockerfile
# Simple but not optimized for production

FROM python:3.11

WORKDIR /app

COPY requirements.txt .

RUN pip install -r requirements.txt

COPY app.py .

ENV APP_VERSION=1.0.0
ENV APP_ENV=production

EXPOSE 5000

CMD ["python", "app.py"]
EOF
```

---

## Lab 1 — Step 4: Build the Single-Stage Image

```bash
docker build -f Dockerfile.single -t lab1-python:single .
```

Watch every step. Notice the layer being created at each instruction.

Check the image size:

```bash
docker images lab1-python:single
```

Write down the size.

---

## Lab 1 — Step 5: Run and Test Locally

```bash
docker run -d \
  --name python-single \
  -p 5000:5000 \
  -e APP_VERSION=1.0.0 \
  -e APP_ENV=development \
  lab1-python:single
```

Verify it is running:

```bash
docker ps
docker logs python-single
```

Test all three endpoints:

```bash
curl http://localhost:5000/
echo ""
curl http://localhost:5000/info
echo ""
curl http://localhost:5000/health
```

You should see JSON responses from all three endpoints.

Access from browser using your VM public IP:

```
http://YOUR_VM_PUBLIC_IP:5000/
http://YOUR_VM_PUBLIC_IP:5000/info
http://YOUR_VM_PUBLIC_IP:5000/health
```

If the browser cannot connect, open the firewall port:

```bash
# On Ubuntu with ufw
sudo ufw allow 5000/tcp

# On cloud VMs (AWS/GCP/Azure) open port 5000 in the security group
```

Stop and remove this container before continuing:

```bash
docker stop python-single && docker rm python-single
```

---

## Lab 1 — Step 6: Write the Multi-Stage Dockerfile

```bash
cat > Dockerfile.multi << 'EOF'
# ============================================================
# STAGE 1: BUILDER
# Install dependencies using full Python image
# This stage is discarded after build
# ============================================================
FROM python:3.11 AS builder

WORKDIR /build

COPY requirements.txt .

# --user installs packages to /root/.local
# Makes them easy to copy to Stage 2
RUN pip install --no-cache-dir --user -r requirements.txt


# ============================================================
# STAGE 2: RUNTIME
# Minimal image — only what is needed to RUN the app
# This is the final image that gets pushed to Docker Hub
# ============================================================
FROM python:3.11-slim AS runtime

WORKDIR /app

# Copy ONLY the installed packages from Stage 1
# Nothing else from Stage 1 comes across
COPY --from=builder /root/.local /root/.local

# Copy application code from local filesystem
COPY app.py .

# Make Python find the packages we copied
ENV PATH=/root/.local/bin:$PATH
ENV APP_VERSION=1.0.0
ENV APP_ENV=production
ENV PYTHONUNBUFFERED=1

# Non-root user for security
RUN groupadd --system appgroup && \
    useradd --system --gid appgroup --no-create-home appuser && \
    chown -R appuser:appgroup /app /root/.local

USER appuser

EXPOSE 5000

CMD ["python", "app.py"]
EOF
```

---

## Lab 1 — Step 7: Build Multi-Stage Image and Compare

```bash
docker build -f Dockerfile.multi -t lab1-python:multi .
```

Compare sizes:

```bash
echo "=== LAB 1 SIZE COMPARISON ==="
docker images | grep lab1-python
```

Check what tools are present in each image:

```bash
echo "--- Single stage: is pip present? ---"
docker run --rm lab1-python:single which pip

echo "--- Multi stage: is pip present? ---"
docker run --rm lab1-python:multi which pip 2>/dev/null || echo "pip NOT present — attack surface reduced"

echo "--- Multi stage: does the app still work? ---"
docker run --rm lab1-python:multi python -c "import flask; print('Flask works:', flask.__version__)"
```

---

## Lab 1 — Step 8: Tag and Push to Docker Hub

Tag the optimized multi-stage image with your Docker Hub username:

```bash
docker tag lab1-python:multi $DOCKERHUB_USERNAME/lab1-python:v1.0.0
docker tag lab1-python:multi $DOCKERHUB_USERNAME/lab1-python:latest
```

Verify tags:

```bash
docker images | grep lab1-python
```

Push both tags to Docker Hub:

```bash
docker push $DOCKERHUB_USERNAME/lab1-python:v1.0.0
docker push $DOCKERHUB_USERNAME/lab1-python:latest
```

Watch the layers upload. Notice Docker says "Layer already exists" for layers shared with the base image.

Verify on Docker Hub:

```
Open browser → hub.docker.com → Your profile → Repositories
You should see lab1-python with tags v1.0.0 and latest
```

---

## Lab 1 — Step 9: Delete Local Image and Pull from Docker Hub

This proves the image is truly on Docker Hub and not just local.

Delete the local image:

```bash
docker rmi $DOCKERHUB_USERNAME/lab1-python:v1.0.0
docker rmi $DOCKERHUB_USERNAME/lab1-python:latest
docker images | grep lab1-python
```

The tagged images are gone from local. Now pull from Docker Hub:

```bash
docker pull $DOCKERHUB_USERNAME/lab1-python:v1.0.0
```

Watch the layers download. Docker Hub is the source now.

---

## Lab 1 — Step 10: Run from Pulled Image and Access via Public IP

```bash
docker run -d \
  --name python-app \
  -p 5000:5000 \
  -e APP_VERSION=1.0.0 \
  -e APP_ENV=production \
  --restart unless-stopped \
  $DOCKERHUB_USERNAME/lab1-python:v1.0.0
```

Verify:

```bash
docker ps
docker logs python-app
curl http://localhost:5000/info
```

Access from browser:

```
http://YOUR_VM_PUBLIC_IP:5000/
http://YOUR_VM_PUBLIC_IP:5000/info
http://YOUR_VM_PUBLIC_IP:5000/health
```

You are now running your own Docker image, pulled from Docker Hub, accessible from the internet.

Clean up before Lab 2:

```bash
docker stop python-app && docker rm python-app
```

---

## Lab 1 — Results Summary

Fill in after completing the lab:

| Metric | Single Stage | Multi Stage |
|--------|-------------|-------------|
| Image size | _______ MB | _______ MB |
| pip present | Yes | No |
| gcc present | Yes | No |
| App works | Yes | Yes |
| On Docker Hub | No | Yes |

---

---

# LAB 2: Java Spring Boot Application

---

## Lab 2 Overview

You will build a Java Spring Boot REST API. Java is where multi-stage builds make the most dramatic difference — the JDK (Java Development Kit) used to compile is hundreds of MB larger than the JRE (Java Runtime Environment) needed to run.

**What the application does:**
- Endpoint `/` returns welcome message
- Endpoint `/info` returns app info and system details
- Endpoint `/health` returns health status

**Key learning:** The JDK compiles Java code. The JRE runs it. Multi-stage build uses JDK in Stage 1 and JRE in Stage 2. The JDK never enters the production image.

---

## Lab 2 — Step 1: Install Java and Maven

Check if Java is installed:

```bash
java -version 2>/dev/null || echo "Java not installed"
mvn -version 2>/dev/null || echo "Maven not installed"
```

Install if needed:

```bash
sudo apt-get update
sudo apt-get install -y openjdk-17-jdk maven
java -version
mvn -version
```

---

## Lab 2 — Step 2: Create Spring Boot Project Structure

```bash
mkdir -p /home/docker-labs/lab2-java
cd /home/docker-labs/lab2-java
```

Create the Maven project structure:

```bash
mkdir -p src/main/java/com/buildautomate/api
mkdir -p src/main/resources
```

---

## Lab 2 — Step 3: Write the Application

Create the main application class:

```bash
cat > src/main/java/com/buildautomate/api/Application.java << 'EOF'
package com.buildautomate.api;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
EOF
```

Create the REST controller:

```bash
cat > src/main/java/com/buildautomate/api/ApiController.java << 'EOF'
package com.buildautomate.api;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import java.net.InetAddress;
import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.Map;

@RestController
public class ApiController {

    @Value("${app.version:1.0.0}")
    private String appVersion;

    @Value("${app.environment:production}")
    private String appEnvironment;

    @GetMapping("/")
    public Map<String, String> home() {
        Map<String, String> response = new HashMap<>();
        response.put("message", "Welcome to Build Automate Architect");
        response.put("app", "Java Spring Boot API");
        response.put("version", appVersion);
        response.put("environment", appEnvironment);
        return response;
    }

    @GetMapping("/info")
    public Map<String, String> info() {
        Map<String, String> response = new HashMap<>();
        try {
            InetAddress ip = InetAddress.getLocalHost();
            response.put("hostname", ip.getHostName());
            response.put("ip_address", ip.getHostAddress());
        } catch (Exception e) {
            response.put("hostname", "unavailable");
            response.put("ip_address", "unavailable");
        }
        response.put("timestamp", LocalDateTime.now().toString());
        response.put("version", appVersion);
        response.put("environment", appEnvironment);
        response.put("message", "Java container is running successfully");
        return response;
    }

    @GetMapping("/health")
    public Map<String, String> health() {
        Map<String, String> response = new HashMap<>();
        response.put("status", "healthy");
        response.put("version", appVersion);
        return response;
    }
}
EOF
```

Create application properties:

```bash
cat > src/main/resources/application.properties << 'EOF'
server.port=8080
app.version=${APP_VERSION:1.0.0}
app.environment=${APP_ENV:production}
EOF
```

Create the Maven pom.xml:

```bash
cat > pom.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
        <relativePath/>
    </parent>

    <groupId>com.buildautomate</groupId>
    <artifactId>api</artifactId>
    <version>1.0.0</version>
    <name>Build Automate Architect API</name>

    <properties>
        <java.version>17</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
EOF
```

---

## Lab 2 — Step 4: Test the App Locally First

```bash
mvn clean package -q
java -jar target/api-1.0.0.jar &
sleep 5
curl http://localhost:8080/
curl http://localhost:8080/info
curl http://localhost:8080/health
```

Stop the local app:

```bash
pkill -f "api-1.0.0.jar"
```

---

## Lab 2 — Step 5: Write the Single-Stage Dockerfile

```bash
cat > Dockerfile.single << 'EOF'
# Single stage — JDK used for both build and run
# JDK is huge — stays in the final image unnecessarily

FROM eclipse-temurin:17-jdk

WORKDIR /app

COPY pom.xml .
COPY src ./src

RUN apt-get update && \
    apt-get install -y maven && \
    mvn clean package -q && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

EXPOSE 8080

CMD ["java", "-jar", "target/api-1.0.0.jar"]
EOF
```

Build it:

```bash
docker build -f Dockerfile.single -t lab2-java:single .
```

This will take 3-5 minutes on first build — Maven downloads dependencies from the internet.

Check the size:

```bash
docker images lab2-java:single
```

---

## Lab 2 — Step 6: Run and Test Single Stage

```bash
docker run -d \
  --name java-single \
  -p 8080:8080 \
  lab2-java:single
```

Wait 15 seconds for Spring Boot to start:

```bash
sleep 15
docker logs java-single
curl http://localhost:8080/
curl http://localhost:8080/info
curl http://localhost:8080/health
```

Stop it:

```bash
docker stop java-single && docker rm java-single
```

---

## Lab 2 — Step 7: Write the Multi-Stage Dockerfile

This is where Java shows the most dramatic size reduction.

```bash
cat > Dockerfile.multi << 'EOF'
# ============================================================
# STAGE 1: BUILDER
# Uses full JDK to compile the Java source code
# Maven downloads dependencies and builds the JAR file
# This entire stage is DISCARDED after build
# ============================================================
FROM eclipse-temurin:17-jdk AS builder

WORKDIR /build

# Copy Maven wrapper and pom.xml first
# Maven dependencies are cached as long as pom.xml does not change
COPY pom.xml .

# Download dependencies (cached layer — only re-runs when pom.xml changes)
RUN apt-get update && \
    apt-get install -y maven --no-install-recommends && \
    mvn dependency:go-offline -q && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Copy source code (separate layer — re-runs when code changes)
COPY src ./src

# Compile and package the application
# -DskipTests skips tests here (run tests in a separate stage or CI)
RUN mvn clean package -DskipTests -q


# ============================================================
# STAGE 2: RUNTIME
# Uses only JRE — not JDK
# JRE = Java Runtime Environment (run Java programs)
# JDK = Java Development Kit (compile + run) — NOT needed here
# This is the final image
# ============================================================
FROM eclipse-temurin:17-jre AS runtime

WORKDIR /app

# Copy ONLY the compiled JAR from Stage 1
# The JDK, Maven, source code, .m2 cache — all left behind
COPY --from=builder /build/target/api-1.0.0.jar app.jar

ENV APP_VERSION=1.0.0
ENV APP_ENV=production

# Non-root user
RUN groupadd --system appgroup && \
    useradd --system --gid appgroup --no-create-home appuser && \
    chown appuser:appgroup app.jar

USER appuser

EXPOSE 8080

# JVM tuning for containers
CMD ["java", \
     "-XX:+UseContainerSupport", \
     "-XX:MaxRAMPercentage=75.0", \
     "-jar", "app.jar"]
EOF
```

Build the multi-stage image:

```bash
docker build -f Dockerfile.multi -t lab2-java:multi .
```

Because Stage 1 dependency layer is cached from the single-stage build, this will be faster.

---

## Lab 2 — Step 8: Compare Sizes

```bash
echo "=== LAB 2 SIZE COMPARISON ==="
docker images | grep lab2-java
```

The difference here is dramatic. JDK vs JRE alone is a 300-400 MB difference.

Check what is in each image:

```bash
echo "--- Single stage: javac (compiler) present? ---"
docker run --rm lab2-java:single which javac

echo "--- Multi stage: javac (compiler) present? ---"
docker run --rm lab2-java:multi which javac 2>/dev/null || echo "javac NOT present — compiler removed from production"

echo "--- Multi stage: can it RUN Java? ---"
docker run --rm lab2-java:multi java -version
```

JDK contains both `java` (run) and `javac` (compile).
JRE contains only `java` (run).

In production you never need to compile. The compiler has no business being in your production image.

---

## Lab 2 — Step 9: Run Multi-Stage and Test

```bash
docker run -d \
  --name java-multi \
  -p 8080:8080 \
  -e APP_VERSION=1.0.0 \
  -e APP_ENV=production \
  lab2-java:multi
```

Wait for startup:

```bash
sleep 15
docker logs java-multi
curl http://localhost:8080/
curl http://localhost:8080/info
curl http://localhost:8080/health
```

---

## Lab 2 — Step 10: Push to Docker Hub

```bash
docker tag lab2-java:multi $DOCKERHUB_USERNAME/lab2-java:v1.0.0
docker tag lab2-java:multi $DOCKERHUB_USERNAME/lab2-java:latest

docker push $DOCKERHUB_USERNAME/lab2-java:v1.0.0
docker push $DOCKERHUB_USERNAME/lab2-java:latest
```

Delete local and pull back:

```bash
docker stop java-multi && docker rm java-multi
docker rmi $DOCKERHUB_USERNAME/lab2-java:v1.0.0
docker rmi $DOCKERHUB_USERNAME/lab2-java:latest

docker pull $DOCKERHUB_USERNAME/lab2-java:v1.0.0
```

Run from pulled image:

```bash
docker run -d \
  --name java-app \
  -p 8080:8080 \
  --restart unless-stopped \
  $DOCKERHUB_USERNAME/lab2-java:v1.0.0

sleep 15
curl http://localhost:8080/info
```

Open in browser:

```
http://YOUR_VM_PUBLIC_IP:8080/
http://YOUR_VM_PUBLIC_IP:8080/info
```

Open firewall if needed:

```bash
sudo ufw allow 8080/tcp
```

Clean up:

```bash
docker stop java-app && docker rm java-app
```

---

## Lab 2 — Results Summary

| Metric | Single Stage | Multi Stage |
|--------|-------------|-------------|
| Image size | _______ MB | _______ MB |
| javac (compiler) present | Yes | No |
| maven present | Yes | No |
| App works | Yes | Yes |
| Size reduction | — | _______ % |

---

---

# LAB 3: Node.js Express Application

---

## Lab 3 Overview

You will build a Node.js Express API. Node.js demonstrates a different multi-stage pattern — the separation between devDependencies (needed to build and test) and production dependencies (needed to run).

**What the application does:**
- Endpoint `/` returns welcome message
- Endpoint `/info` returns system information
- Endpoint `/health` returns health check

**Key learning:** node_modules with all devDependencies can be hundreds of MB. Production only needs a fraction of that. Multi-stage build installs everything in Stage 1 and copies only production modules to Stage 2.

---

## Lab 3 — Step 1: Create Project Structure

```bash
mkdir -p /home/docker-labs/lab3-node
cd /home/docker-labs/lab3-node
```

---

## Lab 3 — Step 2: Write the Application

Create package.json:

```bash
cat > package.json << 'EOF'
{
  "name": "build-automate-architect-api",
  "version": "1.0.0",
  "description": "Node.js Express API for Docker Lab",
  "main": "app.js",
  "scripts": {
    "start": "node app.js",
    "dev": "nodemon app.js",
    "test": "echo 'Tests passed' && exit 0"
  },
  "dependencies": {
    "express": "^4.18.2"
  },
  "devDependencies": {
    "nodemon": "^3.0.2"
  }
}
EOF
```

Create the application:

```bash
cat > app.js << 'EOF'
const express = require('express');
const os = require('os');

const app = express();
const PORT = process.env.PORT || 3000;
const APP_VERSION = process.env.APP_VERSION || '1.0.0';
const APP_ENV = process.env.APP_ENV || 'production';

app.get('/', (req, res) => {
    res.json({
        message: 'Welcome to Build Automate Architect',
        app: 'Node.js Express API',
        version: APP_VERSION,
        environment: APP_ENV
    });
});

app.get('/info', (req, res) => {
    const networkInterfaces = os.networkInterfaces();
    let ipAddress = 'unavailable';

    Object.values(networkInterfaces).forEach(interfaces => {
        interfaces.forEach(iface => {
            if (iface.family === 'IPv4' && !iface.internal) {
                ipAddress = iface.address;
            }
        });
    });

    res.json({
        hostname: os.hostname(),
        ip_address: ipAddress,
        platform: os.platform(),
        node_version: process.version,
        timestamp: new Date().toISOString(),
        version: APP_VERSION,
        environment: APP_ENV,
        message: 'Node.js container is running successfully'
    });
});

app.get('/health', (req, res) => {
    res.json({
        status: 'healthy',
        version: APP_VERSION,
        uptime_seconds: Math.floor(process.uptime())
    });
});

app.listen(PORT, '0.0.0.0', () => {
    console.log(`Server running on port ${PORT}`);
    console.log(`Version: ${APP_VERSION}`);
    console.log(`Environment: ${APP_ENV}`);
});
EOF
```

---

## Lab 3 — Step 3: Write the Single-Stage Dockerfile

```bash
cat > Dockerfile.single << 'EOF'
# Single stage — installs ALL dependencies including devDependencies
# nodemon, testing tools all end up in production image

FROM node:20

WORKDIR /app

COPY package*.json ./

# npm install installs EVERYTHING including devDependencies
RUN npm install

COPY app.js .

ENV APP_VERSION=1.0.0
ENV APP_ENV=production

EXPOSE 3000

CMD ["node", "app.js"]
EOF
```

Build:

```bash
docker build -f Dockerfile.single -t lab3-node:single .
```

Check size:

```bash
docker images lab3-node:single
```

Run and test:

```bash
docker run -d --name node-single -p 3000:3000 lab3-node:single
sleep 3
curl http://localhost:3000/
curl http://localhost:3000/info
curl http://localhost:3000/health
docker stop node-single && docker rm node-single
```

---

## Lab 3 — Step 4: Write the Multi-Stage Dockerfile

```bash
cat > Dockerfile.multi << 'EOF'
# ============================================================
# STAGE 1: DEPENDENCIES
# Install ALL dependencies including devDependencies
# Used for: building, testing, linting
# This stage is DISCARDED
# ============================================================
FROM node:20-alpine AS dependencies

WORKDIR /app

COPY package*.json ./

# Install everything including devDependencies
RUN npm ci


# ============================================================
# STAGE 2: RUNTIME
# Fresh start — installs ONLY production dependencies
# devDependencies like nodemon, jest never enter this image
# This is the FINAL image
# ============================================================
FROM node:20-alpine AS runtime

WORKDIR /app

COPY package*.json ./

# npm ci --only=production skips ALL devDependencies
RUN npm ci --only=production

# Copy application code
COPY app.js .

ENV APP_VERSION=1.0.0
ENV APP_ENV=production
ENV NODE_ENV=production

# Non-root user
RUN addgroup -S appgroup && \
    adduser -S appuser -G appgroup && \
    chown -R appuser:appgroup /app

USER appuser

EXPOSE 3000

CMD ["node", "app.js"]
EOF
```

Build multi-stage:

```bash
docker build -f Dockerfile.multi -t lab3-node:multi .
```

---

## Lab 3 — Step 5: Compare Sizes and Contents

```bash
echo "=== LAB 3 SIZE COMPARISON ==="
docker images | grep lab3-node
```

Check devDependencies:

```bash
echo "--- Single stage: nodemon present (devDependency)? ---"
docker run --rm lab3-node:single ls node_modules | grep nodemon

echo "--- Multi stage: nodemon present? ---"
docker run --rm lab3-node:multi ls node_modules | grep nodemon 2>/dev/null || echo "nodemon NOT present — devDependencies removed"

echo "--- Multi stage: express present (production dep)? ---"
docker run --rm lab3-node:multi ls node_modules | grep express

echo "--- Multi stage: app works? ---"
docker run --rm lab3-node:multi node -e "const express = require('express'); console.log('Express works:', express.version)"
```

Count node_modules in each:

```bash
echo "--- node_modules count single stage ---"
docker run --rm lab3-node:single ls node_modules | wc -l

echo "--- node_modules count multi stage ---"
docker run --rm lab3-node:multi ls node_modules | wc -l
```

---

## Lab 3 — Step 6: Run Multi-Stage and Test

```bash
docker run -d \
  --name node-multi \
  -p 3000:3000 \
  -e APP_VERSION=1.0.0 \
  -e APP_ENV=production \
  lab3-node:multi

sleep 3
docker logs node-multi
curl http://localhost:3000/
curl http://localhost:3000/info
curl http://localhost:3000/health
```

---

## Lab 3 — Step 7: Push to Docker Hub

```bash
docker tag lab3-node:multi $DOCKERHUB_USERNAME/lab3-node:v1.0.0
docker tag lab3-node:multi $DOCKERHUB_USERNAME/lab3-node:latest

docker push $DOCKERHUB_USERNAME/lab3-node:v1.0.0
docker push $DOCKERHUB_USERNAME/lab3-node:latest
```

Delete and pull back:

```bash
docker stop node-multi && docker rm node-multi
docker rmi $DOCKERHUB_USERNAME/lab3-node:v1.0.0
docker rmi $DOCKERHUB_USERNAME/lab3-node:latest

docker pull $DOCKERHUB_USERNAME/lab3-node:v1.0.0
```

Run from pulled image:

```bash
docker run -d \
  --name node-app \
  -p 3000:3000 \
  --restart unless-stopped \
  $DOCKERHUB_USERNAME/lab3-node:v1.0.0

sleep 3
curl http://localhost:3000/info
```

Open in browser:

```
http://YOUR_VM_PUBLIC_IP:3000/
http://YOUR_VM_PUBLIC_IP:3000/info
```

Open firewall if needed:

```bash
sudo ufw allow 3000/tcp
```

---

## Lab 3 — Results Summary

| Metric | Single Stage | Multi Stage |
|--------|-------------|-------------|
| Image size | _______ MB | _______ MB |
| nodemon present | Yes | No |
| node_modules count | _______ | _______ |
| App works | Yes | Yes |

---

---

# Final Session Summary

## What You Did Today

| | Lab 1 Python | Lab 2 Java | Lab 3 Node.js |
|-|-------------|-----------|--------------|
| Application | Flask REST API | Spring Boot REST API | Express REST API |
| Port | 5000 | 8080 | 3000 |
| Single stage base | python:3.11 | eclipse-temurin:17-jdk | node:20 |
| Multi stage base | python:3.11-slim | eclipse-temurin:17-jre | node:20-alpine |
| What stage 1 removes | pip, build cache | JDK, Maven, source | devDependencies |
| On Docker Hub | Yes | Yes | Yes |
| Accessed via public IP | Yes | Yes | Yes |

## The Complete Workflow You Practiced Three Times

```
Write App Code
      ↓
Write Dockerfile (single stage)
      ↓
docker build → test locally
      ↓
Write Dockerfile (multi stage)
      ↓
docker build → compare sizes
      ↓
docker tag → prepare for registry
      ↓
docker push → image live on Docker Hub
      ↓
docker rmi → delete local copy
      ↓
docker pull → pull from Docker Hub
      ↓
docker run -p → run container
      ↓
curl / browser → access via public IP
```

## Key Numbers to Remember

| Language | Typical single stage | Typical multi stage | Reduction |
|----------|---------------------|---------------------|-----------|
| Python | 900 MB | 150 MB | 6x |
| Java | 700 MB | 250 MB | 3x |
| Node.js | 400 MB | 150 MB | 2.5x |
| Go | 850 MB | 15 MB | 57x |

## The Three Rules Every Developer Must Know

```
Rule 1: One Dockerfile can have multiple stages
Rule 2: COPY --from is the ONLY bridge between stages
Rule 3: The final image contains ONLY what Stage 2 explicitly has
        Everything in Stage 1 that is not copied is GONE
```

---

## Cleanup — Remove All Lab Images

Run this after the session to free disk space:

```bash
docker stop $(docker ps -aq) 2>/dev/null
docker rm $(docker ps -aq) 2>/dev/null
docker rmi lab1-python:single lab1-python:multi 2>/dev/null
docker rmi lab2-java:single lab2-java:multi 2>/dev/null
docker rmi lab3-node:single lab3-node:multi 2>/dev/null
docker system prune -f
docker system df
```

---

*End of Session 3 — Next Session: Docker Networking*
