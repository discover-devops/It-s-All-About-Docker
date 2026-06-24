# Module 1: Networking Fundamentals

## Context

Before students learn Docker networking, bridge networks, port mapping, or DNS-based service discovery, they need to understand some basic networking concepts. Otherwise, Docker networking becomes a set of commands to memorize rather than a system they understand.

The goal of this module is not to make them network engineers. The goal is to make them comfortable with four concepts:

* Network
* IP Address
* Port
* DNS

Everything else in Docker networking builds on these concepts. 

---

## Teaching Script

Start by asking the class a simple question:

"Suppose I have two laptops. One laptop is running a web server and the other laptop wants to access it. What is required for them to communicate?"

Let the students answer. Most will say "network."

Then explain that a network is simply a mechanism that allows devices to communicate with each other. The internet is a network. Your office LAN is a network. Your home WiFi is a network. Even Docker creates its own virtual networks.

At a high level, every network requires four things:

* Devices that communicate (hosts)
* A way to connect them
* Rules of communication (protocols)
* A way to identify them (addresses)

Then transition into the apartment analogy from the guide.

Tell them to imagine a large apartment building.

The apartment building is the Host Machine.

Each apartment is a Docker Container.

The hallways and postal system inside the building represent the Network.

To deliver mail successfully, you need an address and a door number. Networking works exactly the same way.

---

## Diagram To Draw

Draw a simple apartment building.

Inside it draw two apartments:

```text
Host Machine

+----------------------+
|                      |
| Apartment A          |
| Container A          |
|                      |
| Apartment B          |
| Container B          |
|                      |
+----------------------+
```

Then explain that communication requires a way to identify each apartment.

That leads naturally into IP addresses.

---

## Teaching Script: IP Address

Tell students that an IP Address is simply a unique identifier for a machine on a network.

Use a house address analogy.

Amazon can deliver a package because your house has an address.

Similarly, a network packet can be delivered because a device has an IP address.

Show examples:

```text
192.168.1.10
10.0.0.25
172.17.0.2
```

Then mention that Docker containers usually receive addresses from the 172.17.x.x range by default. 

---

## Teaching Script: Ports

Now ask:

"If a server has one IP address, how can it run multiple applications?"

For example:

* Web Server
* SSH Server
* PostgreSQL Database

The answer is Ports.

Explain that an IP address identifies the machine, while a port identifies the application running on that machine.

Draw:

```text
192.168.1.10

Port 22   -> SSH
Port 80   -> Web Server
Port 5432 -> PostgreSQL
```

Then connect this to Docker.

When we run:

```bash
docker run -p 8080:80 nginx
```

we are saying:

"Take traffic arriving on Host Port 8080 and forward it to Container Port 80."



---

## Mini Lab

Run:

```bash
docker run -d \
--name web \
-p 8080:80 \
nginx
```

Open:

```text
http://localhost:8080
```

Show the Nginx page.

At this point students understand why ports exist.

---

## Teaching Script: DNS

Now ask:

"Does anyone access Google using an IP address?"

Nobody does.

We use names.

DNS is simply a service that translates names into IP addresses.

```text
google.com
      ↓
142.x.x.x
```

Later in Docker networking, containers will use DNS to find other containers by name.

This becomes extremely important when we discuss custom bridge networks.

---

## Key Takeaway

Students should leave this module understanding:

* IP identifies a machine.
* Port identifies an application.
* DNS translates names into IP addresses.
* Docker containers also get IP addresses, use ports, and participate in networking.

---

## Transition

Now ask:

"If Docker containers get IP addresses, how are those containers connected together?"

That naturally leads to:

**Module 2: Docker Network Types**


