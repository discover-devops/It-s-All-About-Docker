**Concept → Daily Life Story → Then Docker**

**Vulnerability** → "Your house has an old lock that can be opened with a credit card. That old lock is a vulnerability. You don't need to be a criminal to understand that — you just need to know the lock is weak."

**Malicious** → "Imagine someone rings your doorbell pretending to be a delivery person, but they actually want to see inside your house. That 'pretending' is malicious intent."

**Attack** → "When that fake delivery person actually tries to open your door — that's the attack."

**Exploit** → "When they succeed because your lock was weak — that's exploiting the vulnerability."

**Patch / Fix** → "You replace the lock. That's the patch."

---
---

When we hear the word **attack**, many people imagine a hacker wearing a hoodie typing very fast and somehow magically breaking into computers. That is not what an attack means.

An attack simply means **someone trying to do something on your system that you never intended them to do.**

For example, suppose you own a house. You invite your friends through the main door. That is legitimate access. Now imagine someone climbs through the window at night. They were never invited, yet they found another way to enter the house. That is an attack.

The same idea applies to computer systems. Every application is built to perform certain tasks for legitimate users. An attacker looks for unintended ways to make the application do something it was never designed to do.

---

The next concept is **vulnerability**.

A vulnerability is simply a weakness.

Think about your house again. If the front door has a strong lock, it is difficult to break in. But if one of the windows is broken and never repaired, that broken window becomes the weak point. The thief doesn't attack the strongest part of the house; they attack the weakest part.

Software works exactly the same way.

A vulnerability is a mistake, bug, design flaw, or misconfiguration that allows an attacker to perform unauthorized actions.

Sometimes the vulnerability is written by the application developer. Sometimes it exists inside a third-party library. Sometimes it is created by an administrator through a bad configuration.

---

Now students naturally ask:

> "Why do vulnerabilities exist? Can't developers just write perfect software?"

This is a great opportunity to explain that modern applications contain millions of lines of code written by hundreds or thousands of developers. They also depend on hundreds of open-source libraries. It is almost impossible to guarantee that every line of code is perfect. Over time, researchers discover bugs, and those bugs become publicly known vulnerabilities.

That's why security is not just about writing good code. It is also about continuously patching systems, scanning images, updating libraries, and reducing the damage if someone finds a vulnerability before you fix it.

---

The next question is:

> "Why do people attack systems?"

Students often think hacking is only done "for fun." In reality, most attacks today have financial or strategic motives.

Some attackers want to steal customer information such as usernames, passwords, credit card numbers, or medical records. Others deploy ransomware and demand money to unlock the systems. 
Some install cryptocurrency miners and secretly use your servers to generate cryptocurrency. 
In corporate environments, attackers may steal intellectual property or source code. In nation-state attacks, the objective may be espionage or disrupting critical infrastructure.

In almost every case, there is a business motive behind the attack.

---

Now connect this to Docker.

Suppose a company has deployed its online shopping application inside a Docker container.

The application listens on port 443 because customers need to access the website.

An attacker discovers that the application contains a vulnerability.

They send a specially crafted request.

Instead of receiving a normal response, the application accidentally executes the attacker's commands.

At that moment, the attacker has entered the container.

Notice something important here.

The attacker never attacked Docker.

They attacked the **application**.

Docker simply happened to be running that application.

This distinction is extremely important because students often think Docker itself is being attacked first.

---

Now you can introduce the concept of a **malicious request**.

Every request reaching your application is not necessarily malicious.

If you open Amazon's website and search for a laptop, that is a perfectly valid request.

If someone sends a request specifically designed to exploit a bug—for example, inserting SQL commands where a username is expected, uploading malicious code, or sending unexpected input that crashes the application—that becomes a malicious request.

A malicious request is simply a request whose purpose is to exploit a vulnerability rather than use the application normally.

---

Only after this foundation would I introduce the security goals.

Now they make sense.

For example, explain them as a story.

We know attackers exist.

We know vulnerabilities exist.

We also know we cannot guarantee that every vulnerability will be discovered before an attacker finds it.

So how do we defend ourselves?

First, we try to **prevent the initial compromise** by reducing the number of ways an attacker can reach us. If an application only needs port 443, why expose ten other ports?

Second, we assume that prevention may fail one day. If an attacker compromises one container, we want to **contain** them so they cannot move to the host or other containers. This is called limiting the **blast radius**. The term comes from explosions: if a grenade explodes in one room, you don't want the entire building to collapse.

Third, while all this is happening, we need to **detect and respond**. We collect logs, monitor unusual activity, and generate alerts so the security team can react before the damage spreads.

Finally, we **minimize the damage** by following the principle of **least privilege**. Every process, user, or container should receive only the permissions it actually needs. If an attacker compromises that process, they inherit only those limited permissions instead of gaining complete control over the system.

---

This creates a very natural flow:

> **Attacker → Vulnerability → Malicious Request → Initial Compromise → Escalation → Security Controls**

Once students understand that story, every topic in the rest of the Docker Security Hardening guide fits naturally. When you later teach non-root containers, seccomp, AppArmor, user namespaces, or rootless Docker, students will immediately recognize them as controls designed to break that attack chain rather than isolated Docker features.
