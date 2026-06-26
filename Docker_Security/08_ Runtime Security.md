This is another **short chapter**. I wouldn't spend more than **8–10 minutes** here because you've already covered most of the concepts indirectly in the previous sections.

---

# Runtime Security

## Context (1 minute)

Until now we've been securing the **image** before deployment and the **container configuration** before startup.

Now let's assume the container is already running.

The question becomes:

> **"How do we stop one running container from affecting the entire server?"**

This is what Runtime Security is about.

It's about controlling the behavior of a container **while it is running**.

---

# Resource Constraints (4-5 minutes)

Start with a simple real-world example.

Suppose one application has a bug or an attacker intentionally runs a **fork bomb** or an infinite loop.

That single container starts consuming:

* 100% CPU
* All available memory
* Thousands of processes

What happens?

Not only does that application become slow, but other containers running on the same Docker host also start suffering because they all share the same physical resources.

One bad container can impact every other container on that host.

That's why Docker allows us to place limits on the resources a container can consume.

Now explain each option very briefly.

```bash
--cpus=1
```

This tells Docker that the container can use only one CPU core.

Even if the application tries to consume more CPU, Docker won't allow it.

---

```bash
--memory=512m
```

Limits the container to 512 MB of RAM.

If the application exceeds that limit, Docker may terminate the container instead of allowing it to consume all system memory.

---

```bash
--pids-limit=100
```

This limits the number of processes the container can create.

Explain what a fork bomb is in one sentence.

A fork bomb is a malicious or buggy program that continuously creates new processes until the operating system runs out of resources.

By limiting the number of processes, Docker prevents one container from crashing the host.

---

```bash
--device-read-bps /dev/sda:10mb
```

Limits the disk read speed.

This prevents one container from overwhelming the storage subsystem and affecting other containers.

---

Conclude this section with one sentence.

> **Resource limits don't stop an attacker from entering the container, but they stop one compromised container from consuming all the host's resources.**

---

# Dropping Privileges (3-4 minutes)

Students already understand **Non-Root Containers**, so this becomes very easy.

Sometimes applications genuinely need root privileges **only during startup**.

For example,

* Create directories
* Change file ownership
* Set file permissions

After initialization, they no longer need root.

Instead of running the entire application as root, the application starts as root, performs the setup, and then switches to a normal user using tools like:

* `gosu`
* `su-exec`

This follows the Principle of Least Privilege.

Root is used only for a few seconds during startup.

The application itself runs as a normal user.

---

Now explain this command.

```bash
docker run --security-opt=no-new-privileges myapp
```

Tell students this is another safety mechanism.

Normally, a process might gain additional privileges during execution through mechanisms like **setuid binaries**.

The `no-new-privileges` option tells Linux:

> "This process is never allowed to gain more privileges than it started with."

So even if an attacker finds a privilege escalation technique inside the container, Linux refuses to grant additional privileges.

This is another example of enforcing the Principle of Least Privilege.

---

## Lab (2 minutes)

A simple demonstration is enough.

Start a container with memory limits.

```bash
docker run -d \
--memory=128m \
--name memtest \
nginx
```

Then inspect the container.

```bash
docker inspect memtest
```

Search for:

```text
Memory
```

Students will see that Docker has stored the memory limit.

You don't need to demonstrate a real Out-Of-Memory condition during a 2.5-hour session.

---

## One suggestion for the document

I would make one small change to the order.

Move **Dropping Privileges** immediately after **Running Non-Root Containers** in the previous chapter.

Why?

Because they teach the same principle.

* Non-root containers
* Dropping privileges
* `no-new-privileges`

All three are about **reducing privileges**.

Then keep this Runtime Security chapter focused only on **runtime resource controls**, such as CPU, memory, PIDs, and I/O.

That creates a cleaner separation between:

* **Privilege Management** (Container Security)
* **Resource Management** (Runtime Security)

From a teaching perspective, students find that organization easier to follow.
