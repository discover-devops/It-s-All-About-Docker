# What is COntainer: Docker

---

## Introduction

Before we talk about Docker, containers, Kubernetes, or DevOps, let's start with a simple question.

Every application that we use today must run somewhere.

- Netflix must run somewhere.
- Swiggy must run somewhere.
- Paytm must run somewhere.
- Even a simple Python application that we write must eventually run somewhere.

**The first question we should ask is:**

> "Where can we run an application?"

Once we understand the answer to this question, the need for Docker becomes very obvious.

---

## Option 1: Physical Server (Bare Metal)

The most traditional approach is to purchase a physical server and deploy the application directly on it.

**A physical server contains:**
- CPU
- Memory
- Storage
- Network
- Operating System

The application is installed directly on top of the operating system.

### Pros

| # | Advantage |
|---|-----------|
| 1 | Simple — no virtualization layers |
| 2 | Applications get direct hardware access |
| 3 | Maximum performance |

### Cons

| # | Problem | Example |
|---|---------|---------|
| 1 | Resource waste | Server has 16 GB RAM, app uses 4 GB — 12 GB sits idle |
| 2 | No resource capping | One app can consume all CPU and starve others |
| 3 | Security risk | Multiple teams share the same OS |
| 4 | Operational complexity | Running multiple apps on one server causes port conflicts and interference |

**Key insight:** The organization pays for hardware that is not being utilized.

---

## Option 2: Virtual Machines

To solve the bare metal problem, the industry adopted virtualization.

A **hypervisor** sits between hardware and operating systems and allows multiple Virtual Machines (VMs) to run on the same physical server.

**Two types of hypervisor:**
- **Type 1 (Bare Metal):** Runs directly on hardware. Examples: VMware ESXi, Microsoft Hyper-V.
- **Type 2 (Hosted):** Runs on top of a host OS. Examples: VirtualBox, VMware Workstation.

**Each Virtual Machine contains:**
- Guest Operating System
- Libraries and Runtime
- Application Code

### Pros

| # | Advantage |
|---|-----------|
| 1 | Strong isolation — each app has its own OS |
| 2 | Better resource utilization than bare metal |
| 3 | Security improves — apps cannot interfere |

### Cons

| # | Problem | Example |
|---|---------|---------|
| 1 | Heavy | Each VM needs 2 GB+ RAM just for its OS |
| 2 | CAPEX increases | More OS licenses per VM |
| 3 | OPEX increases | More staff needed to manage multiple OS instances |
| 4 | Slow startup | VMs take minutes to boot |

**Key insight:** 10 apps = 10 VMs = 10 OS copies = 20+ GB RAM before your apps even start.

---

## The Important Question

At this point, ask yourself a simple question:

> **What is actually required to run an application?**

Let's take **Zoom** as an example.

To run Zoom, do we need:

| Requirement | Needed? |
|-------------|---------|
| Printer Drivers | ❌ No |
| Microsoft Office | ❌ No |
| Adobe Acrobat | ❌ No |
| Zoom application code | ✅ Yes |
| Required libraries | ✅ Yes |
| Required binaries | ✅ Yes |
| Required dependencies | ✅ Yes |

**Zoom only needs its code + libraries + binaries + dependencies. Nothing more.**

If we can package only these things and run them anywhere, we eliminate the OS overhead completely.

This is the idea that led to **containers**.

---

## Option 3: Containers

A container contains:
- Application Code
- Libraries
- Dependencies
- Runtime

Unlike a Virtual Machine, **a container does not carry an entire operating system**.

Containers share the **host operating system kernel**.

### Pros

| # | Advantage | Example |
|---|-----------|---------|
| 1 | Lightweight | MBs instead of GBs |
| 2 | Fast startup | Milliseconds instead of minutes |
| 3 | Efficient | 10 containers = ~2 GB RAM vs 10 VMs = 40+ GB RAM |
| 4 | Portable | Run on any machine with a container runtime |
| 5 | Better density | More apps on same hardware |

### Comparison: VM vs Container

| | Virtual Machine | Container |
|-|-----------------|-----------|
| OS per unit | Full guest OS | Shared host kernel |
| Size | GBs | MBs |
| Startup time | Minutes | Milliseconds |
| Isolation | Hardware-level | Process-level |
| Resource usage | Heavy | Lightweight |

---

## The Most Important Docker Statement

> **A container is not a Virtual Machine.**
> **A container is a Linux Process.**

This is the single most important concept in Docker.

Most students assume a container is some kind of lightweight VM. It is not.

A container is simply a **process running with additional isolation**.

---

## Demonstration: Proving a Container is a Process

**Run a container:**

```bash
docker run -d --name demo nginx
```

**Observe processes on the Linux host:**

```bash
ps -ef | grep nginx
```

You will see the nginx process listed in the host process table — just like any other Linux process.

**Now kill that process directly:**

```bash
kill -9 <PID>
```

**Check the container:**

```bash
docker ps
```

The container is gone.

### What this proves

> A container lives as long as its primary process lives.
> No process = no running container.

---

## Understanding Linux Processes

A **process** is simply a running program.

When an application starts:
1. Linux loads the program into memory
2. Assigns a unique **Process ID (PID)**
3. Allocates CPU time
4. The program begins executing

Every running application on Linux is ultimately a process. Containers are no exception.

**The difference:** containers run as processes with additional isolation mechanisms applied by the Linux kernel.

---

## Process Isolation: How Containers Stay Separate

If containers are just processes, the obvious question is:

- Why can't one container see all processes on the server?
- Why can't one container consume all CPU and memory?
- Why can't one container interfere with another's filesystem?

The answer lies in two Linux kernel features: **Namespaces** and **cgroups**.

---

## Namespaces — Isolation of View

Namespaces control **what a process can see**.

| Namespace | What it isolates |
|-----------|-----------------|
| PID | Process IDs — container sees only its own processes |
| Network | Network interfaces, IP addresses, ports |
| Mount | Filesystem — container sees only its own files |
| UTS | Hostname — container has its own hostname |
| IPC | Inter-process communication |
| User | User and group IDs |

**Example:** Container thinks its main process is PID 1. The host sees it as PID 8374. Same process, different view.

---

## cgroups — Isolation of Resource Usage

Namespaces control what a process *sees*. cgroups (control groups) control what a process *uses*.

cgroups allow limits on:

| Resource | Example limit |
|----------|--------------|
| CPU | Max 1 core out of 8 |
| Memory | Max 512 MB RAM |
| Disk I/O | Max 100 MB/s read/write |
| Network | Bandwidth throttling |

**Why this matters:** Without cgroups, one container can consume all available CPU and starve every other container — the same problem we had with bare metal.

---

## Docker: Putting It All Together

Docker is **not inventing new technology**.

Docker is making existing Linux kernel features **easy to use**.

When you run `docker run`, Docker automatically:

1. Creates **Namespaces** → process isolation
2. Applies **cgroups** → resource limits
3. Sets up **Union Filesystem layers** → image layering
4. Configures **Networking** → container gets its own IP

All of this is packaged behind a single simple command.

```bash
docker run nginx
```

---

## Key Takeaways 

| Concept | What it means |
|---------|---------------|
| Bare metal | Simple but wasteful and insecure at scale |
| Virtual Machine | Good isolation but heavy — full OS per app |
| Container | Lightweight process with OS-level isolation |
| Container = Process | Not a VM, just a Linux process with extra isolation |
| Namespaces | Control what the process can see |
| cgroups | Control what the process can use |
| Docker | Orchestrates namespaces + cgroups + filesystems behind a simple CLI |

> Docker did not invent containers. Docker made containers easy.

---

*End of Day 1*
