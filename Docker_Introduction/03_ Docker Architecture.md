# Docker Architecture: From Code to Running Container

---

## Where We Are in the Story

In Part 1, we proved that a container is a Linux process.

In Part 2, we understood how Linux processes work, why isolation is necessary, and how namespaces, cgroups, and the union filesystem solve the isolation problem.

Now we are ready for the full picture.

Before we look at Docker architecture, we need to understand one more thing first. What does a container actually run from? Where does it come from? What is the starting point?

---

# Section 1: From Code to Container - The Full Journey

## Concept

Think about this comparison for a moment.

If you want to install an operating system on a laptop, you need an ISO image. That ISO contains everything needed to boot and install the OS. Without the ISO, there is nothing to install.

If you want to launch an EC2 instance on AWS, you need an AMI - an Amazon Machine Image. The AMI is a pre-built template. AWS uses that template to create your virtual machine. Without the AMI, there is no instance.

If you want to install an OS on a bare metal server, you need the OS installation media. Same idea.

Docker containers work exactly the same way.

A container needs a template to start from. That template is called a Docker image.

Just like an ISO is a complete snapshot of an operating system, a Docker image is a complete snapshot of everything your application needs to run - the OS libraries, the runtime, the dependencies, and the application code itself.

Here is the journey from developer to running container.

Step 1. A developer writes application code. This could be Python, Java, Go, Node.js - anything.

Step 2. The developer writes a Dockerfile. A Dockerfile is a plain text file with a set of instructions. It says things like: start from this base image, install these dependencies, copy this application code, run this command on startup. Think of the Dockerfile as a recipe. It describes exactly how to build the image.

Step 3. Docker reads the Dockerfile and builds a Docker image. The image is a read-only, layered package containing everything the application needs. It is portable. It can be stored in a registry and shared with anyone. It can run on any machine that has Docker installed.

Step 4. Docker uses the image as a template to create and run a container. The container is the live, running instance. Just like an AMI is used to launch an EC2 instance, a Docker image is used to launch a container.

This is the relationship:

| Analogy | Template | Running Instance |
|---------|----------|-----------------|
| Operating system install | ISO image | Installed OS |
| AWS virtual machine | AMI | EC2 instance |
| Bare metal server | OS installation media | Running server |
| Docker | Docker image | Running container |

## The Important Rule

An image and a container are not the same thing.

An image is passive. It sits on disk. It never changes. You can create hundreds of containers from the same image. Deleting all those containers does not delete the image.

A container is active. It is a running process. It has a writable layer on top of the image. When the container is deleted, that writable layer is gone. The image remains.

## How a Dockerfile Becomes an Image

A Dockerfile is a simple instruction file. Each instruction creates one layer in the image.

Here is a basic example:

```
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y python3
COPY app.py /app/app.py
CMD ["python3", "/app/app.py"]
```

FROM says which base image to start from. Ubuntu 22.04 already has the OS libraries we need.

RUN executes a command during the build process and saves the result as a new layer. Here it installs Python.

COPY takes a file from the developer's machine and adds it to the image as a new layer.

CMD defines the command that runs when a container starts from this image. This becomes PID 1 inside the container.

When Docker builds this file, it creates four layers stacked on top of each other. The result is a self-contained image that can run anywhere.

We will go deep on Dockerfile syntax in a dedicated session. For now, understand that the Dockerfile is the source of truth for what goes into an image.

---

# Section 2: Docker Architecture Overview

## Concept

Now that we understand where images come from, we can understand the architecture that takes a simple command like this:

```
docker run -d --name mycontainer nginx
```

and turns it into a running, isolated container process.

Docker has a layered architecture. There is not one single program that does everything. There are multiple components, each with a specific responsibility. They communicate with each other through well-defined interfaces.

The components are:

1. Docker CLI - the command line interface
2. Docker Daemon - the main background service, also called dockerd
3. containerd - the container lifecycle manager
4. runc - the low-level container runtime
5. Docker Hub or a container registry - where images are stored

Let us walk through each one.

---

# Section 3: Docker CLI

## Concept

The Docker CLI is what you type commands into. When you run:

```
docker run -d --name mycontainer nginx
```

You are using the Docker CLI.

The CLI is just a client. It does not run containers. It does not manage images. It does not do any heavy lifting at all. Its only job is to take your input, convert it into an API call, and send that API call to the Docker Daemon.

That is it. The CLI is a translation layer between human input and the Docker API.

The Docker API is a REST API that uses the HTTP protocol. The CLI sends an HTTP POST request to the daemon with a JSON body describing what you want to do. The daemon processes the request and sends back a response. The CLI displays that response to you.

This design is important because the CLI and the daemon do not have to run on the same machine. The CLI can talk to a remote daemon running on a different server. This is how tools like Docker Desktop work on Mac and Windows - the CLI runs on your laptop but communicates with a Linux daemon running in a virtual machine.

The Docker socket on a Linux system is located at:

```
/var/run/docker.sock
```

When the CLI and daemon are on the same machine, the CLI communicates through this Unix socket. When they are on different machines, it communicates over TCP.

## Important Note on Security

The Docker socket is effectively a root-level access point to the system. Any process that can write to the Docker socket can create containers, mount host directories, and gain full control of the host. This is why the Docker socket must be protected carefully. We will cover Docker security in a dedicated session.

## Lab

When you run any Docker command, you can see what API call gets made by adding the --debug flag:

```
docker --debug run --name api-demo nginx
```

Watch the output. You will see the HTTP request being made to the daemon, the endpoint being called, and the response coming back.

To see the API directly, you can query the Docker socket with curl:

```
curl --unix-socket /var/run/docker.sock http://localhost/version
```

This returns the daemon version information as JSON - exactly what the CLI receives when it connects.

---

# Section 4: Docker Daemon (dockerd)

## Concept

The Docker Daemon is the heart of Docker. It is a background process that runs continuously on the host and manages everything Docker does.

Its process name is dockerd and it runs as a system service.

When you press Enter after typing a Docker command, the CLI sends an API request to the daemon. The daemon receives that request and decides what to do. For a docker run command, the daemon needs to:

- Check if the requested image exists locally
- If not, pull it from a registry
- Set up the container's filesystem using the union filesystem
- Create the necessary namespaces
- Set up the cgroup for resource limits
- Configure networking
- Start the container process

The daemon does not do all of this itself. For the actual work of starting and stopping containers, it delegates to a lower-level component called containerd.

Think of the daemon as a manager. It receives instructions, makes decisions, and delegates the actual work.

The daemon listens for API requests on the Unix socket at /var/run/docker.sock by default. It can also be configured to listen on a TCP port.

The daemon also manages:
- Docker images stored locally
- Docker volumes for persistent storage
- Docker networks for container communication
- Container logs and metadata

## Lab

Check the status of the Docker daemon:

```
systemctl status docker
```

You will see it running as a system service.

Find the daemon process:

```
ps -ef | grep dockerd
```

View the daemon logs:

```
journalctl -u docker.service -f
```

Open another terminal and run a container while watching the logs. You will see the daemon processing your request in real time.

---

# Section 5: containerd

## Concept

containerd is the component that actually manages the full lifecycle of containers. It sits between the Docker daemon and the low-level runtime.

The Docker daemon delegates container operations to containerd. When the daemon wants to start a container, it tells containerd. When it wants to stop a container, it tells containerd. When it wants to pull an image, containerd handles the download and layer extraction.

containerd is responsible for:

- Pulling and storing container images
- Managing image layers and the content store
- Creating and starting containers
- Stopping and deleting containers
- Managing container state and metadata
- Handling container snapshots

containerd is itself a standalone project and CNCF graduated project. It was originally part of Docker but was separated out so that other container platforms could use it. Kubernetes, for example, can use containerd directly without Docker.

This is an important architectural point. Docker is a tool for developers. containerd is an infrastructure component. They are separate concerns that happen to work together.

When containerd needs to actually create a container - to set up the namespaces, apply cgroups, and start the process - it calls another component: runc.

## Lab

containerd has its own command line tool called ctr. On a system with Docker installed, containerd is running underneath Docker.

Check the containerd process:

```
ps -ef | grep containerd
```

You will see both dockerd and containerd running as separate processes. containerd is a child of the system init, not a child of dockerd. They communicate through a socket.

The containerd socket is at:

```
/run/containerd/containerd.sock
```

---

# Section 6: runc

## Concept

runc is the lowest level component. It is the actual container runtime.

runc is a command-line tool that takes a container specification and creates a running container from it. It is the component that directly calls the Linux kernel to create namespaces, apply cgroups, set up the filesystem, and start the container process.

runc implements the OCI runtime specification. OCI stands for Open Container Initiative. The OCI specification is an industry standard that defines exactly what a container runtime must do. Any OCI-compliant runtime can be used in place of runc.

runc's job is extremely focused:

- Read the container configuration
- Create the specified namespaces
- Apply the specified cgroups
- Mount the specified filesystem
- Start the specified process

After runc starts the container process, runc itself exits. It does not stay running. The container process is now running directly under containerd's supervision. runc is done.

This is why the process tree looks the way it does on a system running containers. The container processes are direct children of containerd, not children of runc.

## The Relationship Between Components

Let us be precise about what each component does:

dockerd: Receives API calls from the CLI. Manages high-level Docker objects like images, networks, and volumes. Delegates container lifecycle operations to containerd.

containerd: Manages the full container lifecycle. Handles image storage and layer management. Calls runc to create containers.

runc: Creates the actual container. Sets up namespaces and cgroups. Starts the process. Then exits.

The running container process: Completely independent of runc. Supervised by containerd. Isolated by the namespaces and cgroups runc created.

## Lab

Find where runc is installed:

```
which runc
runc --version
```

After starting a container, look at the process tree:

```
docker run -d --name runc-demo nginx
pstree -p | grep containerd
```

You will see the nginx process as a child of containerd, not as a child of dockerd or runc.

---

# Section 7: Container Registry and Docker Hub

## Concept

A container registry is a storage system for Docker images. It is where images live when they are not running on your machine.

Docker Hub is the default public registry. It is operated by Docker and contains hundreds of thousands of public images. When you run:

```
docker run nginx
```

and the nginx image is not already on your machine, Docker automatically pulls it from Docker Hub. The full address of that image is docker.io/library/nginx:latest, but Docker knows the default registry and fills in the details automatically.

A registry is organized into repositories. A repository contains all the versions of a specific image. Each version is identified by a tag. The tag latest refers to the most recent published version, but you should always use a specific version tag in production.

```
nginx:latest       - latest version, unpredictable
nginx:1.25.3       - specific version, predictable and safe
ubuntu:22.04       - specific Ubuntu version
python:3.11-slim   - Python 3.11 with a minimal base image
```

Beyond Docker Hub, organizations run private registries. These registries store internal images that should not be public. Common private registries include:

- AWS Elastic Container Registry (ECR)
- Google Artifact Registry
- Azure Container Registry
- Self-hosted registries using the open source registry software

The daemon communicates with registries to push and pull images. When you run docker pull, the daemon contacts the registry, downloads each layer that is not already cached locally, and assembles the image on disk.

## Lab

Search for an image on Docker Hub:

```
docker search nginx
```

Pull an image explicitly:

```
docker pull nginx:1.25.3
```

List all images stored locally:

```
docker images
```

Inspect the layers of an image:

```
docker history nginx:1.25.3
```

See where Docker stores images on disk:

```
ls /var/lib/docker/overlay2/
```

Each directory here is one layer from one or more images.

---

# Section 8: The Complete Journey - docker run from Start to Finish

## Concept

Now we can trace exactly what happens when you type this command and press Enter:

```
docker run -d --name mycontainer -p 8080:80 --memory 512m nginx
```

Follow each step carefully. This is the complete architecture in motion.

### Step 1: Docker CLI parses your command

The Docker CLI reads your input. It identifies the following:

- -d means run in detached mode (in the background)
- --name mycontainer is the container name
- -p 8080:80 means map host port 8080 to container port 80
- --memory 512m is the memory limit
- nginx is the image name

The CLI converts this into an HTTP POST request to the Docker API endpoint /containers/create, with a JSON body containing all these parameters. It sends this request to the Docker daemon via the Unix socket at /var/run/docker.sock.

### Step 2: Docker daemon receives the request

dockerd receives the API call. It begins processing.

First, it checks whether the nginx image exists locally. It looks in its local image store.

If the image is not found locally, the daemon contacts Docker Hub and pulls the image. It downloads each layer individually and stores them in /var/lib/docker/overlay2/. This is why you see a progress bar with multiple layers when you pull an image for the first time.

### Step 3: Docker daemon instructs containerd

The daemon now hands off container creation to containerd. It sends containerd the container specification: which image to use, what command to run, what resource limits to apply, what ports to expose.

containerd takes this specification and prepares the container bundle. A bundle is a directory containing the container's root filesystem and a configuration file called config.json. This config.json describes exactly what the container should look like: its namespaces, its cgroups, its mounts, its process to run.

containerd assembles the root filesystem using OverlayFS. It stacks the nginx image's read-only layers and creates a writable layer on top. This becomes the container's filesystem.

### Step 4: containerd calls runc

containerd invokes runc and passes it the container bundle.

runc reads the config.json and begins creating the container.

runc calls the Linux kernel to create a new PID namespace. Inside this namespace, the container process will appear as PID 1.

runc calls the kernel to create a new network namespace. It gives the container its own virtual ethernet interface.

runc calls the kernel to create a new mount namespace. The container's root filesystem, assembled from the OverlayFS layers, is mounted here.

runc calls the kernel to create a new UTS namespace and sets the hostname.

runc creates the cgroup and applies the memory limit of 512 MB.

runc configures the iptables rules for the port mapping from host port 8080 to container port 80.

runc forks the nginx process inside the namespace. Inside the namespace, this process is PID 1. On the host, it has a different PID.

### Step 5: runc exits

Once the container process is running, runc exits. Its job is done.

The nginx process is now running. It is supervised by containerd. It is isolated by the namespaces runc created. Its resources are limited by the cgroup runc applied.

### Step 6: containerd reports back to the daemon

containerd tells dockerd that the container is running and provides the container ID. The container ID is a long unique hex string that identifies this container. containerd generates this ID when it creates the container.

### Step 7: The daemon responds to the CLI

dockerd sends an HTTP response back to the Docker CLI. The response contains the container ID.

The CLI prints the container ID to your terminal. That long string you see after running docker run -d is the container ID.

The entire sequence from pressing Enter to seeing the container ID typically takes less than one second for a locally cached image.

## The Architecture at a Glance

```
You type: docker run -d nginx

Docker CLI
  - Parses command
  - Builds REST API request
  - Sends to daemon via /var/run/docker.sock

Docker Daemon (dockerd)
  - Receives API request
  - Checks for image locally
  - Pulls from registry if needed
  - Instructs containerd to create container

containerd
  - Prepares container bundle
  - Sets up OverlayFS filesystem
  - Calls runc to create container

runc
  - Creates PID namespace
  - Creates network namespace
  - Creates mount namespace
  - Creates UTS namespace
  - Applies cgroup limits
  - Starts container process
  - Exits

Running container process
  - nginx is running as PID 1 inside the namespace
  - Isolated by namespaces
  - Resource-limited by cgroups
  - Filesystem from OverlayFS layers
```

---

# Section 9: Container ID - What It Is and Why It Exists

## Concept

Every container gets a unique identifier called a container ID.

The container ID is a 64-character hexadecimal string generated when the container is created. Docker also generates a shorter 12-character version of the same ID for convenience. When you run docker ps, you see the short version. When you need to reference the full ID, it is available in docker inspect output.

The container ID is generated by containerd when it creates the container. It is a content-addressable identifier - it is unique across all containers on any Docker host in the world.

The container ID is how the entire Docker system refers to a container internally. The human-readable name you give a container with --name is just an alias. Underneath, Docker uses the ID.

You can use either the name or the ID in Docker commands:

```
docker stop mycontainer
docker stop a3f4b2c1d9e8
```

Both refer to the same container.

Image IDs work on the same principle. Every image layer has a SHA256 hash. The image ID is derived from these hashes. This is what allows Docker to detect when two images share the same layer - the hash is identical.

## Lab

Run a container and capture its full ID:

```
docker run -d --name id-demo nginx
docker inspect id-demo --format '{{.Id}}'
```

Compare the full ID with the short ID:

```
docker ps
```

The first 12 characters of the full ID match the short ID shown in docker ps.

Use the ID directly in commands:

```
docker stop <first 4 characters of ID>
```

Docker accepts any unambiguous prefix of the container ID. Even 3 or 4 characters is usually enough if no other container shares that prefix.

---

# Section 10: Putting It All Together

## The Full Story in One View

You are now holding the complete picture.

A developer writes code. They describe how to package that code in a Dockerfile. Docker builds the Dockerfile into a layered image. The image is stored in a registry.

When you want to run the application, you run docker run. The CLI sends your request to the daemon. The daemon pulls the image if needed, then asks containerd to create the container. containerd sets up the filesystem using OverlayFS and calls runc. runc creates the namespaces, applies cgroups, and starts the process. runc exits. The container runs.

The running container is a Linux process, isolated by namespaces, controlled by cgroups, with a filesystem assembled from union filesystem layers.

That is all of it.

## Component Responsibility Summary

| Component | Responsibility | Location |
|-----------|---------------|----------|
| Dockerfile | Describes how to build an image | Developer's machine |
| Docker image | Layered template for containers | Registry and local disk |
| Docker CLI | Translates user commands to API calls | Developer's machine or server |
| Docker daemon (dockerd) | Manages Docker objects, delegates work | Host server |
| containerd | Container lifecycle and image management | Host server |
| runc | Creates namespaces, cgroups, starts process | Host server |
| Container process | Your running application | Host server, inside namespace |
| Registry (Docker Hub) | Stores and distributes images | Cloud or private server |

## What Comes Next

You now understand the foundation completely.

You know what a container is - a Linux process.

You know how isolation works - namespaces, cgroups, and OverlayFS.

You know the entire path from developer code to running container - Dockerfile, image, daemon, containerd, runc.

The next sessions will go deeper on each piece:

- Dockerfile in detail - every instruction, best practices, multi-stage builds
- Docker images - tagging, pushing, pulling, private registries
- Docker networking - bridge, host, overlay networks
- Docker volumes - persistent storage, bind mounts
- Docker Compose - running multi-container applications

But all of that is built on what you now understand. The rest is just detail on top of a foundation you already have.

---

# Quick Reference

## Component Locations on a Linux System

| Component | Binary Location | Socket / Config |
|-----------|----------------|-----------------|
| Docker CLI | /usr/bin/docker | Connects to daemon socket |
| Docker daemon | /usr/bin/dockerd | /var/run/docker.sock |
| containerd | /usr/bin/containerd | /run/containerd/containerd.sock |
| runc | /usr/bin/runc | Called by containerd directly |
| Image layers | /var/lib/docker/overlay2/ | Managed by daemon |
| Container metadata | /var/lib/docker/containers/ | One directory per container |

## Key Commands for This Section

| Command | What It Shows |
|---------|--------------|
| docker run -d --name X nginx | Start a container in detached mode |
| docker ps | List running containers with short IDs |
| docker ps -a | List all containers including stopped ones |
| docker inspect NAME | Full JSON details of a container |
| docker inspect NAME --format '{{.State.Pid}}' | Host PID of the container process |
| docker history IMAGE | Image layers and their sizes |
| docker images | All locally stored images |
| docker pull IMAGE | Pull image from registry explicitly |
| docker stats | Real-time resource usage per container |
| ps -ef | grep containerd | Verify containerd is running |
| systemctl status docker | Check daemon service status |

---

*End of Part 3 - Next: Dockerfile Deep Dive*
