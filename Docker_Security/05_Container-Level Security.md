# Container-Level Security

## Context (1 minute)

Until now, we've been protecting the Docker Daemon and the Host Operating System.

Now assume the attacker has already compromised one of our containers.

The next question becomes:

> **"How do we stop the attacker from causing more damage inside the container or escaping to the host?"**

This chapter is all about reducing what a compromised container is allowed to do.

The castle analogy fits perfectly here.

> Don't give the guest the master keys (Non-root).
>
> Take away unnecessary tools (Capabilities).
>
> Bolt the furniture to the floor (Read-only filesystem).

Every control here follows one principle:

> **Give the container only the permissions it actually needs.**

---

# Running Non-Root Containers (4-5 min)

### Context

Ask the students:

> "When we start a container, which user does the application run as?"

Most students don't know.

Run this lab.

```bash
docker run --rm -it ubuntu bash
```

Inside the container run:

```bash
whoami
```

Output:

```text
root
```

Now explain.

By default, almost every container starts as the **root user**.

That doesn't immediately mean root on the host, but if the application gets compromised, the attacker now has root privileges **inside the container**.

That is more privilege than most applications actually need.

The solution is simple.

Create a normal user inside the image and switch to that user.

```dockerfile
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

USER appuser
```

Explain:

The first command creates a group and a user.

The second command tells Docker that every process should run as `appuser` instead of `root`.

If you don't control the Dockerfile, Docker also lets you specify the user at runtime.

```bash
docker run --user 1000:1000 nginx
```

In Kubernetes, the same concept is implemented using:

```yaml
runAsNonRoot: true
```

The idea is the same everywhere.

**Applications should never run as root unless absolutely necessary.**

---

# Limiting Capabilities (4 minutes)

### Context

Students usually think Linux has only two permission levels.

Root.

Non-root.

Tell them that Linux is smarter than that.

Instead of giving root every permission, Linux breaks root privileges into small pieces called **Capabilities**.

Think of them as individual permissions.

For example,

One capability allows binding to privileged ports.

Another allows changing network settings.

Another allows mounting file systems.

Instead of giving every capability, Docker allows us to give only the ones an application actually needs.

That's exactly what this command is doing.

```bash
docker run \
--cap-drop=ALL \
--cap-add=NET_BIND_SERVICE \
nginx
```

Explain each option.

`--cap-drop=ALL`

Remove every Linux capability.

Now the container has almost no special privileges.

`--cap-add=NET_BIND_SERVICE`

Only give permission to bind to ports below 1024.

That's all Nginx actually needs.

Everything else remains disabled.

Then immediately explain why this is better than:

```bash
--privileged
```

The `--privileged` flag gives the container almost every Linux capability.

It is the exact opposite of least privilege.

That's why Docker security best practices always recommend avoiding `--privileged` unless absolutely necessary.

---

# Read-Only Root Filesystem (3-4 min)

### Context

Ask the class.

> "Does your application really need to modify its own operating system?"

Usually the answer is no.

Applications mostly:

* Read configuration
* Serve requests
* Write logs
* Store uploaded files

They rarely need to modify `/bin`, `/usr`, `/etc`, etc.

Docker allows us to make the container filesystem read-only.

```bash
docker run \
--read-only \
--tmpfs /tmp \
-v appdata:/data \
myapp
```

Explain each option.

`--read-only`

Makes the container's root filesystem read-only.

The application cannot accidentally or maliciously modify system files.

`--tmpfs /tmp`

Creates temporary writable memory for applications that need `/tmp`.

`-v appdata:/data`

Provides a writable volume for persistent application data.

Now explain the benefit.

Even if an attacker compromises the application, installing malware or modifying system binaries inside the container becomes much harder.

---

# Seccomp Profiles (2-3 min)

### Context

Earlier we learned that containers share the host kernel.

Applications interact with the Linux kernel using **System Calls**.

Examples include:

* opening files
* creating processes
* changing permissions
* mounting devices

Seccomp lets Docker control which system calls a container is allowed to use.

If the application tries to execute a blocked system call, Linux simply denies it.

Docker already provides a secure default Seccomp profile.

Custom profiles are mainly used for highly secure production environments.

The command looks like:

```bash
docker run \
--security-opt seccomp=profile.json \
alpine
```

No lab required.

Just explain the concept.

---

# AppArmor and SELinux (2-3 min)

### Context

Tell the students this.

Everything we've discussed so far is Docker security.

Now we are using **Linux security** to protect Docker.

AppArmor and SELinux are Linux security frameworks.

They define what a process is allowed to access.

For example,

Even if a process is compromised,

AppArmor or SELinux can still prevent it from reading sensitive files, modifying system directories, or accessing devices.

Docker can apply these policies using:

```bash
docker run \
--security-opt apparmor=docker-custom \
myapp
```

You don't need to explain AppArmor syntax.

Students only need to understand:

Docker is using another Linux security mechanism to add one more layer of protection.

---

## Total Time

| Topic                     |  Time |
| ------------------------- | ----: |
| Context                   | 1 min |
| Non-root Containers + Lab | 5 min |
| Capabilities              | 4 min |
| Read-only Filesystem      | 4 min |
| Seccomp                   | 3 min |
| AppArmor / SELinux        | 3 min |

**Total: ~20 minutes**

---

### One suggestion for this chapter

I would add one sentence before you start:

> **"Notice a pattern. Every feature in this chapter follows the Principle of Least Privilege. We keep removing permissions until the application has only what it genuinely needs."**

That one sentence ties together **Non-root Users**, **Capabilities**, **Read-only Filesystem**, **Seccomp**, and **AppArmor/SELinux** under a single security principle. Students remember the principle, and the individual features become examples of applying it.
