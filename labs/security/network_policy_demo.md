# 🧪 Kubernetes NetworkPolicy Demo Guide (Weave Net–friendly)

> This hands‑on guide walks you through **step‑by‑step demos** to teach/verify Kubernetes NetworkPolicies. It assumes a CNI that **supports NetworkPolicy** (e.g., **Weave Net**, Calico, Cilium).

---

## ✅ Prerequisites

- You have `kubectl` access.
- Your cluster network plugin supports NetworkPolicy (**Weave Net does**).
- DNS working inside the cluster (CoreDNS).
- Comfort with creating namespaces, Pods, and Services.

---

## 0) Prepare Test Environment

We’ll create two namespaces, two simple HTTP pods (`app`, `db`) exposed as services in **prod**, and a **test client** in a separate **security** namespace.

```bash
# Namespaces
kubectl create ns prod
kubectl create ns security

# App & DB pods (nginx) + Services in prod
kubectl run app --image=nginx -n prod --labels=role=app --restart=Never --port=80
kubectl expose pod app -n prod --port=80 --name=app

kubectl run db --image=nginx -n prod --labels=role=database --restart=Never --port=80
kubectl expose pod db -n prod --port=80 --name=db

# Test client (curl) in security ns
kubectl run tester --image=curlimages/curl:8.8.0 -n security --restart=Never -- sleep 3600

# Wait until all pods are Ready
kubectl wait --for=condition=ready pod -l role=app -n prod --timeout=90s
kubectl wait --for=condition=ready pod -l role=database -n prod --timeout=90s
kubectl wait --for=condition=ready pod -l run=tester -n security --timeout=90s
```

**Baseline connectivity (before any policies):** everything should be reachable.
```bash
kubectl exec -n security pod/tester -- curl -sS http://app.prod.svc.cluster.local
kubectl exec -n security pod/tester -- curl -sS http://db.prod.svc.cluster.local
```

> If the above works, your cluster networking/DNS is fine and you’re ready to see NetworkPolicies in action.

---

## 1) Default Deny: Block All Ingress & Egress in `prod`

**Goal:** Isolate all pods in `prod` (no inbound or outbound).

```yaml
# 1-deny-all.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: prod
spec:
  podSelector: {}          # applies to ALL pods in 'prod'
  policyTypes:
  - Ingress
  - Egress
```

**Apply & test:**
```bash
kubectl apply -f 1-deny-all.yaml
# Ingress blocked (security → prod)
kubectl exec -n security pod/tester -- curl -m 3 -sS http://app.prod.svc.cluster.local || echo "blocked as expected"
kubectl exec -n security pod/tester -- curl -m 3 -sS http://db.prod.svc.cluster.local || echo "blocked as expected"
```

**Teaching point:** `podSelector: {}` selects *all* pods in the namespace. With `Ingress` and `Egress` set, we’ve made the namespace “zero‑trust by default.”

---

## 2) Allow Ingress to `app` Pods from Anywhere

**Goal:** Open only inbound traffic *to* `app` pods; keep `db` isolated.

```yaml
# 2-allow-ingress-app.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-app
  namespace: prod
spec:
  podSelector:
    matchLabels:
      role: app
  policyTypes:
  - Ingress
  ingress:
  - {}    # allow from any source
```

**Apply & test:**
```bash
kubectl apply -f 2-allow-ingress-app.yaml
kubectl exec -n security pod/tester -- curl -sS http://app.prod.svc.cluster.local  # should succeed
kubectl exec -n security pod/tester -- curl -m 3 -sS http://db.prod.svc.cluster.local || echo "db still blocked"
```

---

## 3) Allow Pod‑to‑Pod: Only `app` → `db`

**Goal:** Permit `app` pods to reach `db` pods (no one else can).

```yaml
# 3-app-to-db.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: app-to-db
  namespace: prod
spec:
  podSelector:
    matchLabels:
      role: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: app
```

**Apply & test:**
```bash
kubectl apply -f 3-app-to-db.yaml

# exec into an app pod and curl db service
APP_POD=$(kubectl -n prod get pod -l role=app -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n prod "$APP_POD" -- sh -c 'apt-get update >/dev/null 2>&1 || true; apt-get install -y curl >/dev/null 2>&1 || true; curl -sS http://db.prod.svc.cluster.local'

# From security ns still blocked
kubectl exec -n security pod/tester -- curl -m 3 -sS http://db.prod.svc.cluster.local || echo "blocked as expected"
```

---

## 4) Namespace Selector: Allow **security → prod**

**Goal:** Allow traffic from any pod in the `security` namespace to any pod in `prod`.

```yaml
# 4-namespace-selector.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: security-to-prod
  namespace: prod
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: security
```

**Apply & test:**
```bash
kubectl apply -f 4-namespace-selector.yaml
kubectl exec -n security pod/tester -- curl -sS http://app.prod.svc.cluster.local  # should succeed
kubectl exec -n security pod/tester -- curl -sS http://db.prod.svc.cluster.local  # should succeed (because ingress now allows ns=security)
```

> You can combine `namespaceSelector` and `podSelector` in a single rule to allow only specific pods from a namespace.

---

## 5) Egress Restriction: Allow Only HTTPS to `1.1.1.1`

**Goal:** Lock down outbound traffic for all `prod` pods to just **1.1.1.1:443**.

```yaml
# 5-egress-only-1-1-1-1.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: prod-egress-only-cloudflare
  namespace: prod
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 1.1.1.1/32
    ports:
    - protocol: TCP
      port: 443
```

**Apply & test:**
```bash
kubectl apply -f 5-egress-only-1-1-1-1.yaml

# Launch an egress tester in prod
kubectl run egress-tester --image=curlimages/curl:8.8.0 -n prod --restart=Never -- sleep 3600
kubectl wait --for=condition=ready pod/egress-tester -n prod --timeout=90s

# Should SUCCEED (allowed)
kubectl exec -n prod pod/egress-tester -- curl -I https://1.1.1.1

# Should FAIL (not allowed)
kubectl exec -n prod pod/egress-tester -- sh -c 'curl -m 3 -I https://8.8.8.8 || echo "blocked as expected"'
```

---

## 6) Limit Ports & Protocols: `app` → `db` only on TCP/80

**Goal:** Allow only port 80/TCP to `db` pods from `app` pods.

```yaml
# 6-app-to-db-80.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: app-to-db-80
  namespace: prod
spec:
  podSelector:
    matchLabels:
      role: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: app
    ports:
    - protocol: TCP
      port: 80
```

**Apply & test:**
```bash
kubectl apply -f 6-app-to-db-80.yaml

# Port 80 works
APP_POD=$(kubectl -n prod get pod -l role=app -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n prod "$APP_POD" -- sh -c 'curl -sS http://db.prod.svc.cluster.local:80'

# Another port (e.g., 81) should fail
kubectl exec -n prod "$APP_POD" -- sh -c 'curl -m 3 -sS http://db.prod.svc.cluster.local:81 || echo "blocked as expected"'
```

---

## 7) `ipBlock` with Exceptions (CIDR allow but exclude a sub‑range)

**Goal:** Allow ingress from 172.17.0.0/16 **except** 172.17.1.0/24 to all pods in `prod`.

```yaml
# 7-ipblock-except.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-17217-except-172171
  namespace: prod
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - ipBlock:
        cidr: 172.17.0.0/16
        except:
        - 172.17.1.0/24
```

> This is useful when you want to allow a broad subnet but carve out a smaller range (e.g., a hostile segment).

---

## 8) Observe & Cleanup

**Observe current policies:**
```bash
kubectl get netpol -A
kubectl describe netpol -n prod deny-all
```

**Reset the lab:**
```bash
kubectl delete ns prod security
```

---

## 🛠️ Troubleshooting Tips

- **“Policies don’t seem to take effect”:** Ensure your CNI **supports** NetworkPolicy (Weave Net ✅). Re‑check that policies are in the **same namespace** as target pods and `podSelector` labels match.
- **DNS name works/not works:** Policy evaluation is by **destination IP**, not hostname. DNS resolution (CoreDNS) may also need egress if you block outbound traffic—allow UDP/TCP to CoreDNS ClusterIP if required.
- **Ingress allowed but still failing:** Verify there isn’t a conflicting **deny** policy that also selects the same pods.
- **Testing tools:** `curlimages/curl` is a minimal image for HTTP tests. For arbitrary ports use `busybox` and `nc` (if compiled) or `nmap` containers for port probes.

---

## 📎 Appendix: One‑shot Validation Script (optional)

```bash
#!/bin/bash
set -euo pipefail
NS="netpol-quickcheck"
kubectl create ns $NS 2>/dev/null || true
kubectl run nginx --image=nginx -n $NS --restart=Never --port=80 2>/dev/null || true
kubectl expose pod nginx -n $NS --port=80 --name=nginx 2>/dev/null || true
kubectl run curl --image=curlimages/curl:8.8.0 -n $NS --restart=Never -- sleep 3600 2>/dev/null || true
kubectl wait --for=condition=ready pod -l run=curl -n $NS --timeout=60s
kubectl wait --for=condition=ready pod -l run=nginx -n $NS --timeout=60s
echo "Pre-policy test:"
kubectl exec -n $NS pod/curl -- curl -sS http://nginx.$NS.svc.cluster.local >/dev/null && echo "OK"
cat <<'YAML' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: netpol-quickcheck
spec:
  podSelector: {}
  policyTypes: [Ingress]
YAML
sleep 3
echo "Post-policy test (should fail):"
kubectl exec -n $NS pod/curl -- sh -c 'curl -m 3 -sS http://nginx.netpol-quickcheck.svc.cluster.local || echo BLOCKED'
kubectl delete ns $NS --ignore-not-found
```

---

### ✔️ Key Takeaways
- Start with **default deny**, then **open only what’s needed**.
- Use **podSelector**, **namespaceSelector**, and **ipBlock** to describe intent.
- Combine **Ingress**/**Egress** + **ports/protocols** for least‑privilege networking.
- With **Weave Net**, these policies are enforced cluster‑wide.
