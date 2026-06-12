# Container Lifecycle: From Creation to Deletion

---

## Where We Are in the Story

In Part 1, we understood why containers exist.

In Part 2, we understood what a container actually is at the kernel level — a Linux process isolated by namespaces, controlled by cgroups, with a filesystem built from union layers.

In Part 3, we understood the Docker architecture — how CLI, daemon, containerd, and runc work together to turn a docker run command into a running process.

Now there is one critical piece missing.

We know how a container is born. We do not yet know how it lives, how it pauses, how it stops, how it restarts, and how it dies. We do not yet know what happens to data at each stage. We do not yet know the difference between a container that is stopped and a container that is deleted. We do not yet know what signals Docker sends to a process when it wants it to stop.

This is the container lifecycle. And if you do not understand this deeply, you will make mistakes in production that are very difficult to debug.

Let us go through every phase, every state, and every rule.

---

# Section 1: The Two Views of Container Lifecycle

## Concept

The container lifecycle can be understood from two different angles. Both are important.

The first view is the state view. At any given moment, a container is in one of five possible states. Understanding these states tells you what is happening to the container right now.

The second view is the operational view. Over its entire lifetime, a container goes through six distinct phases from the moment an image is built to the moment the container is permanently removed. Understanding these phases tells you what happened to the container over time and what happens to your data at each phase.

Most people learn one or the other. You need both.

---

# Section 2: The Five Container States

## Concept

A container is always in exactly one of these five states at any moment.

### State 1: Created

A container enters the Created state when it has been created but not yet started.

What this means technically: Docker has allocated a container ID, prepared the union filesystem by stacking the image layers and adding a writable layer on top, allocated the namespaces, and written the container configuration to disk. Everything is ready. But no process has been started yet.

The container exists. It is just not running.

How you get here: running docker create instead of docker run. The docker create command prepares the container but stops before starting the process.

Why this state is useful: you can prepare a container in advance and start it later. In automated environments, containers are sometimes created during a setup phase and started during an execution phase.

What is true in this state:
- Container has a container ID
- Filesystem is assembled and writable layer exists
- Namespaces are allocated
- No CPU or memory is being consumed
- No PID exists for this container yet

### State 2: Running

A container enters the Running state when its main process, PID 1, is actively executing.

This is the normal, healthy state for a container. The application is running. It is accepting requests, processing data, writing logs, doing whatever it is designed to do.

What is true in this state:
- PID 1 is executing inside the container namespace
- CPU and memory are being consumed
- cgroup limits are actively enforced
- Network is active, ports are bound
- Filesystem writes go to the writable layer
- Container is visible in docker ps output

### State 3: Paused

A container enters the Paused state when its processes are frozen using the cgroups freezer subsystem.

When you pause a container, the kernel suspends all processes in the container's cgroup. They are not killed. They are not sleeping. They are literally frozen in place — suspended mid-execution. The process still exists in memory. Its state is fully preserved. It is just not receiving any CPU time.

The difference between Paused and Stopped is crucial. A paused container can be resumed in milliseconds because nothing needs to be rebuilt. All the process state is still in memory. A stopped container needs to start PID 1 fresh when restarted.

When to use pausing: when you need to temporarily free up CPU resources without losing the container's in-memory state. Useful in testing and development scenarios where you want to freeze a container's state at a specific point.

What is true in this state:
- All processes are frozen via cgroup freezer
- Container is still in memory
- No CPU is being consumed
- Memory is still consumed — the process state is preserved
- The container is visible in docker ps with status Paused

### State 4: Stopped (Exited)

A container enters the Stopped state when its main process has exited, either normally or because it was killed.

This is one of the most important states to understand because of what is preserved and what is not.

When a container stops:
- The process is gone. PID 1 no longer exists.
- The namespaces are torn down.
- The cgroups are emptied.
- CPU and memory are released back to the host.

But the container is NOT deleted. The container's filesystem, including the writable layer with all the files the container wrote during its lifetime, is still on disk. The container metadata — its ID, name, configuration, and exit code — is still recorded by Docker.

This means a stopped container can be restarted. When it restarts, Docker does not create a new container. It reuses the existing container including its writable layer. Any files that were written to the container during its previous run are still there.

This state is also called Exited in docker ps -a output.

What is true in this state:
- No process is running
- No CPU or memory is consumed
- Container filesystem and writable layer are preserved on disk
- Container metadata is preserved
- Container is NOT visible in docker ps but IS visible in docker ps -a
- Container can be restarted with docker start

### State 5: Deleted (Removed)

A container enters the Deleted state when docker rm is run against it.

Deletion is permanent and irreversible.

When a container is deleted:
- The writable layer is destroyed. All files written by the container during its lifetime are gone forever.
- The container metadata is removed.
- The container ID is freed.
- The namespaces and cgroup configurations are cleaned up.

The image that the container was created from is NOT affected. The image layers are read-only and shared. Deleting a container never touches the image.

Important rule: you cannot delete a running container. You must stop it first, then delete it. Or you can use docker rm -f to force-remove a running container, but this sends SIGKILL immediately without a graceful shutdown.

What is true in this state:
- Container no longer exists in Docker's records
- Writable layer and all data is gone
- Image is unaffected

## State Transition Diagram

```
docker create
     |
     v
  [Created]
     |
docker start / docker run
     |
     v
  [Running] <----------- docker unpause
     |    \                    |
     |     docker pause        |
     |          \              |
     |        [Paused] --------
     |
docker stop / process exits / docker kill
     |
     v
  [Stopped / Exited]
     |         ^
     |         |
     |    docker start (restart)
     |
docker rm
     |
     v
  [Deleted]
```

---

# Section 3: The Six Operational Phases

## Concept

The state view tells you what a container is doing right now. The operational view tells you the full story of a container's lifetime from before it even exists to after it is gone. There are six phases.

---

## Phase 1: Image Building

Before a container can exist, an image must exist.

A Docker image is built from a Dockerfile. The Dockerfile is a text file containing a sequence of instructions. Each instruction adds a new layer to the image.

When Docker builds an image, it processes each instruction in order. For each instruction, Docker creates a temporary container, executes the instruction inside it, takes a snapshot of the resulting filesystem changes, and saves that snapshot as a new read-only layer. This layer is then used as the base for the next instruction.

The result is a stack of read-only layers. Each layer contains only the filesystem changes from that one instruction. The layers are stacked from bottom to top, with the most recent instruction on top.

Key properties of an image:
- It is read-only. Every layer in the image is permanently frozen.
- It is immutable. Once built, an image never changes. If you want a different image, you build a new one.
- It is a template. You can create any number of containers from one image.
- Layers are cached. If you rebuild an image and an instruction has not changed, Docker reuses the cached layer instead of rebuilding it. This makes rebuilds fast.

The image is identified by a SHA256 hash derived from its contents. This is the image ID. You can also give it a human-readable name and tag like nginx:1.25.3.

Commands used in this phase:

```
docker build -t myapp:1.0 .
docker images
docker history myapp:1.0
docker inspect myapp:1.0
```

---

## Phase 2: Container Creation

A container is created when Docker takes an image and adds a thin writable layer on top of it.

The creation phase involves three things:

First, Docker assembles the container filesystem using OverlayFS. It takes all the read-only image layers and stacks them. On top of this stack it adds a new empty writable layer. This writable layer is unique to this container. No other container shares it. This is the layer where all filesystem writes from the container will go during its lifetime.

Second, Docker allocates namespaces for the container. The PID namespace, network namespace, mount namespace, UTS namespace, IPC namespace — all are created and configured.

Third, Docker writes the container configuration to disk. This includes the container ID, the container name, the resource limits, the port mappings, the environment variables, the command to run when started, and all other configuration.

At the end of the creation phase, the container exists on disk. But no process has started. The container is in the Created state.

You can explicitly create a container without starting it:

```
docker create --name myapp -p 8080:80 nginx
```

Or you can create and start in one step using docker run, which is what most people use:

```
docker run -d --name myapp -p 8080:80 nginx
```

The difference is that docker create gives you an opportunity to inspect or configure the container before starting it.

Commands used in this phase:

```
docker create --name myapp nginx
docker ps -a
docker inspect myapp
```

---

## Phase 3: Container Execution

The execution phase begins when Docker starts PID 1 inside the container.

When you run docker start on a created container, or when you run docker run, Docker calls containerd, which calls runc, which starts the process defined in the image's CMD or ENTRYPOINT instruction inside the container's namespaces.

That process becomes PID 1 inside the container's PID namespace. From inside the container, this process is the first and most important process. From outside, on the host, this process has a different PID.

The cgroup limits are now active. If you specified a memory limit of 512 MB, the kernel begins enforcing that limit the moment the process starts.

The network is now active. The container's virtual network interface is up, its IP address is assigned, and any port mappings are active.

The container is now Running.

Everything the process does from this point goes through the kernel's isolation mechanisms:
- Any files it reads come from the union filesystem
- Any files it writes go to the writable layer
- Any processes it spawns become children in the same PID namespace
- Any network traffic goes through the container's network namespace
- Any resource consumption is tracked and limited by cgroups

Commands used in this phase:

```
docker start myapp
docker ps
docker logs myapp
docker logs -f myapp
docker exec -it myapp bash
docker stats myapp
docker top myapp
```

---

## Phase 4: Container Stopping

This phase is one of the most important to understand correctly. Most engineers learn only part of it and pay for that ignorance in production.

When you run docker stop on a running container, Docker does NOT immediately kill the process. It follows a two-step shutdown sequence.

Step 1: Docker sends SIGTERM to PID 1.

SIGTERM is the graceful shutdown signal. It is a request, not a command. A well-written application that receives SIGTERM will finish its current work, close open connections, flush buffers to disk, and exit cleanly. A web server receiving SIGTERM will stop accepting new requests, finish serving requests already in progress, and then shut down.

Step 2: Docker waits for a grace period. The default grace period is 10 seconds.

If PID 1 exits on its own within those 10 seconds, Docker records the exit code and moves the container to Stopped state. Clean shutdown complete.

If PID 1 has NOT exited after 10 seconds, Docker sends SIGKILL.

SIGKILL is not a request. It is a command from the kernel. The process cannot catch it, cannot ignore it, and cannot handle it. The kernel immediately terminates the process. No cleanup happens. Open connections are dropped. Unflushed buffers are lost. The process simply stops existing.

This is a critical point: if your application does not handle SIGTERM, Docker will kill it with SIGKILL after 10 seconds, and it will lose any data that was not written to disk.

You can change the grace period:

```
docker stop --time 30 myapp
```

This gives the application 30 seconds to shut down gracefully before SIGKILL.

You can also bypass the grace period entirely and send SIGKILL immediately:

```
docker kill myapp
```

This is the same as docker stop but with a zero grace period. It sends SIGKILL directly. Use this only when you need to force-stop an unresponsive container.

What is preserved after stopping:
- The writable layer and all files written by the container are still on disk
- The container configuration and metadata are still in Docker's records
- The container ID and name are still valid
- The exit code of PID 1 is recorded

What is released after stopping:
- CPU and memory are returned to the host
- Network ports are released
- The process no longer exists

Commands used in this phase:

```
docker stop myapp
docker stop --time 30 myapp
docker kill myapp
docker ps -a
docker inspect myapp --format '{{.State.ExitCode}}'
```

---

## Phase 5: Container Restart

When a stopped container is restarted, Docker does not create a new container. It reuses the existing one.

This is important. The same container ID, the same writable layer, the same configuration — all of it is reused. Docker simply starts PID 1 again inside the existing container.

This means any files that were written to the container's writable layer during its previous run are still there after restart. If your application wrote a configuration file to /etc/myapp/config.json during its first run, that file is still there when it starts again.

The restart does not create new namespaces from scratch. The container's namespaces are reconfigured and PID 1 is started fresh inside them.

There is also a feature called restart policies. A restart policy tells Docker what to do automatically when a container stops.

The four restart policies are:

no — the default. Docker does nothing when the container stops. It stays in Stopped state.

on-failure — Docker automatically restarts the container if it exited with a non-zero exit code. A zero exit code means the application exited cleanly on purpose. A non-zero exit code means something went wrong. You can limit restart attempts: --restart on-failure:5 means try at most 5 times.

always — Docker always restarts the container when it stops, regardless of exit code. If Docker daemon itself restarts (after a server reboot, for example), containers with this policy will also start automatically.

unless-stopped — same as always, except Docker will not restart the container if it was manually stopped with docker stop before the daemon restarted.

```
docker run -d --restart unless-stopped --name myapp nginx
```

This is the restart policy most commonly used in production. Your application container will survive server reboots automatically, but you can still manually stop it when needed.

Commands used in this phase:

```
docker start myapp
docker restart myapp
docker run -d --restart unless-stopped --name myapp nginx
docker inspect myapp --format '{{.HostConfig.RestartPolicy}}'
```

---

## Phase 6: Container Removal

Container removal is permanent.

When you delete a container, the writable layer is destroyed. Every file that was ever written to that container during its lifetime is gone. There is no recycle bin. There is no recovery.

This is the most important data rule in Docker:

Data written inside a container exists only as long as the container exists.

If you delete the container, you delete the data.

This is not a bug. It is a design principle. Containers are meant to be ephemeral. They are meant to be disposable and replaceable. You should be able to delete a container and create a new one from the same image and have everything work exactly as before.

For data that must survive beyond the lifetime of a container — database files, user uploads, log archives, configuration files — you use Docker volumes or bind mounts. These are storage locations outside the container's writable layer that are mounted into the container. When the container is deleted, the volume persists. We will cover this in detail in the Docker Volumes session.

You cannot delete a running container:

```
docker rm myapp
Error response from daemon: You cannot remove a running container...
```

You must stop it first:

```
docker stop myapp
docker rm myapp
```

Or force-remove it, which sends SIGKILL and deletes in one step:

```
docker rm -f myapp
```

To remove all stopped containers at once:

```
docker container prune
```

When a container is removed:
- The writable layer and all its data are deleted
- Container metadata and logs are deleted
- The container ID is freed
- The image is completely unaffected

Commands used in this phase:

```
docker rm myapp
docker rm -f myapp
docker container prune
docker system prune
```

---

# Section 4: The Four Lifecycle Rules Every Engineer Must Know

## Rule 1: Container lives as long as PID 1 lives

This rule governs everything about container uptime.

The moment PID 1 exits — whether it exited cleanly, crashed, or was killed — the container stops. There is no way to keep a container running without a running PID 1. The container does not have an independent existence separate from its main process.

This is why when you write a Dockerfile, the CMD or ENTRYPOINT must define a foreground process. If your startup command launches a background service and then exits, PID 1 is gone and the container stops immediately even though your service might still be running as an orphaned process.

Wrong approach — starts nginx in background, then the shell exits, container stops:

```
CMD service nginx start
```

Correct approach — nginx runs in foreground as PID 1, container stays alive:

```
CMD ["nginx", "-g", "daemon off;"]
```

This distinction causes more container confusion than any other single concept.

## Rule 2: Stopping a container preserves data. Deleting a container destroys data.

A stopped container is not a deleted container.

When a container stops, the writable layer stays on disk. You can restart the container and all the data from its previous run is still there.

When a container is deleted, the writable layer is gone. The data is gone.

This means:
- docker stop — safe, reversible, data preserved
- docker rm — permanent, data destroyed
- docker rm -f — immediate kill plus permanent deletion, data destroyed

Many engineers confuse these. They stop a container thinking the data is safe, but then someone runs docker container prune and everything is gone. Use volumes for anything that matters.

## Rule 3: Images are immutable. Containers are mutable.

An image never changes. From the moment it is built, every layer in the image is frozen. You cannot modify an image. If you want a different image, you build a new one.

A container can change. Anything written to the container's writable layer represents a change from the original image state. The container is a live, mutable instance of the image.

The relationship is like a class and an object in programming. The image is the class — it defines the template. The container is the object — it is a live instance with its own state.

This has a practical implication: never rely on changes made inside a running container for anything important. If you install a package inside a running container, that change lives only in the writable layer of that specific container. When the container is deleted, the change is gone. If you want that change to persist, rebuild the image with the change baked into a new layer.

## Rule 4: One process per container is a best practice, not a hard rule

Docker does not enforce one process per container. You can run multiple processes inside a container using a process manager or an init system.

But the principle exists for good reasons.

When a container runs one process, it has one job. It is easy to scale — you just run more containers. It is easy to replace — you delete and recreate without side effects on other services. It is easy to monitor — one process, one set of logs, one health check.

When a container runs multiple processes, you lose these benefits. Scaling becomes complicated. Failures are harder to diagnose. The container becomes a mini-server, which is exactly what containers were designed to replace.

The rule is: one process per container. One concern per container. Design your application as separate containers and connect them — that is what Docker Compose and Kubernetes are for.

---

# Section 5: Container Lifecycle and Data — The Complete Picture

## Concept

Let us be completely precise about what happens to data at every stage.

| Phase | What happens to writable layer data |
|-------|-------------------------------------|
| Created | Writable layer exists but is empty |
| Running | All container writes go to writable layer |
| Paused | Writable layer unchanged, process frozen in memory |
| Stopped | Writable layer fully preserved on disk |
| Restarted | Writable layer from previous run still present |
| Deleted | Writable layer permanently destroyed |

| Storage type | Survives container stop | Survives container delete |
|---|---|---|
| Container writable layer | Yes | No |
| Docker volume | Yes | Yes |
| Bind mount (host directory) | Yes | Yes |

The rule is simple. If data lives inside the container's writable layer, it is as temporary as the container itself. If data lives in a volume or bind mount, it is independent of the container lifecycle.

---

# Section 6: Exit Codes — Reading What the Container Tells You

## Concept

When PID 1 exits, it returns an exit code. Docker records this exit code. The exit code tells you whether the container stopped cleanly or because of a problem.

| Exit Code | Meaning |
|-----------|---------|
| 0 | Clean exit. Process completed successfully and exited on its own. |
| 1 | Application error. The process exited because of an error in the application code. |
| 137 | Process was killed by SIGKILL (128 + 9). Docker sent SIGKILL, or the OOM killer terminated it. |
| 143 | Process received SIGTERM (128 + 15) and exited. Graceful shutdown completed. |
| 125 | Docker run itself failed. The container could not start. |
| 126 | Command found but could not be executed. |
| 127 | Command not found inside the container. |

Exit code 137 is the one that matters most in production. If you see a container repeatedly exiting with code 137, it is being killed by the out-of-memory killer. Your container is exceeding its memory limit and the kernel is terminating it. The fix is either to increase the memory limit or to fix the memory leak in your application.

Read exit codes with:

```
docker inspect myapp --format '{{.State.ExitCode}}'
docker ps -a
```

---

# Section 7: Complete Lab — Walking the Full Lifecycle

## Lab

This lab walks through every state and every phase in sequence.

Start with a clean environment:

```
docker rm -f $(docker ps -aq) 2>/dev/null || true
```

Phase 1 — Build or pull an image:

```
docker pull nginx:1.25
docker images
docker history nginx:1.25
```

Phase 2 — Create without starting:

```
docker create --name lifecycle-demo -p 8080:80 nginx:1.25
docker ps
docker ps -a
```

Note that docker ps shows nothing. docker ps -a shows the container in Created state.

Phase 3 — Start the container:

```
docker start lifecycle-demo
docker ps
```

Now it appears in docker ps as running. Find its PID on the host:

```
docker inspect lifecycle-demo --format '{{.State.Pid}}'
ps -ef | grep nginx
```

Write a file inside the running container:

```
docker exec lifecycle-demo bash -c "echo 'I was written at runtime' > /tmp/testfile.txt"
docker exec lifecycle-demo cat /tmp/testfile.txt
```

Pause the container:

```
docker pause lifecycle-demo
docker ps
```

Try to reach the container while paused. It will not respond — the process is frozen.

Unpause the container:

```
docker unpause lifecycle-demo
docker exec lifecycle-demo cat /tmp/testfile.txt
```

The file is still there. The process resumed exactly where it was.

Phase 4 — Stop the container gracefully:

```
docker stop lifecycle-demo
docker ps
docker ps -a
```

The container is now in Exited state. Check the exit code:

```
docker inspect lifecycle-demo --format '{{.State.ExitCode}}'
```

Exit code 0 means nginx received SIGTERM and exited cleanly.

Phase 5 — Restart and verify data:

```
docker start lifecycle-demo
docker exec lifecycle-demo cat /tmp/testfile.txt
```

The file is still there. The writable layer was preserved during the stop and reused on restart.

Phase 6 — Delete the container:

```
docker stop lifecycle-demo
docker rm lifecycle-demo
docker ps -a
```

Now create a fresh container from the same image and check for the file:

```
docker run -d --name fresh-start nginx:1.25
docker exec fresh-start cat /tmp/testfile.txt
```

The file does not exist. Fresh container from the same image, clean writable layer. The file lived in the deleted container's writable layer and is gone.

Demonstrate restart policy:

```
docker run -d --name always-on --restart unless-stopped nginx:1.25
docker stop always-on
docker ps -a
docker start always-on
docker inspect always-on --format '{{.HostConfig.RestartPolicy.Name}}'
```

Demonstrate force removal:

```
docker rm -f always-on
docker ps -a
```

Gone in one command, no separate stop required.

---

# Section 8: Key Takeaways

## The Five States

| State | Process running | Data on disk | CPU consumed |
|-------|----------------|--------------|--------------|
| Created | No | Yes (empty writable layer) | No |
| Running | Yes | Yes (growing) | Yes |
| Paused | Frozen in memory | Yes | No |
| Stopped | No | Yes (preserved) | No |
| Deleted | No | No | No |

## The Six Phases

| Phase | Command | What happens |
|-------|---------|--------------|
| Image Building | docker build | Layers created, image assembled |
| Container Creation | docker create / docker run | Writable layer added, namespaces allocated |
| Container Execution | docker start | PID 1 starts, cgroups active |
| Container Stopping | docker stop | SIGTERM then SIGKILL, writable layer preserved |
| Container Restart | docker start | Same container reused, writable layer intact |
| Container Removal | docker rm | Writable layer destroyed permanently |

## The Four Rules

1. Container lives as long as PID 1 lives. PID 1 exits, container stops.
2. Stopping preserves data. Deleting destroys data.
3. Images are immutable. Containers are mutable instances of images.
4. One process per container. Keep containers focused on a single job.

---

*End of Part 4 - Next: Basic Linux Commands to Understand Containers*
