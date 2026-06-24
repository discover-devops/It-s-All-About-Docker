* Module 1 = Networking Fundamentals 
* Module 2 = Docker Network Drivers Deep Dive (Bridge, User Defined Bridge, Host, None, Macvlan, Overlay) 
* Module 3 = Networking Internals, Debugging, Security & Troubleshooting

This module answers questions like:

> How does `-p 8080:80` actually work?

> Why can I access a container from my browser?

> How does Docker DNS resolve container names?

> Why can a container reach the internet?

> How do I debug networking issues?

---

# Module 3: Networking Internals, Debugging, Security & Troubleshooting

## Context

By now students have seen:

* docker0 bridge
* User-defined bridge
* Host network
* None network
* Macvlan

They have created networks and attached containers.

The next question naturally becomes:

> How does Docker actually make all this work?

Because if we don't understand the internals, the first networking issue in production becomes difficult to troubleshoot.

Today's goal is not to become Linux networking experts.

Today's goal is to understand enough internals that when networking breaks, we know where to look.

---

## Part 1: Port Mapping Deep Dive

Start by asking:

"Yesterday we ran:

```bash
docker run -d -p 8080:80 nginx
```

and we opened:

```text
http://localhost:8080
```

How did that request reach port 80 inside the container?"

Most students won't know.

Perfect.

Now we create curiosity.

---

### Draw This First

On Draw.io:

```text
Browser
   |
localhost:8080
   |
Host Machine
   |
???
   |
Container:80
```

Ask:

"What is the ???"

That is what we're going to discover.

---

## Lab 1: Understanding Port Mapping

### Objective

Understand how Host Port 8080 reaches Container Port 80.

### Step 1

Start nginx.

```bash
docker run -d \
--name web \
-p 8080:80 \
nginx
```

---

### Step 2

Verify container is running.

```bash
docker ps
```

Observe:

```text
0.0.0.0:8080->80/tcp
```

---

### Step 3

Open browser.

```text
http://localhost:8080
```

Nginx page appears.

---

### Step 4

Inspect port mapping.

```bash
docker port web
```

Expected output:

```text
80/tcp -> 0.0.0.0:8080
```

---

### Learning

Students now know:

```text
Host Port 8080
        ↓
Container Port 80
```

But we still don't know how.

---

### Step 5

Show NAT rules.

```bash
sudo iptables -t nat -L -n
```

(Depending on Ubuntu version nftables may be underneath, but for teaching this is fine.)

Show Docker-created rules.

---

### Teaching Script

Explain:

Docker creates DNAT rules automatically.

When traffic arrives on:

```text
Host:8080
```

Docker rewrites the destination.

```text
Host:8080
       ↓
ContainerIP:80
```

This process is called:

```text
DNAT
Destination NAT
```

---

### Final Learning

Students should leave with:

```text
-p does not expose a port magically.

Docker creates NAT rules
that forward traffic.
```

---

# Part 2: Docker DNS Deep Dive

Now ask:

"When we created a custom bridge network, why could one container ping another using the container name?"

Example:

```bash
ping db
```

instead of:

```bash
ping 172.20.0.3
```

---

## Lab 2: Docker DNS

### Objective

Understand Docker's embedded DNS server.

### Step 1

Create network.

```bash
docker network create mynet
```

---

### Step 2

Run first container.

```bash
docker run -dit \
--name db \
--network mynet \
ubuntu bash
```

---

### Step 3

Run second container.

```bash
docker run -dit \
--name web \
--network mynet \
ubuntu bash
```

---

### Step 4

Install ping.

```bash
docker exec -it web bash
```

Inside:

```bash
apt update
apt install iputils-ping -y
```

---

### Step 5

Ping container name.

```bash
ping db
```

Observe:

```text
PING db (172.x.x.x)
```

---

### Step 6

Check DNS configuration.

```bash
cat /etc/resolv.conf
```

Observe:

```text
nameserver 127.0.0.11
```

From the guide. 

---

### Teaching Script

Explain:

Docker runs an embedded DNS service.

Inside the container:

```text
127.0.0.11
```

is Docker's DNS server.

When container asks:

```text
Where is db?
```

Docker replies:

```text
db = 172.x.x.x
```

---

### Learning

Students understand:

```text
Container Name
      ↓
Docker DNS
      ↓
Container IP
```

This is called Service Discovery.

---

# Part 3: Debugging Container Networking

Now transition.

Say:

"Everything looks easy when networking works.

What happens when it breaks?"

This is where real engineering starts.

---

## Lab 3: Investigating A Container

### Objective

Learn how to inspect networking from inside a container.

### Step 1

Enter container.

```bash
docker exec -it web bash
```

---

### Step 2

Check interfaces.

```bash
ip addr show
```

Observe:

```text
lo
eth0
```

---

### Step 3

Check routes.

```bash
ip route
```

Observe default gateway.

---

### Step 4

Check DNS.

```bash
cat /etc/resolv.conf
```

---

### Step 5

Check connectivity.

```bash
ping google.com
```

---

### Learning

Whenever networking fails:

```text
Check Interface
Check Route
Check DNS
Check Connectivity
```

Always in that order.

---

# Part 4: Security

Now ask:

"Should my database be reachable from the internet?"

Students:

"No."

Perfect.

---

## Teaching Script

In production we separate workloads.

Example:

Frontend Network

```text
Web
API
```

Backend Network

```text
API
Database
```

Database should never be directly reachable from users.

From the guide's security section. 

---

## Lab 4: Internal Network

### Objective

Create a network with no internet access.

### Step 1

Create network.

```bash
docker network create \
--internal \
internal-net
```

---

### Step 2

Run container.

```bash
docker run -dit \
--network internal-net \
ubuntu bash
```

---

### Step 3

Enter container.

```bash
docker exec -it <container-id> bash
```

---

### Step 4

Test internet.

```bash
ping google.com
```

Expected:

```text
Fails
```

---

### Learning

Internal network means:

```text
Containers can communicate internally

No internet access
```

---

# Part 5: Troubleshooting Decision Tree

This is how I would finish.

Tell students:

Whenever networking fails, don't panic.

Use a process.

---

### Problem 1

Cannot access container from browser.

Check:

```bash
docker ps
docker port container
```

Verify published port.

---

### Problem 2

Container cannot reach another container.

Check:

```bash
docker network inspect mynet
```

Verify both containers are attached.

---

### Problem 3

Container cannot resolve names.

Check:

```bash
cat /etc/resolv.conf
```

Verify:

```text
127.0.0.11
```

---

### Problem 4

Container cannot reach internet.

Check:

```bash
ip route
ping 8.8.8.8
```

---

## Final Takeaway

By the end of Module 3, students should understand:

* How port mapping works.
* What DNAT is.
* How Docker DNS works.
* How service discovery works.
* How to inspect container networking.
* How to create isolated networks.
* How to troubleshoot networking issues systematically.

And most importantly:

> Docker networking is not magic.

It is Linux networking, Linux namespaces, Linux bridges, routing, DNS and NAT wrapped behind Docker commands.
