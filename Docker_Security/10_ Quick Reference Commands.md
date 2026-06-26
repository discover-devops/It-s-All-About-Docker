These are your **closing slides**. Don't teach them like new topics because there is nothing new here. Everything has already been covered. Use them as a **5–7 minute recap**.

---

# Quick Reference Commands (3-4 minutes)

## Context

Tell the students:

> **"We've learned many security concepts today. Now let's put them together and look at the most commonly used commands. Think of this as your cheat sheet."**

Don't explain every command in detail because you've already covered them.

---

## Scanning & Signing

Start with Trivy.

```bash
trivy image myimage:latest
```

Tell them:

"This scans the image and reports known vulnerabilities before deploying it."

Next,

```bash
export DOCKER_CONTENT_TRUST=1
```

"This tells Docker to accept only signed images."

Finally,

```bash
docker trust sign myrepo/myimage:v1.0
```

"This digitally signs the image so others can verify it hasn't been modified."

Done.

30 seconds each.

---

## Secure Container Run

Now tell them something important.

> **"This command is not introducing anything new. It combines almost everything we learned today into a single secure `docker run` command."**

Now go one option at a time.

```bash
--read-only
```

Protects the container filesystem.

---

```bash
--tmpfs /tmp
```

Provides temporary writable storage.

---

```bash
--cap-drop=ALL
```

Removes all Linux capabilities.

---

```bash
--cap-add=NET_BIND_SERVICE
```

Adds back only the capability required by the application.

---

```bash
--security-opt=no-new-privileges
```

Prevents privilege escalation.

---

```bash
--security-opt seccomp=profile.json
```

Restricts system calls.

---

```bash
--user 1000:1000
```

Runs the application as a non-root user.

---

```bash
--memory=512m
```

Limits memory usage.

---

```bash
--cpus=1
```

Limits CPU usage.

---

```bash
--pids-limit=100
```

Prevents fork bomb attacks.

---

```bash
-v appdata:/data
```

Provides a writable location for application data.

Then conclude with one sentence.

> **"If you understand every option in this command, you've understood almost the entire Docker Security session."**

---

# Real-World Secure Setup Example (4-5 minutes)

This is probably the best slide to end the session.

I would not read the bullets.

Instead explain the complete journey.

---

Suppose tomorrow you're asked to deploy a production application.

What would you do?

First, build a secure image.

Use a minimal base image.

Scan it for vulnerabilities.

Don't store secrets inside it.

Then deploy the container securely.

Run it as a non-root user.

Drop unnecessary capabilities.

Make the root filesystem read-only.

Next, secure the Docker host.

Keep Linux updated.

Enable logging.

Protect the Docker daemon.

Finally, secure the network.

Expose only the required ports.

Use custom bridge networks.

Control communication between containers.

If you're running Kubernetes, add RBAC and security policies on top.

Notice something.

No single feature makes Docker secure.

Security comes from applying multiple layers together.

That's exactly what **Defense in Depth** means.

---

## Key Principles

I would spend about one minute here.

Don't explain each principle separately.

Relate them to what students already learned.

**Least Privilege**

Give every user, process and container only the permissions they actually need.

We've applied this throughout the session using Non-Root Containers, Capabilities and Rootless Docker.

---

**Immutability**

Don't modify running containers.

If something changes, rebuild the image and redeploy.

That's why we use Read-Only Filesystems and reproducible builds.

---

**Defense in Depth**

Don't depend on one security feature.

Use multiple layers.

If one layer fails, another layer still protects the system.

---

**Assume Breach**

Never design your system assuming attackers will never get in.

Assume one day a container will be compromised.

Your goal is to stop the attack from spreading.

---

**Continuous Monitoring**

Security isn't finished after deployment.

Keep scanning images.

Review logs.

Apply patches.

Audit regularly.

---

## Final Closing (30 seconds)

I would end exactly like this.

> **"Today we didn't learn ten different Docker security features. We learned one idea in ten different ways: reduce the attack surface, apply least privilege, and never trust by default. Whether you're securing the Docker Host, the Docker Image, or the running Container, you're always following these same principles."**

---

# One suggestion for your entire session

After reviewing your complete document, I think your timing will look like this:

* Foundation + Threat Model: **20–25 min**
* Host Security: **20 min**
* Container Security: **20 min**
* Image Security + Image Trust: **25 min**
* Runtime Security: **10 min**
* Compliance + Checklist: **10 min**
* Quick Reference + Final Architecture: **10 min**
* Labs throughout: **25–30 min**
* Break: **10 min**
* Q&A: **15 min**

That puts you almost exactly at **2.5 hours**. I wouldn't add any more topics—the content is already well balanced for the time you have.
