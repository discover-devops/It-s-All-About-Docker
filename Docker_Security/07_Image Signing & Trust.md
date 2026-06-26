# Image Signing & Trust

### Context (1 minute)

Start with a simple question.

> **"How do you know that the image you downloaded is really the image published by the original developer?"**

Students usually answer:

> "Because I downloaded it from Docker Hub."

Now ask another question.

> **"What if someone modified the image after the developer uploaded it?"**

How would Docker know?

That's the problem Image Signing solves.

---

### Concept (2-3 minutes)

Think about downloading software from the Internet.

When you download software from Microsoft or Adobe, the installer is digitally signed. Your operating system verifies that signature before trusting the software.

Docker applies the same idea to container images.

When an image publisher creates an image, they digitally sign it using their **private key**.

That signature is stored separately in a **Notary Server**.

Later, when someone tries to pull that image, Docker verifies the signature.

If the signature matches, Docker knows:

* The image really came from the expected publisher.
* The image has not been modified after it was signed.

If someone tampers with the image, the signature verification fails and Docker refuses to trust it.

The important point to tell students is:

> **Image Signing does not scan for vulnerabilities.**

It only verifies **authenticity and integrity**.

Students often confuse Image Signing with Image Scanning.

Keep them separate.

* **Image Scanning** answers:

  > "Does this image contain known vulnerabilities?"

* **Image Signing** answers:

  > "Is this the original image published by the trusted publisher?"

Those are two completely different security checks.

---

### Enable Docker Content Trust

The command is simple.

```bash
export DOCKER_CONTENT_TRUST=1
```

Explain what this does.

This sets an environment variable telling the Docker CLI:

> "From now on, only trust signed images."

Now if someone tries to pull an unsigned image, Docker rejects it.

The application never even starts because Docker refuses to download an image that cannot be verified.

---

### Real-world Example (1 minute)

Suppose your company has an internal registry containing approved images.

Every image is signed by the Platform Team.

A developer accidentally tries to deploy an unsigned image downloaded from an unknown registry.

Since Docker Content Trust is enabled, Docker immediately rejects it.

Instead of discovering the problem in production, the deployment fails immediately.

That is exactly what we want.

---

### One thing I would add while teaching

After explaining Image Signing, spend **30 seconds** clarifying the difference between three terms because students often mix them up:

| Concept             | Purpose                                                                |
| ------------------- | ---------------------------------------------------------------------- |
| **Image Hardening** | Build a secure image                                                   |
| **Image Scanning**  | Find vulnerabilities in the image                                      |
| **Image Signing**   | Verify who published the image and ensure it hasn't been tampered with |

This simple comparison removes a lot of confusion and makes the previous three sections feel connected.

---

### Time Required

* Context: **1 minute**
* Concept: **3 minutes**
* Command: **30 seconds**
* Real-world example: **1 minute**

**Total: ~5 minutes**

For a 2.5-hour Docker Security session, that's exactly the right amount of depth. Students understand **why** image signing exists without getting lost in cryptography or the internals of Notary.
