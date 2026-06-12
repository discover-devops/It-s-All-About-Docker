# Linux Commands: Seeing Containers From the Inside Out

---

## Where We Are in the Story

In Part 1, we understood why containers exist.

In Part 2, we proved a container is a Linux process and understood namespaces, cgroups, and OverlayFS.

In Part 3, we understood the full Docker architecture from CLI to runc.

In Part 4, we walked every state and phase of the container lifecycle.

Now we do something different.

We stop taking Docker's word for it.

Everything we have discussed — namespaces, cgroups, union filesystems, process isolation, network separation — all of it is visible on the Linux host using standard commands. No special tools. No Docker-specific commands. Just the same Linux commands that have existed for decades.

This section is about building the habit of looking underneath Docker and seeing what is actually happening at the kernel level. Engineers who have this habit debug problems faster, design better systems, and never feel like containers are a black box.

By the end of this section, you will be able to look at any running container and answer these questions using Linux commands alone:

- What is the actual PID of this container's process on the host?
- What namespaces is this container using?
- What are the exact resource limits the kernel is enforcing?
- What filesystem layers make up this container?
- What network interface does this container have and what is its IP address?
- What ports is this container listening on?

---

# Section 1: The Master Workflow

## Concept

Before we go through individual commands, understand the master workflow. This is the sequence you will use every time you want to look inside a container from the host.

```
Step 1: docker ps                        Find the container name or ID
Step 2: docker inspect <name>            Get the host PID and full configuration
Step 3: ps -ef | grep <PID>              Verify the process on the host
Step 4: lsns -p <PID>                    See all namespaces for this process
Step 5: cat /proc/<PID>/cgroup           See the cgroup assignments
Step 6: mount | grep overlay             See the filesystem layers
Step 7: ip addr show                     Compare host network with container network
Step 8: ss -tulpn                        See what ports are actually bound
```

Run through this sequence once and you will never see containers as magical again.

---

# Section 2: Process Inspection Commands

## Concept

These commands reveal container processes as they truly exist on the host. The container thinks it owns PID 1. The host sees the real story.

---

### Command: ps aux

ps aux shows every running process on the system with resource usage.

The columns are:

| Column | Meaning |
|--------|---------|
| USER | The OS user running the process |
| PID | Process ID on the host |
| %CPU | Percentage of CPU being used |
| %MEM | Percentage of memory being used |
| VSZ | Virtual memory size in kilobytes |
| RSS | Resident Set Size — actual physical memory in use |
| TTY | Terminal associated with process. ? means no terminal |
| STAT | Process state — S is sleeping, R is running, Z is zombie |
| START | When the process started |
| TIME | Total CPU time consumed |
| COMMAND | The command that started the process |

## Lab

Start a container:

```
docker run -d --name ps-demo nginx
```

Run ps aux and find the nginx processes:

```
ps aux | grep nginx
```

You will see something like this:

```
root      4821  0.0  0.0  nginx: master process nginx -g daemon off;
101       4892  0.0  0.0  nginx: worker process
```

These are real host processes. The container does not hide them from the host. The container hides the host processes from the container. That is the direction namespaces work — inward, not outward.

Notice the USER column. The master process runs as root. The worker runs as user 101. This is what the nginx image was built to do. Understanding which user your container runs as matters for security.

---

### Command: ps -ef

ps -ef shows every process with the parent-child relationship visible through the PPID column.

The columns are:

| Column | Meaning |
|--------|---------|
| UID | User ID of the process owner |
| PID | Process ID |
| PPID | Parent Process ID — who created this process |
| C | CPU utilization |
| STIME | Start time |
| TTY | Terminal |
| TIME | Cumulative CPU time |
| CMD | Full command with arguments |

## Lab

Run ps -ef and look for the container process:

```
ps -ef | grep nginx
```

Now look at the PPID of the nginx master process. That PPID is the containerd-shim process. The containerd-shim is a small process that containerd creates to act as the parent of each container process. It keeps the container process running independently of the containerd daemon itself.

Run this to see the full chain:

```
ps -ef | grep -E "containerd|nginx"
```

You will see:

```
containerd            (the main containerd process)
  containerd-shim     (one per container, child of containerd)
    nginx master      (child of containerd-shim, PID 1 inside container)
      nginx worker    (child of nginx master)
```

This is the real process tree. Docker → daemon → containerd → shim → your container process. The shim exists so that if containerd restarts, your container process does not die. The shim holds the container alive independently.

---

### Command: top and htop

top shows a live, continuously updating view of processes sorted by CPU usage by default.

## Lab

Run top while a container is running:

```
top
```

Press M to sort by memory usage. Press P to sort by CPU. Press q to quit.

You will see container processes listed exactly like any other system process. There is no visual indicator that a process belongs to a container. From top's perspective, a container process is just a process.

htop is an improved version of top with color, mouse support, and a process tree view. Install it if it is not present:

```
apt-get install htop -y
htop
```

In htop, press F5 to switch to tree view. You will see the full process hierarchy including containerd, the shim, and your container processes nested inside.

---

### Command: pstree

pstree displays the process hierarchy as a tree diagram. It makes the parent-child relationships immediately visible.

## Lab

```
pstree -p
```

Find the containerd branch. Under it you will see the shim processes. Under each shim you will see the container process. This tree view makes it obvious that containers are just branches of the normal Linux process tree, rooted at PID 1.

To see only the docker-related branch:

```
pstree -p | grep -A 5 containerd
```

---

# Section 3: Namespace Inspection Commands

## Concept

These commands let you see exactly which namespaces a container process is using. This is the direct proof that container isolation exists at the kernel level.

---

### Command: lsns

lsns lists all namespaces currently in use on the system.

The columns are:

| Column | Meaning |
|--------|---------|
| NS | Namespace inode number — a unique identifier |
| TYPE | The type of namespace |
| NPROCS | Number of processes using this namespace |
| PID | PID of the process that created this namespace |
| USER | User who owns this namespace |
| COMMAND | Command associated with the creating process |

The namespace types you will see:

| Type | What it isolates |
|------|-----------------|
| pid | Process IDs |
| net | Network interfaces, IP addresses, routing |
| mnt | Filesystem mount points |
| uts | Hostname and domain name |
| ipc | Inter-process communication |
| user | User and group IDs |

## Lab

First see all namespaces on the system with no containers running:

```
lsns
```

Now start a container:

```
docker run -d --name ns-demo nginx
```

Run lsns again:

```
lsns
```

You will see new namespace entries appear. These are the namespaces Docker created for this container.

Get the host PID of the container:

```
docker inspect ns-demo --format '{{.State.Pid}}'
```

Now list only the namespaces belonging to that specific process:

```
lsns -p <PID>
```

You will see exactly six namespaces listed — one of each type — all belonging to the container process. These are the walls of isolation around your container.

Run the same for a second container:

```
docker run -d --name ns-demo2 nginx
docker inspect ns-demo2 --format '{{.State.Pid}}'
lsns -p <PID of second container>
```

Compare the namespace inode numbers between the two containers. Every number is different. Every container lives in completely separate namespaces. The kernel enforces this.

---

### Command: Reading namespaces from /proc

Every process on Linux has a directory under /proc that contains everything the kernel knows about it. The namespace information lives in /proc/PID/ns/.

## Lab

Get the container PID:

```
docker inspect ns-demo --format '{{.State.Pid}}'
```

List the namespace files for that process:

```
ls -la /proc/<PID>/ns/
```

You will see symbolic links like:

```
lrwxrwxrwx ipc -> ipc:[4026532456]
lrwxrwxrwx mnt -> mnt:[4026532454]
lrwxrwxrwx net -> net:[4026532459]
lrwxrwxrwx pid -> pid:[4026532457]
lrwxrwxrwx uts -> uts:[4026532455]
lrwxrwxrwx user -> user:[4026531837]
```

The number in brackets is the namespace inode. Compare the net namespace inode of your container with the net namespace inode of a host process:

```
ls -la /proc/1/ns/net
ls -la /proc/<container PID>/ns/net
```

Different inode numbers means different network namespaces. That is why the container has a different IP address — it is literally in a different network namespace at the kernel level.

---

### Command: ip netns

ip netns is the iproute2 tool for working with network namespaces directly.

## Lab

List network namespaces visible to the system:

```
ip netns list
```

Note: Docker-created network namespaces may not appear here by default because Docker does not register them with the system namespace list. But you can still enter them using nsenter.

Enter the container's network namespace using nsenter:

```
docker inspect ns-demo --format '{{.State.Pid}}'
nsenter -t <PID> -n ip addr show
```

nsenter runs a command inside a specific namespace. The -t flag specifies the target PID. The -n flag says to enter the network namespace of that process. Then ip addr show runs inside that namespace.

Compare this output with ip addr show run on the host:

```
ip addr show
```

The container has a different interface name and a different IP address. Same physical machine. Different namespace. Different network view.

---

# Section 4: cgroup Inspection Commands

## Concept

cgroups are visible on the filesystem under /sys/fs/cgroup. The kernel exposes cgroup information as files you can read directly. This means you can see the exact resource limits being enforced — not what Docker thinks is happening, but what the kernel is actually doing.

---

### Command: Reading cgroups from /proc

Every process's cgroup memberships are listed in /proc/PID/cgroup.

## Lab

Start a container with explicit resource limits:

```
docker run -d --name cgroup-demo --memory 256m --cpus 0.5 nginx
```

Get the container PID:

```
docker inspect cgroup-demo --format '{{.State.Pid}}'
```

Read the cgroup file for this process:

```
cat /proc/<PID>/cgroup
```

You will see output like:

```
12:memory:/docker/a3f4b2c1d9e8...
11:cpu,cpuacct:/docker/a3f4b2c1d9e8...
10:blkio:/docker/a3f4b2c1d9e8...
```

Each line shows which cgroup hierarchy the process belongs to and the path within that hierarchy. The long hex string is the container ID.

---

### Command: Reading actual limits from /sys/fs/cgroup

The cgroup filesytem under /sys/fs/cgroup contains the actual enforcement values the kernel is using.

## Lab

Read the memory limit the kernel is enforcing:

```
cat /sys/fs/cgroup/memory/docker/<container-id>/memory.limit_in_bytes
```

You will see a number like 268435456, which is exactly 256 MB in bytes. The kernel will not allow this container to use more than this. This is not a Docker-level restriction. This is enforced at the kernel level.

Read the current memory usage:

```
cat /sys/fs/cgroup/memory/docker/<container-id>/memory.usage_in_bytes
```

Read the CPU quota:

```
cat /sys/fs/cgroup/cpu/docker/<container-id>/cpu.cfs_quota_us
cat /sys/fs/cgroup/cpu/docker/<container-id>/cpu.cfs_period_us
```

The quota divided by the period gives you the CPU allocation as a fraction of one core. With --cpus 0.5, you will see quota of 50000 and period of 100000, giving 0.5 CPU.

Read memory statistics in detail:

```
cat /sys/fs/cgroup/memory/docker/<container-id>/memory.stat
```

This file contains a complete breakdown of how memory is being used by the container — cache, RSS, mapped files, and more. This is exactly what monitoring tools read to generate container memory metrics.

---

### Command: docker stats

docker stats gives you a live view of container resource usage. It reads from the cgroup filesystem and presents it in a human-readable format.

## Lab

```
docker stats
```

Or for a specific container:

```
docker stats cgroup-demo
```

The columns are:

| Column | Source |
|--------|--------|
| CPU % | cpu.stat in cgroup cpu subsystem |
| MEM USAGE / LIMIT | memory.usage_in_bytes / memory.limit_in_bytes |
| MEM % | Calculated from the above two |
| NET I/O | Network interface statistics from network namespace |
| BLOCK I/O | blkio.throttle statistics from cgroup blkio subsystem |
| PIDS | pids.current from cgroup pids subsystem |

Everything in docker stats comes from the cgroup filesystem. There is no separate monitoring agent. Docker reads the same kernel files you just read manually.

---

# Section 5: Filesystem Commands

## Concept

These commands reveal the union filesystem structure that makes container images work. You can see the exact layers, where they are stored on disk, and how OverlayFS assembles them into the container's view.

---

### Command: mount

mount shows all filesystems currently mounted on the system. For containers, the critical entries are the overlay mounts.

## Lab

Start a container:

```
docker run -d --name fs-demo nginx
```

Show only the overlay mounts:

```
mount | grep overlay
```

You will see an entry like:

```
overlay on /var/lib/docker/overlay2/<id>/merged type overlay
(rw,lowerdir=/var/lib/docker/overlay2/<layer1>:
           /var/lib/docker/overlay2/<layer2>:
           /var/lib/docker/overlay2/<layer3>,
 upperdir=/var/lib/docker/overlay2/<id>/diff,
 workdir=/var/lib/docker/overlay2/<id>/work)
```

Break this down:

lowerdir is the stack of read-only image layers, separated by colons. Each entry is one layer from the nginx image. The order matters — layers are listed from top to bottom. The leftmost layer in lowerdir is the most recently added layer. The rightmost is the base layer.

upperdir is the writable layer for this specific container. This is where all writes from the container go. This is the layer that gets destroyed when the container is deleted.

workdir is an internal working directory required by OverlayFS for atomic file operations. You can ignore this.

merged is the unified filesystem view that the container sees. When you exec into a container, the filesystem you are walking is this merged directory.

---

### Command: Exploring Docker's storage on disk

## Lab

See all overlay2 layer directories Docker has stored:

```
ls /var/lib/docker/overlay2/
```

Each directory here is either an image layer or a container writable layer. Count the directories. You will see more directories than containers because many directories are shared image layers used by multiple containers.

Look inside one layer directory:

```
ls /var/lib/docker/overlay2/<layer-id>/
```

You will see diff, link, lower, and work directories. The diff directory contains the actual filesystem content of that layer — the files that were added or changed by that layer's instruction.

Look at the diff directory of one layer:

```
ls /var/lib/docker/overlay2/<layer-id>/diff/
```

You are looking at the raw files that make up one layer of the nginx image. This is what gets stacked by OverlayFS to create the container's filesystem.

Start two containers from the same image and check how many new layers appear:

```
docker run -d --name fs-demo2 nginx
docker run -d --name fs-demo3 nginx
ls /var/lib/docker/overlay2/ | wc -l
```

Only two new directories will have appeared for two new containers — one writable layer per container. The image layers are shared. This is the storage efficiency of union filesystems made visible.

---

### Command: df and du

df shows disk space usage per filesystem. du shows disk space usage per directory.

## Lab

See the total disk usage of all Docker data:

```
du -sh /var/lib/docker/
```

See the disk usage broken down by category:

```
du -sh /var/lib/docker/overlay2/
du -sh /var/lib/docker/containers/
du -sh /var/lib/docker/volumes/
```

See Docker's own summary of disk usage:

```
docker system df
```

This shows how much space is used by images, containers, volumes, and build cache, and how much of that is reclaimable by pruning.

---

### Command: chroot (conceptual demonstration)

chroot changes the root directory of a process. It is one of the oldest forms of process isolation in Linux and is conceptually the ancestor of container filesystem isolation.

## Lab

This is a conceptual demonstration to show where the container filesystem idea originated.

Create a minimal directory structure:

```
mkdir -p /tmp/miniroot/{bin,lib,lib64}
cp /bin/bash /tmp/miniroot/bin/
ldd /bin/bash | grep -o '/lib[^ ]*' | xargs -I{} cp {} /tmp/miniroot/lib/ 2>/dev/null || true
```

Enter the chroot jail:

```
chroot /tmp/miniroot /bin/bash
```

Inside, try to navigate above the root:

```
ls /
cd ..
ls /
```

You cannot escape. The process's view of the filesystem starts at /tmp/miniroot on the host but appears as / from inside.

This is the conceptual foundation of mount namespaces. A container's mount namespace is a more sophisticated version of chroot — it provides a private root filesystem view using OverlayFS instead of a copied directory tree.

Exit the chroot:

```
exit
```

---

# Section 6: Network Commands

## Concept

These commands reveal the network isolation that container network namespaces provide. You can see that a container truly has its own network stack — its own interfaces, its own IP address, its own routing table, and its own open ports.

---

### Command: ip addr show

ip addr show displays all network interfaces and their IP addresses.

## Lab

Show host network interfaces:

```
ip addr show
```

Note the interfaces. You will likely see:
- lo — the loopback interface
- eth0 or ens3 — the main host network interface
- docker0 — the Docker bridge network. This is a virtual switch created by Docker.
- veth interfaces — one created per running container, connecting the container to docker0

Now show the network interfaces from inside a container:

```
docker exec fs-demo ip addr show
```

Inside the container you will see:
- lo — the container's own loopback
- eth0 — the container's virtual ethernet interface, connected to the veth on the host

The container's eth0 has a different IP address than the host. It is on the 172.17.0.0/16 subnet by default — Docker's internal bridge network. This is the network namespace doing exactly what it was designed to do.

---

### Command: ip link show

ip link show displays network interfaces without IP address information — just the interface names and their state.

## Lab

On the host:

```
ip link show
```

Look for entries starting with veth. Each veth interface is one end of a virtual ethernet pair. The other end lives inside a container's network namespace. This is how traffic flows between the host and the container — through a veth pair bridged by docker0.

Count the veth interfaces and count the running containers:

```
ip link show | grep veth | wc -l
docker ps | tail -n +2 | wc -l
```

The numbers will match. One veth pair per running container.

---

### Command: ss and netstat

ss shows socket statistics — what ports are open and what process owns them. It is the modern replacement for netstat.

## Lab

Show all listening ports on the host:

```
ss -tulpn
```

The flags mean:
- t — show TCP sockets
- u — show UDP sockets
- l — show only listening sockets
- p — show the process that owns the socket
- n — show numeric addresses instead of resolving hostnames

Start a container with a port mapping:

```
docker run -d --name port-demo -p 8080:80 nginx
```

Run ss again and look for port 8080:

```
ss -tulpn | grep 8080
```

You will see Docker proxy listening on 0.0.0.0:8080. This is the Docker userspace proxy that forwards traffic from the host port to the container port.

Now check what is listening inside the container:

```
docker exec port-demo ss -tulpn
```

Inside the container, nginx is listening on port 80. On the host, the traffic arrives on port 8080 and Docker forwards it to port 80 inside the container's network namespace.

---

### Command: ip route

ip route shows the routing table — how the system decides where to send network traffic.

## Lab

Show the host routing table:

```
ip route
```

You will see a route for 172.17.0.0/16 via docker0. This tells the host that traffic destined for the Docker internal network should go through the docker0 bridge interface.

Show the routing table from inside a container:

```
docker exec port-demo ip route
```

Inside the container, the default route goes through 172.17.0.1, which is the docker0 interface on the host. The container uses the host as its gateway to reach the outside world.

---

# Section 7: The Complete Proof Sequence

## Lab

This is the sequence to run every time you want to fully understand what a container is doing. Run this on any container and you will know everything about it.

Start with a container to examine:

```
docker run -d --name proof-demo --memory 256m --cpus 0.5 -p 9090:80 nginx
```

Step 1 — Find the container:

```
docker ps
```

Step 2 — Get the host PID and full details:

```
docker inspect proof-demo --format 'PID: {{.State.Pid}}'
docker inspect proof-demo --format 'IP: {{.NetworkSettings.IPAddress}}'
docker inspect proof-demo --format 'Memory: {{.HostConfig.Memory}}'
```

Step 3 — Confirm the process exists on the host:

```
ps -ef | grep nginx
```

The PID from docker inspect and the PID in ps output will match.

Step 4 — Examine the namespaces:

```
lsns -p <PID>
```

Six namespaces. This process is isolated in every dimension Linux supports.

Step 5 — Read the filesystem namespaces from /proc:

```
ls -la /proc/<PID>/ns/
```

Step 6 — Confirm the cgroup limits at the kernel level:

```
cat /proc/<PID>/cgroup
cat /sys/fs/cgroup/memory/docker/<container-id>/memory.limit_in_bytes
```

Step 7 — See the overlay filesystem:

```
mount | grep overlay
```

Step 8 — Compare network from host and container:

```
ip addr show | grep -A2 docker0
docker exec proof-demo ip addr show
```

Step 9 — See the open ports:

```
ss -tulpn | grep 9090
docker exec proof-demo ss -tulpn
```

You have now fully verified from the Linux host that this container:
- Is a real Linux process with a real host PID
- Is isolated in six namespaces
- Has its resource consumption capped by cgroups at the kernel level
- Has a filesystem assembled from OverlayFS layers
- Has its own network interface and IP address
- Has port mapping connecting host port 9090 to container port 80

Nothing was taken on faith. You read it directly from the kernel.

---

# Section 8: Quick Reference Card

## Process Commands

| Command | What it shows |
|---------|--------------|
| ps aux | All processes with resource usage |
| ps -ef | All processes with parent-child relationships |
| pstree -p | Process tree with PIDs |
| top | Live process viewer sorted by CPU |
| htop | Enhanced live process viewer with tree mode |

## Namespace Commands

| Command | What it shows |
|---------|--------------|
| lsns | All namespaces on the system |
| lsns -p PID | Namespaces of a specific process |
| ls -la /proc/PID/ns/ | Namespace inode numbers for a process |
| nsenter -t PID -n ip addr show | Run command inside process network namespace |
| ip netns list | Network namespaces registered with iproute2 |

## cgroup Commands

| Command | What it shows |
|---------|--------------|
| cat /proc/PID/cgroup | cgroup memberships of a process |
| cat /sys/fs/cgroup/memory/docker/ID/memory.limit_in_bytes | Enforced memory limit |
| cat /sys/fs/cgroup/memory/docker/ID/memory.usage_in_bytes | Current memory usage |
| cat /sys/fs/cgroup/cpu/docker/ID/cpu.cfs_quota_us | CPU quota in microseconds |
| docker stats | Live resource usage from cgroup data |

## Filesystem Commands

| Command | What it shows |
|---------|--------------|
| mount | grep overlay | All OverlayFS mounts for running containers |
| ls /var/lib/docker/overlay2/ | All image and container layers on disk |
| docker system df | Disk usage by images, containers, volumes |
| du -sh /var/lib/docker/ | Total Docker disk usage |

## Network Commands

| Command | What it shows |
|---------|--------------|
| ip addr show | All network interfaces and IPs on host |
| ip link show | grep veth | Virtual ethernet pairs — one per container |
| ip route | Routing table including Docker network routes |
| ss -tulpn | All listening sockets with owning process |
| docker exec NAME ip addr show | Network interfaces inside container |
| docker exec NAME ss -tulpn | Ports listening inside container |

---

*End of Part 5 - Day 1 Complete*

*Day 2: Dockerfile Deep Dive — Writing, Building, and Optimizing Container Images*
