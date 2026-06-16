# Multi-Stage Builds: The Complete Deep Dive
# From the Question That Stopped the Session

---

## Why We Are Here

During the session, multi-stage builds raised more questions than answers.

What actually happens in Phase 1?
How does Phase 2 know what Phase 1 built?
Does the build time stay the same or get faster?
What about startup time, attack surface, storage cost, and transfer time?

These are exactly the right questions. They show the audience is thinking like engineers, not just following commands.

This document answers every one of those questions. We go from first principles to a fully working lab with measurements you can actually see.

---

# Section 1: The Problem Multi-Stage Builds Solve

## Concept

To understand multi-stage builds, you first need to feel the pain they solve.

Think about how software gets built in the real world.

You write Python code. To run it, you need Python. But to install the dependencies, you also need pip. To compile anything, you need gcc. To run tests, you need pytest and a dozen dev tools. To debug, you want curl, vim, netstat.

None of those tools are needed when the application is actually serving requests in production. The application just needs Python and the libraries it imports. That is it.

But here is the trap most engineers fall into.

They install everything — build tools, compilers, test frameworks, debug utilities — in the same image that goes to production. The image starts at 70 MB (the Python base) and ends up at 900 MB after all the tools are in.

That 900 MB image now:
- Takes 3 minutes to build
- Takes 4 minutes to push and pull over the network
- Takes 8 seconds to start cold because the layers need to be loaded
- Contains curl, wget, apt, bash, gcc — every tool an attacker needs
- Costs real money to store in a registry at scale

The solution is to separate the build environment from the runtime environment.

Multi-stage builds are Docker's mechanism for doing this inside a single Dockerfile.

---

# Section 2: The Mental Model — Two Factories

## Concept

Before looking at any code, get this mental model locked in.

Imagine two separate factories.

**Factory 1 — The Builder**

This factory has every tool available. Welding equipment, lathes, drills, testing rigs, quality control stations. Workers go in, raw materials go in, a finished product comes out. The factory itself is large, expensive, and complex.

**Factory 2 — The Shipper**

This factory receives only the finished product from Factory 1. It packages it and ships it to customers. There are no manufacturing tools here. No welding equipment. No drill presses. Just the product and the packaging. The factory is small, clean, and fast.

The customer never sees Factory 1. They receive only what Factory 2 ships them.

This is exactly what multi-stage builds do.

Stage 1 is Factory 1. It has the compiler, the package manager, the build tools. It produces the finished artifact — a compiled binary, an installed set of libraries, a built front-end bundle.

Stage 2 is Factory 2. It receives only the finished artifact from Stage 1. It contains only what is needed to run. Nothing from Stage 1 comes along except what you explicitly copy.

The final Docker image is Stage 2 only. Factory 1 is used and discarded.

---

# Section 3: What Happens in Each Phase — In Exact Detail

## Concept

This is the question that stopped the session. Let us answer it precisely.

### Phase 1 — The Builder Stage

When Docker processes Phase 1, it does everything a normal docker build does.

It creates a fresh container from the base image (golang, python, node — whatever you specify). It runs every instruction inside that container. It captures each instruction as a read-only layer. By the end of Phase 1, there is a complete, fully-built image in memory with all your build tools, all your source code, all your compiled output.

This image exists temporarily. Docker keeps it in memory and on disk only long enough to use it as a source for Phase 2.

Phase 1 has a name. You give it a name using the AS keyword:

```
FROM golang:1.21 AS builder
```

That name — builder — is how Phase 2 refers back to it.

### Phase 2 — The Runtime Stage

Phase 2 starts completely fresh. It does not inherit anything from Phase 1 automatically. It starts from its own base image — usually something minimal like alpine or python:3.11-slim.

The COPY --from instruction is the bridge between the two phases.

```
COPY --from=builder /app/server /server
```

This tells Docker: go into the Phase 1 image (named builder), find the file at /app/server, and copy it into the current Phase 2 image at /server.

That is the only mechanism for transferring anything from Phase 1 to Phase 2. Nothing else crosses over. Not the build tools. Not the source code. Not the package manager. Not the intermediate files. Not the environment variables from Phase 1. Nothing — unless you explicitly copy it.

After Phase 2 is built, Docker discards Phase 1 from the final image. The final image that you push, pull, and run is Phase 2 only.

### What This Means for the Final Image

When you docker run a multi-stage image:
- The container starts from the Phase 2 image
- The Phase 2 filesystem is what the container sees
- Phase 1 does not exist inside the running container at all
- The container has no access to build tools, source code, or anything from Phase 1

Phase 1 is a build-time construct. It exists only to produce artifacts. Phase 2 is the runtime construct. It is what actually runs.

---

# Section 4: The Build Time Question

## Concept

"If we are building two stages, does the build take twice as long?"

This is a fair and important question. The answer is no, and here is why.

### First Build — Yes, It Takes Longer

The very first time you build a multi-stage image from scratch, it takes longer than a single-stage build because Docker must complete both stages. If Stage 1 takes 2 minutes and Stage 2 takes 30 seconds, the first build takes approximately 2 minutes 30 seconds.

### Second Build Onward — Layer Cache Changes Everything

After the first build, every layer from both stages is cached. On subsequent builds, Docker only re-executes layers whose inputs have changed.

In practice, here is what this means:

You change your application code (app.py or main.go). Docker rebuilds only the COPY instruction in Phase 1 that copied the code, then rebuilds from that point forward. The dependency installation layer in Phase 1 (the slow part — pip install, go mod download, npm install) is still cached because requirements.txt did not change.

Result: your second, third, tenth build is fast because the expensive parts are cached.

### The Comparison That Matters

Do not compare multi-stage build time to single-stage build time of the same Dockerfile. Compare it to the alternative: a single-stage image that installs everything together.

A badly ordered single-stage Dockerfile can take 5 minutes on every build because every code change invalidates the dependency installation layer. A well-ordered multi-stage Dockerfile consistently runs in under 30 seconds for code-only changes because the dependency layer is properly cached.

Multi-stage builds, done correctly, are faster in practice because they encourage correct layer ordering.

### The Build Time Breakdown for a Go App

| Scenario | First Build | Code Change | Dependency Change |
|----------|-------------|-------------|-------------------|
| Single-stage naive | 4 minutes | 4 minutes | 4 minutes |
| Multi-stage well ordered | 3 minutes | 15 seconds | 3 minutes |

The "code change" scenario is what happens dozens of times per day in a development team. Multi-stage wins by a large margin in real usage.

---

# Section 5: The Four Concerns — Full Answers

## Startup Time

Every time a container starts cold (the image is not already on the node), Docker must load the image layers from disk before starting the process. The more layers, the more data, the longer this takes.

A 900 MB single-stage image has many large layers. On a Kubernetes node receiving this image for the first time, it can take 30-60 seconds just to pull and load the image before the application even starts.

A 15 MB multi-stage image pulls in under 5 seconds. In an auto-scaling scenario where 10 new pods must start in response to a traffic spike, this difference between 30 seconds and 5 seconds per pod is the difference between absorbing the spike and dropping requests.

In Kubernetes, startup time directly affects:
- Horizontal Pod Autoscaler response time
- Rolling deployment speed
- Node recovery time after a failure

## Attack Surface

Every binary, library, and tool in a container image is a potential attack vector.

A single-stage Python image that includes pip, apt, curl, wget, bash, gcc, and build tools gives an attacker who compromises the container a full toolkit. They can download additional tools with curl. They can install new packages with pip or apt. They can compile exploit code with gcc. They can explore the filesystem with bash.

A multi-stage image that contains only python, your app.py, and your specific libraries gives an attacker almost nothing. If they compromise the container, they find a minimal environment with no package manager to install tools, no compiler to build exploits, often no shell to run commands.

The attack surface reduction is the security argument for multi-stage builds. It is not theoretical — it directly limits what an attacker can do after a successful compromise.

## Storage Cost

In a production environment with 50 microservices, each with 30 versions retained in the registry, you might have 1500 images stored.

At 900 MB per image: 1500 x 900 MB = 1.35 TB of registry storage

At 50 MB per image: 1500 x 50 MB = 75 GB of registry storage

Registry storage on ECR, GCR, or Azure costs money per GB per month. A 18x reduction in storage cost is not trivial at scale.

On individual developer machines and CI/CD nodes, disk space is finite. Large images fill disks faster and require more frequent pruning. Smaller images are simply easier to manage.

## Transfer Time

Every deployment requires transferring the image from the registry to the server. Every developer pulling an image for local development is waiting for that transfer.

On a CI/CD pipeline that builds and deploys 50 times per day:

900 MB image: 50 transfers x 900 MB = 45 GB of transfer per day
50 MB image: 50 transfers x 50 MB = 2.5 GB of transfer per day

Transfer costs money (egress fees from cloud registries) and takes time (deployment pipeline speed). 

But the real saving is smarter than that calculation suggests. Docker transfers layers, not whole images. If 10 of your 50 microservices share the same python:3.11-slim base (60 MB), that base is transferred once and cached. Only the unique layers transfer on each deploy.

Multi-stage images have fewer layers and smaller unique layers. The total transfer on a cache-warm system can be under 5 MB per deployment for a code-only change.

---

# Section 6: Complete Lab — Step by Step With Measurements

## Lab Setup

This lab builds three versions of the same Python application — naive single-stage, bad multi-stage (wrong), and correct multi-stage — and measures the difference across all four concerns: image size, build time, startup time, and what is exposed inside the container.

### Step 1: Create the application

```bash
mkdir -p /home/multistage-lab && cd /home/multistage-lab
```

Create requirements.txt:

```bash
cat > requirements.txt << 'EOF'
flask==3.0.0
requests==2.31.0
EOF
```

Create app.py:

```bash
cat > app.py << 'EOF'
from flask import Flask, jsonify
import requests
import os
import time

app = Flask(__name__)
START_TIME = time.time()

@app.route('/')
def home():
    return jsonify({
        'status': 'running',
        'uptime_seconds': round(time.time() - START_TIME, 2),
        'version': os.environ.get('APP_VERSION', '1.0.0')
    })

@app.route('/health')
def health():
    return jsonify({'healthy': True})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
EOF
```

---

### Step 2: Build Version 1 — The Naive Single-Stage Image

This is what most developers build the first time. It works. But it is a mess.

```bash
cat > Dockerfile.naive << 'EOF'
FROM ubuntu:22.04

RUN apt-get update
RUN apt-get install -y python3 python3-pip curl wget vim git build-essential

COPY . /app
WORKDIR /app

RUN pip3 install -r requirements.txt

EXPOSE 8080

CMD python3 app.py
EOF
```

Build it and time the build:

```bash
time docker build -f Dockerfile.naive -t myapp:naive . 2>&1 | tee /tmp/naive-build.log
```

Check the size:

```bash
docker images myapp:naive
```

Write down the size. It will be between 500 MB and 900 MB.

Look inside — what tools are available to an attacker:

```bash
docker run --rm myapp:naive which curl
docker run --rm myapp:naive which wget
docker run --rm myapp:naive which vim
docker run --rm myapp:naive which gcc
docker run --rm myapp:naive which pip3
docker run --rm myapp:naive which apt-get
docker run --rm myapp:naive python3 --version
```

Every single one of those will return a path. Every one of those is a tool an attacker can use.

---

### Step 3: Build Version 2 — The Wrong Multi-Stage

This is the mistake people make when they first learn about multi-stage builds. They create two stages but put too little in Stage 2.

```bash
cat > Dockerfile.multistage-wrong << 'EOF'
FROM python:3.11 AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

FROM python:3.11 AS runtime
WORKDIR /app
COPY --from=builder /app .
COPY app.py .
EXPOSE 8080
CMD ["python3", "app.py"]
EOF
```

The mistake here is using python:3.11 (900 MB) for both stages instead of python:3.11-slim for the runtime. The multi-stage structure is correct but the base image choice ruins the savings.

Build it:

```bash
time docker build -f Dockerfile.multistage-wrong -t myapp:multistage-wrong . 2>&1 | tee /tmp/wrong-build.log
```

Check the size:

```bash
docker images myapp:multistage-wrong
```

It will still be close to 900 MB. Multi-stage builds alone do not save space if you use the wrong base image for Stage 2.

The lesson: multi-stage builds only help if Stage 2 uses a minimal base image.

---

### Step 4: Build Version 3 — The Correct Multi-Stage

This is the right approach. Stage 1 uses the full Python image for building. Stage 2 uses the slim image for running.

```bash
cat > Dockerfile.multistage-correct << 'EOF'
# ============================================================
# STAGE 1: BUILDER
# Purpose: Install all dependencies
# Base: Full Python image with all tools available
# This stage will NOT appear in the final image
# ============================================================
FROM python:3.11 AS builder

WORKDIR /build

# Copy only the dependency file first
# This layer is cached as long as requirements.txt does not change
COPY requirements.txt .

# Install dependencies into a specific directory
# --user installs to /root/.local instead of the system Python
# This makes it easy to copy just the installed packages to Stage 2
RUN pip install --no-cache-dir --user -r requirements.txt

# ============================================================
# STAGE 2: RUNTIME
# Purpose: Run the application
# Base: Minimal slim image — no build tools, no compilers
# This IS the final image
# ============================================================
FROM python:3.11-slim AS runtime

WORKDIR /app

# THE BRIDGE: Copy ONLY the installed packages from Stage 1
# --from=builder means: go into the Stage 1 image and copy from there
# /root/.local contains all the pip-installed packages and their binaries
# Nothing else from Stage 1 comes with this copy
COPY --from=builder /root/.local /root/.local

# Copy the application code from the LOCAL filesystem (not from Stage 1)
COPY app.py .

# Make sure Python can find the packages we copied from Stage 1
ENV PATH=/root/.local/bin:$PATH

# Non-root user for security
RUN groupadd --system appgroup && \
    useradd --system --gid appgroup --no-create-home appuser && \
    chown -R appuser:appgroup /app /root/.local

USER appuser

EXPOSE 8080

CMD ["python3", "app.py"]
EOF
```

Build it:

```bash
time docker build -f Dockerfile.multistage-correct -t myapp:correct . 2>&1 | tee /tmp/correct-build.log
```

Check the size:

```bash
docker images myapp:correct
```

---

### Step 5: Measure and Compare

Run all three images and compare sizes side by side:

```bash
echo "=== IMAGE SIZE COMPARISON ==="
docker images | grep myapp | awk '{printf "%-35s %s\n", $1":"$2, $7" "$8}'
```

Count the layers in each image:

```bash
echo ""
echo "=== LAYER COUNT COMPARISON ==="
echo "Naive layers:    $(docker history myapp:naive --no-trunc -q | wc -l)"
echo "Wrong MS layers: $(docker history myapp:multistage-wrong --no-trunc -q | wc -l)"
echo "Correct layers:  $(docker history myapp:correct --no-trunc -q | wc -l)"
```

Check what tools are available inside the correct image:

```bash
echo ""
echo "=== ATTACK SURFACE COMPARISON ==="
echo "--- Naive image ---"
docker run --rm myapp:naive sh -c "which curl wget vim gcc apt-get pip3 2>/dev/null || echo 'tool found'" 2>/dev/null

echo ""
echo "--- Correct multi-stage image ---"
docker run --rm myapp:correct sh -c "which curl 2>/dev/null || echo 'curl: NOT FOUND'"
docker run --rm myapp:correct sh -c "which wget 2>/dev/null || echo 'wget: NOT FOUND'"
docker run --rm myapp:correct sh -c "which vim 2>/dev/null || echo 'vim: NOT FOUND'"
docker run --rm myapp:correct sh -c "which gcc 2>/dev/null || echo 'gcc: NOT FOUND'"
docker run --rm myapp:correct sh -c "which apt-get 2>/dev/null || echo 'apt-get: NOT FOUND'"
docker run --rm myapp:correct sh -c "which pip3 2>/dev/null && echo 'pip3: FOUND' || echo 'pip3: NOT FOUND'"
```

Verify that the application still works correctly despite having none of those tools:

```bash
echo ""
echo "=== FUNCTIONALITY CHECK ==="
docker run -d --name correct-test -p 8080:8080 myapp:correct
sleep 2
curl -s http://localhost:8080/ | python3 -m json.tool
curl -s http://localhost:8080/health | python3 -m json.tool
docker stop correct-test && docker rm correct-test
```

The application returns valid JSON responses even though the image has no curl, no wget, no vim, no gcc. The application does not need those tools to run. It only needed them to be built.

---

### Step 6: Prove What is in Stage 1 and What is in Stage 2

This step makes the boundary between stages physically visible.

Build only Stage 1 (the builder):

```bash
docker build -f Dockerfile.multistage-correct --target builder -t myapp:stage1-only .
```

List what is in Stage 1:

```bash
echo "=== STAGE 1 CONTENTS ==="
docker run --rm myapp:stage1-only sh -c "pip3 list | head -20"
docker run --rm myapp:stage1-only sh -c "which pip3 gcc curl"
docker run --rm myapp:stage1-only sh -c "ls /root/.local/lib/python3.11/site-packages/ | head -10"
docker run --rm myapp:stage1-only sh -c "ls /build/"
```

Stage 1 has pip, the installed packages, the requirements.txt. It has everything.

Now check what Stage 2 has:

```bash
echo ""
echo "=== STAGE 2 CONTENTS ==="
docker run --rm myapp:correct sh -c "ls /root/.local/lib/python3.11/site-packages/ | head -10"
docker run --rm myapp:correct sh -c "which pip3 2>/dev/null || echo 'pip3 not here'"
docker run --rm myapp:correct sh -c "ls /app/"
docker run --rm myapp:correct sh -c "python3 -c 'import flask; print(flask.__version__)'"
```

Stage 2 has the installed packages (because we copied them with COPY --from). It has the application code. It does NOT have pip itself, because pip is a tool we used to install packages — the packages are what we need, not the installer.

---

### Step 7: Prove the COPY --from Bridge

This is the exact answer to "how does Phase 2 know what Phase 1 built?"

The COPY --from instruction is the only bridge. Let us see it in action explicitly.

Create a modified Dockerfile that makes the bridge obvious:

```bash
cat > Dockerfile.bridge-demo << 'EOF'
FROM python:3.11 AS builder
WORKDIR /build
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt
# Create a file that only exists in Stage 1
RUN echo "I was built in Stage 1 at $(date)" > /build/stage1-proof.txt

FROM python:3.11-slim AS runtime
WORKDIR /app
# This copies the installed packages from Stage 1
COPY --from=builder /root/.local /root/.local
# This copies the proof file from Stage 1
COPY --from=builder /build/stage1-proof.txt /app/stage1-proof.txt
# This copies app code from the LOCAL FILESYSTEM (not Stage 1)
COPY app.py .
ENV PATH=/root/.local/bin:$PATH
CMD ["python3", "app.py"]
EOF
```

Build it:

```bash
docker build -f Dockerfile.bridge-demo -t myapp:bridge-demo .
```

Look inside Stage 2:

```bash
# The proof file was explicitly copied from Stage 1 — it's here
docker run --rm myapp:bridge-demo cat /app/stage1-proof.txt

# Flask was installed in Stage 1 and copied — it works
docker run --rm myapp:bridge-demo python3 -c "import flask; print('Flask:', flask.__version__)"

# pip was NOT copied from Stage 1 — it's gone
docker run --rm myapp:bridge-demo which pip3 2>/dev/null || echo "pip3 is NOT in Stage 2"

# The requirements.txt from Stage 1 was NOT copied — it's gone
docker run --rm myapp:bridge-demo ls /build/ 2>/dev/null || echo "/build does not exist in Stage 2"
```

This makes the bridge visible. COPY --from is surgical. It takes exactly what you specify and nothing else.

---

### Step 8: Measure Build Time With Cache

This answers "does the build time stay the same?"

Build the correct multi-stage image a second time with no code changes:

```bash
echo "=== BUILD WITH FULL CACHE ==="
time docker build -f Dockerfile.multistage-correct -t myapp:correct . 2>&1 | grep -E "Step|Using cache|FINISHED|real"
```

Every step will show "Using cache". Build time will be under 1 second.

Now simulate a code change (change the version in app.py):

```bash
sed -i "s/'1.0.0'/'1.0.1'/" app.py
echo "=== BUILD AFTER CODE CHANGE ==="
time docker build -f Dockerfile.multistage-correct -t myapp:correct . 2>&1 | grep -E "Step|Using cache|FINISHED|real"
```

Only the COPY app.py step rebuilds. Stage 1 is fully cached. The pip install step is cached. Build completes in 2-3 seconds.

Now simulate a dependency change (add a new package):

```bash
echo "python-dateutil==2.8.2" >> requirements.txt
echo "=== BUILD AFTER DEPENDENCY CHANGE ==="
time docker build -f Dockerfile.multistage-correct -t myapp:correct . 2>&1 | grep -E "Step|Using cache|FINISHED|real"
```

Now COPY requirements.txt and RUN pip install both rebuild. This is the slow path. But it should still be faster than the naive Dockerfile because the base image layers are cached and Stage 2 rebuild is fast.

Reset the file for clean state:

```bash
git checkout -- app.py requirements.txt 2>/dev/null || true
```

---

### Step 9: Measure Startup Time

Save the images to tar files and measure load time as a proxy for startup:

```bash
echo "=== IMAGE SIZES ON DISK ==="
docker save myapp:naive -o /tmp/naive.tar
docker save myapp:correct -o /tmp/correct.tar
ls -lh /tmp/naive.tar /tmp/correct.tar
```

Measure actual container startup time:

```bash
echo ""
echo "=== STARTUP TIME COMPARISON ==="

echo "Naive image startup:"
time docker run --rm myapp:naive python3 -c "print('started')"

echo ""
echo "Correct multi-stage startup:"
time docker run --rm myapp:correct python3 -c "print('started')"
```

The difference will be visible. The slim image starts measurably faster even on a local machine. On a production node pulling the image cold over a network, the difference is dramatic.

---

### Step 10: Final Scorecard

Run this complete comparison:

```bash
echo "============================================"
echo "MULTI-STAGE BUILD — COMPLETE COMPARISON"
echo "============================================"

NAIVE_SIZE=$(docker inspect myapp:naive --format='{{.Size}}' 2>/dev/null)
CORRECT_SIZE=$(docker inspect myapp:correct --format='{{.Size}}' 2>/dev/null)

echo ""
echo "IMAGE SIZES:"
docker images | grep "^myapp" | awk '{printf "  %-35s %s %s\n", $1":"$2, $7, $8}'

echo ""
echo "LAYER COUNTS:"
printf "  %-35s %s\n" "myapp:naive" "$(docker history myapp:naive -q | wc -l) layers"
printf "  %-35s %s\n" "myapp:correct" "$(docker history myapp:correct -q | wc -l) layers"

echo ""
echo "TOOLS AVAILABLE (ATTACK SURFACE):"
echo "  myapp:naive:"
for tool in curl wget vim gcc apt-get pip3 python3; do
  result=$(docker run --rm myapp:naive which $tool 2>/dev/null)
  if [ -n "$result" ]; then
    echo "    $tool: PRESENT ($result)"
  fi
done

echo "  myapp:correct:"
for tool in curl wget vim gcc apt-get pip3; do
  result=$(docker run --rm myapp:correct which $tool 2>/dev/null)
  if [ -n "$result" ]; then
    echo "    $tool: PRESENT"
  else
    echo "    $tool: NOT PRESENT (attack surface reduced)"
  fi
done
result=$(docker run --rm myapp:correct which python3 2>/dev/null)
echo "    python3: PRESENT (needed to run)"

echo ""
echo "APPLICATION WORKS:"
docker run -d --name final-test -p 8081:8080 myapp:correct 2>/dev/null
sleep 2
RESPONSE=$(curl -s http://localhost:8081/ 2>/dev/null)
if [ -n "$RESPONSE" ]; then
  echo "  HTTP response: OK"
  echo "  $RESPONSE"
else
  echo "  Could not reach container"
fi
docker stop final-test && docker rm final-test 2>/dev/null

echo ""
echo "============================================"
echo "SUMMARY"
echo "============================================"
echo ""
echo "  Naive image:   attack tools present, large, slow to pull"
echo "  Correct image: minimal attack surface, small, fast"
echo ""
echo "  The application is IDENTICAL."
echo "  The difference is what surrounds it."
echo "============================================"
```

---

# Section 7: The Four Concerns — Answered With Numbers

After running the lab, fill in this table with your actual measurements:

| Concern | Naive Image | Multi-Stage Correct | Improvement |
|---------|-------------|---------------------|-------------|
| Image size | ~600-900 MB | ~150-180 MB | 4-6x smaller |
| Layer count | 15-20 layers | 8-10 layers | Fewer, cleaner |
| Build time (first) | ~4 min | ~3.5 min | Similar |
| Build time (code change) | ~4 min | ~5 seconds | 50x faster |
| Startup time (cold) | Slow (large layers) | Fast (small layers) | 3-5x faster |
| Attack tools present | curl, wget, vim, gcc, apt, pip | None | Drastically reduced |
| Storage cost at 1000 images | ~700 GB | ~150 GB | ~5x less |
| Transfer per deploy | ~600 MB | ~5 MB (cached) | 100x less |

---

# Section 8: The Three Rules of Multi-Stage Builds

### Rule 1: Stage 1 is for building. Stage 2 is for running.

Never put runtime concerns in Stage 1. Never put build tools in Stage 2. The separation must be clean.

### Rule 2: COPY --from is the only bridge. Be deliberate about what crosses.

Every file that crosses from Stage 1 to Stage 2 is a conscious choice. If you are copying the right thing, Stage 2 will be minimal. If you copy too broadly (COPY --from=builder /usr /usr), you defeat the purpose.

### Rule 3: Stage 2 base image determines your final size. Choose it as aggressively as your application allows.

python:3.11-slim: 125 MB — good for most Python apps
python:3.11-alpine: 50 MB — better if your dependencies support musl
gcr.io/distroless/python3: 20 MB — best if you can sacrifice debuggability

---

# Section 9: Common Mistakes and How to Avoid Them

## Mistake 1: Using full base image for Stage 2

```
# Wrong — Stage 2 uses the heavy image
FROM python:3.11 AS runtime    # 900 MB, includes compiler and build tools

# Correct
FROM python:3.11-slim AS runtime   # 125 MB, just the runtime
```

## Mistake 2: Copying too much from Stage 1

```
# Wrong — copies the entire /usr directory including compilers
COPY --from=builder /usr /usr

# Correct — copies only the installed packages
COPY --from=builder /root/.local /root/.local
```

## Mistake 3: Forgetting to set PATH after copying packages

When you install with pip --user, packages go to /root/.local/bin. If you copy that to Stage 2 but do not set PATH, Python cannot find the package executables.

```
# Correct
COPY --from=builder /root/.local /root/.local
ENV PATH=/root/.local/bin:$PATH
```

## Mistake 4: Copying source code from Stage 1 when it is already in the build context

```
# Unnecessary — app.py is in your build context, copy it directly
COPY --from=builder /build/app.py /app/app.py

# Correct — copy directly from local filesystem
COPY app.py /app/app.py
```

Use COPY --from for artifacts produced by Stage 1 (installed packages, compiled binaries, built assets). For source files that exist locally, copy directly.

## Mistake 5: Not naming stages

```
# Without names
FROM python:3.11
...
FROM python:3.11-slim
COPY --from=0 /root/.local /root/.local    # 0 means first stage

# With names — clearer and maintainable
FROM python:3.11 AS builder
...
FROM python:3.11-slim AS runtime
COPY --from=builder /root/.local /root/.local
```

Always name your stages. Referencing by index (0, 1, 2) breaks when you add or reorder stages.

---

# Section 10: Key Takeaways

## What Multi-Stage Builds Do

1. Stage 1 builds the application using a full-featured image with all necessary tools
2. Stage 2 starts completely fresh from a minimal base image
3. COPY --from is the only mechanism to transfer files from Stage 1 to Stage 2
4. The final image contains only Stage 2 — Stage 1 is discarded entirely
5. Nothing from Stage 1 enters the final image unless explicitly copied

## What Multi-Stage Builds Improve

| Concern | How multi-stage helps |
|---------|----------------------|
| Build time | Layer cache makes code-only rebuilds extremely fast |
| Startup time | Smaller image = fewer layers to load = faster container start |
| Attack surface | No build tools, no package manager, no compiler in production |
| Storage cost | 4-10x smaller images = 4-10x less registry storage |
| Transfer time | Smaller layers = faster pulls = faster deployments |

## The One Sentence Summary

Multi-stage builds let you use an expensive, tool-heavy environment to build your application and then copy only the result into a minimal, secure runtime image — giving you the best of both worlds without any compromise.

---

*End of Multi-Stage Builds Deep Dive*
