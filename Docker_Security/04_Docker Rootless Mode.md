

## Docker Rootless Mode (3 minutes)

### Context

Till now, our Docker Daemon has been running as the **root** user. The problem is that if someone finds a vulnerability in the Docker Daemon, they immediately get root privileges on the host.

### Concept

Rootless Docker changes this behavior. Instead of running the Docker Daemon as the root user, it runs it as a normal Linux user.

So even if someone compromises the Docker Daemon, they don't automatically become root on the host. The attack is still possible, but its impact is much smaller.

That's why we say **Rootless Docker improves isolation and reduces the blast radius.**

### Limitation

Since it is not running as root, some operations require extra configuration. For example, binding to privileged ports like 80 or 443, or using some storage drivers.

**No lab.** Just explain the concept.

---

# Restricting Docker Socket Access (4 minutes)

### Context

Earlier we learned that `/var/run/docker.sock` is the communication channel between the Docker CLI and the Docker Daemon.

If an attacker gets access to this socket, they can send Docker commands directly to the Docker Daemon.

### Concept

That is why people say:

> **Access to the Docker Socket is almost equivalent to root access on the host.**

So we must protect this socket.

### Explain the solutions briefly

**Don't mount the socket.**

If your application doesn't need Docker, don't expose the socket.

**Socket Proxy.**

If an application really needs Docker access, place a proxy in between so only selected Docker API calls are allowed.

**Read-only mount.**

Mounting with `:ro` provides limited protection but should not be considered a complete security solution.

**File permissions.**

The Docker socket is protected using normal Linux permissions. Only trusted users should belong to the `docker` group.

### Lab

One command is enough.

```bash
ls -l /var/run/docker.sock
```

Explain:

```
srw-rw---- root docker
```

Owner is `root`.

Group is `docker`.

Only trusted users should belong to that group.

Done.

---

# Securing the Underlying OS (4–5 minutes)

### Context

Until now we've been securing Docker.

Now let's secure the Linux server on which Docker is running.

If the operating system is compromised, Docker security won't help.

### Explain each line in one sentence

**Keep the system updated**

Security patches fix known vulnerabilities in Linux.

**Minimal OS**

Install only what you need. Fewer packages mean fewer vulnerabilities.

**Firewall**

Open only the ports your applications actually need.

**Disable unnecessary services**

Every running service is another possible attack surface.

**Secure SSH**

Disable root login and use SSH keys instead of passwords.

**SELinux/AppArmor**

These Linux security frameworks provide an additional protection layer around containers and processes.

**Kernel hardening**

Enable Linux security settings to reduce common network and kernel attacks.

**Audit logging**

Record important system events so you can investigate security incidents later.

**No lab.**

---

See the difference?

Instead of spending **15 minutes** on this section, you can comfortably cover it in **10–12 minutes**, which is exactly what an instructor-led session needs.

I also noticed a pattern in your document. Some topics deserve deep explanations (for example, **Non-Root Containers**, **Capabilities**, **Docker Socket**, **Image Hardening**), while others are awareness topics where a concise explanation is enough. I'll optimize accordingly so you can realistically finish the entire guide, including your break and Q&A, within the 2.5-hour session.
