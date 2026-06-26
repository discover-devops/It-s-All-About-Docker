

# Compliance and Best Practices

## Context (1 minute)

I would start with a simple question.

> **"Suppose you've secured your Docker environment. How do you know you haven't missed anything?"**

In large organizations, nobody relies on memory. Instead, they follow industry standards and security checklists.

That's where **CIS** and **NIST** come in.

Tell students:

> **These are not Docker features. They are security guidelines published by industry organizations to help us build secure container environments.**

---

## CIS Benchmarks (3 minutes)

Explain CIS in one paragraph.

The **Center for Internet Security (CIS)** publishes security best practices for many technologies, including Docker. Think of it as a checklist that tells us whether our Docker environment follows recommended security practices.

Now briefly explain the recommendations.

* Separate partition for containers means Docker data should be stored separately so that if containers consume excessive disk space, they don't fill up the entire operating system partition.

* Harden the container host means keep the Linux host updated, remove unnecessary software, and secure SSH access.

* Don't use the default bridge network because every container attached to it can communicate freely. Instead, create user-defined bridge networks.

* Enable Content Trust so only trusted images are deployed.

* Audit the Docker daemon and important Docker directories so that administrative activities are recorded.

* Restrict network traffic between containers to prevent lateral movement if one container is compromised.

* Enable User Namespace Remapping so root inside the container doesn't become root on the host.

Finally tell them:

> Instead of checking these manually, tools like **docker-bench-security** automatically compare your Docker host against the CIS Benchmark and tell you what needs improvement.

That is enough.

---

## NIST Container Security Guidelines (3 minutes)

Now introduce NIST.

NIST stands for **National Institute of Standards and Technology**. It publishes security standards that are widely followed across governments and enterprises.

For containers, NIST groups security into five major areas.

Rather than memorizing them, explain that these are exactly the topics we covered today.

Image threats are solved by using minimal base images, vulnerability scanning, and image signing.

Registry threats are addressed by using private registries, role-based access control, and cleaning up old or unused images.

Orchestrator threats relate to Kubernetes or Docker Swarm. These platforms should restrict privileges and control what containers are allowed to do.

Container threats are mitigated by running containers as non-root users, dropping unnecessary capabilities, enabling Seccomp, and using a read-only root filesystem.

Host OS threats are reduced by keeping the operating system updated, using a minimal Linux installation, and enabling firewalls.

Now conclude with one sentence.

> **Notice that NIST isn't introducing anything new. It is simply organizing all the security practices we've already learned into different categories.**

---

# Security Checklist

## Context (1 minute)

Tell students:

> **This is probably the most valuable slide in the entire presentation.**

Why?

Because after today's session, you won't remember every command we discussed.

But before deploying any application, you can quickly review this checklist.

If every item is checked, your Docker deployment is significantly more secure.

---

Now don't read every checkbox one by one.

Instead group them.

### Image Security

Everything related to building a secure image.

Ask yourself:

* Am I using a trusted base image?
* Is the image small?
* Have I scanned it?
* Did I accidentally include secrets?
* Am I avoiding `latest`?

If the answer is yes, your image is in good shape.

---

### Container Runtime

Everything related to the running container.

Ask yourself:

* Is it running as a non-root user?
* Have I removed unnecessary capabilities?
* Is the filesystem read-only?
* Am I using Seccomp and AppArmor?
* Have I configured resource limits?
* Am I avoiding `--privileged` and host networking?

These checks protect the running container.

---

### Host Security

Everything related to the Docker host.

Ask yourself:

* Is the operating system patched?
* Is Rootless Docker or User Namespace Remapping enabled where appropriate?
* Is the Docker Daemon protected?
* Is TLS enabled for remote access?
* Are logging and firewall configured?

These checks protect the server running Docker.

---

### Network & Compliance

Finally, validate the overall environment.

Use user-defined bridge networks instead of the default bridge.

Expose only the ports that are actually required.

Run automated CIS benchmark scans.

Integrate security scanning into your CI/CD pipeline so vulnerabilities are detected before deployment.

---

## Final Closing (1 minute)

I would finish the session with this sentence:

> **Throughout this session we learned many Docker security features—non-root users, capabilities, Seccomp, image scanning, Rootless Docker, Content Trust, and resource limits. Individually they solve different problems, but together they follow one simple principle: Reduce the attack surface, apply least privilege, and assume that one day something will fail. Security is about building multiple layers of defense so that a single mistake doesn't compromise the entire system.**

---

## My only recommendation for the document

I would add **one final slide** after the Security Checklist titled:

> **"Docker Security in One Picture"**

```text
Secure Image
      │
      ▼
Trusted Registry
      │
      ▼
Image Scan
      │
      ▼
Signed Image
      │
      ▼
Docker Host
      │
      ▼
Non-Root Container
      │
      ▼
Resource Limits
      │
      ▼
Monitoring & Audit Logs
```

That single diagram gives students a complete mental picture of everything they learned in the last 2.5 hours. It also leaves them with a structured way to remember the entire session instead of isolated commands and features. I think that would be a much stronger ending than finishing on a checklist alone.
