# Topic 3: Kubernetes Services — The Stable Endpoint
**Session time:** ~12–15 min (2 min analogy, 2 min what-it-does/doesn't-do, 8-10 min lab)

---

## 1. The 60-Second Analogy

> "Pods are like **employees who keep changing desks** — one gets sick and is replaced, another gets promoted and moves teams, a new one joins. If customers had to memorize each employee's desk number, chaos.
>
> So the company sets up a **main post office / front desk**. Customers always send mail to 'Front Desk.' The front desk knows, at any given moment, which employees are actually at their desks and forwards accordingly. Employees come and go — the Front Desk address **never changes.**
>
> That's a Kubernetes **Service**. Pods are the employees. The Service is the Front Desk."

- Employees who rotate = **Pods** (IPs change every time one restarts)
- Front Desk address = **Service ClusterIP** (stable forever)
- Front Desk's "who's currently at their desk" list = **Endpoints**

---

## 2. What a Service Does / Does NOT Do (say fast, put on slide)

| Does | Does NOT |
|---|---|
| Gives a stable ClusterIP that never changes | Know Pod IPs directly — it uses **label selectors** |
| Gives a stable DNS name mapped to that IP | Go down when a Pod restarts |
| Load-balances across all healthy matching Pods | Need you to update anything when Pods change |
| Tracks Pod IPs in/out automatically via Endpoints | |

**One-liner for the room:** *"You give the Service a label to look for. It does the rest — forever."*

---

## 3. Live Lab

### Step 1 — Create the Deployment (3 Pods)
```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
EOF

kubectl rollout status deployment/web-app
```

### Step 2 — Expose it as a Service
```bash
kubectl expose deployment web-app \
  --port=80 \
  --target-port=80 \
  --name=web-service

kubectl get service web-service
```
*Talking point:* "That's the Front Desk being built. One stable IP, no matter how many desks (Pods) are behind it."

### Step 3 — Explain port vs targetPort (30 sec, point at YAML)
```yaml
ports:
- port: 80          # what CALLERS use to reach the Service
  targetPort: 80    # what the Service forwards to on the POD
```
*Say:* "`port` is the Front Desk's public number. `targetPort` is the actual employee's desk extension. They don't have to match — that's the decoupling."

### Step 4 — Prove it tracks Pods automatically
```bash
# See which Pod IPs the Service is currently forwarding to
kubectl get endpoints web-service
```

```bash
# Kill one Pod
kubectl delete pod $(kubectl get pods -l app=web-app \
  -o jsonpath='{.items[0].metadata.name}')

# Watch endpoints update live
kubectl get endpoints web-service -w
```
**Narrate live:** "Old Pod IP disappears. New Pod IP appears the moment its replacement passes readiness checks. **I changed nothing.** The Service noticed and fixed itself."

---

## 4. Quick Check (rapid-fire)
1. What stays the same even when every Pod behind a Service is replaced? *(The ClusterIP)*
2. How does a Service know which Pods belong to it? *(Label selector, not direct IP tracking)*
3. If `port: 8080` and `targetPort: 3000` — which one does the calling app use? *(8080 — the Service's port)*
4. Who updates the Endpoints list when a Pod dies — you, or Kubernetes? *(Kubernetes, automatically)*

---

*Ready for Topic 4 (DNS-Based Service Discovery) whenever you say go.*
