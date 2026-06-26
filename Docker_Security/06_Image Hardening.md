# Image Hardening

## Context (1 minute)

Until now we've been protecting the Docker Host and the running Container.

Now let's move one step earlier in the lifecycle.

Ask the students:

> **"Can we make a container secure if the image itself is already insecure?"**

The answer is **No**.

Remember,

> **Every container starts from an image.**

If the image already contains vulnerabilities, malware, secrets, or unnecessary software, every container created from that image inherits those problems.

That's why we harden the image **before** deploying it.

---

# Image Security Standards (2 minutes)

The first principle is very simple.

> **Start with a trusted foundation.**

Suppose you are building a house.

Would you buy cement from an unknown roadside shop?

Probably not.

Similarly, don't build production images from random images available on the Internet.

Organizations usually maintain a list of approved base images called **Blessed Images**.

These images are regularly patched, scanned, and maintained by the platform or security team.

The second recommendation is to remove everything you don't need.

Every additional package increases the attack surface.

More packages mean more software.

More software means more vulnerabilities (CVEs).

A simple rule students can remember is:

> **Smaller image = Smaller attack surface.**

---

# Minimal Base Images (3 minutes)

Here you're simply choosing the right base image.

Explain all three.

### Alpine

Very small Linux distribution.

Around 5 MB.

Good for most lightweight applications.

Mention one point because students may ask later.

Alpine uses **musl libc** instead of **glibc**, so a few applications may require extra testing.

Don't go deeper.

---

### Distroless

Even smaller.

No shell.

No package manager.

No utilities like `bash`.

This makes the image much harder for attackers to explore if they compromise the container.

It's an excellent production choice.

---

### Scratch

Empty image.

Contains nothing.

Only suitable for statically compiled applications like Go binaries.

Not useful for most Java, Python or Node applications.

---

# Multi-Stage Builds (4 minutes)

Students already know Multi-stage builds.

Now explain the security benefit.

During development we need:

* Maven
* Node
* GCC
* Build tools

But in production we only need the application.

Multi-stage builds ensure that only the final application is copied into the production image.

Everything used during compilation is left behind.

Explain the Dockerfile.

```dockerfile
FROM node:18 AS builder
```

This stage performs the build.

It installs dependencies.

Compiles the application.

Produces the final artifacts.

Then:

```dockerfile
FROM node:18-alpine
```

Starts a fresh image.

Now we only copy:

```dockerfile
COPY --from=builder /app/dist ./dist
```

The final image contains only the application.

No source code.

No build tools.

No npm cache.

No unnecessary packages.

This makes the image smaller, faster and more secure.

---

# Dependency Scanning (3 minutes)

Even if you write perfect code,

your image still contains hundreds of third-party libraries.

Some of them may have known vulnerabilities.

Security scanners compare your image against public vulnerability databases and tell you which packages are vulnerable.

Briefly introduce the tools.

* Trivy (most popular, fast, free)
* Grype
* Docker Scan
* Clair

No need to explain all four.

Tell students:

> **In real projects, Trivy is probably the tool you'll encounter most often.**

### Small Lab

```bash
trivy image nginx
```

Show the CVEs.

Students love this demo.

---

# Reducing Attack Surface (3 minutes)

This is mostly common sense.

Remove package managers if they are no longer needed.

Pin image versions.

Instead of:

```dockerfile
FROM python:latest
```

Use:

```dockerfile
FROM python:3.9.18-alpine3.18
```

Now every build is predictable.

Explain `.dockerignore`.

Students already know it.

Now relate it to security.

Without `.dockerignore`, we may accidentally copy:

* `.git`
* `.env`
* SSH keys
* AWS credentials

into the image.

The image may eventually reach production or a public registry.

Those sensitive files become part of the image forever.

---

# Removing Build-Time Secrets (4 minutes)

This is a very practical topic.

Suppose a developer writes:

```dockerfile
ENV AWS_SECRET_ACCESS_KEY=abcd123
```

The application works.

Everyone is happy.

Months later someone downloads the image.

They execute:

```bash
docker history myimage
```

The secret is still present in the image layers.

Deleting it in a later Dockerfile instruction doesn't help because Docker images are immutable layers.

Once a secret enters a layer, it remains in the image history.

That's why we never bake secrets into images.

Instead use:

* BuildKit Secrets
* Runtime environment variables
* Docker Swarm Secrets

The principle is simple:

> **Secrets should be injected at runtime, not stored inside the image.**

---

## Total Time

| Topic                 |  Time |
| --------------------- | ----: |
| Context               | 1 min |
| Blessed Images        | 2 min |
| Minimal Images        | 3 min |
| Multi-stage Builds    | 4 min |
| Dependency Scanning   | 3 min |
| Reduce Attack Surface | 3 min |
| Secrets               | 4 min |

**Total: ~20 minutes**

---

## One thing I would slightly improve in the document

Since you've already taught `.dockerignore` and Multi-stage Builds in earlier Docker sessions, don't reteach them from scratch. Spend **30 seconds reminding students** what they already know, then answer the new question:

> **"Earlier we learned Multi-stage Builds to reduce image size. Today we're learning that they also improve security because build tools and unnecessary software never make it into the production image."**

That connection is powerful because it shows students that one Docker feature can solve multiple problems—performance, image size, and security.
