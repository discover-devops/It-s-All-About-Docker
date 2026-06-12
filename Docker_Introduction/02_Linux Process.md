# Docker Day 1 - Part 2
# Linux Process

---

## Where We Are in the Story

In Part 1, we proved something important.

We ran a container. We found its PID on the host. We killed that PID. The container disappeared.

That one demonstration tells us everything we need to know about what a container actually is.

A container is a Linux process.

But if we stop there, we are only halfway through the story. Because now a new question appears.

If a container is just a process, what exactly is a process? How does Linux create it, track it, and manage it? And if containers are just processes, why do they not interfere with each other?

That is what Part 2 is about.

We will understand Linux processes from the ground up. Then we will understand why isolation is necessary. Then we will see exactly how Docker solves the isolation problem using three kernel features: namespaces, cgroups, and the union filesystem.

By the end of this section, Docker will feel obvious. Not magical. Obvious.

---

# Section 1: Program vs Process

## Concept

Most people use the words program and process interchangeably. They are not the same thing.

A program is a file sitting on disk. It is passive. It does nothing on its own. When you look at /usr/bin/nginx on a Linux system, you are looking at a program. It is bytes on disk. Nothing more.

A process is what happens when Linux loads that program into memory and starts executing it. The moment you run nginx, Linux reads the binary from disk, loads it into RAM, creates a data structure to track it, and begins executing the instructions. That running instance is a process.

The same program can produce multiple processes. You can run nginx twenty times. Each run creates a separate process with its own memory space, its own state, and its own lifecycle. They all came from the same binary on disk, but they are completely independent processes.

This distinction matters for containers because a container image is the program. It sits on disk. The running container is the process. It lives in memory. When the process stops, the container stops. The image stays on disk exactly as it was.

## Summary

| | Program | Process |
|-|---------|---------|
| Where it lives | Disk | Memory |
| State | Passive | Active, executing |
| Created by | Compiler or build tool | Kernel, when you run it |
| Multiple instances | One copy on disk | Many processes from one program |
| Container equivalent | Docker image | Running container |

---

# Section 2: What is a PID

## Concept

When Linux creates a process, it assigns a unique number to it called a Process ID, or PID.

The PID is how the kernel tracks every running process on the system. Every operation the kernel performs on a process uses the PID as the identifier. Sending a signal, allocating memory, scheduling CPU time, killing a process - all of it goes through the PID.

PIDs are assigned in sequence. The very first process that Linux starts when the system boots receives PID 1. This is the init process, and in modern Linux systems it is typically systemd. Every other process on the system is a descendant of PID 1.

After PID 1, the kernel assigns PIDs in increasing order. When the counter reaches its maximum value, it wraps back around and reuses PIDs that are no longer in use.

## Why PID Matters for Containers

Here is the interesting part.

When you run a container, the process inside the container thinks it is PID 1. From inside the container, there appears to be only one process running, and it has PID 1. But if you look from the host, that same process has a completely different PID - something like 4821 or 11304.

The same process has two different PIDs depending on where you look from.

This is not a trick. This is namespaces working exactly as designed. We will cover this in detail shortly.

## Lab

Open a terminal and run the following commands.

List all running processes and their PIDs:

```
ps -ef
```

You will see PID 1 at the top. Every other process has a higher PID. Notice the PPID column - that is the Parent Process ID, which we will cover next.

Now run a container in the background:

```
docker run -d --name process-demo nginx
```

Find the PID of that container process on the host:

```
ps -ef | grep nginx
```

Note the PID. Now exec into the container and check the PID from inside:

```
docker exec -it process-demo ps -ef
```

Inside the container, nginx appears as PID 1. On the host, it has a different PID entirely. Same process. Two different views.

---

# Section 3: Parent and Child Processes

## Concept

Linux processes do not exist in isolation. Every process except PID 1 was created by another process. The process that creates a new process is called the parent. The newly created process is called the child.

When a parent process creates a child, Linux uses a system call called fork. The fork call creates an almost exact copy of the parent process. The child gets its own PID, but it inherits many properties from the parent - its environment variables, file descriptors, and memory state at the moment of creation.

After fork, the child process usually calls another system call called exec. The exec call replaces the child's memory with a new program. This fork-then-exec pattern is how almost every process on Linux comes into existence.

This creates a tree of processes. PID 1 is the root. Everything else branches down from there.

## Why This Matters for Containers

A container has a main process. When you run a container, Docker starts one primary process. That process becomes PID 1 inside the container's namespace. Any additional processes the container starts are children of that main process.

This is where the most important rule in containers comes from:

When the main process of a container exits, the container stops.

There is no background service keeping the container alive. There is no container daemon running inside. The container is alive exactly as long as its main process is alive. When the main process exits, the kernel cleans up the namespace, releases the cgroup resources, and the container is gone.

This is why Dockerfiles end with a CMD or ENTRYPOINT instruction - that instruction defines the main process. That process is what keeps the container running.

## Lab

Check the process tree on your system:

```
pstree -p
```

You will see the full hierarchy starting from PID 1, branching into every running process.

Run a container:

```
docker run -d --name child-demo nginx
```

Find the main nginx process:

```
ps -ef | grep nginx
```

You will see a master nginx process and worker nginx processes. The workers are children of the master. If you kill the master, the entire container stops. If a worker dies, nginx restarts it automatically.

Now see what happens when the main process exits:

```
docker run --name exit-demo ubuntu echo "hello"
```

The echo command runs, prints hello, and exits. Check the container:

```
docker ps -a
```

The container is in Exited state. No main process means no running container.

---

# Section 4: The Need for Isolation

## Concept

Now that we understand what a process is, we need to understand why isolation is necessary.

By default, Linux processes run in a shared environment. Every process can see every other process. Every process competes for the same CPU and memory. Every process can open files anywhere on the filesystem. This is perfectly fine for a general-purpose operating system. It is a serious problem when you want to run multiple independent applications on the same machine.

There are four specific problems that arise when processes have no isolation.

## Problem 1: Process Visibility

By default, any process can see every other process running on the system.

Run ps -ef on a Linux machine and you will see everything. Web servers, database processes, background jobs, other users applications - all of it is visible.

This creates a security problem. If you are running a multi-tenant environment - where different customers or teams share the same server - any process can observe what other processes are running. A malicious process can watch for sensitive process names, read environment variables from /proc, and gather information it should never have access to.

For containers, this means if isolation were not applied, a containerized application could see every other container running on the host. Application A could read the process names and arguments of Application B. That is not acceptable.

## Problem 2: Resource Conflict

Without resource limits, processes compete for CPU and memory with no rules.

Imagine running two applications on the same server. Application A starts processing a large batch job and consumes all available CPU. Application B, which is serving customer requests, slows to a crawl or stops entirely. Neither application did anything wrong. The kernel simply gave CPU to whoever asked for it.

The same problem exists with memory. An application with a memory leak will eventually consume all available RAM. The kernel will start killing other processes to reclaim memory. Your database gets killed because someone else's application leaked memory.

In a shared environment, one badly behaving process can bring down everything else.

## Problem 3: Security Concern

Without isolation, processes can interfere with each other in dangerous ways.

A process running as root has access to everything on the system. It can read any file, kill any process, change any configuration. Even a non-root process can do damage - reading files it should not see, sending signals to processes in other applications, or exploiting kernel vulnerabilities to escalate privileges.

When multiple applications share a single operating system, the blast radius of a security breach is the entire machine. A vulnerability in one application potentially gives an attacker access to all other applications on the same host.

## Problem 4: Dependency Conflict

Different applications often require different versions of the same library.

Application A requires Python 3.8. Application B requires Python 3.11. On a single operating system, only one version of Python can be the default. You cannot easily run both applications without complex workarounds.

The same problem exists with any shared system library. glibc versions, OpenSSL versions, database client libraries - any of these can create conflicts when multiple applications share the same OS environment.

This is the dependency hell problem. It is the primary reason developers say it works on my machine. Their machine has the right library versions. The server does not.

## Summary of Problems

| Problem | What Happens Without Isolation |
|---------|-------------------------------|
| Process Visibility | Any process can see all other processes on the system |
| Resource Conflict | One process can consume all CPU and memory |
| Security Concern | A breach in one app can expose all other apps |
| Dependency Conflict | Two apps needing different library versions cannot coexist |

---

# Section 5: How Docker Solves These Problems

## Concept

Docker does not invent solutions to these problems. Linux already had the solutions built into the kernel. Docker packages those kernel features and makes them easy to use.

Three kernel features make containers work:

1. Namespaces - solve the visibility problems
2. cgroups - solve the resource conflict problem
3. Union Filesystem - solve the dependency conflict problem

Each one addresses a specific problem from the list above.

---

# Section 6: Namespaces

## Concept

A namespace wraps a global system resource and makes it appear to a process that it has its own isolated instance of that resource.

The key word is appear. The resource is still shared at the kernel level. But from the perspective of any process inside a namespace, it looks like that process is the only one using it.

Linux has six types of namespaces that Docker uses.

### PID Namespace

Solves: Process Visibility

Inside a PID namespace, a process can only see other processes in the same namespace. It cannot see any processes outside.

When a container starts, Docker creates a new PID namespace. The main container process is assigned PID 1 inside that namespace. From inside the container, that process sees itself as PID 1 and sees no other processes from the host or other containers.

From the host, the kernel knows the real PID. The container process might be PID 4821 on the host. Inside the container, it is PID 1. Same process, two different identities depending on perspective.

### Network Namespace

Solves: Port conflicts, network isolation

Each container gets its own network namespace. This means its own virtual network interface, its own IP address, its own routing table, and its own set of ports.

Two containers can both listen on port 8080 because they are in separate network namespaces. Port 8080 inside Container A is completely separate from port 8080 inside Container B. To expose a container port to the host, Docker maps the container port to a host port using iptables rules.

### Mount Namespace

Solves: Filesystem isolation

A mount namespace gives each container its own filesystem view. The container sees a complete Linux filesystem - /bin, /lib, /etc, /var - but this filesystem is built from the container image. It is separate from the host filesystem.

A process inside a container cannot access /etc/passwd on the host. It can only access /etc/passwd inside its own mount namespace, which contains the image's version of that file. Unless explicitly mounted, host directories are completely invisible to the container.

### UTS Namespace

Solves: Hostname isolation

UTS stands for Unix Timesharing System. The UTS namespace allows each container to have its own hostname. When a container starts, its hostname is typically set to the container ID or a name you specify. The container has no knowledge of the host's real hostname.

### IPC Namespace

Solves: Inter-process communication isolation

IPC namespace isolates inter-process communication resources like shared memory segments and semaphores. This prevents processes in one container from communicating with or interfering with processes in another container using these mechanisms.

### User Namespace

Solves: Privilege escalation risk

User namespaces allow a process to have root privileges inside the container namespace while mapping to an unprivileged user on the host. Even if a process inside a container believes it is running as root, the kernel knows it is actually running as a non-root user on the host.

This significantly limits the damage a compromised container can do.

## Lab

Run a container:

```
docker run -d --name ns-demo nginx
```

List all namespaces on the host:

```
lsns
```

You will see separate namespaces for the container's PID, network, mount, and UTS.

Find the container's PID on the host:

```
docker inspect ns-demo --format '{{.State.Pid}}'
```

View the namespaces for that specific process:

```
lsns -p <PID>
```

Compare the network namespace of the container with the host:

```
ip addr show
docker exec ns-demo ip addr show
```

The container has a completely different IP address and network interface. Same kernel. Separate view.

---

# Section 7: cgroups

## Concept

cgroups stands for control groups. Where namespaces control what a process can see, cgroups control what a process can use.

cgroups is a Linux kernel feature that limits, accounts for, and isolates the resource usage of a collection of processes.

When Docker starts a container, it creates a cgroup for that container's process tree. Any resource limits you specify in the docker run command are applied through this cgroup. The kernel enforces these limits at a hardware level. A process cannot exceed its cgroup limits regardless of how hard it tries.

### CPU Control

You can limit a container to a specific number of CPU cores or a percentage of available CPU time.

If a container is limited to 0.5 CPU, it can use at most 50 percent of one CPU core, no matter how many cores the host has. Even if the container's process is running at 100 percent CPU, it will never take more than its allocated share.

This is what prevents one container from starving all others on a shared host.

### Memory Control

You can set a hard memory limit on a container.

If a container is limited to 512 MB of RAM and it tries to allocate more, the kernel will trigger an out-of-memory event. The kernel will kill the container's process rather than allow it to consume memory belonging to other containers or the host.

Without this limit, a memory-leaking application would consume all available RAM on the host and bring down every other container and the host operating system itself.

### Disk I/O Control

cgroups can also limit how fast a container reads from or writes to disk. This prevents a container running a disk-intensive workload from saturating storage and slowing down all other containers.

### Resource Accounting

Beyond limits, cgroups also tracks actual usage. The kernel records exactly how much CPU time, memory, and I/O each cgroup has consumed. This data powers monitoring tools and enables accurate billing in cloud environments.

## Lab

Run a container with memory and CPU limits:

```
docker run -d --name cgroup-demo --memory 256m --cpus 0.5 nginx
```

Inspect the resource limits applied:

```
docker inspect cgroup-demo | grep -i memory
docker inspect cgroup-demo | grep -i cpu
```

Check real-time resource usage:

```
docker stats cgroup-demo
```

On the host, find the cgroup created for this container:

```
cat /sys/fs/cgroup/memory/docker/<container-id>/memory.limit_in_bytes
```

The kernel is enforcing these limits directly. There is no application-level throttling happening. The limit is in the kernel itself.

---

# Section 8: Union Filesystem

## Concept

The union filesystem solves the dependency conflict problem. It is also what makes Docker images efficient and portable.

A union filesystem allows multiple directories to be stacked on top of each other and presented as a single unified view. Each layer is read-only except for the top layer, which is writable.

This is how Docker images work.

A Docker image is not a single large file. It is a stack of read-only layers. Each layer represents one instruction in the Dockerfile. When you run a container from that image, Docker adds a thin writable layer on top.

Here is a concrete example.

Consider an image built with the following steps:

- Start from Ubuntu base - this is layer 1
- Install Python - this adds layer 2
- Copy application code - this adds layer 3
- Set the startup command - this adds layer 4

Each layer contains only the files that changed in that step. Layer 1 has the full Ubuntu filesystem. Layer 2 has only the Python files that were installed. Layer 3 has only the application code files. Layer 4 has only a small configuration change.

When you run a container, Docker stacks all these read-only layers and adds a writable layer on top. Any files the container writes go into the writable layer. The layers below remain untouched.

### Why Layers Matter

Layers are shared across containers. If you run ten containers from the same Ubuntu-Python image, all ten containers share the same underlying layers. The Ubuntu layer is stored once on disk, not ten times. Only the writable layer is unique per container.

This makes Docker storage extremely efficient. It is also why pulling an image is fast when you already have some of its layers - Docker only downloads the layers you do not already have.

### OverlayFS

Modern Docker uses OverlayFS as its union filesystem driver. OverlayFS works by defining two directories: a lower directory containing all the read-only layers, and an upper directory that is the writable layer. The kernel merges these into a single unified view called the merged directory. That merged view is what the container sees as its filesystem.

When a container reads a file, OverlayFS looks in the upper directory first. If the file is not there, it falls through to the lower layers. When a container writes to a file that exists in a lower layer, OverlayFS performs a copy-on-write operation: it copies the file to the upper layer and writes the changes there. The original in the lower layer is never modified.

## Lab

Pull a simple image and inspect its layers:

```
docker pull nginx
docker history nginx
```

You will see each layer, its size, and the command that created it.

Run two containers from the same image:

```
docker run -d --name c1 nginx
docker run -d --name c2 nginx
```

Check the overlay mounts:

```
mount | grep overlay
```

You will see two separate overlay mounts, but they reference the same lower directory - the shared image layers. Only the upper directory is different per container.

Inspect where Docker stores the layers:

```
ls /var/lib/docker/overlay2/
```

You will see layer directories. Multiple containers share the same layer directories. Only their diff directories are unique.

---

# Section 9: How Docker Puts It All Together

## Concept

When you run this command:

```
docker run -d --name myapp -p 8080:80 --memory 512m nginx
```

Docker performs the following sequence automatically.

Step 1. Docker checks if the nginx image exists locally. If not, it pulls the image layers from the registry.

Step 2. Docker creates a new mount namespace and sets up the union filesystem. It stacks the image's read-only layers and adds a writable layer on top. The container now has its own isolated filesystem.

Step 3. Docker creates a new PID namespace. The container process will see itself as PID 1 with no other processes visible.

Step 4. Docker creates a new network namespace. It creates a virtual ethernet interface, assigns an IP address from Docker's internal network, and sets up the port mapping rule so that host port 8080 forwards to container port 80.

Step 5. Docker creates a new UTS namespace and sets the container's hostname.

Step 6. Docker creates a cgroup for the container and applies the memory limit of 512 MB.

Step 7. Docker starts the nginx process inside all these namespaces with the cgroup limits active.

The result is a process that feels like it is running on its own machine with its own filesystem, its own network, its own process tree, and its own resource allocation - but it is actually just a Linux process with kernel-level isolation applied.

## The Full Picture

| Container Feature | How It Works | Linux Mechanism |
|-------------------|--------------|-----------------|
| Process isolation | Can only see its own processes | PID namespace |
| Network isolation | Has its own IP and ports | Network namespace |
| Filesystem isolation | Sees only its own files | Mount namespace |
| Own hostname | Has an independent hostname | UTS namespace |
| Resource limits | Cannot exceed CPU and memory caps | cgroups |
| Efficient storage | Shares image layers across containers | OverlayFS |

---

# Section 10: Putting the Story Together

We started with a question. Where can we run an application?

We found that bare metal wastes resources. Virtual machines solve isolation but introduce heavy overhead. Containers solve both problems.

Then we proved that a container is a Linux process. We killed the PID and the container died.

Then we asked the right question. If containers are just processes, how do they remain isolated? How do they not see each other's files? How do they not steal each other's resources?

The answer is that Linux already knew how to do this. The kernel has had namespaces since 2002, cgroups since 2007, and OverlayFS for well over a decade. These features were always available. They were just difficult to use directly.

Docker took these three kernel features and packaged them into a tool that any developer can use with a single command.

That is the entire foundation of Docker.

Everything that comes next - images, Dockerfiles, volumes, networks, Docker Compose, Kubernetes - is built on top of this foundation. If you understand that a container is a process isolated by namespaces, controlled by cgroups, and built from union filesystem layers, you understand everything.

---

# Quick Reference

## Commands Used in This Section

| Command | What It Does |
|---------|--------------|
| ps -ef | List all processes with PID and PPID |
| pstree -p | Show process hierarchy as a tree |
| lsns | List all namespaces on the system |
| lsns -p PID | Show namespaces for a specific process |
| docker inspect NAME --format '{{.State.Pid}}' | Get host PID of a container |
| docker exec NAME ps -ef | List processes from inside a container |
| ip addr show | Show network interfaces and IPs |
| mount | grep overlay | Show overlay filesystem mounts |
| docker stats NAME | Real-time resource usage |
| docker history IMAGE | Show image layers |
| cat /sys/fs/cgroup/memory/docker/ID/memory.limit_in_bytes | Read cgroup memory limit |

## Key Rules

1. A container is a Linux process. Not a VM.
2. A container lives as long as its main process lives. PID 1 exits, container stops.
3. Namespaces control what the process can see.
4. cgroups control what the process can use.
5. OverlayFS controls what the process can read and write.
6. Docker automates all three of these kernel features with a single command.

---

*End of Part 2 - Next: Docker Architecture, the Daemon, and the Image System*
