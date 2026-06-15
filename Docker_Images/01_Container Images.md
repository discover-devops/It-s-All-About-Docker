# From Dockerfile to Docker Image: How Container Images Are Built

---

## Where We Are in the Story

In Session 1, we proved that a container is a Linux process. We understood namespaces, cgroups, OverlayFS, and the full Docker architecture from CLI to runc.

But we never answered one question completely.

Where does the container get its filesystem from?

When runc creates a container, it needs a root filesystem to mount. That root filesystem comes from a Docker image. And that image comes from a Dockerfile.

In this session, we build the complete picture of how a developer's code becomes a running container. We start at the Dockerfile, which is the source of truth, and we follow every step through to the final running container with its layered filesystem.

By the end of Part 1, you will understand:
- What a Dockerfile is and what every instruction does
- How Docker builds an image from a Dockerfile step by step
- What intermediate containers are and why they exist
- How image layers are created, cached, and stored on disk
- What an image manifest is and why it matters
- Where images live on your filesystem and how to inspect them

---

# Section 1: What Is a Docker Image

## Concept

Before we write a Dockerfile, we need to be precise about what we are trying to build.

A Docker image is a read-only, layered, self-contained package that includes everything an application needs to run. Not just the application code. Everything.

Everything means:

| What | Why |
|------|-----|
| Base operating system files | Libraries, system binaries the app depends on |
| Application runtime | Python interpreter, Java JVM, Node.js binary |
| Dependencies | pip packages, npm modules, maven jars |
| Application code | Your actual program |
| Configuration | Environment variables, default settings |
| Startup command | What process to run when a container starts |

The critical properties of an image:

**Immutable.** Once an image is built, it never changes. You cannot modify it. If you want a different image, you build a new one. This immutability is what makes containers reproducible and reliable.

**Layered.** An image is not a single large file. It is a stack of layers. Each layer represents one change to the filesystem. Layers are stacked from bottom to top to produce the final filesystem view.

**Shareable.** An image can be pushed to a registry and pulled by any machine anywhere in the world. The same image that works on your laptop will work on a production server.

**Versioned.** Images are tagged with version identifiers so you can reference specific, known-good versions of an application.

## The Analogy That Makes It Click

Think about what you need to install an operating system on a laptop. You need an ISO image. The ISO contains everything — the bootloader, the OS files, the installer. You cannot run a laptop without it. The ISO itself does not run. It is a template. When you install from the ISO, the installation is the running instance.

A Docker image works the same way:

| World | Template | Running Instance |
|-------|----------|-----------------|
| Operating system | ISO image | Installed OS |
| AWS cloud | AMI | EC2 instance |
| Docker | Docker image | Running container |

The image is passive. The container is active. The image is what you build and store. The container is what runs.

---

# Section 2: What Is a Dockerfile

## Concept

A Dockerfile is a plain text file containing a sequence of instructions that tell Docker exactly how to build an image.

It is the source of truth for your container image. Every decision about what goes into the image — which base OS, which dependencies, which code, which startup command — is captured in the Dockerfile. This makes your image reproducible. Anyone with the Dockerfile and the source code can build the exact same image.

Think of the Dockerfile as a recipe. The image is the dish you cook from that recipe. The container is the serving of that dish that you actually eat. You can cook the dish as many times as you want from the same recipe, and you can serve as many portions as you want from each cooking.

## Dockerfile Instructions

Every line in a Dockerfile is either a comment (starting with #) or an instruction. Instructions are processed in order from top to bottom. Each instruction has a specific job.

### FROM

FROM defines the base image. It is always the first instruction in a Dockerfile.

```
FROM ubuntu:22.04
```

Every image is built on top of another image. Ubuntu is built on top of scratch (empty). Python is built on top of Debian. Your application image is built on top of Python. This creates a chain of layers going all the way down to a base.

You can also build from scratch if you want a completely empty starting point:

```
FROM scratch
```

This is used for images containing static binaries with no OS dependencies at all.

### RUN

RUN executes a command during the build process. The result of the command — any files created, modified, or deleted — is captured as a new layer.

```
RUN apt-get update && apt-get install -y python3 pip
```

RUN is used to install packages, compile code, create directories, set permissions, and any other build-time operations.

Important: each RUN instruction creates one layer. Two separate RUN instructions create two layers. Combining them into one RUN instruction with && creates one layer. This matters for image size optimization, which we cover in Part 3.

### COPY

COPY copies files from the build context (your local directory) into the image filesystem.

```
COPY requirements.txt /app/requirements.txt
COPY app.py /app/app.py
```

The first argument is the source path relative to your build context. The second argument is the destination path inside the image.

### ADD

ADD does everything COPY does, plus two extra capabilities: it can extract tar archives automatically, and it can download files from URLs.

```
ADD myarchive.tar.gz /app/
ADD https://example.com/config.json /config/
```

Best practice: use COPY for simple file copying. Use ADD only when you specifically need the tar extraction or URL download capability. ADD has more implicit behavior and is harder to reason about.

### ENV

ENV sets environment variables that will be available both during the build process and inside running containers.

```
ENV APP_PORT=8080
ENV NODE_ENV=production
ENV DB_HOST=localhost
```

Environment variables set with ENV are baked into the image. Every container created from this image will have these variables by default. They can be overridden at runtime with docker run -e.

### ARG

ARG defines variables that are available only during the build process. Unlike ENV, ARG variables do not persist into the running container.

```
ARG BUILD_VERSION=1.0
RUN echo "Building version $BUILD_VERSION"
```

ARG values are passed at build time with docker build --build-arg BUILD_VERSION=2.0. Use ARG for things that vary between builds but should not be in the final image, like version numbers or build flags.

### WORKDIR

WORKDIR sets the working directory for all subsequent instructions. If the directory does not exist, Docker creates it.

```
WORKDIR /app
```

After this instruction, all RUN, COPY, ADD, CMD, and ENTRYPOINT instructions operate relative to /app. This is cleaner than writing absolute paths in every instruction.

### EXPOSE

EXPOSE documents which network port the container listens on. It does not actually publish or open the port — it is metadata.

```
EXPOSE 8080
```

The actual port mapping happens at runtime with docker run -p. EXPOSE is documentation for whoever reads the Dockerfile and for tools that automatically discover container ports.

### VOLUME

VOLUME creates a mount point and marks it as externally mountable. It tells Docker that this directory should be managed as a volume.

```
VOLUME /data
```

When a container starts from an image with VOLUME defined, Docker creates an anonymous volume for that path. This ensures data written there is not lost in the writable layer but is managed separately.

### CMD

CMD defines the default command to run when a container starts.

```
CMD ["python3", "/app/app.py"]
```

CMD can be overridden at runtime. If you run docker run myimage python3 /app/other.py, the CMD from the Dockerfile is ignored and python3 /app/other.py runs instead.

There can only be one CMD instruction. If you write multiple CMD instructions, only the last one takes effect.

### ENTRYPOINT

ENTRYPOINT also defines what runs when a container starts, but with a crucial difference: it is not overridden by arguments passed to docker run. Arguments passed to docker run are appended to the ENTRYPOINT command instead.

```
ENTRYPOINT ["python3"]
CMD ["/app/app.py"]
```

With this configuration:
- docker run myimage runs python3 /app/app.py
- docker run myimage /app/other.py runs python3 /app/other.py

ENTRYPOINT defines what the container IS. CMD defines the default argument. Together they give you a container that behaves like an executable with a sensible default.

## CMD vs ENTRYPOINT — The Definitive Comparison

This is the question that confuses almost everyone. Here is the complete answer.

| Scenario | CMD | ENTRYPOINT |
|----------|-----|------------|
| Can be overridden by docker run args | Yes, completely replaced | No, args are appended |
| Use case | Default arguments, easily swappable | Fixed executable, container acts as command |
| Typical use | Web server default config | CLI tools, scripts |
| Combined use | Default arguments to ENTRYPOINT | Fixed executable |

Use CMD when you want a default that users might commonly override.
Use ENTRYPOINT when the container has one specific job and that job should not change.
Use both together when you have a fixed executable with a configurable default argument.

### LABEL

LABEL adds metadata to the image as key-value pairs.

```
LABEL maintainer="rahul@example.com"
LABEL version="1.0"
LABEL description="My application image"
```

Labels are visible in docker inspect output and can be used to filter images. They add zero bytes to the image size because they are stored in the image manifest, not the filesystem.

### USER

USER sets the username or UID for running subsequent instructions and for the container's main process.

```
RUN groupadd -r appgroup && useradd -r -g appgroup appuser
USER appuser
```

By default, everything in a Dockerfile runs as root. This is a security risk. Setting a non-root user with USER means the container process runs without root privileges. This limits what an attacker can do if the application is compromised.

## A Complete Dockerfile Example

```
FROM python:3.11-slim

LABEL maintainer="rahul@example.com"
LABEL version="1.0"

WORKDIR /app

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

RUN groupadd -r appgroup && useradd -r -g appgroup appuser
USER appuser

EXPOSE 8080

CMD ["python3", "app.py"]
```

Read this top to bottom and trace what each instruction does. This is a clean, production-appropriate Dockerfile. We will optimize it further in Part 3.

---

# Section 3: How Docker Builds an Image

## Concept

When you run docker build, Docker does not simply package your files. It executes a precise, ordered process where each instruction becomes a distinct layer in the final image.

Understanding this process at depth changes how you write Dockerfiles and how you debug build failures.

## The Build Process Step by Step

### Step 1: Build Context

When you run:

```
docker build -t myapp:1.0 .
```

The . at the end is the build context. Docker packages everything in the current directory and sends it to the Docker daemon. The daemon cannot access your local filesystem directly — it only has access to what was sent in the build context.

This is important for two reasons. First, it means COPY instructions can only reference files that were in the build context. Second, it means a large directory full of unnecessary files slows down every build because all those files are transmitted to the daemon even if they are never used. The .dockerignore file solves this problem, covered in Part 3.

### Step 2: Parsing

The daemon reads the Dockerfile from top to bottom and builds an execution plan. It validates the syntax and checks that all referenced files exist in the build context.

### Step 3: Base Image

Docker processes the FROM instruction. If the base image is not in the local cache, it is pulled from the registry. The base image's layers become the foundation of your new image.

### Step 4: Processing Each Instruction — Intermediate Containers

This is the most important part to understand deeply.

For each instruction after FROM, Docker:

1. Creates a temporary container from the image state so far
2. Executes the instruction inside that temporary container
3. Takes a snapshot of the resulting filesystem changes
4. Saves that snapshot as a new read-only layer
5. Deletes the temporary container
6. Uses the new layer as the base for the next instruction

These temporary containers are called intermediate containers. They are created and destroyed during the build process. You can see them if you run docker build without the --rm flag (Docker removes them by default after each step).

This is the mechanism by which each instruction becomes a layer. The intermediate container is the execution environment. The snapshot of what changed is the layer.

## Lab — Watching the Build in Action

Create a project directory:

```
mkdir /home/myapp && cd /home/myapp
```

Create a simple requirements.txt:

```
flask==3.0.0
```

Create a simple app.py:

```
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello():
    return 'Hello from Docker Session 2'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

Create the Dockerfile:

```
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
EXPOSE 8080
CMD ["python3", "app.py"]
```

Build the image with verbose output:

```
docker build -t myapp:1.0 .
```

Watch the output carefully. You will see:

```
Step 1/7 : FROM python:3.11-slim
 ---> a4b8c2d1e9f0
Step 2/7 : WORKDIR /app
 ---> Running in 3f4a2b1c8d9e
Removing intermediate container 3f4a2b1c8d9e
 ---> b5c3d2e1f0a9
Step 3/7 : COPY requirements.txt .
 ---> c6d4e3f2a1b0
Step 4/7 : RUN pip install --no-cache-dir -r requirements.txt
 ---> Running in 7a8b9c0d1e2f
...
Removing intermediate container 7a8b9c0d1e2f
 ---> d7e5f4a3b2c1
```

Every line that says Running in is an intermediate container being created. Every line that says Removing intermediate container is that container being deleted after its layer is captured. Every line with an arrow followed by a hash is a new layer being created.

The final hash after the last step is your image ID.

---

# Section 4: Image Layers in Depth

## Concept

Every instruction that modifies the filesystem creates a new layer. Understanding layers is what separates engineers who write efficient Dockerfiles from those who write bloated ones.

## What Becomes a Layer

Not every instruction creates a filesystem layer. Only instructions that change the filesystem create layers with actual content.

| Instruction | Creates filesystem layer | Notes |
|-------------|------------------------|-------|
| FROM | Yes | Brings in all base image layers |
| RUN | Yes | Captures filesystem changes from command execution |
| COPY | Yes | Adds files to the filesystem |
| ADD | Yes | Adds files to the filesystem |
| WORKDIR | Sometimes | Only if directory must be created |
| ENV | No | Stored in image config metadata |
| ARG | No | Build-time only, not in image |
| EXPOSE | No | Stored in image config metadata |
| LABEL | No | Stored in image manifest metadata |
| CMD | No | Stored in image config metadata |
| ENTRYPOINT | No | Stored in image config metadata |
| USER | No | Stored in image config metadata |

## Layer Sharing — The Storage Efficiency Mechanism

Layers are identified by their SHA256 hash. If two images have a layer with the same hash, they share that layer on disk. It is stored once, no matter how many images reference it.

Consider this scenario:

Image A — Ubuntu + Python + App1 = 3 layers
Image B — Ubuntu + Python + App2 = 3 layers
Image C — Ubuntu + Node + App3 = 3 layers

On disk:
- Ubuntu layer: stored once, referenced by A, B, and C
- Python layer: stored once, referenced by A and B
- App1 layer: unique to A
- App2 layer: unique to B
- Node layer: unique to C
- App3 layer: unique to C

Total unique layers on disk: 6
Total layers if stored separately: 9

As the number of images grows, the savings compound. A fleet of 20 microservices all based on the same python:3.11-slim base shares that base layer across all 20 images. The base is downloaded once, stored once.

## Lab — Inspecting Layers

After building myapp:1.0, inspect its layers:

```
docker history myapp:1.0
```

You will see each layer, its size, and the instruction that created it. Notice that some layers show 0B — those are the metadata-only instructions like CMD, EXPOSE, ENV.

Inspect the layer IDs in detail:

```
docker inspect myapp:1.0
```

Look for the RootFS section. The Layers array shows the SHA256 hash of every layer in the image from bottom to top. These hashes are what Docker uses to check whether a layer already exists on disk before downloading it.

See where layers are stored:

```
ls /var/lib/docker/overlay2/
```

Each directory here is one layer. Count them. Build a second image based on the same python:3.11-slim and count again. The python:3.11-slim layers are not duplicated.

Compare layer IDs between two images from the same base:

```
docker inspect myapp:1.0 --format '{{.RootFS.Layers}}'
docker pull nginx
docker inspect nginx --format '{{.RootFS.Layers}}'
```

---

# Section 5: Layer Caching

## Concept

Docker caches every layer it builds. When you rebuild an image, Docker checks whether each instruction and its inputs are identical to a previously built layer. If they are, Docker reuses the cached layer instead of rebuilding it. If they are not, Docker rebuilds from that point and invalidates the cache for all subsequent layers.

This is called the layer cache. Understanding it is the single most impactful optimization for build speed.

## Cache Invalidation Rules

Cache is used when: the instruction text is identical AND all files referenced by the instruction are unchanged AND the previous layer's cache was used.

Cache is invalidated when: the instruction text changes OR any file referenced by COPY or ADD changes OR the previous layer's cache was invalidated.

The third rule is the important one. Cache invalidation cascades downward. If layer 4 is rebuilt, layers 5, 6, 7, and every layer after it must also be rebuilt, even if nothing in those layers changed.

## The Correct Order of Instructions

This cache behavior defines the correct order for Dockerfile instructions:

Put instructions that change rarely at the top.
Put instructions that change frequently at the bottom.

The most common mistake is this:

```
FROM python:3.11-slim
WORKDIR /app
COPY . .                              <- copies everything including app code
RUN pip install -r requirements.txt  <- runs after code copy
```

Every time any file in your project changes, the COPY . . layer is invalidated. pip install then runs again even though requirements.txt did not change. A pip install might take 2 minutes. This adds 2 minutes to every single build.

The correct order:

```
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .              <- copy only the dependency file first
RUN pip install -r requirements.txt  <- cached as long as requirements.txt unchanged
COPY . .                             <- copy code last, only invalidates code layer
```

Now pip install is cached unless requirements.txt changes. Changing app.py only rebuilds the COPY . . layer, which is fast. Typical build time drops from 2 minutes to under 10 seconds for code-only changes.

## Lab — Experiencing the Cache

Build myapp:1.0 a second time immediately after the first build:

```
docker build -t myapp:1.0 .
```

This time, every step shows Using cache. The build completes in milliseconds.

Now change one line in app.py (change the return string):

```
docker build -t myapp:1.1 .
```

Watch which steps show Using cache and which steps rebuild. Only COPY app.py . and any layers after it rebuild. The pip install layer is still cached.

Now change requirements.txt (add a new package):

```
docker build -t myapp:1.2 .
```

Now COPY requirements.txt ., the RUN pip install layer, and all subsequent layers rebuild. pip install runs in full.

This demonstrates exactly how layer cache invalidation works in practice.

---

# Section 6: Image Manifests and Metadata

## Concept

A Docker image is not just a collection of filesystem layers. It also contains metadata — information about the image itself. This metadata lives in two JSON documents: the image manifest and the image config.

## Image Manifest

The manifest is the index of the image. It is a JSON document that describes:

- The list of layers in the image, identified by their SHA256 digest
- A reference to the image config file
- The architecture the image is built for (amd64, arm64, etc.)
- The operating system (linux, windows)
- The total size of all layers

The manifest is what the registry uses when you pull an image. Docker downloads the manifest first, reads the layer list, then downloads only the layers it does not already have cached locally.

Multi-architecture images use a manifest list (also called a fat manifest) — a manifest that contains references to separate manifests for each architecture. When you pull nginx on an x86 machine, Docker reads the manifest list, finds the amd64 manifest, and downloads those layers. When you pull on an ARM machine, it finds the arm64 manifest. Same image name, correct layers for each platform, automatically.

## Image Config

The image config is a separate JSON document containing all the metadata about how the image was configured:

- Environment variables defined with ENV
- The WORKDIR
- The EXPOSE ports
- The CMD and ENTRYPOINT
- Labels
- The user set by USER
- The complete build history — every instruction, in order

## Digests vs Tags

Every layer and every manifest has a SHA256 digest. The digest is computed from the content. If the content changes, the digest changes. If the content is identical, the digest is identical, on any machine anywhere in the world.

A tag (like nginx:1.25 or myapp:v2.0) is a human-readable pointer to a specific manifest digest. Tags are mutable — you can move a tag to point to a different manifest. A digest is immutable — it permanently identifies one specific set of content.

| | Tag | Digest |
|-|-----|--------|
| Example | nginx:1.25 | sha256:abc123...def456 |
| Mutable | Yes — can be moved | No — permanently fixed |
| Human readable | Yes | No |
| Production safety | Risk — can change | Safe — always the same image |

In production, the safest way to reference an image is by digest:

```
docker run nginx@sha256:abc123...def456
```

This guarantees you are running exactly the same image every time, with no risk of the tag being moved to a different image.

## Lab — Inspecting Manifests and Config

View the full image config and manifest data:

```
docker inspect myapp:1.0
```

The output is a large JSON document. Find these sections:

Config.Env — the environment variables
Config.ExposedPorts — the exposed ports
Config.Cmd — the CMD instruction
Config.WorkingDir — the WORKDIR
RootFS.Layers — the layer SHA256 hashes

Get the image digest:

```
docker inspect myapp:1.0 --format '{{.RepoDigests}}'
```

If the image has not been pushed to a registry, this will be empty. Push it and it will show the registry digest.

Pull an image by digest:

```
docker pull ubuntu@sha256:$(docker inspect ubuntu --format '{{index .RepoDigests 0}}' | cut -d@ -f2)
```

---

# Section 7: How Images Are Stored on Disk

## Concept

Docker stores images on the local filesystem under /var/lib/docker/. Understanding this storage structure helps you manage disk space and understand what Docker is doing when you pull, build, and delete images.

The storage driver Docker uses on modern Linux systems is overlay2. We briefly covered OverlayFS in Session 1. Here we go deeper into how it actually lays out image data on disk.

## The Directory Structure

```
/var/lib/docker/
├── image/                     Image metadata and manifest database
│   └── overlay2/
│       ├── imagedb/           Image config JSON files, one per image
│       ├── layerdb/           Layer metadata, checksums, parent relationships
│       └── repositories.json  Tag to image ID mapping
├── overlay2/                  Actual layer content (the filesystems)
│   ├── <layer-hash>/
│   │   ├── diff/              The actual files in this layer
│   │   ├── link               Short link name for this layer
│   │   ├── lower              Reference to the parent layer
│   │   └── work/              OverlayFS work directory
│   └── <layer-hash>-init/     Container init layer (networking config)
└── containers/                Container metadata
    └── <container-id>/        One directory per container
        ├── config.v2.json     Container configuration
        ├── hostconfig.json    Host-level configuration
        └── <id>-json.log      Container logs
```

## Lab — Exploring Docker's Storage

See the total Docker disk usage:

```
docker system df
```

This shows images, containers, volumes, and build cache with actual and reclaimable sizes.

Count the overlay2 layers:

```
ls /var/lib/docker/overlay2/ | wc -l
```

Look inside one layer:

```
ls /var/lib/docker/overlay2/
cd /var/lib/docker/overlay2/<any-layer-id>/diff
ls
```

You are looking at the actual files that make up one layer of one image. This is the real filesystem content, on disk, uncompressed.

See the layer chain for a specific image:

```
docker inspect myapp:1.0 --format '{{.GraphDriver.Data}}'
```

This shows the LowerDir, UpperDir, WorkDir, and MergedDir that OverlayFS uses to assemble the filesystem for a running container based on this image.

Find where a container's writable layer is:

```
docker run -d --name storage-demo myapp:1.0
docker inspect storage-demo --format '{{.GraphDriver.Data.UpperDir}}'
```

Write a file inside the container:

```
docker exec storage-demo bash -c "echo 'written at runtime' > /app/runtime.txt"
```

Now look in the UpperDir on the host:

```
ls $(docker inspect storage-demo --format '{{.GraphDriver.Data.UpperDir}}')
```

You will find runtime.txt there. That file exists in the container's writable layer on the host filesystem. You can read it directly from the host.

---

# Section 8: Key Takeaways — Part 1

## The Complete Picture

```
Developer writes code
        |
        v
Writes Dockerfile
(FROM, RUN, COPY, CMD ...)
        |
        v
docker build -t myapp:1.0 .
        |
        v
For each instruction:
  Create intermediate container
  Execute instruction
  Snapshot filesystem changes -> new read-only layer
  Delete intermediate container
        |
        v
Stack all layers -> Docker Image
  Layer 1: Base OS (FROM)
  Layer 2: Dependencies installed (RUN)
  Layer 3: App code (COPY)
  Metadata: ENV, CMD, EXPOSE (no filesystem layer)
        |
        v
Image stored in /var/lib/docker/overlay2/
Each layer = one directory with diff/ containing the files
        |
        v
docker run myapp:1.0
  Stack read-only image layers
  Add writable layer on top
  Mount as container filesystem
  Start PID 1
        |
        v
Running container with unified filesystem view
```

## Reference Table: Dockerfile Instructions

| Instruction | Creates Layer | Build-time | Runtime | Purpose |
|-------------|--------------|------------|---------|---------|
| FROM | Yes | Yes | - | Base image |
| RUN | Yes | Yes | - | Execute commands |
| COPY | Yes | Yes | - | Copy files in |
| ADD | Yes | Yes | - | Copy/extract/download |
| WORKDIR | Sometimes | Yes | Yes | Set working directory |
| ENV | No | Yes | Yes | Set environment variable |
| ARG | No | Yes | No | Build-time variable |
| EXPOSE | No | No | Docs | Document port |
| LABEL | No | Yes | Yes | Add metadata |
| CMD | No | No | Yes | Default command |
| ENTRYPOINT | No | No | Yes | Fixed executable |
| USER | No | Yes | Yes | Set run user |
| VOLUME | No | No | Yes | Define mount point |

---

*End of Part 1 - Next: Part 2 — Running Containers and Essential Commands*
