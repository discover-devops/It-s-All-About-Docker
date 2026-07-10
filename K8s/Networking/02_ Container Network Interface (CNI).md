# Topic 2: Container Network Interface (CNI) — How Pods Get Their IPs
**Session time:** ~12–15 min (2 min analogy, 3 min flow/plugins, 7-10 min lab)

---

## 1. The 60-Second Analogy

> "Kubernetes is like a **hotel manager**. When a guest (Pod) checks in, the manager doesn't personally run cables and assign a room number — he calls **the utility company** and says 'set this room up.'
>
> The **utility company is the CNI plugin.** It picks the room number (IP), wires up the phone jack (virtual interface), and updates the building directory (routing rules) so anyone in the hotel can dial that room directly.
>
> When the guest checks out, the manager calls the utility company again: 'tear it down, free up that room number for the next guest.'
>
> Kubernetes never touches the wiring itself. It just gives instructions and records the result."

- Hotel manager = **Kubernetes / kubelet**
- Utility company = **CNI plugin** (Calico, Flannel, Weave, Cilium...)
- Room number = **Pod IP**
- Phone jack = **virtual network interface (veth)**
- Building directory = **routing rules**

---

## 2. The Flow (say it as a story, don't just read the list)

**Pod created:**
"Kubernetes says *'set up networking'* → CNI picks an IP from the node's CIDR slice → CNI creates a virtual interface inside the Pod → CNI configures routes on the node → CNI reports the IP back → Kubernetes stores it in etcd."

**Pod deleted:**
"Kubernetes says *'tear it down'* → CNI removes the interface → releases the IP → cleans up the routes."

**One-line takeaway:** *Kubernetes decides WHAT should happen. CNI actually makes it happen.*

---

## 3. Plugin Cheat Sheet (show as table, don't dwell)

| Plugin | Personality | Best for |
|---|---|---|
| **Calico** | Security guard — enforces Network Policies | Enterprise, multi-tenant |
| **Flannel** | Simple overlay, no frills | Dev / learning clusters |
| **Weave** | Mesh, cloud-agnostic | Multi-cloud / hybrid |
| **Cilium** | eBPF-powered, sees everything | High-performance, observability-heavy |

**Ask the room:** *"If you're just learning Kubernetes on a laptop, which one would you reach for?"* → Flannel. *"If you need to write 'deny traffic from namespace X'?"* → Calico.

---

## 4. Live Lab

### Step 1 — See what CIDR each node was handed
```bash
kubectl get nodes -o wide
kubectl describe node worker-node-1 | grep PodCIDR
```
*Talking point:* "This CIDR is the 'block of room numbers' CNI is allowed to hand out on this node."

### Step 2 — Confirm every Pod IP fits inside its node's CIDR
```bash
kubectl get pods -o wide
```
Match each Pod's IP against the Node's PodCIDR from Step 1 — it should always fall inside that range.

### Step 3 — See the actual "phone jacks" CNI created (needs node access)
```bash
ip addr show
```
Point out the `veth...` interfaces — one per Pod on this node. *"This is the CNI plugin's handiwork, not Kubernetes."*

### Step 4 — See the routing table CNI built
```bash
ip route
```
Point out routes to **other nodes' Pod CIDRs**. *"This is literally the building directory — how does traffic from a Pod on Node A find a Pod on Node B? These routes."*

---

## 5. Quick Check (rapid-fire)
1. Who assigns the actual Pod IP — Kubernetes or CNI? *(CNI)*
2. What does Kubernetes do with the IP once CNI reports it? *(Stores it in etcd)*
3. Which plugin would you pick for enforcing Network Policies in a multi-tenant cluster? *(Calico)*
4. What's a `veth` interface, in one word? *(The Pod's network "jack"/interface)*

---

*Ready for Topic 3 whenever you share it.*
