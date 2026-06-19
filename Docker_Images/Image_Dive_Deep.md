
# Topic: 1


1. Understanding Image Layers

Priya knows images have layers, but what exactly ARE they?

What is a Layer?

A layer is simply a tar file containing filesystem changes.

Think of it like this: Each Dockerfile instruction creates a new layer with only the changes made by that command.

Example:

Visual representation:

Why Layers Matter

Sharing saves space: If you have 10 images all based on ubuntu:20.04, that base layer is stored only once!

Caching speeds up builds:

Change your app code → Only the last layer rebuilds → Fast builds!

Layer Immutability

Once created, layers never change.

This is crucial because:
Multiple containers can safely share the same layers

You can roll back to previous versions

Docker knows exactly what changed between versions



# Understanding Docker Image Layers

## Context

Most beginners think a Docker image is a single file. It is not.

Let's prove it.

Pull an Ubuntu image:

```bash
docker pull ubuntu:22.04
```

Now inspect its history:

```bash
docker history ubuntu:22.04
```

Example output:

```text
IMAGE          CREATED BY                                      SIZE
4f838adc7181   CMD ["/bin/bash"]                               0B
<missing>      ADD file:xxxx                                   87.7MB
<missing>      LABEL org.opencontainers...                     0B
<missing>      ARG LAUNCHPAD_BUILD_ARCH                        0B
<missing>      ARG RELEASE                                     0B
```

Notice that the image is made up of multiple entries, not a single file.

---

## Concept

A Docker image is a collection of read-only layers stacked together.

Think of an image like a stack of transparent sheets:

```text
+-------------------+
| Layer 4           |
+-------------------+
| Layer 3           |
+-------------------+
| Layer 2           |
+-------------------+
| Layer 1           |
+-------------------+
```

Docker combines all layers and presents them as a single filesystem.

---

## What is a Layer?

A layer is a read-only filesystem snapshot that stores only the changes introduced by a Dockerfile instruction.

Example:

```dockerfile
FROM ubuntu:22.04
RUN apt update
RUN apt install -y curl
COPY app.py /app.py
```

Creates approximately:

```text
Layer 1 → Ubuntu Base OS

Layer 2 → apt update changes

Layer 3 → curl installation

Layer 4 → app.py
```

Each layer stores only the difference from the previous layer.

---

## Why Layers Matter

### Storage Efficiency

If 10 images use the same Ubuntu base image, Docker stores that Ubuntu layer only once.

### Faster Builds

If only app.py changes:

```dockerfile
COPY app.py /app.py
```

Docker reuses all previous layers and rebuilds only the last layer.

### Immutability

Layers never change after creation.

Benefits:

* Safe sharing between containers
* Efficient caching
* Easy rollback to previous versions

---

## Verify the Layers

View image history:

```bash
docker history ubuntu:22.04
```

View actual layer IDs:

```bash
docker inspect ubuntu:22.04
```

Look for:

```json
"RootFS": {
  "Type": "layers",
  "Layers": [
    "sha256:..."
  ]
}
```

Docker is literally telling us that the image consists of multiple layers.

---

## Key Takeaway

A Docker image is not a single file. It is a stack of immutable, read-only filesystem layers. Each layer stores only the changes introduced by a Dockerfile instruction, which helps Docker save storage, speed up builds, and efficiently distribute images.
