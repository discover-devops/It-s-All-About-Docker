# From Image to Running Container: Essential Commands and Operations

---

## Where We Are in the Story

In Part 1, we built a Docker image from a Dockerfile. We watched every instruction become a layer. We inspected those layers on disk.

Now the image exists. It is sitting on disk. It is doing nothing.

The next question is: how does that image become a running container? And once it is running, how do we operate it — inspect it, interact with it, monitor it, stop it, remove it?

This is the operational layer. Every Docker user needs these commands to the point of muscle memory. By the end of Part 2, you will be comfortable with the full image and container command set.

---

# Section 1: From Image to Running Container

## Concept

When you run docker run, Docker takes a static image on disk and turns it into a running process. Here is precisely what happens between the image and the container.

The image provides:
- A stack of read-only filesystem layers
- The CMD or ENTRYPOINT defining what process to run
- Environment variables, working directory, exposed ports

Docker adds:
- A writable layer on top of the image layers (the container layer)
- Six Linux namespaces for isolation
- A cgroup for resource limits
- A virtual network interface with an IP address

The result is a container — a live, running process with a complete filesystem that looks and behaves like its own isolated system, but is actually a process on the host.

The filesystem the container sees is called the Unified Filesystem. It is the result of OverlayFS merging all the read-only image layers with the writable container layer into a single coherent view.

```
Writable Layer (container layer)   <- writes go here, destroyed on docker rm
----------------------------------
Image Layer 4: app code            <- read-only
Image Layer 3: dependencies        <- read-only
Image Layer 2: base packages       <- read-only
Image Layer 1: base OS             <- read-only
```

The container sees all of this as a single filesystem starting at /. Files can be read from any layer. Writes always go to the writable layer. If a container modifies a file from a read-only layer, the kernel performs a copy-on-write operation — the file is copied to the writable layer and the modification is made there. The original in the read-only layer is never touched.

This is why:
- Containers start in milliseconds — the read-only layers are already on disk, only a writable layer needs to be created
- Multiple containers from the same image share the same read-only layers — only their writable layers are unique
- Deleting a container destroys the writable layer — all writes are lost unless persisted to a volume

---

# Section 2: Essential Container Commands

## Concept

Every Docker operator needs these commands fluently. We will go through each one with its complete syntax, all important flags, and labs that demonstrate real behavior.

---

## docker run

docker run is the single most important command in Docker. It creates a container from an image and starts it in one step.

Full syntax:

```
docker run [options] image [command] [arguments]
```

### The Most Important Flags

**-d (detached mode)**

Runs the container in the background. The terminal is returned to you immediately. The container continues running.

```
docker run -d nginx
```

Without -d, the container runs in the foreground. The terminal is attached to the container's stdout. You must Ctrl+C to detach, which also stops the container.

**-it (interactive + TTY)**

Allocates a pseudo-TTY and keeps stdin open. Used when you want an interactive shell inside the container.

```
docker run -it ubuntu bash
```

-i keeps stdin open even when not attached. -t allocates a TTY. Together, they give you a proper interactive terminal session inside the container.

**--name**

Assigns a human-readable name to the container. Without this, Docker generates a random name like laughing_pasteur or focused_wiles.

```
docker run -d --name webserver nginx
```

Always name your containers in any serious environment. Named containers are much easier to manage.

**-p (port mapping)**

Maps a port on the host to a port inside the container.

```
docker run -d -p 8080:80 nginx
```

Format is hostPort:containerPort. Port 8080 on the host forwards to port 80 inside the container. Multiple -p flags for multiple ports.

```
docker run -d -p 8080:80 -p 8443:443 nginx
```

**-e (environment variable)**

Sets an environment variable inside the container at runtime. Overrides ENV values from the Dockerfile.

```
docker run -d -e DB_HOST=prod-db.example.com -e DB_PORT=5432 myapp:1.0
```

**-v (volume or bind mount)**

Mounts a host directory or Docker volume into the container.

```
docker run -d -v /host/data:/container/data myapp:1.0
docker run -d -v myvolume:/container/data myapp:1.0
```

The first form is a bind mount — the host directory is mounted directly. The second form uses a named Docker volume managed by Docker. Data written to the mounted path persists beyond the container's lifetime.

**--rm**

Automatically removes the container when it exits. Clean up without a separate docker rm command.

```
docker run --rm ubuntu echo "hello and goodbye"
```

Perfect for one-off tasks and commands where you do not need the container to persist.

**--memory and --cpus**

Resource limits applied via cgroups.

```
docker run -d --memory 512m --cpus 1.5 myapp:1.0
```

--memory accepts values like 256m, 1g, 512M. --cpus accepts decimal values where 1.0 is one full core and 0.5 is half a core.

**--restart**

Defines the restart policy.

```
docker run -d --restart unless-stopped myapp:1.0
```

Restart policies: no (default), on-failure, on-failure:5 (max 5 retries), always, unless-stopped.

**--network**

Connects the container to a specific Docker network.

```
docker run -d --network mynetwork myapp:1.0
```

**--entrypoint**

Overrides the ENTRYPOINT defined in the Dockerfile.

```
docker run --entrypoint /bin/bash myapp:1.0
```

## Lab — Exploring docker run

Run a container with multiple options:

```
docker run -d \
  --name myapp \
  -p 8080:8080 \
  -e APP_ENV=development \
  --memory 256m \
  --cpus 0.5 \
  --restart unless-stopped \
  myapp:1.0
```

Verify it is running:

```
docker ps
curl http://localhost:8080
```

---

## docker ps

docker ps lists running containers. docker ps -a lists all containers including stopped ones.

```
docker ps
docker ps -a
```

Column breakdown:

| Column | Meaning |
|--------|---------|
| CONTAINER ID | Short 12-character container ID |
| IMAGE | Image the container was created from |
| COMMAND | The command running as PID 1 |
| CREATED | How long ago the container was created |
| STATUS | Current state: Up X minutes, Exited (0) 5 minutes ago |
| PORTS | Port mappings — host:container |
| NAMES | Human-readable name |

The STATUS column tells you everything:
- Up 2 hours — running, healthy
- Exited (0) — stopped with clean exit
- Exited (137) — killed by SIGKILL, likely OOM
- Exited (1) — stopped due to application error
- Restarting — crash loop

Useful flags:

```
docker ps -q              # only container IDs, useful for scripting
docker ps -a -q           # all container IDs including stopped
docker ps --filter status=exited    # only stopped containers
docker ps --filter name=myapp       # filter by name
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"  # custom output
```

---

## docker logs

docker logs retrieves the stdout and stderr output from a container.

```
docker logs myapp
```

Important flags:

```
docker logs -f myapp              # follow — stream logs in real time like tail -f
docker logs --tail 100 myapp      # show only last 100 lines
docker logs -t myapp              # include timestamps
docker logs --since 1h myapp      # logs from the last hour
docker logs --since 2024-01-15 myapp   # logs since a specific date
```

## Lab

Generate some logs then view them:

```
docker run -d --name log-demo nginx
curl http://localhost:8080  # if you have port mapping
docker logs log-demo
docker logs -f log-demo
```

In a second terminal, make requests and watch the logs stream in the first terminal.

---

## docker exec

docker exec runs a command inside an already running container. This is how you interact with a container without stopping it.

```
docker exec -it myapp bash
```

-it gives you an interactive terminal. bash is the command to run inside the container. If the image does not have bash, try sh.

Other useful exec commands:

```
docker exec myapp ps -ef                          # see processes inside container
docker exec myapp env                             # see environment variables
docker exec myapp cat /etc/hosts                  # read a file
docker exec myapp ls /app                         # list directory
docker exec -it myapp python3 manage.py shell     # Django shell
docker exec -u root myapp bash                    # enter as root even if USER is set
```

The difference between docker exec and docker run:

- docker run creates a new container and starts it
- docker exec runs a command inside an existing running container

docker exec does not affect PID 1. The container stays running. You are just running an additional process inside the same namespace.

## Lab

Get a shell inside myapp:

```
docker exec -it myapp bash
```

Inside the container:

```
ps -ef          # you will see only processes inside this namespace
env             # see environment variables including those from Dockerfile
ls /app         # verify application files are present
cat /etc/hosts  # see the container's hosts file
exit
```

---

## docker stop and docker kill

docker stop sends SIGTERM to PID 1, waits 10 seconds, then sends SIGKILL if the process has not exited.

```
docker stop myapp
docker stop --time 30 myapp     # 30 second grace period
```

docker kill sends a signal directly without waiting. Default signal is SIGKILL.

```
docker kill myapp               # immediate SIGKILL
docker kill --signal SIGTERM myapp    # send specific signal
docker kill --signal SIGHUP myapp     # send HUP for config reload
```

When to use which:

- docker stop — always use this first. Allows graceful shutdown.
- docker kill — only when the container is unresponsive and docker stop is not working.

---

## docker start and docker restart

docker start starts a stopped container. The same container, same writable layer, same configuration.

```
docker start myapp
docker start -i myapp     # attach stdin to the container
```

docker restart stops and starts a container in one command.

```
docker restart myapp
docker restart --time 5 myapp    # 5 second grace period for stop before restart
```

---

## docker inspect

docker inspect returns the complete JSON configuration of a container or image. This is the primary debugging tool.

```
docker inspect myapp
```

The output is hundreds of lines. Use --format to extract specific fields:

```
docker inspect myapp --format '{{.State.Status}}'
docker inspect myapp --format '{{.State.Pid}}'
docker inspect myapp --format '{{.State.ExitCode}}'
docker inspect myapp --format '{{.NetworkSettings.IPAddress}}'
docker inspect myapp --format '{{.HostConfig.Memory}}'
docker inspect myapp --format '{{.HostConfig.RestartPolicy.Name}}'
docker inspect myapp --format '{{.Mounts}}'
docker inspect myapp --format '{{json .NetworkSettings}}' | python3 -m json.tool
```

## Lab

Inspect the running container and extract key information:

```
docker inspect myapp --format 'Status: {{.State.Status}}'
docker inspect myapp --format 'PID: {{.State.Pid}}'
docker inspect myapp --format 'IP: {{.NetworkSettings.IPAddress}}'
docker inspect myapp --format 'Memory Limit: {{.HostConfig.Memory}}'
```

Verify the PID from inspect matches what you see on the host:

```
ps -ef | grep $(docker inspect myapp --format '{{.State.Pid}}')
```

---

## docker stats

docker stats shows live resource usage for running containers.

```
docker stats
docker stats myapp
docker stats --no-stream myapp     # single snapshot, no live update
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
```

Column meanings:

| Column | Source | Meaning |
|--------|--------|---------|
| CPU % | cgroup cpu.stat | Percentage of allocated CPU being used |
| MEM USAGE / LIMIT | cgroup memory | Current memory used vs limit |
| MEM % | Calculated | Memory used as percentage of limit |
| NET I/O | Network namespace | Network bytes sent and received |
| BLOCK I/O | cgroup blkio | Disk bytes read and written |
| PIDS | cgroup pids | Number of processes inside the container |

---

## docker top

docker top shows the processes running inside a container, as seen from the host.

```
docker top myapp
docker top myapp aux    # pass ps flags
```

This is equivalent to running ps aux on the host and filtering for the container's processes. The PIDs shown are host PIDs, not the container's internal PIDs.

---

## docker cp

docker cp copies files between a container and the host filesystem.

```
docker cp myapp:/app/app.py ./app.py           # container to host
docker cp ./newconfig.py myapp:/app/config.py  # host to container
```

This works on both running and stopped containers. It is the correct way to get files out of a container for inspection or backup.

---

## docker rm

docker rm removes stopped containers.

```
docker rm myapp
docker rm -f myapp           # force remove even if running
docker rm $(docker ps -aq)   # remove all stopped containers
docker container prune       # remove all stopped containers with confirmation
```

---

# Section 3: Essential Image Commands

## Concept

These commands manage the images on your local system — listing, inspecting, pulling, tagging, pushing, and removing them.

---

## docker images

Lists all images stored locally.

```
docker images
docker image ls              # same command, newer syntax
```

Column breakdown:

| Column | Meaning |
|--------|---------|
| REPOSITORY | Image name, including registry prefix if not Docker Hub |
| TAG | Version tag |
| IMAGE ID | Short 12-character image ID |
| CREATED | When the image was built |
| SIZE | Total size of all layers |

Useful flags:

```
docker images -a             # include intermediate layers
docker images -q             # only image IDs
docker images nginx          # filter by name
docker images --filter dangling=true    # only untagged images
docker images --format "table {{.Repository}}:{{.Tag}}\t{{.Size}}"
```

---

## docker pull

Downloads an image from a registry to local storage.

```
docker pull nginx
docker pull nginx:1.25.3
docker pull nginx:1.25.3-alpine
docker pull ubuntu@sha256:abc123...    # pull by digest, exact image
```

Docker downloads only the layers it does not already have locally. If you pull a new version of nginx and it shares layers with a previously pulled version, only the new layers are downloaded.

---

## docker build

Builds an image from a Dockerfile.

```
docker build -t myapp:1.0 .
docker build -t myapp:1.0 -f path/to/Dockerfile .
docker build -t myapp:1.0 --no-cache .              # ignore cached layers
docker build -t myapp:1.0 --build-arg VERSION=2 .
docker build -t myapp:1.0 --target build-stage .    # multi-stage: stop at named stage
```

The . at the end is the build context. It can also be a URL to a Git repository:

```
docker build -t myapp:1.0 https://github.com/user/repo.git
```

---

## docker tag

Creates a new tag for an existing image. Does not copy the image — just creates a new name pointing to the same content.

```
docker tag myapp:1.0 myapp:latest
docker tag myapp:1.0 registry.example.com/myteam/myapp:1.0
docker tag myapp:1.0 myapp:stable
```

One image can have many tags. All tags point to the same layers. No extra storage is used.

---

## docker push

Uploads an image to a registry.

```
docker push myusername/myapp:1.0
docker push registry.example.com/myteam/myapp:1.0
```

Before pushing to Docker Hub, you must log in:

```
docker login
docker login registry.example.com
```

Docker only uploads layers that are not already present in the registry. This makes pushing subsequent versions of the same image fast.

---

## docker rmi

Removes an image from local storage.

```
docker rmi myapp:1.0
docker rmi -f myapp:1.0                    # force remove even if containers exist
docker rmi $(docker images -q)             # remove all images
docker image prune                         # remove dangling (untagged) images
docker image prune -a                      # remove all unused images
```

An image cannot be removed if a container (including stopped containers) is using it. Remove the containers first.

---

## docker history

Shows the build history of an image — every instruction and the layer it created.

```
docker history myapp:1.0
docker history --no-trunc myapp:1.0    # show full commands without truncation
```

---

## docker save and docker load

Export an image to a tar file and import it back. Used to move images between systems without a registry.

```
docker save -o myapp.tar myapp:1.0
docker load -i myapp.tar
```

---

## docker export and docker import

Export a container's filesystem (not an image — the actual running container's filesystem) to a tar file.

```
docker export myapp -o myapp-container.tar
docker import myapp-container.tar myapp:from-export
```

The difference from save/load: export flattens all layers into one. The resulting image has no build history. Use save/load for images. Use export/import rarely, only when you need the flattened container filesystem.

---

# Section 4: The Complete Command Reference Lab

## Lab

This lab exercises every command in sequence on a single container.

Start fresh:

```
docker system prune -f
```

Build an image:

```
cd /home/myapp
docker build -t myapp:1.0 .
```

List images:

```
docker images
docker history myapp:1.0
docker inspect myapp:1.0 --format '{{.RootFS.Layers}}'
```

Tag the image multiple ways:

```
docker tag myapp:1.0 myapp:latest
docker tag myapp:1.0 myapp:stable
docker images myapp
```

Run a container:

```
docker run -d \
  --name myapp \
  -p 8080:8080 \
  -e APP_ENV=development \
  --memory 256m \
  --restart unless-stopped \
  myapp:1.0
```

Verify it is running:

```
docker ps
docker top myapp
docker stats --no-stream myapp
```

Inspect the container:

```
docker inspect myapp --format 'PID: {{.State.Pid}}'
docker inspect myapp --format 'IP: {{.NetworkSettings.IPAddress}}'
docker inspect myapp --format 'Memory: {{.HostConfig.Memory}}'
```

Interact with the container:

```
docker exec myapp env
docker exec myapp ps -ef
docker exec -it myapp bash
# Inside container:
ls /app
cat /etc/hosts
exit
```

View logs:

```
docker logs myapp
docker logs --tail 20 -t myapp
```

Copy a file out:

```
docker cp myapp:/app/app.py ./extracted_app.py
cat extracted_app.py
```

Stop the container:

```
docker stop myapp
docker ps
docker ps -a
```

Verify the container still exists (writable layer preserved):

```
docker inspect myapp --format '{{.State.Status}}'
```

Restart and verify data:

```
docker start myapp
docker ps
docker exec myapp ls /app
```

Stop and remove:

```
docker stop myapp
docker rm myapp
docker ps -a
```

Remove the image:

```
docker rmi myapp:1.0 myapp:latest myapp:stable
docker images
```

---

# Section 5: Key Takeaways — Part 2

## Container Runtime Behavior

| What happens | Technical reality |
|-------------|------------------|
| docker run | Creates writable layer, starts PID 1 inside namespaces |
| Container running | PID 1 active, cgroups enforcing limits, writes to writable layer |
| docker stop | SIGTERM to PID 1, wait 10s, SIGKILL if not exited |
| Container stopped | Process gone, writable layer preserved on disk |
| docker start | PID 1 started again, same writable layer reused |
| docker rm | Writable layer destroyed, container metadata deleted |

## Command Decision Tree

```
Need to run a new container?
  -> docker run

Need to start an existing stopped container?
  -> docker start

Need to run a command in a running container?
  -> docker exec

Need to see what is happening inside a container?
  -> docker logs, docker top, docker stats

Need to see container configuration?
  -> docker inspect

Need to safely stop a container?
  -> docker stop

Need to force-stop an unresponsive container?
  -> docker kill

Need to permanently remove a container?
  -> docker rm (after stopping)

Need to remove an image?
  -> docker rmi (after removing all containers using it)
```

---

*End of Part 2 - Next: Part 3 — Image Optimization, Registries, and Versioning*
