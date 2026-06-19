
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


---
---
---

# Topic 2

# How Docker Stacks Layers

## Context

In the previous section we learned that a Docker image consists of multiple read-only layers.

For example:

```dockerfile
FROM ubuntu:22.04
RUN apt update
RUN apt install -y curl
COPY app.py /app.py
```

This creates multiple layers:

```text
Layer 4 → app.py
Layer 3 → curl installation
Layer 2 → apt update
Layer 1 → Ubuntu Base OS
```

Now the question is:

> If the image consists of multiple layers, how does the container see a single filesystem?

When we enter a container, we don't see four filesystems.

We see one filesystem:

```text
/
├── bin
├── etc
├── usr
├── var
└── app.py
```

So how does Docker combine all these layers together?

https://en.wikipedia.org/wiki/OverlayFS

---

## Concept

Docker uses a Linux filesystem technology called OverlayFS.

OverlayFS stacks multiple layers together and presents them as a single unified filesystem.

Think of transparent sheets stacked on top of each other.

```text
Layer 4 → app.py

Layer 3 → curl

Layer 2 → apt update

Layer 1 → Ubuntu
```

When stacked together, the container sees:

```text
One Unified Filesystem
```

The container does not know there are multiple layers underneath.

---

## Layer Visibility

An upper layer can hide files from lower layers.

Think of transparent animation sheets.

If the top sheet contains a tree, you cannot see what is directly underneath that tree on lower sheets.

Similarly, OverlayFS always gives priority to the upper layer.

```text
Top Layer
    ↓
Middle Layer
    ↓
Bottom Layer
```

The container sees the topmost version of a file.

---

## The Writable Layer

All image layers are read-only.

Docker never modifies image layers.

When a container starts, Docker adds one thin writable layer on top.

```text
Writable Container Layer
------------------------
Image Layer 4
Image Layer 3
Image Layer 2
Image Layer 1
```

This writable layer stores:

* New files
* Modified files
* Deleted files

All container changes go here.

The image layers remain unchanged.

---

## Why This Is Powerful

Suppose you start 1000 containers from the same image.

Docker does NOT create 1000 copies of the image.

Instead:

```text
Shared Read-Only Image Layers
                ↓
Container 1 Writable Layer

Container 2 Writable Layer

Container 3 Writable Layer

...

Container 1000 Writable Layer
```

Benefits:

* Saves disk space
* Saves memory
* Containers start very quickly
* Images remain immutable

---

## Demonstration

Run a container:

```bash
docker run -it ubuntu:22.04 bash
```

Inside the container:

```bash
echo "Hello Docker" > test.txt
```

Exit:

```bash
exit
```

Start a new container:

```bash
docker run -it ubuntu:22.04 bash
```

Check:

```bash
ls
```

Output:

```text
test.txt not found
```

Why?

Because the file was written only to the first container's writable layer.

The image itself was never modified.

---

## Key Takeaway

Docker images consist of multiple read-only layers.

OverlayFS stacks these layers together and presents them as a single filesystem.

When a container starts, Docker adds a thin writable layer on top. All container changes are stored there, while the underlying image layers remain unchanged.

