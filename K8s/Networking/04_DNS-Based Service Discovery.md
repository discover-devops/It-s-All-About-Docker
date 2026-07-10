# Topic 4: DNS-Based Service Discovery
**Session time:** ~12–15 min (2 min analogy, 3 min how-it-works, 8-10 min lab)

---

## 1. The 60-Second Analogy

> "Before the internet, finding a business in an Indian city meant knowing someone who knew the number, or flipping through a phone book. **JustDial** fixed that — call one number, say a business name, get the current contact. If the business changed its landline, JustDial's listing updated. **The name stayed stable. The number could change.**
>
> **CoreDNS is JustDial for your cluster.** Apps ask for a Service by **name**. CoreDNS looks it up and hands back the current ClusterIP. Even in the rare case the ClusterIP changed, nobody would notice — they never call the number directly, only the name."

- JustDial directory = **CoreDNS**
- Business name = **Service DNS name** (`web-service`)
- Current phone number = **ClusterIP** (`10.96.45.123`)

---

## 2. How It Actually Works (use the diagram, trace the colors)

**Point at the diagram, walk the two sides:**

**Left side — Internal Resolution (Blue + Green):**
"Client Pod asks CoreDNS for `my-service` → CoreDNS has the record memorized (`my-service → 10.96.0.20`) → sends it straight back → Pod sends traffic to that Service IP → Service load-balances across its backend Pods."

**Right side — External Resolution (Green + Orange):**
"Client Pod asks CoreDNS for `google.com` → CoreDNS doesn't know that one, so it **forwards** the query upstream (8.8.8.8 or the node's DNS) → gets the real public IP back → hands it to the Pod → Pod talks directly to the external service."

**One-line takeaway:** *"CoreDNS answers cluster names itself. Anything it doesn't recognize, it forwards outward — same as your laptop's DNS resolver, just cluster-aware."*

### Naming rules (say fast, put on slide)
Create Service `web-service` in namespace `default` → CoreDNS auto-creates:
```
web-service.default.svc.cluster.local → <ClusterIP>
```

| From namespace `default` | Works? |
|---|---|
| `web-service` | ✅ short name |
| `web-service.default` | ✅ |
| `web-service.default.svc` | ✅ |
| `web-service.default.svc.cluster.local` | ✅ FQDN |

| From namespace `monitoring` | Works? |
|---|---|
| `web-service` | ❌ resolves inside `monitoring`, not `default` |
| `web-service.default` | ✅ |
| `web-service.default.svc.cluster.local` | ✅ |

**Ask the room:** *"Why would `web-service` alone fail from another namespace?"* → Because short names always resolve relative to your **own** namespace — same as calling an extension without the area code.

---

## 3. Live Lab

### Step 1 — DNS lookup from inside the cluster
```bash
kubectl run dns-tester --image=busybox:1.36 \
  --rm -it --restart=Never -- sh
```
Inside the temp pod:
```bash
nslookup web-service
# Server: 10.96.0.10                          <- CoreDNS
# Name:   web-service.default.svc.cluster.local
# Address: 10.96.45.123                       <- Service ClusterIP

nslookup web-service.default.svc.cluster.local
# Same result

wget -qO- web-service
# Returns nginx welcome page from one of the 3 pods
exit
```
*Talking point:* "Notice the app never touched an IP. Just a name — exactly like dialing JustDial."

### Step 2 — See what Kubernetes injects into every Pod
```bash
kubectl exec pod-beta -- cat /etc/resolv.conf
```
Point out the `nameserver` line pointing at CoreDNS, and the `search` domains that let short names resolve.

### Step 3 — Cross-namespace resolution (prove the failure AND the fix)
```bash
kubectl create namespace production
kubectl create deployment prod-web --image=nginx:1.25 -n production
kubectl expose deployment prod-web --port=80 -n production

kubectl run cross-ns-test --image=busybox:1.36 \
  --rm -it --restart=Never -- sh
```
Inside the test pod:
```bash
nslookup prod-web
# ** server can't find prod-web: NXDOMAIN     <- fails, wrong namespace assumed

nslookup prod-web.production.svc.cluster.local
# Address: 10.96.78.234                       <- works, full name given
exit
```
*Narrate live:* "Same CoreDNS, same cluster — the only difference is whether we gave it the area code."

---

## 4. Quick Check (rapid-fire)
1. What does CoreDNS do when it doesn't recognize a name like `google.com`? *(Forwards the query upstream)*
2. Will `web-service` (short name) resolve correctly from a different namespace? *(No — need namespace or FQDN)*
3. What's injected into every Pod so short names work at all? *(`/etc/resolv.conf` with nameserver + search domains)*
4. In the JustDial analogy, what plays the role of the "phone number that can change"? *(The ClusterIP)*

---

*Ready for Topic 5 (kube-proxy) whenever you say go.*
