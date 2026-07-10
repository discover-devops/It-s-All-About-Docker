# Topic 1: The Pod Network Model — IP-per-Pod Architecture
**Session time:** ~12–15 min (2 min analogy, 3 min rules, 7-10 min lab)

---

## 1. The 60-Second Analogy (say this, don't read it)

> "Imagine an old office building with **one shared phone line**. To reach Team A, you dial the main number then **extension 8080**. To reach Team B, same number, **extension 8081**. If two teams grab the same extension by mistake — conflict, confusion, someone's call gets dropped.
>
> Kubernetes tears that building down and gives **every team its own direct phone number**. No extensions, no shared line, no conflicts. You just dial the person directly, from anywhere in the company."

- Shared phone + extensions = **Docker/host-port model** (Container A:8080, Container B:8081)
- Direct phone number per team = **Kubernetes Pod IP model**

That's the whole mental model. Everything else is just consequences of "every Pod gets its own number."

---

## 2. The Three Rules (write on board / slide, 1 line each)

| Rule | One-liner |
|---|---|
| 1 | Every Pod has its own unique IP — no sharing |
| 2 | Any Pod can reach any Pod, no NAT — flat network |
| 3 | The IP a Pod sees itself as = the IP others use to reach it |

**Ask the room:** *"So if my Pod is on Node 1 and yours is on Node 3, what do I need to know to talk to you?"* → Answer: just the Pod IP. Nothing else.

---

## 3. Live Lab (drive from terminal, students follow along)

### Step 1 — Create two Pods
```bash
kubectl run pod-alpha --image=nginx:1.25
kubectl run pod-beta --image=nginx:1.25
```

### Step 2 — Wait until both are Running
```bash
kubectl get pods -w
```
*Talking point while waiting:* "Watch — no port numbers being assigned here. Just pods coming up."

### Step 3 — Reveal the IPs
```bash
kubectl get pods -o wide
```
**Point at the output columns:** `IP` and `NODE`.

Expected observation:
- `pod-alpha` → e.g. `10.244.1.15` on Node A
- `pod-beta` → e.g. `10.244.2.20` on Node B
- **Different IPs. Possibly different Nodes. No port needed to tell them apart.**

### Step 4 — Prove Rule 2 (no NAT, flat network)
```bash
kubectl exec -it pod-alpha -- curl <pod-beta-IP>
```
It works — directly, with **no port-forwarding, no NAT, no extension number**. That's the "direct phone call" from the analogy, live on screen.

### Step 5 — The contrast callback (10 sec)
> "In plain Docker, both of these containers would've shared the **host's IP** and needed **different ports**. Here — two IPs, same port 80, zero conflict. That's the whole topic."

---

## 4. Quick Check (rapid-fire, no need to write answers)
1. Can two Pods in the cluster ever have the same IP? *(No)*
2. Do I need NAT to go from Pod A to Pod B? *(No)*
3. What's the Kubernetes replacement for "host port"? *(Nothing — you just use the Pod IP directly)*

---

*Ready for Topic 2 whenever you share it — I'll keep the same short-analogy + rules + lab format.*
