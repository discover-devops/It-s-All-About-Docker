
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

---
---
---

# Topic 3

Yes, exactly. You're connecting the dots correctly.

## Is OverlayFS a Linux Kernel Feature?

**Yes.**

OverlayFS is implemented inside the Linux kernel.

Verify:

```bash id="3cwct5"
cat /proc/filesystems | grep overlay
```

Output:

```text id="c9ybxu"
nodev   overlay
```

This proves the kernel supports OverlayFS.

---

## What Problem Does OverlayFS Solve?

Suppose you have:

```text id="rya7mx"
Layer 1
---------
/bin/bash
/etc/passwd

Layer 2
---------
curl
wget

Layer 3
---------
app.py
```

Without OverlayFS:

```text id="jq6v0k"
3 separate filesystems
```

Container cannot use them directly.

OverlayFS merges them into:

```text id="s3k3hx"
/bin/bash
/etc/passwd
curl
wget
app.py
```

Container sees one filesystem.

---

## Is OverlayFS Same as Union Filesystem?

### Short Answer

**Yes, conceptually.**

OverlayFS is a type of Union Filesystem.

---

### What is a Union Filesystem?

A Union Filesystem combines multiple filesystems into one unified view.

```text id="glw7kt"
Filesystem A
      +
Filesystem B
      +
Filesystem C
      ↓
Unified Filesystem
```

That's the fundamental idea.

---

### Historical Timeline

Before OverlayFS, Linux had other Union Filesystems:

```text id="h5b7kp"
UnionFS
AUFS
OverlayFS
```

Docker initially used:

```text id="uaxd4k"
AUFS
```

because OverlayFS didn't exist in the kernel yet.

Later:

```text id="r9el4r"
OverlayFS
```

was merged into the Linux kernel and became the preferred choice.

Today most Docker installations use:

```text id="0c7sn0"
overlay2
```

which is built on OverlayFS.

---

## How Docker Uses OverlayFS

Docker image:

```text id="4w7f1m"
Layer 3
Layer 2
Layer 1
```

Container start:

```text id="sx22db"
Writable Layer
      ↓
Layer 3
      ↓
Layer 2
      ↓
Layer 1
```

OverlayFS merges everything.

Container sees:

```text id="69iq5r"
One Unified Filesystem
```

---

## The Most Important Teaching Point

I would tell students:

> Docker did not invent image layering. Docker leverages a Linux kernel feature called OverlayFS. OverlayFS is a Union Filesystem implementation that merges multiple read-only image layers and a writable container layer into a single filesystem view.

---

## Nice Interview Question

Ask students:

> Is OverlayFS a Docker feature?

Answer:

```text id="5wcbdh"
No.
```

Ask:

> Then what is it?

Answer:

```text id="knj2c8"
A Linux Kernel filesystem feature.
```

Ask:

> Why does Docker use it?

Answer:

```text id="u3g1mv"
To combine multiple image layers and the writable container layer into one filesystem visible to the container.
```

That's usually enough depth for Docker internals without diving into kernel source code.

---
---
---

# Topic 4


# Copy-on-Write (CoW) Simplified

## Context

In the previous section, we learned that Docker images consist of multiple read-only layers.

When a container starts, Docker adds one writable layer on top.

Question:

> If image layers are read-only, how can a container modify files?

The answer is Copy-on-Write.

---

## The Core Idea

Docker does not copy files unless a write operation occurs.

This makes containers:

* Fast to start
* Lightweight
* Efficient in storage usage

---

## Reading a File

Suppose the image contains:

```text
/app/config.json
```

Container reads the file:

```bash
cat /app/config.json
```

Docker checks:

```text
Step 1: Writable Layer → File not found

Step 2: Image Layers → File found

Step 3: Read file directly
```

Result:

```text
No file copy occurs
```

Reading is very fast.

---

## Creating a New File

Container creates:

```bash
touch /app/data.db
```

Docker stores it directly in:

```text
Writable Container Layer
```

Result:

```text
Image layers remain unchanged
```

---

## Modifying an Existing File

Suppose:

```text
/app/config.json
```

exists in a read-only image layer.

Container runs:

```bash
echo "new config" > /app/config.json
```

Docker cannot modify the image layer because it is read-only.

Instead:

```text
Step 1: Locate file in image layer

Step 2: Copy file to writable layer

Step 3: Modify copied file

Step 4: Use modified version
```

This process is called:

```text
Copy-on-Write
```

---

## Visual Representation

```text
Before Write

Writable Layer
----------------
(empty)

Image Layer
----------------
config.json
```

After Write:

```text
Writable Layer
----------------
config.json (modified)

Image Layer
----------------
config.json (original)
```

Container always sees:

```text
Modified Version
```

because the writable layer sits on top.

---

## Why Docker Uses Copy-on-Write

### Faster Container Startup

When a container starts:

```text
No files are copied
```

Docker simply mounts layers.

Container starts almost instantly.

---

### Storage Efficiency

100 containers can share:

```text
Ubuntu Layer
Python Layer
Application Layer
```

Only writable changes consume additional space.

---

### Image Protection

Original image layers never change.

Benefits:

* Predictable behavior
* Safe rollbacks
* Consistent deployments

---

## Demonstration

Start a container:

```bash
docker run -it --name cow-demo ubuntu:22.04 bash
```

Create a file:

```bash
echo "Docker Rocks" > /tmp/test.txt
```

Verify:

```bash
cat /tmp/test.txt
```

Exit:

```bash
exit
```

Commit the container:

```bash
docker commit cow-demo cow-image
```

Inspect image history:

```bash
docker history cow-image
```

Notice Docker creates a new layer containing your changes.

---

## Why Databases Should Use Volumes

Copy-on-Write works well for applications.

However, databases constantly modify files.

Every write operation must pass through the container layer.

For heavy-write workloads:

```text
MySQL
PostgreSQL
MongoDB
Redis
```

Use:

```text
Docker Volumes
```

Volumes bypass the image layer system and write directly to the host filesystem.

---

## Key Takeaway

Docker images are read-only.

When a container modifies an existing file, Docker copies that file from the image layer into the container's writable layer and then modifies the copy.

This behavior is called Copy-on-Write and is one of the key reasons containers are fast, lightweight, and storage-efficient.


---
---
---

# Topic 5

# Where Docker Stores Everything & Inspecting Images

## Context

We have learned:

* Images consist of layers
* OverlayFS combines layers
* Containers use Copy-on-Write

A common question is:

> Where does Docker store all this information?

Docker stores its data under:

```bash id="zax8ow"
/var/lib/docker
```

This is Docker's working directory.

---

## Explore Docker Storage

View the Docker root directory:

```bash id="jlwmg8"
docker info | grep "Docker Root Dir"
```

Example:

```text id="xghx9n"
Docker Root Dir: /var/lib/docker
```

Explore:

```bash id="y7pgjv"
cd /var/lib/docker

ls -l
```

Example:

```text id="zyr3ib"
buildkit/
containers/
image/
network/
volumes/
```

---

## Important Directories

### containers/

Stores container-specific information.

Each container gets its own directory.

```bash id="cr6vym"
docker ps -a
```

Copy a container ID and inspect:

```bash id="g9xqcw"
ls /var/lib/docker/containers
```

Students will see container IDs matching Docker containers.

---

### volumes/

Stores Docker volumes.

Create a volume:

```bash id="tcz84k"
docker volume create myvolume
```

Verify:

```bash id="7lwdz7"
docker volume ls
```

Now check:

```bash id="7jnj5s"
ls /var/lib/docker/volumes
```

---

### buildkit/

Stores build cache and build metadata.

Check build cache usage:

```bash id="db7pmu"
docker system df
```

---

## Important Warning

Never manually edit files inside:

```text id="c7v0i4"
/var/lib/docker
```

Always use Docker commands.

Docker manages these files internally.

---

# Inspecting Images

Docker provides commands to understand images without directly browsing internal directories.

---

## docker history

Shows how an image was built.

Example:

```bash id="njlwmg"
docker history ubuntu:22.04
```

Output:

```text id="t6gv5g"
IMAGE
CMD ["/bin/bash"]
ADD filesystem
LABEL ...
```

Students learn:

* Which instructions created layers
* Layer sizes
* Image build history

---

## docker inspect

Shows complete metadata in JSON format.

Example:

```bash id="dyxgca"
docker inspect ubuntu:22.04
```

Look for:

```json id="s3pnmj"
"RootFS": {
   "Type": "layers"
}
```

This proves:

```text id="b3iswm"
Image = Collection of Layers
```

---

## Extract Specific Information

Show image size:

```bash id="pbslst"
docker inspect ubuntu:22.04 \
--format='{{.Size}}'
```

Show architecture:

```bash id="wwjgyk"
docker inspect ubuntu:22.04 \
--format='{{.Architecture}}'
```

Show default command:

```bash id="bdgn3q"
docker inspect ubuntu:22.04 \
--format='{{.Config.Cmd}}'
```

---

# Bonus Tool: dive

dive is one of the best Docker troubleshooting tools.

It allows you to explore images layer by layer.

Install:

```bash id="7rj6ko"
wget https://github.com/wagoodman/dive/releases/latest/download/dive_0.12.0_linux_amd64.deb

apt install ./dive_0.12.0_linux_amd64.deb
```

Launch:

```bash id="jgb6z4"
dive ubuntu:22.04
```

---

## Why dive is Useful

Students can:

* Navigate layer by layer
* See files added in each layer
* Identify wasted space
* Understand image growth
* Discover unnecessary files

Typical findings:

```text id="v90fvp"
node_modules accidentally copied

Large log files

Temporary build files

Duplicate content
```

---

# Demonstration

Show image history:

```bash id="l3b89u"
docker history ubuntu:22.04
```

Show image metadata:

```bash id="rztfnd"
docker inspect ubuntu:22.04
```

Show container directories:

```bash id="wstffz"
ls /var/lib/docker/containers
```

Show volumes:

```bash id="d7lnvt"
ls /var/lib/docker/volumes
```

Show build cache:

```bash id="7f7h5h"
docker system df
```

---

# Key Takeaway

Docker stores all images, containers, volumes, networks, and build cache under `/var/lib/docker`.

Instead of manually exploring these files, use:

```bash id="7tf9v0"
docker history
docker inspect
docker system df
dive
```

These tools provide a safer and clearer view of Docker internals.

---
---
---

# Topic 6

# Container Runtime Architecture

## Context

Earlier, we learned:

```text
Docker Image
    ↓
Docker Container
```

We used commands like:

```bash
docker pull nginx
docker run nginx
docker ps
docker stop
```

But a natural question arises:

> What actually happens when I run `docker run nginx`?

Does Docker itself create the container?

Does the Linux kernel create it?

Why do we have Docker, containerd, runc, and the Linux kernel?

Let's find out.

---

## The Runtime Stack

When we run:

```bash
docker run nginx
```

multiple components work together.

```text
Docker CLI
     ↓
Docker Daemon
     ↓
containerd
     ↓
runc
     ↓
Linux Kernel
```

Finally:

```text
Container Starts
```

---

## Docker CLI

This is what we interact with every day.

Examples:

```bash
docker run nginx
docker ps
docker images
docker stop
```

Docker CLI does not create containers.

Its job is simply:

```text
Accept User Commands
```

and send requests to the Docker Daemon.

---

## Docker Daemon (dockerd)

The Docker Daemon is the management layer.

Responsibilities:

* Image management
* Container management
* Network management
* Volume management
* API handling

Think of it as:

```text
Docker Control Plane
```

When you type:

```bash
docker run nginx
```

the CLI sends the request to dockerd.

---

## containerd

Docker Daemon does not directly create containers.

Instead it delegates to:

```text
containerd
```

containerd is an industry-standard container runtime.

Responsibilities:

* Container lifecycle management
* Image pulling
* Container execution
* Container monitoring

Many Kubernetes distributions talk directly to containerd.

---

## runc

containerd delegates container creation to:

```text
runc
```

This is where the real magic happens.

runc creates:

* Namespaces
* Cgroups
* Mounts
* Container process

In simple terms:

```text
runc creates the isolated environment
```

---

## Linux Kernel

The Linux kernel provides the actual isolation mechanisms.

Docker does not implement isolation.

Linux does.

Key kernel features:

```text
Namespaces
Cgroups
OverlayFS
Capabilities
Seccomp
```

Without Linux kernel support:

```text
No Containers
```

---

## What Happens During docker run nginx?

Step-by-step:

```text
docker run nginx
        ↓
Docker CLI
        ↓
Docker Daemon
        ↓
containerd
        ↓
runc
        ↓
Linux Kernel
        ↓
Container Process Starts
```

This entire process usually takes less than a second.

---

## Why So Many Layers?

Students often ask:

> Why can't Docker directly talk to the kernel?

Good question.

The answer is modularity.

Each component has one responsibility.

```text
CLI         → User Interface

Docker      → Management

containerd  → Runtime Management

runc        → Container Creation

Kernel      → Isolation
```

This design makes Docker easier to maintain and extend.

---

## The Secret Hero: containerd-shim

This is a fascinating internal component.

Question:

> What happens if Docker Daemon crashes?

Most students assume:

```text
Containers Stop
```

Wrong.

Containers continue running.

Why?

Because of:

```text
containerd-shim
```

The shim process sits between containerd and the container.

Even if:

```text
dockerd
```

is restarted,

the container keeps running.

This is one of the reasons Docker is highly reliable.

---

## Demonstration

Start a container:

```bash
docker run -d --name web nginx
```

Find processes:

```bash
ps -ef | grep nginx
```

Show containerd:

```bash
ps -ef | grep containerd
```

Show Docker daemon:

```bash
ps -ef | grep dockerd
```

Students can now see the runtime stack on the host.

---

## WOW Demonstration

Run:

```bash
docker run -d nginx
```

Get container PID:

```bash
docker inspect -f '{{.State.Pid}}' <container-id>
```

Example:

```text
2451
```

Now:

```bash
ps -fp 2451
```

Students discover:

```text
Container = Linux Process
```

This is usually the biggest mindset shift in the entire Docker course.

Containers are not mini virtual machines.

They are isolated Linux processes.

---

## Interview Question

Question:

Who actually creates the container?

Answer:

```text
runc
```

Question:

Who provides isolation?

Answer:

```text
Linux Kernel
```

Question:

What keeps containers alive if Docker daemon restarts?

Answer:

```text
containerd-shim
```

---

## Key Takeaway

Docker is not a single component.

When we run a container, Docker CLI talks to the Docker Daemon, which delegates to containerd. containerd uses runc to create an isolated process using Linux kernel features such as namespaces, cgroups, and OverlayFS. The result is a lightweight container running as a Linux process.



