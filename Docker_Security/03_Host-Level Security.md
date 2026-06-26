This is actually the first place where I would **restructure the teaching slightly**.

The document is technically correct, but if you start with **"Use TLS for remote access"**, students will immediately ask:

> "Remote access to what?"

> "Who is connecting?"

> "Why would anyone expose Docker remotely?"

So before teaching the individual controls, I would spend 5 minutes building the context.

---

## Context

I would begin by reminding the students about the previous topic.

> "In the Threat Model, we saw that an attacker ultimately wants to compromise the Host Operating System. But before they reach the host, they usually try to control the Docker Daemon because the Docker Daemon is responsible for creating, stopping, deleting, and managing every container on that server."

Then draw this simple architecture.

```text
Docker CLI
      │
      ▼
Docker Daemon
      │
      ▼
Containers
```

Then ask the class:

> "If I can completely control the Docker Daemon, what can I do?"

Students will answer:

* Create containers.
* Stop containers.
* Delete containers.
* Pull images.

Exactly.

Then conclude:

> "That is why our first objective is to protect the Docker Daemon."

Now the entire Host-Level Security chapter has a purpose.

---

## Castle Analogy

The analogy in your document is good and I wouldn't change it. 

I would simply expand it slightly.

The Host Operating System is your castle.

The Docker Daemon is the main gatekeeper. Every person entering or leaving the castle must pass through this gate. If the gatekeeper is compromised, the attacker no longer needs to climb the walls. They can simply walk through the front gate.

Therefore, Host-Level Security is really about protecting the gatekeeper before we worry about protecting individual rooms (containers).

---

Now comes something important.

I would **not** teach all these controls together.

I would split this chapter into three logical sections.

---

# Section 1 - Protect the Docker Daemon

Everything that directly protects the daemon belongs together.

So I would teach:

* TLS for Remote Access
* User Namespace Remapping
* Rootless Docker
* Docker Socket

These four topics are closely related.

---

# Section 2 - Secure the Host Operating System

Now move to:

* Firewall
* SSH
* SELinux
* AppArmor
* Kernel Updates
* Auditd

Students immediately understand these are host-level controls rather than Docker-specific controls.

---

# Section 3 - Reduce the Attack Surface

Finally explain:

* Disable legacy features
* Content Trust
* Audit Logging

Now the chapter flows much more naturally.

---

# Let's start with the first topic.

## Use TLS for Remote Access

This is where I would spend some time because almost everyone gets confused.

---

### Context

Ask the students:

> "When you execute"

```bash
docker ps
```

> "Who actually performs the work?"

Students should already know.

The Docker CLI simply sends the request.

The Docker Daemon performs the operation.

Normally both components exist on the same machine.

Communication happens through

```text
/var/run/docker.sock
```

Everything is local.

---

Now ask another question.

> "Suppose I have 50 Docker servers in my data center. Do I need to SSH into each server every time I want to create a container?"

Obviously not.

Large organizations often allow administrators or automation tools to communicate with the Docker Daemon remotely.

Instead of using the Unix socket, the daemon listens on a TCP port.

For example:

```text
tcp://0.0.0.0:2376
```

Now another machine can communicate with the Docker Daemon over the network.

The communication now looks like this.

```text
Laptop
   │
   │
Network
   │
   ▼
Docker Daemon
```

This is called **Remote Docker API**.

---

Now explain the problem.

Imagine you expose the Docker Daemon like this:

```text
tcp://0.0.0.0:2375
```

without authentication.

Anyone who can reach that server can execute Docker commands.

They can create containers.

Delete containers.

Stop production applications.

Launch privileged containers.

Eventually they may gain complete control over the host.

This is exactly the same as exposing an SSH server with no password.

---

Now introduce TLS.

TLS solves two problems.

First, it encrypts the communication so attackers cannot read or modify packets travelling over the network.

Second, it authenticates both parties using certificates.

Only clients that possess valid certificates are allowed to communicate with the Docker Daemon.

So when someone sends a Docker command,

the daemon first asks,

> "Who are you?"

If the certificate is trusted,

communication continues.

Otherwise,

the request is rejected.

That is exactly what these configuration parameters are doing:

```json
{
  "tls": true,
  "tlscert": "/etc/docker/server-cert.pem",
  "tlskey": "/etc/docker/server-key.pem",
  "tlscacert": "/etc/docker/ca.pem",
  "tlsverify": true,
  "hosts": ["tcp://0.0.0.0:2376"]
}
```

Explain each property one by one instead of reading the JSON.

* `tls: true` enables encrypted communication.
* `tlscert` is the server's public certificate.
* `tlskey` is the server's private key.
* `tlscacert` specifies the Certificate Authority used to validate client certificates.
* `tlsverify` tells Docker to verify every connecting client.
* `hosts` configures the Docker Daemon to listen for remote TCP connections on port **2376**, which is the standard TLS-secured Docker port.

Also mention an important distinction:

* **2375** → Docker Remote API **without TLS** (insecure).
* **2376** → Docker Remote API **with TLS** (secure).

Many students notice these ports in documentation but don't know why there are two.

---

### Lab

This topic is difficult to demonstrate fully because generating certificates takes time, but you can still give students a practical understanding.

First, ask them to verify which interfaces the Docker Daemon is currently listening on:

```bash
sudo ss -lntp | grep dockerd
```

or

```bash
sudo netstat -tulpn | grep dockerd
```

They will usually see that Docker is listening only on the local Unix socket and **not** on a TCP port.

Then show the Docker daemon configuration file:

```bash
cat /etc/docker/daemon.json
```

If the file doesn't exist, explain that Docker uses default settings.

Finally, show the official TLS configuration from the document and explain what each field represents rather than actually enabling it during class. Configuring certificates is better suited for an advanced lab because it involves generating a CA, server certificates, client certificates, restarting the daemon, and testing client authentication.

---

### My only recommendation

I would add **one slide before this topic** titled:

> **"When do we need Remote Docker API?"**

Answer it with real-world examples:

* Jenkins building Docker images on a remote Docker host.
* Centralized automation managing multiple Docker servers.
* Operations teams administering Docker hosts from a management workstation.

Once students understand *why* remote access exists, TLS stops looking like "just another security feature" and instead becomes the obvious way to secure that communication.
