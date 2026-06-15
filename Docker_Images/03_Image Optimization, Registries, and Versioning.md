# Image Optimization, Registries, and Versioning

---

## Where We Are in the Story

In Part 1, we built images from Dockerfiles and understood layers.

In Part 2, we learned to operate containers and manage images with commands.

Now we answer the question every engineer eventually asks: my image works, but it is 2 GB. How do I make it 50 MB? And once it is optimized, where do I store it so my team and my production systems can use it reliably?

Part 3 covers the production concerns — optimization strategies that reduce image size by 10x to 100x, the registry ecosystem for storing and distributing images, and versioning practices that prevent the kind of incidents that end careers.

---

# Section 1: Why Image Size Matters

## Concept

Before we optimize, we need to understand exactly why image size matters. It is not just about storage cost.

**Build time.** Every byte that does not need to be in the image is a byte that does not need to be built. Smaller images mean faster builds, which means faster developer feedback loops and faster CI/CD pipelines.

**Transfer time.** Images are pulled from registries to servers. Every time you deploy, the image must be transferred. A 2 GB image on a slow connection can block a deployment for minutes. A 50 MB image transfers in seconds. In Kubernetes environments where many nodes pull the same image simultaneously, this difference is amplified.

**Startup time.** When a container starts, the image layers must be available on the node. If the image is not cached, it must be pulled first. Smaller images mean faster cold starts, which matters for auto-scaling scenarios where new nodes must spin up quickly to handle load.

**Attack surface.** Every binary, library, and tool included in an image is a potential attack vector. A base image that includes curl, wget, bash, apt, and build tools gives an attacker many options if they compromise the container. A minimal image with only the application binary and its runtime dependencies gives an attacker almost nothing.

**Storage cost.** At scale, image storage costs are real. A registry storing hundreds of versions of dozens of services, each multiple gigabytes, costs money. Optimized images reduce this cost significantly.

## The Problem in Practice

A developer starting from a comfortable base image and installing tools the way they would on a development server can easily produce a 2 GB image for an application that could run in 20 MB.

Here is where the bloat typically comes from:

| Source | Typical size | Necessary? |
|--------|-------------|------------|
| Full Ubuntu base | 77 MB | Often no |
| Build tools (gcc, make) | 200+ MB | Only at build time |
| Package manager cache (apt lists) | 50-100 MB | No |
| pip download cache | Variable | No |
| Development dependencies | Variable | No |
| Test files | Variable | No |
| .git directory in build context | Variable | No |
| node_modules in build context | 100+ MB | Only needed in image, not context |

The optimization strategies below address each of these sources.

---

# Section 2: Strategy 1 — Choose the Right Base Image

## Concept

The base image is the foundation of your image. Everything else is built on top of it. Choosing the wrong base image is the single biggest contributor to bloated images.

Most tutorials tell you to start with ubuntu:latest or debian:latest because they are familiar. But familiarity is not a good reason. Choose the smallest base image that provides everything your application needs.

## Base Image Comparison

| Base Image | Size | What it includes | Best for |
|------------|------|-----------------|----------|
| ubuntu:22.04 | 77 MB | Full Ubuntu userland, apt | Development, complex apps needing many system tools |
| debian:12-slim | 74 MB | Minimal Debian, apt | Apps needing Debian packages but not Ubuntu-specific |
| alpine:3.18 | 7 MB | BusyBox utilities, musl libc, apk | Most production workloads |
| distroless/python3 | 20-40 MB | Python runtime only, no shell | Python apps in production |
| distroless/static | 2 MB | SSL certificates, timezone data only | Statically compiled binaries |
| scratch | 0 MB | Nothing at all | Static Go binaries, minimal containers |

## Alpine Linux

Alpine is the most common choice for production images. It is 7 MB compared to Ubuntu's 77 MB. It uses musl libc instead of glibc, which is a different implementation of the C standard library.

The musl vs glibc difference matters for some applications. Most standard programs work identically on Alpine. Some programs compiled specifically for glibc will fail on Alpine. Test your application on Alpine before committing to it for production.

Alpine uses apk as its package manager:

```
FROM alpine:3.18
RUN apk add --no-cache python3 py3-pip
```

The --no-cache flag tells apk not to cache the package index, saving additional space.

## Distroless Images

Distroless images, maintained by Google, contain only the application runtime and its dependencies. They have no shell (no bash, no sh), no package manager, no system utilities.

This is the most secure option for production. An attacker who compromises a distroless container has no tools to work with — they cannot run commands, cannot install software, cannot explore the filesystem interactively.

```
FROM gcr.io/distroless/python3
COPY app.py /app/app.py
CMD ["/app/app.py"]
```

The tradeoff is debuggability. You cannot exec into a distroless container and run commands because there is no shell. Use these in production where stability and security matter most, and keep a debug image with a shell for troubleshooting.

## Language-Specific Slim Images

Most official language images offer a slim variant that strips unnecessary components:

```
FROM python:3.11          # 900 MB — includes compilers, build tools
FROM python:3.11-slim     # 125 MB — just the Python runtime
FROM python:3.11-alpine   # 50 MB — Python on Alpine
```

Always check for slim and alpine variants before using the default.

## Lab

Compare sizes across base images:

```
docker pull ubuntu:22.04
docker pull debian:12-slim
docker pull alpine:3.18
docker pull python:3.11
docker pull python:3.11-slim
docker pull python:3.11-alpine

docker images | grep -E "ubuntu|debian|alpine|python"
```

Sort by size and see the difference directly.

---

# Section 3: Strategy 2 — Multi-Stage Builds

## Concept

Multi-stage builds are the single most powerful image optimization technique. They solve a fundamental problem: the tools needed to build an application are much larger than the tools needed to run it.

A compiled Go application is a single binary of a few megabytes. To build that binary, you need the Go compiler, which is hundreds of megabytes. Without multi-stage builds, you either include the compiler in your production image (wasteful and insecure) or you manually manage a separate build environment.

Multi-stage builds let you use multiple FROM instructions in one Dockerfile. Each FROM starts a new build stage with a fresh filesystem. You can copy artifacts from one stage to another. The final image contains only what you explicitly copied into the last stage.

The build tools, source code, intermediate files, and everything else from earlier stages are completely discarded. They never enter the final image.

## Example 1: Go Application

```
# Stage 1: Build
FROM golang:1.21 AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /app/server .

# Stage 2: Run
FROM alpine:3.18
RUN apk add --no-cache ca-certificates
COPY --from=builder /app/server /app/server
EXPOSE 8080
CMD ["/app/server"]
```

What happens:

Stage 1 (builder) uses the full golang:1.21 image (850 MB). It downloads dependencies and compiles the application. The output is a single binary at /app/server.

Stage 2 starts fresh from alpine:3.18 (7 MB). It copies only the compiled binary from the builder stage. Nothing else from stage 1 enters stage 2.

Result: Final image is 12 MB instead of 850 MB. A 70x reduction.

## Example 2: Python Application

Python does not compile to a binary, but you can still use multi-stage builds to separate installation from runtime:

```
# Stage 1: Install dependencies
FROM python:3.11-slim AS dependencies
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# Stage 2: Runtime
FROM python:3.11-slim AS runtime
WORKDIR /app
COPY --from=dependencies /root/.local /root/.local
COPY app.py .
ENV PATH=/root/.local/bin:$PATH
EXPOSE 8080
CMD ["python3", "app.py"]
```

This separates pip install from the final image layers. If requirements.txt has not changed, the first stage is fully cached. The runtime stage only rebuilds when code changes.

## Example 3: Node.js Application

```
# Stage 1: Build
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Production
FROM node:20-alpine AS production
WORKDIR /app
ENV NODE_ENV=production
COPY package*.json ./
RUN npm ci --only=production
COPY --from=build /app/dist ./dist
EXPOSE 3000
USER node
CMD ["node", "dist/index.js"]
```

The first stage installs all dependencies (including devDependencies) and runs the build. The second stage installs only production dependencies and copies only the built output. Test files, source maps, dev tools, and source code are not in the production image.

## Targeting Specific Stages

You can build a specific stage instead of the final stage. This is useful for running tests or debugging the build process:

```
docker build --target builder -t myapp:builder .
docker build --target production -t myapp:prod .
```

## Lab

Create a multi-stage Dockerfile for the Python application:

```
cat > /home/myapp/Dockerfile.optimized << 'EOF'
FROM python:3.11-slim AS base
WORKDIR /app
COPY requirements.txt .

FROM base AS dependencies
RUN pip install --no-cache-dir -r requirements.txt

FROM base AS production
COPY --from=dependencies /usr/local/lib/python3.11/site-packages /usr/local/lib/python3.11/site-packages
COPY app.py .
RUN groupadd -r appgroup && useradd -r -g appgroup appuser
USER appuser
EXPOSE 8080
CMD ["python3", "app.py"]
EOF
```

Build and compare:

```
docker build -t myapp:original .
docker build -f Dockerfile.optimized -t myapp:optimized .
docker images | grep myapp
```

---

# Section 4: Strategy 3 — Minimize Layers and Clean Up in the Same RUN

## Concept

Every RUN instruction creates a layer. A layer captures the filesystem state at the end of the instruction. This means that even if you install a package and then delete the package manager cache in separate RUN instructions, the layer created by the install instruction still contains the cache — it is in a lower, read-only layer.

This is a common mistake:

```
# Wrong — cache files preserved in layer 1 even though layer 2 deletes them
RUN apt-get update && apt-get install -y python3
RUN apt-get clean && rm -rf /var/lib/apt/lists/*
```

Layer 1 is frozen with the cache files in it. Layer 2 adds a delete operation on top. Docker sees the deletion, but the original files are still in layer 1 on disk. The image contains both the files and their deletion, so the cache still occupies disk space.

The correct approach — install and clean in the same RUN instruction so the cache never exists in any layer:

```
# Correct — single RUN instruction, cache files never exist in any layer
RUN apt-get update && \
    apt-get install -y python3 pip && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

## Combining Related Commands

Group logically related commands into single RUN instructions:

```
# Before: 4 layers
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y git
RUN apt-get clean

# After: 1 layer
RUN apt-get update && \
    apt-get install -y curl git && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

But do not go too far. Combining unrelated commands makes the Dockerfile harder to read and reduces cache effectiveness. Only combine commands that logically belong together and that you would always run as a group.

---

# Section 5: Strategy 4 — Use .dockerignore

## Concept

When you run docker build, Docker sends the entire build context (the directory you specify) to the daemon. The daemon cannot see your local filesystem except through this build context. If the build context contains unnecessary files, those files are transmitted to the daemon on every build, even if they are never used by any Dockerfile instruction.

A 200 MB node_modules directory, a 1 GB .git history, or a directory full of log files all add to the build context if they are present and not excluded.

The .dockerignore file works exactly like .gitignore. It lists patterns of files and directories that should not be included in the build context.

## A Complete .dockerignore File

Create .dockerignore in the same directory as your Dockerfile:

```
# Version control
.git
.gitignore
.gitattributes

# CI/CD and editor files
.github
.gitlab-ci.yml
.circleci
Jenkinsfile
.vscode
.idea
*.swp

# Test and documentation
tests/
test/
spec/
docs/
README.md
*.md

# Dependency directories
node_modules/
vendor/
__pycache__/
*.pyc
*.pyo
.venv/
venv/
env/

# Build outputs
dist/
build/
*.egg-info/
target/

# Logs and temporary files
*.log
logs/
tmp/
temp/

# Docker files themselves (optional)
Dockerfile*
docker-compose*
.dockerignore

# Environment files
.env
.env.*
*.local
```

## Lab

Check your build context size before and after .dockerignore:

Initialize a git repository to simulate a real project:

```
cd /home/myapp
git init
git add .
git commit -m "initial commit"
```

Build and note the context size in the output:

```
docker build -t myapp:before .
```

The first line of output shows: Sending build context to Docker daemon X.XXkB

Create .dockerignore:

```
cat > .dockerignore << 'EOF'
.git
*.md
__pycache__
*.pyc
.venv
venv
tests/
EOF
```

Build again:

```
docker build -t myapp:after .
```

Compare the context sizes. Even in a small project the difference is visible. In real projects with node_modules or .git histories, the difference can be hundreds of megabytes.

---

# Section 6: Strategy 5 — Pin Versions and Run as Non-Root

## Concept

These two practices are not about size optimization. They are about reliability and security. But they belong in every production Dockerfile.

## Pin Specific Versions

Using latest or unspecified tags means your builds are not reproducible. A build that works today might fail tomorrow because the base image changed.

```
# Unpredictable — base image changes without warning
FROM python:latest
RUN pip install flask

# Reproducible — exact versions specified
FROM python:3.11.7-slim-bookworm
RUN pip install flask==3.0.0
```

For production images, pin:
- The base image to a specific version tag
- All package installations to specific versions
- All pip/npm/apt packages to exact versions

## Run as Non-Root

By default, everything in a container runs as root. Root inside a container is still a significant security risk even with namespaces. An application running as root can read any file in the container, modify system configurations, and in some misconfiguration scenarios can affect the host.

Always create and use a non-root user:

```
RUN groupadd --system appgroup && \
    useradd --system --gid appgroup --no-create-home appuser
USER appuser
```

Or in Alpine:

```
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

Place the USER instruction near the end of the Dockerfile, after all installation steps (which often require root) and before CMD/ENTRYPOINT.

---

# Section 7: The Optimized Dockerfile — Before and After

## Concept

Let us take a realistic before state and apply every strategy to arrive at a production-ready Dockerfile.

## Before — Developer's First Attempt

```
FROM ubuntu:latest

RUN apt-get update
RUN apt-get install -y python3 python3-pip curl wget vim

COPY . /app
WORKDIR /app

RUN pip install -r requirements.txt

EXPOSE 8080

CMD python3 app.py
```

Problems:
- ubuntu:latest — 77 MB base, no version pin
- Two separate RUN for apt — apt cache preserved as a layer
- Installing curl, wget, vim — not needed at runtime
- COPY . /app — copies everything including .git, __pycache__, logs
- pip install after COPY . — cache invalidated on any file change
- Running as root
- No cleanup of pip cache

Estimated size: 600-800 MB

## After — Production-Ready

```
FROM python:3.11.7-slim-bookworm

LABEL maintainer="rahul@example.com"
LABEL version="1.0"

WORKDIR /app

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

RUN groupadd --system appgroup && \
    useradd --system --gid appgroup --no-create-home appuser && \
    chown -R appuser:appgroup /app

USER appuser

EXPOSE 8080

CMD ["python3", "app.py"]
```

Changes made:
- python:3.11.7-slim-bookworm — 125 MB, version-pinned, slim variant
- Single ENV instruction with all variables
- requirements.txt copied before app code — pip install cached
- --no-cache-dir — pip does not write cache to disk
- Non-root user created and applied
- Proper JSON array syntax for CMD
- No unnecessary tools installed

Estimated size: 150-180 MB

With python:3.11-alpine as base instead: approximately 60-80 MB.

## Lab — Measuring the Difference

Build both versions and compare:

```
docker build -f Dockerfile.before -t myapp:before .
docker build -f Dockerfile.after -t myapp:after .
docker images | grep myapp
```

The size difference will be dramatic even in this simple example. On a real application with many dependencies, the gap is larger.

---

# Section 8: Docker Registries

## Concept

A registry is a service that stores and distributes Docker images. It is the bridge between your local machine (or CI/CD system) and production servers.

The registry workflow:

```
Developer machine          Registry              Production server
      |                       |                        |
docker build            images stored            docker pull
docker tag       ->     with versions     ->     docker run
docker push             and metadata
```

## Registry Concepts

**Registry** — the server. Example: docker.io, registry.example.com, 123456789.dkr.ecr.ap-south-1.amazonaws.com

**Repository** — a collection of related images within a registry. Example: nginx, myteam/myapp, library/ubuntu

**Tag** — a version identifier within a repository. Example: latest, 1.25.3, stable, v2.0.0-alpine

**Full image reference** — combines all three: registry/repository:tag

```
docker.io/library/nginx:1.25.3
123456789.dkr.ecr.ap-south-1.amazonaws.com/myteam/myapp:v2.0.0
ghcr.io/myorg/myservice:stable
```

When you write just nginx, Docker expands it to docker.io/library/nginx:latest.

## Popular Registries

### Docker Hub (docker.io)

The default public registry. Every official image (nginx, ubuntu, python, redis) lives here under the library namespace. Free tier allows unlimited public images and limited private repositories.

```
docker pull nginx              # same as docker pull docker.io/library/nginx:latest
docker pull myusername/myapp   # user-namespaced image
```

### Amazon Elastic Container Registry (ECR)

AWS-native private registry. Integrates with IAM for authentication. Images are stored in the same region as your ECS or EKS workloads, minimizing transfer costs and latency.

```
# Authenticate
aws ecr get-login-password --region ap-south-1 | \
  docker login --username AWS --password-stdin \
  123456789.dkr.ecr.ap-south-1.amazonaws.com

# Push
docker tag myapp:1.0 123456789.dkr.ecr.ap-south-1.amazonaws.com/myapp:1.0
docker push 123456789.dkr.ecr.ap-south-1.amazonaws.com/myapp:1.0
```

### Google Artifact Registry

GCP-native registry. Successor to Google Container Registry. Supports multi-format artifacts (Docker images, npm packages, Maven artifacts) in one place.

### GitHub Container Registry (ghcr.io)

Integrated with GitHub Actions and repositories. Public or private. Authentication uses GitHub personal access tokens.

```
docker login ghcr.io -u USERNAME --password-stdin <<< $GITHUB_TOKEN
docker push ghcr.io/myorg/myapp:1.0
```

### Private Self-Hosted Registry

For organizations that cannot use cloud registries. Options include Harbor (open source, feature-rich, scanning, RBAC) and the basic Docker Registry (open source, minimal).

## Authentication

Always use tokens instead of passwords for registry authentication.

```
docker login                                    # Docker Hub, prompts for credentials
docker login registry.example.com              # private registry
docker login -u AWS -p $(aws ecr get-login-password) 123456789.dkr.ecr.ap-south-1.amazonaws.com
```

Credentials are stored in ~/.docker/config.json. In CI/CD pipelines, use credential helpers or environment variables instead of storing credentials in files.

## Lab — Working with a Registry

Tag your local image for Docker Hub:

```
docker tag myapp:1.0 yourdockerhubusername/myapp:1.0
```

Log in to Docker Hub:

```
docker login
```

Push the image:

```
docker push yourdockerhubusername/myapp:1.0
```

Delete the local copy and pull it back:

```
docker rmi yourdockerhubusername/myapp:1.0
docker pull yourdockerhubusername/myapp:1.0
docker run -d -p 8080:8080 yourdockerhubusername/myapp:1.0
```

You have completed the full build-push-pull-run cycle.

---

# Section 9: Image Tags and Versioning

## Concept

The way you version and tag images is one of the most consequential decisions you make in a Docker environment. Bad tagging practices cause production incidents. Good tagging practices give you reliability, traceability, and safe rollbacks.

## The latest Problem

latest is the default tag when no tag is specified. It is also the most misunderstood.

latest does NOT mean most recent version. It means whatever was most recently tagged as latest. The latest tag on an image can point to any version — there is no mechanism that automatically keeps it updated.

The production problem with latest:

```
# You deploy with latest
docker run -d myapp:latest
# The image is v1.2 because that's what was latest when you deployed

# A week later, your coworker builds and pushes v2.0 tagged as latest
# Now "latest" means v2.0

# You scale up — new servers pull "latest"
docker run -d myapp:latest  # now running v2.0, different from existing servers

# You now have a mixed-version deployment
# Some servers running v1.2, some running v2.0
# Bugs appear that only affect some users
# You cannot reproduce them because your environment is inconsistent
```

Rule: never use latest in production. Always use a specific, immutable version tag.

## Semantic Versioning

Semantic versioning (SemVer) is the standard approach for versioning software. The format is MAJOR.MINOR.PATCH.

```
MAJOR — breaking changes. Old clients may not work with new version.
MINOR — new features. Backwards compatible. Old clients still work.
PATCH — bug fixes only. No new features, no breaking changes.
```

Examples:
- 2.0.0 — major release, breaking changes from 1.x
- 2.1.0 — new feature added to 2.0
- 2.1.7 — seventh bug fix release on 2.1

For Docker images:

```
docker tag myapp:2.1.7 registry.example.com/myapp:2.1.7
docker tag myapp:2.1.7 registry.example.com/myapp:2.1    # track latest patch
docker tag myapp:2.1.7 registry.example.com/myapp:2      # track latest minor
```

In production, always reference the full version: 2.1.7. Let the higher-level tags (2.1, 2) float naturally but never deploy using them.

## Git-Based Tags

Tagging images with the Git commit SHA ties every image to the exact code that produced it.

```
GIT_SHA=$(git rev-parse --short HEAD)
docker build -t myapp:${GIT_SHA} .
```

With the commit SHA in the image tag, you can always answer: what code is running in production? You check the running container's image tag, look up that commit in Git, and see exactly what code was deployed.

This is invaluable for debugging production issues.

## Environment Tags

Tag images with the environment they are deployed to:

```
docker tag myapp:2.1.7 myapp:2.1.7-dev
docker tag myapp:2.1.7 myapp:2.1.7-staging
docker tag myapp:2.1.7 myapp:2.1.7-prod
```

Or tag with base image variant for multi-architecture awareness:

```
docker tag myapp:2.1.7 myapp:2.1.7-alpine
docker tag myapp:2.1.7 myapp:2.1.7-debian
```

## A Complete Versioning Workflow

This is Maya's complete workflow from the source document, refined for production:

```
# Build with version and SHA
VERSION="v2.1.7"
GIT_SHA=$(git rev-parse --short HEAD)

docker build -t myapp:${VERSION} .

# Tag with multiple identifiers
docker tag myapp:${VERSION} myapp:${GIT_SHA}
docker tag myapp:${VERSION} myapp:stable
docker tag myapp:${VERSION} registry.example.com/myteam/myapp:${VERSION}
docker tag myapp:${VERSION} registry.example.com/myteam/myapp:${GIT_SHA}

# Push all tags
docker push registry.example.com/myteam/myapp:${VERSION}
docker push registry.example.com/myteam/myapp:${GIT_SHA}
docker push registry.example.com/myteam/myapp:stable
```

## Safe Rollback

When versioning is done correctly, rollback is a single command:

```
# Current production deployment
docker run -d --name myapp registry.example.com/myteam/myapp:v2.1.7

# v2.1.7 has a critical bug. Rollback to v2.1.6:
docker stop myapp
docker rm myapp
docker run -d --name myapp registry.example.com/myteam/myapp:v2.1.6
```

Because v2.1.6 is a specific, immutable tag pointing to the exact image that ran in production before, the rollback gives you precisely the same environment as before. No surprises.

## Tagging Best Practices Summary

| Practice | Reason |
|----------|--------|
| Never deploy with latest | latest is mutable, unreliable in production |
| Always use SemVer in production | Immutable, communicates change impact |
| Tag with Git SHA | Traceable to exact code commit |
| Never overwrite existing version tags | Version tags should be immutable once pushed |
| Use floating tags (stable, 2.1) separately | Useful for non-production environments, never production |
| Include architecture in tag when needed | myapp:2.1.7-amd64, myapp:2.1.7-arm64 |

## Lab — Versioning Practice

Build and tag with proper versioning:

```
VERSION="v1.0.0"
GIT_SHA=$(git log --format="%h" -n 1)

docker build -t myapp:${VERSION} /home/myapp/

docker tag myapp:${VERSION} myapp:${GIT_SHA}
docker tag myapp:${VERSION} myapp:stable
docker tag myapp:${VERSION} myapp:1.0

docker images myapp
```

You will see one image with four different tags. All four point to the same image ID. No extra storage used.

Change a line of code in app.py, rebuild as v1.0.1:

```
VERSION="v1.0.1"
GIT_SHA=$(git log --format="%h" -n 1)

docker build -t myapp:${VERSION} /home/myapp/
docker tag myapp:${VERSION} myapp:${GIT_SHA}
docker tag myapp:${VERSION} myapp:stable  # stable now points to 1.0.1
docker tag myapp:${VERSION} myapp:1.0

docker images myapp
```

Notice that stable now points to v1.0.1. The v1.0.0 tag still points to the original image. Both are available for rollback or inspection at any time.

---

# Section 10: Key Takeaways — Part 3

## Optimization Checklist

Before shipping any image to production, verify:

| Check | Question to ask |
|-------|----------------|
| Base image | Am I using the smallest base that works? |
| Multi-stage | Have I separated build tools from runtime? |
| Layer cleanup | Am I cleaning caches in the same RUN instruction? |
| Layer order | Are frequently changing instructions last? |
| .dockerignore | Have I excluded unnecessary files from the build context? |
| Version pins | Are all versions pinned explicitly? |
| Non-root user | Is the container running as a non-root user? |
| Unnecessary tools | Are there tools in the image that are not needed at runtime? |

## Registry Decision Guide

| Situation | Recommended Registry |
|-----------|---------------------|
| Learning and public images | Docker Hub |
| AWS-based production workloads | Amazon ECR |
| GCP-based production workloads | Google Artifact Registry |
| GitHub-based CI/CD | GitHub Container Registry |
| On-premises, air-gapped environments | Harbor (self-hosted) |
| Azure-based workloads | Azure Container Registry |

## Versioning Rules

1. Never use latest in production. Ever.
2. Always use SemVer — MAJOR.MINOR.PATCH.
3. Tag every image with its Git commit SHA.
4. Never overwrite a version tag once it has been pushed.
5. Keep previous versions in the registry for rollback.
6. In production, always reference the full version (2.1.7, not 2.1 or 2).

---

# Complete Session 2 Summary

## What We Covered

| Part | Topics | Core Output |
|------|--------|-------------|
| Part 1 | Dockerfile instructions, build process, intermediate containers, layers, caching, manifests, disk storage | You can write and understand any Dockerfile and explain exactly how an image is built |
| Part 2 | Unified filesystem, docker run flags, container operations, image commands | You can operate containers confidently from day one to day one hundred |
| Part 3 | Optimization strategies, base images, multi-stage builds, registries, versioning | You can build production-grade images and manage them across environments |

## The Complete Journey

```
Developer writes code
        |
        v
Writes Dockerfile with pinned versions, non-root user, .dockerignore
        |
        v
docker build with multi-stage — build tools stay in stage 1, only runtime in final
        |
        v
docker tag with SemVer, Git SHA, and environment tags
        |
        v
docker push to private registry (ECR, GCR, ghcr.io)
        |
        v
Production server: docker pull specific version, docker run
        |
        v
Monitor with docker stats, docker logs, docker inspect
        |
        v
Rollback: docker stop, docker rm, docker run previous-version
```

---

*End of Session 2*

*Session 3: Docker Networking — Bridge, Host, Overlay, and Container Communication*
