# Kubernetes ReplicaSet — Quick Lab & Demo

This lab covers **ReplicaSet (RS)**: what it is, how it differs from a Deployment, and step-by-step commands and YAML you can run to demo it.

---

## What is a ReplicaSet?
A **ReplicaSet** ensures *a specified number of identical Pods* are running at any time. If a Pod dies, the ReplicaSet creates a replacement. ReplicaSets are usually **not** created directly in production — Deployments manage ReplicaSets and provide rolling updates, history, and declarative updates. But RS is fundamental to understand pod replication and recovery.

**Key points**
- Ensures N replicas of a Pod template are running.  
- Uses a **label selector** to manage pods.  
- Does **not** provide rollout strategies (Deployment wraps RS for that).  

---

## When to use a ReplicaSet
- Learning/education and demos.  
- Very specialized cases where you want raw control of RS without Deployment features (rare).

---

## Files in this demo
- `replicaset-demo.yaml` — a ReplicaSet running `nginx:1.23` with 3 replicas.
- `replicaset.md` — this file (explanation + commands).

---

## 1. ReplicaSet YAML (save as `replicaset-demo.yaml`)
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
      tier: frontend
  template:
    metadata:
      labels:
        app: nginx
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.23
        ports:
        - containerPort: 80
```

Notes:
- `selector.matchLabels` must match the labels in `template.metadata.labels`.
- If selector and template labels don't match, RS won't manage the Pods.

---

## 2. Create the ReplicaSet
```bash
kubectl apply -f replicaset-demo.yaml
```

Verify:
```bash
kubectl get rs
kubectl get pods -l app=nginx -o wide
kubectl describe rs nginx-rs
```

Expected:
- `kubectl get rs` shows `nginx-rs 3` replicas.
- `kubectl get pods -l app=nginx` shows 3 pods (nginx-rs-xxxxx).

---

## 3. Demo actions (commands and what to expect)

### A. Simulate Pod failure (RS auto-heals)
Delete one pod:
```bash
P=$(kubectl get pod -l app=nginx -o jsonpath='{.items[0].metadata.name}')
kubectl delete pod $P
kubectl get pods -l app=nginx -w
```
**Expect:** RS will create a new Pod to maintain the count at 3.

### B. Scale the ReplicaSet
Scale up:
```bash
kubectl scale rs nginx-rs --replicas=5
kubectl get rs
kubectl get pods -l app=nginx
```
Scale down:
```bash
kubectl scale rs nginx-rs --replicas=2
kubectl get rs
kubectl get pods -l app=nginx
```

### C. Update the Pod template (image change)
ReplicaSet will reconcile the pods to match the new template, but **ReplicaSet does not give you Deployment-style rollouts** (no pauses, no progress history). Changing the pod template typically results in new pods being created and old ones deleted to match the new spec.

Patch image (imperative):
```bash
kubectl patch rs nginx-rs --type='json' -p='[{"op":"replace","path":"/spec/template/spec/containers/0/image","value":"nginx:1.25"}]'
kubectl get pods -l app=nginx -o wide
```
**Expect:** New pods will be created with `nginx:1.25` and old pods removed as RS reconciles to the spec.

### D. Inspect ReplicaSet controller events & status
```bash
kubectl describe rs nginx-rs
kubectl get events --sort-by='.metadata.creationTimestamp'
```

### E. Delete the ReplicaSet
Delete RS and cascade delete its Pods:
```bash
kubectl delete rs nginx-rs
# pods are removed by default when RS is deleted with cascading deletion
kubectl get pods -l app=nginx
```

If you want to delete the RS but keep the Pods (orphan them), use:
```bash
kubectl delete rs nginx-rs --cascade=false
# pods remain running, but RS is gone
kubectl get pods -l app=nginx
```

---

## 4. Troubleshooting & tips

- **Selector mismatch**: If RS selector doesn't match template labels, RS won't manage pods. Always ensure `.spec.selector` matches `.spec.template.metadata.labels`.
- **Use Deployments for production**: Deployments provide rolling updates, rollbacks, and history and will create/manage ReplicaSets for you.
- **Viewing which RS controls a Pod**: Look at Pod owner references:
  ```bash
  kubectl get pod <pod-name> -o yaml | yq '.metadata.ownerReferences'
  ```
- **Multiple controllers warning**: Ensure only one controller manages a set of labels. Do not manually create pods that match an RS selector unless intended.

---

## 5. Quick demo script (copy-paste)
```bash
# apply
kubectl apply -f replicaset-demo.yaml

# show status
kubectl get rs
kubectl get pods -l app=nginx

# simulate failure (delete one pod)
kubectl delete pod $(kubectl get pod -l app=nginx -o jsonpath='{.items[0].metadata.name}')
sleep 2
kubectl get pods -l app=nginx

# scale up
kubectl scale rs nginx-rs --replicas=4
kubectl get pods -l app=nginx

# patch image
kubectl patch rs nginx-rs --type='json' -p='[{"op":"replace","path":"/spec/template/spec/containers/0/image","value":"nginx:1.25"}]'
kubectl get pods -l app=nginx -o wide

# cleanup (delete RS and pods)
kubectl delete rs nginx-rs
```

---

## 6. Summary
- ReplicaSet ensures a desired number of pod replicas.  
- ReplicaSets are the low-level building block — Deployments manage ReplicaSets for safe, declarative updates.  
- Great for demos and learning; use Deployments in production.
