# Docker Security Lab 1: Image Hardening — From Vulnerable to Production-Ready

> **Session Type:** Instructor-Led Live Lab + Student Runbook  
> **Difficulty:** Intermediate  
> **Estimated Time:** 90–120 minutes  
> **Prerequisites:** Basic Docker knowledge, Linux command line, sudo access

---

## 🏰 The Castle Analogy — Before We Begin

Before you build a castle, you choose your materials carefully.

A castle built from rotting wood with open windows, no gatehouse, and a treasure map nailed to the front door is not a castle — it's an invitation.

Most Docker images in the wild are exactly that. They start from bloated base images packed with unnecessary tools. They run everything as root. They bake secrets directly into their layers. They have no health checks.

**Today we fix that — layer by layer.**

We will start with the most dangerous possible Dockerfile, deliberately broken in every way that matters. Then we will harden it through eight concrete steps, scanning and measuring our improvement at each stage.

By the end, you will have:
- A 96% reduction in vulnerabilities
- An 88% reduction in image size
- Zero secrets baked into the image
- A container that runs as a non-root user with a health check

---

## Environment Setup

### 🧠 Context and Concept

Before we write a single Dockerfile, we need three tools:

| Tool | Purpose |
|---|---|
| **Trivy** | Scans Docker images for known CVEs (vulnerabilities) in OS packages and dependencies |
| **Docker Bench Security** | Audits your Docker host and running containers against CIS Docker Benchmark best practices |
| **Falco** | Runtime threat detection (used in Lab 2 — installed now while we have the setup context) |

**Why scan images?** Every package you include is a potential vulnerability. A single `ubuntu:20.04` base image carries hundreds of packages you never asked for — many with known exploits. Trivy makes this visible.

---

### Install Required Tools

**Update your system first:**

```bash
sudo apt update && sudo apt upgrade -y
```

**Install Trivy (vulnerability scanner):**

```bash
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | \
  sudo sh -s -- -b /usr/local/bin
```

**Verify Trivy installed:**

```bash
trivy --version
```

> ✅ Expected: version output like `Version: 0.5x.x`

**Install Docker Bench Security:**

```bash
cd ~
git clone https://github.com/docker/docker-bench-security.git
```

> We will use Docker Bench at the end of the lab to validate our hardened image.

**Install Falco (for Lab 2 — install now while setting up):**

```bash
# Remove any old Falco repo entries
sudo rm -f /etc/apt/sources.list.d/falcosecurity.list

# Add GPG key
curl -fsSL https://falco.org/repo/falcosecurity-packages.asc | \
  sudo gpg --dearmor -o /usr/share/keyrings/falcosecurity.gpg

# Add repository
echo "deb [signed-by=/usr/share/keyrings/falcosecurity.gpg] https://download.falco.org/packages/deb stable main" | \
  sudo tee /etc/apt/sources.list.d/falcosecurity.list

# Update and install
sudo apt update
sudo apt install -y falco
```

**Add your user to the Docker group (avoid needing sudo for docker commands):**

```bash
sudo usermod -aG docker $USER
newgrp docker
```

**Verify Docker works without sudo:**

```bash
docker ps
```

> ✅ Expected: An empty container list with no permission errors.

---

## Step 1: Create the Lab Directory and Vulnerable App

### 🧠 Context and Concept

We are not studying a toy example. The vulnerable app we are about to create mirrors the kind of real-world mistakes found in actual production codebases every day:

- **Hardcoded secrets** — API keys and database passwords baked into source code and Dockerfile ENV statements
- **Running as root** — the default if you forget to add a user
- **Bloated base images** — using `ubuntu:20.04` when you only need Python
- **Secrets in `.env` files copied into the image** — leaks credentials into every image layer permanently

You need to see the problem before you can fix it.

---

**Create the lab directory:**

```bash
mkdir -p ~/docker-security-lab/lab1-image-hardening
cd ~/docker-security-lab/lab1-image-hardening
```

**Create the vulnerable Flask application (`app.py`):**

```bash
cat > app.py << 'EOF'
# app.py
from flask import Flask
import os

app = Flask(__name__)

# Hardcoded secrets — NEVER do this in real code
DATABASE_URL = "postgresql://admin:SuperSecret123@db:5432/mydb"
API_KEY = "sk_live_abc123xyz789"

@app.route('/')
def hello():
    return f"Hello! Running as user: {os.getuid()}"

@app.route('/config')
def config():
    # This endpoint exposes all secrets — demonstrates the risk
    return {
        "database": DATABASE_URL,
        "api_key": API_KEY
    }

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF
```

**Create `requirements.txt`:**

```bash
cat > requirements.txt << 'EOF'
Flask==2.3.0
EOF
```

**Create a fake `.env` file (simulating secrets a developer committed):**

```bash
cat > .env << 'EOF'
DATABASE_URL=postgresql://admin:SuperSecret123@db:5432/mydb
API_KEY=sk_live_abc123xyz789
AWS_ACCESS_KEY=AKIAIOSFODNN7EXAMPLE
AWS_SECRET_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
EOF
```

> ⚠️ These are fake credentials used for learning. Never commit real secrets to any file.

---

**Create the vulnerable Dockerfile (`Dockerfile.vulnerable`):**

```bash
cat > Dockerfile.vulnerable << 'EOF'
# Dockerfile.vulnerable
# WARNING: This Dockerfile is intentionally insecure for learning purposes

# Bad practice #1: Full OS base image — brings hundreds of unnecessary packages
FROM ubuntu:20.04

# Bad practice #2: Installing everything including attack tools
RUN apt-get update && apt-get install -y \
    python3 \
    python3-pip \
    curl \
    wget \
    vim \
    netcat \
    nmap \
    telnet \
    git

# Install Python packages
COPY requirements.txt /app/
RUN pip3 install -r /app/requirements.txt

# Copy application
COPY app.py /app/

# Bad practice #3: Copying a secrets file directly into the image
COPY .env /app/

# Bad practice #4: Hardcoded secrets as environment variables (visible in image history)
ENV DATABASE_URL="postgresql://admin:SuperSecret123@db:5432/mydb"
ENV API_KEY="sk_live_abc123xyz789"

WORKDIR /app

# Bad practice #5: No USER directive = runs as root
CMD ["python3", "app.py"]
EOF
```

---

## Step 2: Build and Scan the Vulnerable Image

### 🧠 Context and Concept

Building and scanning the vulnerable image is not just an exercise in counting CVEs. It is about making the *abstract* risk *concrete*.

When your team says "our images have vulnerabilities," what does that actually mean? It means packages like `openssl`, `glibc`, or `expat` have known exploits with published CVE numbers. Trivy maps your installed packages against the CVE database and tells you exactly what you are carrying.

**Image layers are permanent.** Even if you later run `RUN rm /app/.env` in a subsequent Dockerfile layer, the `.env` file still exists in the earlier layer and can be extracted by anyone with the image. This is a critical concept: secrets in layers cannot be erased retroactively.

---

**Build the vulnerable image:**

```bash
docker build -f Dockerfile.vulnerable -t myapp:vulnerable .
```

> This will take a minute — it is pulling a full Ubuntu image and installing many packages.

**Check the image size:**

```bash
docker images myapp:vulnerable
```

> 📝 Note the size. It will likely be **500 MB or more**.

**Scan for vulnerabilities with Trivy:**

```bash
trivy image myapp:vulnerable
```

> ⏳ First run downloads the vulnerability database — this is normal. Subsequent scans are fast.

**Get just the summary line:**

```bash
trivy image --severity HIGH,CRITICAL myapp:vulnerable | grep -E "Total|CRITICAL|HIGH"
```

> 📝 Record the `Total` number. You will compare this against the hardened image at the end.
> Expected: something like `Total: 200+ (HIGH: 40+, CRITICAL: 10+)`

**Prove secrets are baked into image layers:**

```bash
# View the image build history — secrets are visible in ENV layers
docker history myapp:vulnerable --no-trunc | grep -i "secret\|api_key\|password\|database"
```

```bash
# Extract the image and search all layers for secrets
docker save myapp:vulnerable -o vuln-image.tar
mkdir vuln-extract && tar -xf vuln-image.tar -C vuln-extract
grep -r "SuperSecret" vuln-extract/
grep -r "API_KEY" vuln-extract/
```

> ✅ You will find the secrets in multiple places across the image layers. This is exactly what attackers do when they get access to an image.

**Clean up extraction:**

```bash
rm -rf vuln-extract vuln-image.tar
```

**Run the vulnerable container and prove it runs as root:**

```bash
docker run -d -p 5000:5000 --name vuln-app myapp:vulnerable
sleep 2

# Check the user — this should show root
docker exec vuln-app id
```

> ❌ Expected: `uid=0(root) gid=0(root)` — running as root is dangerous. If this container is compromised, the attacker has root.

**Prove secrets are exposed via the API:**

```bash
curl http://localhost:5000/config
```

> ❌ Expected: JSON response showing the database URL and API key. A misconfigured endpoint + hardcoded secrets = instant credential leak.

**Cleanup:**

```bash
docker rm -f vuln-app
```

**Document what you found:**

```
❌ Image size:              500+ MB
❌ Runs as:                 root (uid=0)
❌ Vulnerabilities (H/C):  200+
❌ Secrets in ENV layers:  Yes — visible in docker history
❌ Secrets in .env layer:  Yes — extractable from image tar
❌ Unnecessary packages:   vim, nmap, netcat, telnet, git
❌ Health check:           None
❌ Base image:             ubuntu:20.04 (full OS)
❌ Multi-stage build:      No
```

---

## Step 3: Apply Multi-Stage Builds — Cut the Fat

### 🧠 Context and Concept

When you build a Python application, you need two different sets of tools:

1. **Build-time tools** — compilers, pip, build dependencies. Needed to install packages.
2. **Runtime tools** — just Python and your application. Needed to run the app.

A naive Dockerfile uses one image for both — and ships all the build tools into production. That's like sending your entire construction crew, scaffolding, cement mixers, and raw materials into the finished building just because they were involved in building it.

**Multi-stage builds** solve this by using two `FROM` statements:
- **Stage 1 (builder):** Install everything needed to build. Does the heavy lifting.
- **Stage 2 (runtime):** Start fresh with a clean minimal image. Copy only the finished artifact from Stage 1.

The final image contains *only* what Stage 2 has — no build tools, no pip, no compilers.

---

**Create the multi-stage Dockerfile:**

```bash
cat > Dockerfile.multistage << 'EOF'
# Dockerfile.multistage
# Stage 1: Builder — has pip and build tools
FROM python:3.11-slim AS builder

WORKDIR /app

# Install dependencies into a user-local directory
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# Stage 2: Runtime — clean slate, no build tools
FROM python:3.11-slim

WORKDIR /app

# Copy ONLY the installed packages from the builder stage
COPY --from=builder /root/.local /root/.local

# Copy the application
COPY app.py .

# Make installed packages accessible
ENV PATH=/root/.local/bin:$PATH

CMD ["python", "app.py"]
EOF
```

**Build and compare sizes:**

```bash
docker build -f Dockerfile.multistage -t myapp:multistage .
docker images | grep myapp
```

> 📝 Compare: `myapp:vulnerable` vs `myapp:multistage`
> Expected: `myapp:multistage` is approximately **150 MB** — already ~70% smaller.

**Scan the multi-stage image:**

```bash
trivy image --severity HIGH,CRITICAL myapp:multistage | grep -E "Total|CRITICAL|HIGH"
```

> 📝 Record the new Total. Fewer packages = fewer vulnerabilities.

---

## Step 4: Switch to Alpine — Shrink Further

### 🧠 Context and Concept

`python:3.11-slim` is already much better than `ubuntu:20.04`. But it is still based on Debian and includes a general-purpose Linux userspace.

**Alpine Linux** is a security-oriented minimal Linux distribution. It uses `musl libc` instead of `glibc` and `busybox` utilities. Its entire OS footprint is about 5 MB. It was built from the ground up with the philosophy: *only ship what you need*.

The tradeoff: some Python packages have C extensions that need to be compiled against `musl libc` rather than `glibc`. The builder stage handles this by including the required Alpine build tools (`gcc`, `musl-dev`) — which again stay out of the final runtime image.

---

**Create the Alpine Dockerfile:**

```bash
cat > Dockerfile.alpine << 'EOF'
# Dockerfile.alpine
# Stage 1: Builder — Alpine with build tools for compiling Python packages
FROM python:3.11-alpine AS builder

WORKDIR /app

# Install C build tools needed for some Python packages
RUN apk add --no-cache gcc musl-dev

# Install Python packages
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# Stage 2: Runtime — minimal Alpine, no build tools
FROM python:3.11-alpine

WORKDIR /app

# Copy only the installed packages
COPY --from=builder /root/.local /root/.local

# Copy the application
COPY app.py .

ENV PATH=/root/.local/bin:$PATH

CMD ["python", "app.py"]
EOF
```

**Build and compare:**

```bash
docker build -f Dockerfile.alpine -t myapp:alpine .
docker images | grep myapp
```

> 📝 Expected:
> ```
> myapp:vulnerable    ~500 MB
> myapp:multistage    ~150 MB
> myapp:alpine         ~60 MB
> ```

**Scan Alpine:**

```bash
trivy image --severity HIGH,CRITICAL myapp:alpine | grep -E "Total|CRITICAL|HIGH"
```

> 📝 Vulnerability count drops again significantly with Alpine.

---

## Step 5: Distroless — The Maximum Security Base Image

### 🧠 Context and Concept

Alpine is minimal. But it still has a shell (`sh`), a package manager (`apk`), and standard Linux utilities. An attacker who gains code execution inside an Alpine container still has tools to work with.

**Distroless images**, created by Google, take minimalism to its logical conclusion: they contain *only* the language runtime and your application. No shell. No package manager. No `ls`, no `cat`, no `bash`. Nothing an attacker can use interactively.

The security impact:
- No shell = attacker cannot run `docker exec ... sh` interactively
- No package manager = attacker cannot install additional tools
- Massively reduced attack surface

The tradeoff: debugging distroless containers requires using a separate debug sidecar image. This is a deliberate design choice — if debugging requires extra steps, the environment is inherently more secure at runtime.

---

**Create the Distroless Dockerfile:**

```bash
cat > Dockerfile.distroless << 'EOF'
# Dockerfile.distroless
# Stage 1: Build on slim (has pip)
FROM python:3.11-slim AS builder

WORKDIR /app

COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# Stage 2: Distroless runtime — no shell, no utilities
FROM gcr.io/distroless/python3-debian11

WORKDIR /app

# Copy installed packages and application from builder
COPY --from=builder /root/.local /root/.local
COPY app.py .

ENV PATH=/root/.local/bin:$PATH
ENV PYTHONPATH=/root/.local/lib/python3.11/site-packages

# Distroless expects the script name directly (no 'python' prefix)
CMD ["app.py"]
EOF
```

**Build:**

```bash
docker build -f Dockerfile.distroless -t myapp:distroless .
docker images | grep myapp
```

> 📝 Expected: `myapp:distroless` at approximately **50 MB**

**Prove there is no shell — this is the key demo:**

```bash
docker run -it myapp:distroless sh
```

> ✅ Expected: `Error: no such file or directory` or similar — *there is no shell to run*. This is a feature, not a bug.

```bash
docker run -it myapp:distroless bash
```

> ✅ Same result — no bash either.

**Scan distroless:**

```bash
trivy image --severity HIGH,CRITICAL myapp:distroless | grep -E "Total|CRITICAL|HIGH"
```

---

## Step 6: Remove Hardcoded Secrets

### 🧠 Context and Concept

Up to this point, we have been shrinking the image and reducing CVEs. But we have not yet fixed the most critical developer mistake: **hardcoded secrets**.

Secrets in a Docker image are a permanent problem. Even if you patch the CVE, rotate the API key, and rebuild the image — the old image with the old key still exists in your registry, in your CI/CD logs, in anyone's local cache. The damage is done.

The correct model: **secrets never enter the image**. They are injected at runtime via:
- Environment variables passed at `docker run`
- Docker Secrets (in Swarm)
- Kubernetes Secrets or a Vault sidecar
- AWS Secrets Manager, Azure Key Vault, etc.

The application reads from the environment. The image itself has no idea what the values will be.

We also fix the `/config` endpoint — it should never return secret values, regardless of where they come from.

---

**Create the secrets-safe application (`app-secure.py`):**

```bash
cat > app-secure.py << 'EOF'
# app-secure.py
from flask import Flask
import os

app = Flask(__name__)

# Read from environment at runtime — image has no knowledge of values
DATABASE_URL = os.getenv('DATABASE_URL', 'not-configured')
API_KEY = os.getenv('API_KEY', 'not-configured')

@app.route('/')
def hello():
    return f"Hello! Running as user: {os.getuid()}"

@app.route('/config')
def config():
    # Never expose secrets via API — return only status
    return {
        "database_configured": DATABASE_URL != 'not-configured',
        "api_key_configured": API_KEY != 'not-configured'
    }

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF
```

**Create the no-secrets Dockerfile:**

```bash
cat > Dockerfile.no-secrets << 'EOF'
# Dockerfile.no-secrets
FROM python:3.11-alpine AS builder

WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

FROM python:3.11-alpine

WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY app-secure.py app.py

ENV PATH=/root/.local/bin:$PATH

# No DATABASE_URL. No API_KEY. No .env. Nothing.
CMD ["python", "app.py"]
EOF
```

**Build:**

```bash
docker build -f Dockerfile.no-secrets -t myapp:no-secrets .
```

**Prove secrets are gone from the image:**

```bash
# Check build history — no secret values should appear
docker history myapp:no-secrets --no-trunc | grep -i "secret\|api_key\|password\|database"
```

> ✅ Expected: No output — nothing found.

```bash
# Extract and search all layers
docker save myapp:no-secrets -o nosecrets-image.tar
mkdir nosecrets-extract && tar -xf nosecrets-image.tar -C nosecrets-extract
grep -r "SuperSecret" nosecrets-extract/ 2>/dev/null && echo "FOUND - BAD!" || echo "Clean - no secrets found"
grep -r "sk_live" nosecrets-extract/ 2>/dev/null && echo "FOUND - BAD!" || echo "Clean - no secrets found"
```

> ✅ Expected: `Clean - no secrets found` for both searches.

**Clean up extraction:**

```bash
rm -rf nosecrets-extract nosecrets-image.tar
```

**Run with secrets injected at runtime (the correct pattern):**

```bash
docker run -d -p 5000:5000 \
  -e DATABASE_URL="postgresql://user:pass@db:5432/mydb" \
  -e API_KEY="sk_live_runtime_key" \
  --name nosecrets-app \
  myapp:no-secrets

sleep 2

# Verify the app works and config endpoint no longer leaks values
curl http://localhost:5000/
curl http://localhost:5000/config
```

> ✅ The `/config` endpoint should now return `{"database_configured": true, "api_key_configured": true}` — no actual values.

**Cleanup:**

```bash
docker rm -f nosecrets-app
```

---

## Step 7: Add a Non-Root User — The Final Hardening Layer

### 🧠 Context and Concept

Everything we have done so far reduces what an attacker can attack. This step reduces what they can *do* if they get in.

By default, every `docker run` runs as `root` inside the container unless the Dockerfile specifies otherwise. Root inside a container is not the same as root on the host — but it is dangerously close. With capabilities like `SYS_ADMIN`, root-in-container can escape to the host.

**The principle of least privilege:** a process should have only the permissions it needs to do its job. A Flask web server needs to read its files and bind to a port — that is all. It does not need root.

We also add a **HEALTHCHECK** at this stage. Docker uses the health check to know if your container is actually serving traffic, not just running. Orchestration tools like Kubernetes and Docker Swarm use health status to route traffic and restart unhealthy containers.

---

**Create the final secure Dockerfile:**

```bash
cat > Dockerfile.secure << 'EOF'
# Dockerfile.secure — Production-ready hardened image
FROM python:3.11-alpine AS builder

WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

FROM python:3.11-alpine

# Create a dedicated non-root system group and user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Copy installed packages with correct ownership
COPY --from=builder --chown=appuser:appgroup /root/.local /root/.local

# Copy application with correct ownership
COPY --chown=appuser:appgroup app-secure.py app.py

# Switch to non-root user — all subsequent operations run as appuser
USER appuser

ENV PATH=/root/.local/bin:$PATH

# Document which port the app uses (informational — does not publish it)
EXPOSE 5000

# Health check — Docker will probe this every 30 seconds
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget -q --spider http://localhost:5000/ || exit 1

CMD ["python", "app.py"]
EOF
```

**Build the final secure image:**

```bash
docker build -f Dockerfile.secure -t myapp:secure .
```

**Verify it runs as non-root:**

```bash
docker run -d -p 5000:5000 --name final-app myapp:secure
sleep 3

# Check the user — this should NOT be root
docker exec final-app id
```

> ✅ Expected: `uid=100(appuser) gid=101(appgroup) groups=101(appgroup)`
> ❌ NOT `uid=0(root)`

**Check the health check is active:**

```bash
docker inspect final-app --format='{{.State.Health.Status}}'
```

> ✅ Expected: `healthy` (may show `starting` for the first 5 seconds)

**Verify the app works:**

```bash
curl http://localhost:5000/
```

**Build the runtime-test tag (used in Lab 2):**

```bash
# Tag the secure image as runtime-test for Lab 2
docker tag myapp:secure myapp:runtime-test
```

> This ensures Lab 2 has the `myapp:runtime-test` image ready to use.

**Cleanup:**

```bash
docker rm -f final-app
```

---

## Step 8: Scan and Compare All Versions

### 🧠 Context and Concept

Measurement is the last step, but arguably the most important one. Security improvements that cannot be quantified cannot be communicated to stakeholders, managers, or auditors.

Run the full comparison scan now.

---

**Scan all images and compare:**

```bash
echo ""
echo "=== VULNERABILITY COMPARISON ==="
echo ""

echo "--- Vulnerable (ubuntu:20.04 base) ---"
trivy image --severity HIGH,CRITICAL --quiet myapp:vulnerable 2>/dev/null | grep "Total"

echo ""
echo "--- Multi-stage (python:3.11-slim) ---"
trivy image --severity HIGH,CRITICAL --quiet myapp:multistage 2>/dev/null | grep "Total"

echo ""
echo "--- Alpine (python:3.11-alpine) ---"
trivy image --severity HIGH,CRITICAL --quiet myapp:alpine 2>/dev/null | grep "Total"

echo ""
echo "--- Secure (hardened alpine + non-root) ---"
trivy image --severity HIGH,CRITICAL --quiet myapp:secure 2>/dev/null | grep "Total"
```

**Compare image sizes:**

```bash
echo ""
echo "=== IMAGE SIZE COMPARISON ==="
docker images | grep myapp | awk '{printf "%-25s %s\n", $1":"$2, $7$8}'
```

**Generate your comparison report:**

```bash
cat > comparison.md << 'EOF'
# Docker Image Hardening — Before vs After

| Metric | Vulnerable | Secure | Improvement |
|---|---|---|---|
| Image Size | ~500 MB | ~60 MB | 88% smaller |
| Vulnerabilities (H/C) | 200+ | <5 | 97%+ reduction |
| Base Image | ubuntu:20.04 | python:3.11-alpine | Minimal OS |
| Runs as Root | Yes (uid=0) | No (appuser) | ✅ Fixed |
| Secrets in Image | Yes (ENV + .env) | No | ✅ Fixed |
| Secrets in Layers | Yes | No | ✅ Fixed |
| Unnecessary Tools | vim, nmap, netcat, telnet | None | ✅ Removed |
| Multi-stage Build | No | Yes | ✅ Applied |
| Health Check | No | Yes | ✅ Added |
| Shell Available | Yes | No (Alpine minimal) | ✅ Hardened |

## Key Principle
Image size ↓  =  Attack surface ↓  =  Vulnerabilities ↓
EOF

cat comparison.md
```

---

## Step 9: Run Docker Bench Security

### 🧠 Context and Concept

Trivy scans for CVEs in packages. Docker Bench checks something different: whether your Docker *host* and *containers* follow the **CIS Docker Benchmark** — a set of industry best practices for securely operating Docker in production.

It checks things like:
- Are containers running as non-root?
- Does the Docker daemon have a TLS certificate?
- Are container capabilities restricted?
- Do containers have memory limits?

Think of it as a configuration audit rather than a vulnerability scan.

---

```bash
cd ~/docker-bench-security
```

**Run the full benchmark:**

```bash
sudo sh docker-bench-security.sh 2>/dev/null | tee bench-results.txt
```

> This takes about 30–60 seconds.

**View results focused on container user checks:**

```bash
grep -E "WARN|PASS|INFO" bench-results.txt | grep -i "user\|root\|privilege" | head -20
```

**Now compare: run the vulnerable container and check it:**

```bash
cd ~/docker-security-lab/lab1-image-hardening

# Start vulnerable container
docker run -d --name test-vuln myapp:vulnerable

# Check user configuration
echo "Vulnerable container USER setting:"
docker inspect test-vuln --format='User: {{.Config.User}}'
```

> ❌ Expected: `User:` (empty — means root)

```bash
# Start secure container
docker run -d --name test-secure myapp:secure

# Check user configuration
echo "Secure container USER setting:"
docker inspect test-secure --format='User: {{.Config.User}}'
```

> ✅ Expected: `User: appuser`

**Cleanup:**

```bash
docker rm -f test-vuln test-secure
```

---

## Lab Summary

### What You Built

| Stage | Dockerfile | Change Made |
|---|---|---|
| Baseline | `Dockerfile.vulnerable` | Ubuntu + root + hardcoded secrets + bloat |
| Stage 3 | `Dockerfile.multistage` | Multi-stage build — no build tools in runtime |
| Stage 4 | `Dockerfile.alpine` | Alpine base — 60 MB instead of 500 MB |
| Stage 5 | `Dockerfile.distroless` | No shell at all — maximum hardening |
| Stage 6 | `Dockerfile.no-secrets` | Secrets removed from image entirely |
| Stage 7 | `Dockerfile.secure` | Non-root user + health check — production-ready |

### The Core Principle

Every package you remove is a vulnerability that cannot be exploited.  
Every tool you exclude is a weapon an attacker cannot use.  
Every secret you keep out of the image is a credential that cannot be stolen from it.

---

## Assignment Questions

Answer these based on what you ran today. Use your terminal output as evidence.

1. **Image size:** Run `docker images | grep myapp` and record all four image sizes. Calculate the percentage reduction from `myapp:vulnerable` to `myapp:secure`. Show your working.

2. **Secrets in layers:** You ran `docker history myapp:vulnerable --no-trunc`. What did you find? Now run the same command against `myapp:secure`. What is different and why?

3. **Root vs non-root:** Run `docker exec <container> id` against both the vulnerable and secure containers. What are the exact UIDs in each case? What is the security impact of running as `uid=0` in a container?

4. **Distroless shell test:** What happened when you ran `docker run -it myapp:distroless sh`? Why is the absence of a shell a security feature rather than a limitation?

5. **Trivy output:** Copy the `Total:` line from your Trivy scan of `myapp:vulnerable` and `myapp:secure`. What categories of vulnerabilities were reduced the most (CRITICAL, HIGH, MEDIUM)?

6. **Design question:** Your team's CI/CD pipeline builds a new Docker image every time code is pushed. A developer suggests that since secrets are injected at runtime (via `-e` flags), it is safe to keep a `.env.example` file with dummy values in the repository as documentation. What is your security assessment of this proposal? What would you recommend instead?

---

## Quick Reference — Key Commands

```bash
# Build
docker build -f Dockerfile.secure -t myapp:secure .

# Scan
trivy image myapp:secure
trivy image --severity HIGH,CRITICAL myapp:secure

# Inspect
docker images | grep myapp                         # Compare sizes
docker history myapp:secure --no-trunc             # View layers
docker inspect <container> --format='{{.Config.User}}'  # Check user
docker exec <container> id                         # Runtime user check

# Secrets search
docker save myapp:secure -o image.tar
tar -xf image.tar -C extract/
grep -r "secret_value" extract/

# Health check
docker inspect <container> --format='{{.State.Health.Status}}'

# Docker Bench
cd ~/docker-bench-security
sudo sh docker-bench-security.sh
```

---

> **End of Lab 1**  
> Next session: Lab 2 — Logging, Auditing, and Runtime Threat Detection with Falco.
> Make sure `myapp:runtime-test` is available: `docker images | grep runtime-test`
