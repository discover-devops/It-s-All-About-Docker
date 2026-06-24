# Module 2: Docker Network Types + Working With Networks + Bridge Deep Dive

## Context

Before talking about Docker networking, I want to show you that Docker networking is not magic.

Docker is simply using Linux networking primitives and automating them for us.

Today we will start from a clean Ubuntu machine.

Before Docker installation, we'll inspect the machine and understand what network interfaces currently exist.

Then we'll install Docker and observe what changes Docker introduces into the operating system.

By the end of this session, you should understand:

* Why docker0 gets created
* What a bridge network is
* What a user-defined bridge network is
* How containers get IP addresses
* How packet flow works
* How NAT works
* How container-to-container communication works
* Why DNS works on custom bridge networks
* What Host, None, Macvlan and Overlay networks actually do

Most importantly, you should understand what Linux is doing underneath Docker.

---

## Lab 1: Observe Ubuntu Before Docker Installation

Start with a clean Ubuntu machine.

Run:

ip addr show

Ask students:

"How many interfaces do you see?"

Typically they will see:

* lo
* ens3 / eth0

depending on cloud provider.

Draw:

Internet
|
|
Host NIC
|
Ubuntu

Tell students:

At this point Docker is not installed.

There is no docker0 bridge.

There are no container networks.

This is a normal Linux machine.

---

## Lab 2: Install Docker

Install Docker.

After installation run:

ip addr show

Immediately students will notice:

docker0

Ask:

"Who created this interface?"

Answer:

Docker.

Now run:

docker network ls

Students will see:

bridge
host
none

This becomes the perfect entry point into Docker Network Drivers.

At this point explain:

Docker ships with multiple network drivers.

The driver determines how networking behaves.

Exactly like volume drivers determine how storage behaves.

The most important drivers are:

* bridge
* host
* none
* macvlan
* overlay

We'll study them one by one.

---

## Bridge Network (Most Important)

Tell students:

This is the default network driver.

If you create a container and do nothing, Docker automatically attaches it to the bridge network.

Run:

docker run -dit --name c1 ubuntu bash

Inspect:

docker inspect c1

Search for:

IPAddress

Students will see something like:

172.17.0.2

Now draw:

Container
|
eth0
|
veth pair
|
docker0
|
Host NIC
|
Internet

Now introduce a very important Linux concept.

A container does not get a physical NIC.

A container gets a virtual network interface.

That interface exists because Docker creates a veth pair.

One end lives inside the container namespace.

One end lives on the host bridge.

---

## Namespace Discussion

This is where Linux namespaces come in.

Container networking is built primarily using Network Namespaces.

Ask:

"How can two containers both have an eth0?"

Because each container lives inside its own network namespace.

Draw:

Namespace A

eth0
127.0.0.1

Namespace B

eth0
127.0.0.1

Same names.

Different namespaces.

No conflict.

Explain:

Namespaces isolate networking resources exactly the same way they isolate process tables and mount points.

This is why every container believes it owns its own network stack.

---

## Lab 3: Observe Container Interface

Run:

docker exec -it c1 bash

Inside:

ip addr show

Students will see:

lo
eth0

Ask:

"Is this eth0 a real NIC?"

Answer:

No.

This is a virtual ethernet device.

The other end exists on the host.

Exit container.

On host:

ip link

Students will notice veth interfaces.

Explain:

One end in container.

One end connected to docker0 bridge.

---

## How Container Reaches Google

Now explain packet flow.

Inside container:

ping google.com

Draw packet journey.

Container eth0
|
|
veth
|
|
docker0
|
|
Host Routing Table
|
|
Host NIC
|
Internet

Return packet follows the reverse path.

Students finally understand that containers do not magically access the internet.

Linux networking is doing all the work.

---

## NAT Discussion

Ask:

"Can the internet route 172.17.0.2?"

Answer:

No.

Private IP.

So Docker performs NAT.

Draw:

Container IP
172.17.0.2

translated to

Host IP
10.x.x.x

before sending traffic to internet.

Explain:

This is exactly why containers can access the internet even though their addresses are private.

---

## Lab 4: Container To Container Communication

Create:

docker run -dit --name c1 ubuntu bash

docker run -dit --name c2 ubuntu bash

Get IPs.

Ping:

docker exec c1 ping <c2-ip>

Draw:

c1
|
docker0
|
c2

Important observation:

Traffic never leaves host.

Everything switches locally via docker0.

This is exactly what a bridge does.

A bridge is a virtual switch.

---

## User Defined Bridge Network

Now create:

docker network create mynet

Run:

docker run -dit --name web --network mynet ubuntu bash

docker run -dit --name db --network mynet ubuntu bash

Explain:

Custom bridge networks provide DNS.

Default bridge network does not.

This is one of the most important interview questions.

Inside web:

ping db

Students will see name resolution working.

Explain:

Docker's embedded DNS server resolves:

db
↓
Container IP

This is why Docker recommends custom bridge networks.

---

## Lab 5: Custom CIDR Range

Create:

docker network create 
--subnet=172.20.0.0/16 
mynet2

Run container.

Inspect IP.

Students will see:

172.20.x.x

Explain:

Docker IP allocation comes from the network subnet.

Not random.

---

## Host Network

Now discuss Host Driver.

Run:

docker run --network host nginx

Ask:

"What IP does container get?"

Answer:

None.

Container uses host networking directly.

Draw:

Container
|
Host Network Stack

No docker0.

No NAT.

No port mapping.

Maximum performance.

Minimum isolation.

---

## None Network

Run:

docker run --network none ubuntu

Inside:

ip addr show

Students see:

lo

Only loopback.

Explain:

Completely isolated container.

No internet.

No communication.

Useful for highly restricted workloads.

---

## Macvlan (Excellent Demo)

This is where students get excited.

Explain:

Instead of receiving a private Docker IP,

the container receives a real IP from the physical network.

Draw:

Router
|
Switch
|
Container

Container appears as a real machine.

Create Macvlan network.

Assign container IP.

Run nginx.

Access:

http://container-ip

Students directly access container.

No port mapping.

No NAT.

The router believes the container is a separate machine.

This is the biggest difference from bridge networking.

---

## Overlay Network

Keep this conceptual.

Explain:

Bridge works on one host.

Overlay works across multiple hosts.

Draw:

Host A
|
Overlay Network
|
Host B

Containers communicate as if they were on the same switch.

Used by:

Docker Swarm
Kubernetes CNI Solutions

Do not spend much time here.

---

## Key Takeaways

Docker networking is Linux networking.

Bridge networks are virtual switches.

Containers use network namespaces.

Containers get virtual interfaces, not physical NICs.

docker0 is a Linux bridge.

NAT allows containers to access the internet.

Custom bridge networks provide DNS.

Macvlan provides real network identities.

Overlay enables multi-host communication.
