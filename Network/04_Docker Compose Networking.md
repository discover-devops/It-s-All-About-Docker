# Module 4: Docker Compose Networking

## Context

Up until now, everything we have done has been manually.

We created networks manually.

```bash
docker network create mynet
```

We attached containers manually.

```bash
docker run --network mynet
```

We inspected networks manually.

```bash
docker network inspect mynet
```

This is fine for learning.

But in real projects nobody runs:

```bash
docker run ...
docker run ...
docker run ...
docker run ...
```

Imagine a real application:

```text
Frontend
Backend API
Database
Redis
RabbitMQ
```

Are we going to start all of these manually?

No.

This is where Docker Compose comes in.

Docker Compose not only starts multiple containers together but also automatically creates networking between them.

Today's goal is to understand:

1. How Compose creates networks
2. How Service Discovery works
3. How containers communicate using names
4. Multi-tier architecture networking

---

# Lab 1: Create Your First Compose Network

## Objective

Understand that Docker Compose automatically creates a network.

---

### Step 1

Create a directory.

```bash
mkdir compose-lab

cd compose-lab
```

---

### Step 2

Create a compose file.

```yaml
version: '3'

services:

  web:
    image: nginx

  db:
    image: mysql
    environment:
      MYSQL_ROOT_PASSWORD: password
```

Save as:

```text
docker-compose.yml
```

or

```text
compose.yaml
```

---

### Step 3

Start the application.

```bash
docker compose up -d
```

---

### Step 4

Check containers.

```bash
docker ps
```

Observe:

```text
compose-lab-web-1
compose-lab-db-1
```

---

### Step 5

Now check networks.

```bash
docker network ls
```

Students will observe something new.

Example:

```text
compose-lab_default
```

---

## Teaching Script

Ask:

"Did we create this network?"

Answer:

No.

Docker Compose created it automatically.

This is one of the biggest advantages of Compose.

---

## Learning

Students understand:

```text
docker compose up
         ↓
Network Created Automatically
```

No need for:

```bash
docker network create
```

---

# Lab 2: Service Discovery

## Objective

Understand why containers can communicate using names.

---

### Step 1

Enter web container.

```bash
docker exec -it compose-lab-web-1 bash
```

If bash unavailable:

```bash
docker exec -it compose-lab-web-1 sh
```

---

### Step 2

Install ping.

```bash
apt update

apt install iputils-ping -y
```

---

### Step 3

Ping database container.

```bash
ping db
```

Observe:

```text
PING db (172.x.x.x)
```

---

## Teaching Script

Ask:

"How did web know the IP address of db?"

Nobody configured DNS.

Nobody configured IP.

Nobody edited hosts file.

Yet it worked.

Why?

Because Docker Compose automatically registers services inside Docker DNS.

---

Draw:

```text
web
 |
 |
Docker DNS
 |
 |
db
```

---

Explain:

Compose service name becomes DNS name.

So:

```text
Service Name
      ↓
DNS Name
```

---

## Learning

Students understand:

Instead of:

```text
172.18.0.5
```

we can simply use:

```text
db
```

This is called:

```text
Service Discovery
```

---

# Lab 3: Verify DNS Configuration

## Objective

Show Docker embedded DNS.

---

### Step 1

Inside container:

```bash
cat /etc/resolv.conf
```

Observe:

```text
nameserver 127.0.0.11
```

---

## Teaching Script

Explain:

This is Docker's embedded DNS server.

Every service registration happens here.

When web asks:

```text
Where is db?
```

Docker DNS answers:

```text
db = 172.x.x.x
```

---

## Learning

Students understand:

```text
Container Name
      ↓
Docker DNS
      ↓
Container IP
```

---

# Lab 4: Multi-Tier Application Design

## Objective

Understand how production applications are designed.

---

Draw:

```text
Internet
    |
    |
 Frontend
    |
    |
 Backend
    |
    |
 Database
```

Ask:

"Should users access the database directly?"

Answer:

No.

---

### Create Network Design

Draw:

```text
Frontend Network

web
api

------------------

Backend Network

api
db
```

Notice:

```text
api
```

belongs to both networks.

Database belongs only to backend.

---

### Compose Example

```yaml
version: '3'

services:

  web:
    image: nginx
    networks:
      - frontend

  api:
    image: nginx
    networks:
      - frontend
      - backend

  db:
    image: mysql
    environment:
      MYSQL_ROOT_PASSWORD: password
    networks:
      - backend

networks:

  frontend:

  backend:
```

---

## Teaching Script

Explain:

Users can reach:

```text
web
```

Web can reach:

```text
api
```

API can reach:

```text
db
```

But:

```text
web
```

cannot directly talk to:

```text
db
```

because they are not on the same network.

This is how production environments are segmented.

---

## Learning

Students understand:

```text
Multiple Networks
      ↓
Security
      ↓
Isolation
```

---

# Lab 5: Verify Network Membership

## Objective

Observe which container belongs to which network.

---

### Step 1

Inspect frontend network.

```bash
docker network inspect frontend
```

Observe:

```text
web
api
```

---

### Step 2

Inspect backend network.

```bash
docker network inspect backend
```

Observe:

```text
api
db
```

---

## Learning

Students understand:

Networks determine communication boundaries.

Not containers.

---

# Real World Example

Most applications look like:

```text
User
  |
Load Balancer
  |
Frontend
  |
Backend API
  |
Database
```

Every layer typically sits on a different network.

Exactly the same principle used in:

* Docker Compose
* Kubernetes
* AWS VPC Design
* Azure VNets
* OCI VCN Architecture

---

# Key Takeaways

By the end of this module students should understand:

Docker Compose automatically creates networks.

Service names automatically become DNS names.

Docker provides DNS-based service discovery.

Applications should communicate using names, not IP addresses.

Multiple networks provide isolation and security.

Compose networking is simply an automation layer on top of everything they already learned about Docker bridge networks.

---

# Transition To Final Session

Now that students understand:

* Networking Fundamentals
* Network Drivers
* Bridge Internals
* DNS
* NAT
* Port Mapping
* Service Discovery
* Docker Compose Networking

the final section is:

# Networking Summary, Interview Questions & Troubleshooting Scenarios

where we connect all concepts together and prepare students for real-world troubleshooting and interviews.
