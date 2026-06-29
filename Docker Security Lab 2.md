# Docker Security Lab 2: Logging, Auditing & Runtime Threat Detection

> **Session Type:** Instructor-Led Live Lab + Student Runbook  
> **Difficulty:** Intermediate  
> **Estimated Time:** 90–120 minutes  
> **Prerequisites:** Docker installed, `myapp:runtime-test` image built from Lab 1

---

## 🏰 The Castle Analogy — Before We Begin

Imagine your Docker host as a medieval castle.

In the previous labs, we **built the walls** — hardened images, dropped capabilities, ran as non-root. But walls alone are not enough. A smart castle lord also does three more things:

1. **Keeps a logbook** of every gate entry and exit → *Container Logging*
2. **Runs an audit trail** of who touched what, when → *auditd*
3. **Stations guards inside the castle walls** who raise an alarm the moment something suspicious happens → *Falco Runtime Detection*

Today's lab covers all three layers. By the end, you will have a fully observable Docker environment — one where nothing happens silently.

---

## Lab Environment Check

Before you start, verify your environment is ready.

```bash
# Confirm Docker is running
docker version

# Confirm your test image exists
docker images | grep myapp

# Confirm you are in the right location
pwd
```

> ✅ Expected: Docker version output, `myapp:runtime-test` listed, and you are in your home directory.

If `myapp:runtime-test` is missing, rebuild it from Lab 1 before proceeding.

---

## Step 1: Container Logging — The Castle Logbook

### 🧠 Context and Concept

Every process that runs inside a container writes output to **stdout** and **stderr**. Docker captures this output through a component called a **log driver**.

Think of the log driver as the scribe sitting at the castle gate. Whatever the container shouts, the scribe writes it down — but *where* and *how* the scribe records it depends on which log driver you configure.

Docker ships with several log drivers:

| Driver | What it does | Best used for |
|---|---|---|
| `json-file` | Writes logs to JSON files on the host | Default; good for dev and small deployments |
| `syslog` | Ships logs to a syslog daemon | Traditional Linux syslog infrastructure |
| `journald` | Writes to systemd journal | Systems already using systemd |
| `fluentd` | Ships to Fluentd aggregator | Production centralized logging |
| `awslogs` | Ships directly to CloudWatch | AWS-hosted workloads |

**Why does this matter for security?**
- Without log limits, a single noisy container can **fill your disk** — a classic denial-of-service vector.
- Without centralized logging, an attacker can **destroy evidence** by simply removing the container.
- Audit-grade logging requires logs to go somewhere the container *cannot reach*.

---

### 1.1 — Set Up the Lab Directory

```bash
cd ~
mkdir -p docker-security-lab/lab2-logging-monitoring
cd docker-security-lab/lab2-logging-monitoring
```

---

### 1.2 — JSON File Log Driver (Default Driver)

The `json-file` driver stores container logs as JSON on the host filesystem under `/var/lib/docker/containers/<container-id>/`.

**The security problem with defaults:** Without size limits, logs grow forever. An attacker who triggers thousands of errors can exhaust your disk.

**Run a container with safe log settings:**

```bash
docker run -d \
  --name app-json-logs \
  --log-driver json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  --log-opt labels=env,version \
  --label env=production \
  --label version=1.0 \
  myapp:runtime-test
```

> **What these flags do:**
> - `--log-driver json-file` → explicitly sets the driver (don't rely on defaults)
> - `--log-opt max-size=10m` → each log file is capped at 10 MB
> - `--log-opt max-file=3` → keep only 3 rotated files (max 30 MB total per container)
> - `--log-opt labels=env,version` → inject container labels into every log line for filtering

**View the live logs:**

```bash
docker logs app-json-logs
```

**Find the raw JSON log file on the host:**

```bash
# Get the container's full ID
CONTAINER_ID=$(docker inspect app-json-logs -f '{{.Id}}')
echo "Container ID: $CONTAINER_ID"

# List the log file
sudo ls -lh /var/lib/docker/containers/${CONTAINER_ID}/
```

**Peek at the raw JSON structure:**

```bash
sudo cat /var/lib/docker/containers/${CONTAINER_ID}/${CONTAINER_ID}-json.log | head -5
```

> 🔍 Notice each log line is a JSON object with `log`, `stream`, and `time` fields. This is what log aggregators like Fluentd parse.

**Generate some traffic to populate logs:**

```bash
# Send a few requests to the app
curl http://localhost:5000/ 2>/dev/null || echo "App may not be serving HTTP — check port mapping"
```

**Cleanup:**

```bash
docker rm -f app-json-logs
```

---

### 1.3 — Syslog Driver (Centralized Logging)

The syslog driver sends logs to a syslog server — *outside* the container. This is the first step toward tamper-resistant logging, because the container cannot delete what it cannot access.

**Start a syslog server container:**

```bash
docker run -d \
  --name syslog-server \
  -p 514:514/udp \
  -v /tmp/syslog:/var/log \
  pbertera/syslogserver
```

> This runs a lightweight syslog daemon. Logs land in `/tmp/syslog` on your host — mounted into the syslog container.

**Start your app with the syslog driver:**

```bash
docker run -d \
  --name app-syslog \
  --log-driver syslog \
  --log-opt syslog-address=udp://localhost:514 \
  --log-opt tag="{{.Name}}/{{.ID}}" \
  myapp:runtime-test
```

> **Key option:** `tag="{{.Name}}/{{.ID}}"` — every log line is prefixed with the container name and ID. When you have 50 containers all logging to the same syslog server, this tag is how you filter them.

**Generate some activity:**

```bash
curl http://localhost:5000/ 2>/dev/null
curl http://localhost:5000/config 2>/dev/null
sleep 2
```

**View logs on the syslog server:**

```bash
docker exec syslog-server tail -20 /var/log/messages
```

> ✅ You should see log entries tagged with your container name.

**Cleanup:**

```bash
docker rm -f app-syslog syslog-server
```

---

### 1.4 — Set Daemon-Wide Logging Defaults

Instead of setting log options on every `docker run`, configure them once in Docker's daemon configuration. This is the production-grade approach — no container can accidentally run without log limits.

```bash
sudo tee /etc/docker/daemon.json > /dev/null <<EOF
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
EOF
```

**Restart Docker to apply:**

```bash
sudo systemctl restart docker
```

**Verify Docker came back up:**

```bash
docker info | grep -A3 "Logging Driver"
```

> ✅ You should see `Logging Driver: json-file` with your options applied.

> ⚠️ **Important for live sessions:** `systemctl restart docker` will stop all running containers. In production, use `--live-restore` in `daemon.json` to avoid this.

---

## Step 2: Container Audit Logs — The Forensic Trail

### 🧠 Context and Concept

Container logs tell you *what the application said*. But there's a deeper question: **who called Docker, when, and what did they touch?**

This is what **auditd** answers.

`auditd` is the Linux kernel's audit subsystem. It watches system calls and file access events at the kernel level — completely independent of Docker. It cannot be fooled by containers. Even if an attacker gains root inside a container and deletes application logs, `auditd` has already recorded the system calls on the host.

**What we audit for Docker:**
- Every execution of the `docker` binary
- Any read/write of the Docker socket (`/var/run/docker.sock`)
- Changes to Docker's configuration in `/etc/docker`
- Container runtime binaries (`containerd`, `runc`)

**Why the Docker socket?** Because the socket IS root. Anyone who can write to `/var/run/docker.sock` can start a privileged container and escape to the host. Auditing it tells you exactly who touched it and when.

---

### 2.1 — Install and Start auditd

```bash
sudo apt-get install -y auditd audispd-plugins
```

```bash
sudo systemctl start auditd
sudo systemctl enable auditd
sudo systemctl is-active auditd
```

> ✅ Expected output: `active`

---

### 2.2 — Configure Docker Audit Rules

```bash
sudo tee /etc/audit/rules.d/docker.rules > /dev/null <<EOF
# Watch the Docker binary — log every time docker is executed
-w /usr/bin/docker -p rwxa -k docker

# Watch the Docker socket — critical: socket access = potential root access
-w /var/run/docker.sock -p rwxa -k docker

# Watch Docker's configuration directory
-w /etc/docker -p rwxa -k docker

# Watch Docker's data directory
-w /var/lib/docker -p rwxa -k docker

# Watch container runtime binaries
-w /usr/bin/containerd -p rwxa -k docker
-w /usr/bin/runc -p rwxa -k docker
EOF
```

> **Flag meanings:**
> - `-w` → watch this path
> - `-p rwxa` → log read, write, execute, and attribute-change events
> - `-k docker` → tag these events with the key `docker` (makes searching easy)

**Load the rules:**

```bash
sudo augenrules --load
```

**Verify rules are active:**

```bash
sudo auditctl -l | grep docker
```

> ✅ You should see your 6 rules listed.

---

### 2.3 — Generate Audit Events

Now let's actually create some Docker activity so we have events to examine.

```bash
# Start a container
docker run -d --name audit-test nginx

# Wait a moment
sleep 2

# Stop it
docker stop audit-test

# Remove it
docker rm audit-test
```

---

### 2.4 — View and Search Audit Logs

```bash
# Search all events tagged with 'docker'
sudo ausearch -k docker | tail -50
```

```bash
# Narrow to events from today only
sudo ausearch -k docker -ts today
```

```bash
# Look for docker run events
sudo ausearch -k docker | grep "docker"
```

**Generate a summary report:**

```bash
sudo aureport -k docker --summary
```

**Check for any failed/denied events:**

```bash
sudo aureport -k docker --failed
```

> 🔍 **What to look for in an audit log:**
> - `uid=` — which user invoked Docker
> - `exe=` — the exact binary that was executed
> - `key=docker` — confirms this is a Docker-tagged event
> - `success=yes/no` — whether the operation succeeded

> 💡 **Assignment note:** In a real security investigation, you would search these logs to answer: *"At 2:47am, who started a container on this host?"*

---

## Step 3: Runtime Threat Detection with Falco — The Guards Inside the Walls

### 🧠 Context and Concept

Logging and auditing tell you what happened *after the fact*. But what if you want to know **in real time** — the moment a container does something dangerous?

That's what **Falco** does.

Falco is an open-source runtime security tool from the CNCF (Cloud Native Computing Foundation). It hooks into the Linux kernel using eBPF and watches every system call made by every process on the host — including processes inside containers. When a system call matches a **rule**, Falco fires an alert.

**Think of it this way:**
- `auditd` is the logbook — reviewed later
- `Falco` is the guard shouting *"Intruder!"* — in real time

**What kinds of things does Falco detect?**
- A shell being spawned inside a running container (`docker exec -it ... bash`)
- A container trying to read `/etc/shadow` (password file)
- A process making an unexpected outbound network connection
- Package managers (`apt`, `yum`) running inside a container at runtime
- A container writing to filesystem paths it shouldn't touch

**Why does Falco matter?**
Containers are supposed to be immutable. If something is running `bash` or `wget` inside your container at 3am, that's almost certainly an attacker — or a misconfigured container that's a security liability.

---

### 3.1 — Install Falco

```bash
# Remove any old or broken Falco repository entry
sudo rm -f /etc/apt/sources.list.d/falcosecurity.list
```

```bash
# Add the Falco GPG key using the modern method
curl -fsSL https://falco.org/repo/falcosecurity-packages.asc | \
  sudo gpg --dearmor -o /usr/share/keyrings/falcosecurity.gpg
```

```bash
# Add the Falco apt repository
echo "deb [signed-by=/usr/share/keyrings/falcosecurity.gpg] https://download.falco.org/packages/deb stable main" | \
  sudo tee /etc/apt/sources.list.d/falcosecurity.list
```

```bash
# Update package list and install Falco
sudo apt update
sudo apt install -y falco
```

> During installation, if prompted about the kernel driver method, select **modern eBPF** (no kernel module compilation needed).

---

### 3.2 — Start Falco

```bash
# Start the Falco service (modern eBPF mode)
sudo systemctl start falco-modern-bpf

# Enable it to start on boot
sudo systemctl enable falco-modern-bpf

# Confirm it's running
sudo systemctl is-active falco-modern-bpf
```

> ✅ Expected: `active`

**Quick sanity check — view Falco's startup output:**

```bash
sudo journalctl -u falco-modern-bpf --since "2 minutes ago" | tail -20
```

> You should see Falco reporting how many rules it loaded.

---

### 3.3 — Browse the Built-in Rules

Falco ships with a large ruleset covering the most common attack patterns. Let's take a quick look.

```bash
# Page through the default rules file
sudo cat /etc/falco/falco_rules.yaml | grep "^- rule:" | head -30
```

> This shows you the rule names. Rules like `Terminal shell in container`, `Read sensitive file untrusted`, and `Unexpected outbound connection destination` are included by default.

**Look at one rule in detail:**

```bash
sudo grep -A 10 "Terminal shell in container" /etc/falco/falco_rules.yaml | head -15
```

> 🔍 Notice the structure: every Falco rule has a `condition` (what to watch for), an `output` (what to log), and a `priority` (severity level).

---

### 3.4 — Create Custom Rules

Falco's built-in rules are good, but your environment has specific needs. You extend Falco through a **local rules file** — this file overrides or adds to the defaults without touching the base ruleset.

```bash
sudo tee /etc/falco/falco_rules.local.yaml > /dev/null <<'EOF'
# Rule 1: Detect suspicious network tools running inside containers
- rule: Suspicious Container Network Tool
  desc: >
    Detects known attack tools or unexpected network utilities
    being executed inside a running container.
  condition: >
    container.id != host and
    (proc.name in (nc, ncat, nmap, telnet, wget, curl) or
     (spawned_process and proc.cmdline contains "bash -i"))
  output: >
    Suspicious network tool detected in container
    (user=%user.name container=%container.name
     image=%container.image.repository command=%proc.cmdline)
  priority: WARNING
  tags: [network, container, custom]

# Rule 2: Detect writes outside /tmp in containers
- rule: Unexpected Write in Container Filesystem
  desc: >
    Detects file write attempts outside /tmp in containers.
    Immutable containers should never write to their own filesystem at runtime.
  condition: >
    container.id != host and
    evt.type in (open, openat) and
    evt.is_open_write=true and
    fd.typechar='f' and
    not fd.name startswith /tmp and
    not fd.name startswith /proc and
    not fd.name startswith /dev
  output: >
    Write to non-temp path detected in container
    (file=%fd.name container=%container.name
     image=%container.image.repository command=%proc.cmdline)
  priority: WARNING
  tags: [filesystem, container, custom]
EOF
```

**Restart Falco to load the custom rules:**

```bash
sudo systemctl restart falco-modern-bpf
sleep 3
sudo systemctl is-active falco-modern-bpf
```

> ✅ Expected: `active`

**Verify both rules loaded:**

```bash
sudo journalctl -u falco-modern-bpf --since "1 minute ago" | grep -i "rule\|loaded\|error"
```

---

### 3.5 — Trigger Alerts (Two-Terminal Exercise)

This is the most important part of the lab. We are going to *prove* Falco works by deliberately triggering its rules.

---

**Open Terminal 1 — Watch Falco in real time:**

```bash
sudo journalctl -fu falco-modern-bpf
```

> Leave this running. Every Falco alert will appear here as it fires.

---

**Open Terminal 2 — Run the test container and trigger alerts:**

**Start the test container:**

```bash
docker run -d --name falco-test myapp:runtime-test
sleep 2
```

---

**Trigger 1 — Shell in container (built-in rule: "Terminal shell in container")**

```bash
docker exec -it falco-test sh
```

> In Terminal 1 you should see a WARNING fire immediately.
> Type `exit` to leave the shell.

```bash
exit
```

---

**Trigger 2 — Suspicious network tool (your custom rule)**

```bash
docker exec falco-test wget http://example.com
```

> In Terminal 1: watch for your custom `Suspicious Container Network Tool` rule firing.

---

**Trigger 3 — Write outside /tmp (your custom rule)**

```bash
docker exec falco-test touch /app/suspicious.txt
```

> In Terminal 1: watch for `Unexpected Write in Container Filesystem`.

---

**Trigger 4 — Read sensitive file (built-in rule)**

```bash
docker exec falco-test cat /etc/shadow
```

> This may fail with permission denied inside the container — but Falco will still log the *attempt*.

---

**Cleanup:**

```bash
docker rm -f falco-test
```

---

### 3.6 — Analyze and Count Alerts

After triggering alerts, switch back to a single terminal and analyze what Falco recorded.

**View alerts from the last 10 minutes:**

```bash
sudo journalctl -u falco-modern-bpf --since "10 minutes ago" \
  | grep -i "warning\|notice\|critical"
```

**Count alerts grouped by rule (shows which rules fired most):**

```bash
sudo journalctl -u falco-modern-bpf --since "10 minutes ago" \
  | grep -i "warning\|notice\|critical" \
  | grep -oP 'rule=\K[^,]+' \
  | sort | uniq -c | sort -rn
```

**Confirm Falco is still healthy after all the activity:**

```bash
sudo systemctl status falco-modern-bpf --no-pager
```

---

## Lab Summary — What You Built Today

| Layer | Tool | What it does |
|---|---|---|
| Application logs | Docker log drivers | Captures stdout/stderr from containers with size limits |
| Centralized logs | Syslog driver | Ships logs outside the container to a remote server |
| Host-level audit | auditd | Records every Docker binary call and socket access at kernel level |
| Runtime detection | Falco (eBPF) | Watches system calls in real time and alerts on suspicious behavior |

**The security principle behind all of this:** Defense in depth. Walls (hardening) + Logbook (logging) + Audit trail (auditd) + Guards (Falco) = a system where attackers cannot act silently.

---

## Assignment Questions

Answer these based on what you ran today. Use your terminal output and logs as evidence.

1. **Log rotation:** What happens if you set `max-file=1` and `max-size=1m` on a very noisy container? What is the risk of setting `max-file` too low from a security/compliance perspective?

2. **auditd:** Run `sudo ausearch -k docker -ts today` and identify one audit event. Explain what `uid`, `exe`, and `success` fields tell you about that event.

3. **Syslog centralization:** Why is sending container logs to an external syslog server more secure than relying on `json-file` alone? What specific attack does it protect against?

4. **Falco rule logic:** Look at the custom rule `Suspicious Container Network Tool`. Explain in plain English what the `condition` block is checking. What does `container.id != host` mean and why is that check important?

5. **Trigger analysis:** After running Trigger 1 (shell in container), what exact Falco output did you see in Terminal 1? Copy the alert line and identify the `container`, `user`, and `command` fields.

6. **Design question:** You are running a production e-commerce app in Docker. The app legitimately uses `curl` to call payment APIs. How would you modify the `Suspicious Container Network Tool` rule so that your app's `curl` calls are allowed, but a `curl` call from any *other* container is still flagged?

---

## Quick Reference — Key Commands

```bash
# Logging
docker logs <container>                          # View container logs
docker inspect <container> -f '{{.Id}}'         # Get full container ID
sudo ls /var/lib/docker/containers/<id>/         # Find raw log files

# auditd
sudo auditctl -l | grep docker                  # List active audit rules
sudo ausearch -k docker -ts today               # Search today's Docker audit events
sudo aureport -k docker --summary               # Summary report

# Falco
sudo systemctl status falco-modern-bpf          # Check Falco status
sudo journalctl -fu falco-modern-bpf            # Live alert stream
sudo journalctl -u falco-modern-bpf --since "10 minutes ago"  # Recent alerts
sudo cat /etc/falco/falco_rules.local.yaml      # View your custom rules
```

---

> **End of Lab 2**  
> Next session: Docker Network Security — segmenting containers, internal networks, and preventing lateral movement.
