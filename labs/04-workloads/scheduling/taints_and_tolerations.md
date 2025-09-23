# Kubernetes Taints and Tolerations — Notes & Demo

## 🔹 What are Taints?
- Taints are applied **on Nodes**.  
- They mark a node so that **only specific Pods** (with matching tolerations) can be scheduled there.  
- Think: **“Keep pods away unless they explicitly tolerate me.”**

Command to add a taint:
```bash
kubectl taint nodes <node-name> key=value:effect
```

## 🔹 Effects of Taints
- **NoSchedule** → New pods will not be scheduled unless they tolerate the taint.  
- **PreferNoSchedule** → Scheduler avoids placing pods here but not a strict rule.  
- **NoExecute** → New pods are not scheduled + existing pods without toleration are evicted.  

Remove taint:
```bash
kubectl taint nodes <node-name> key:effect-
```

---

## 🔹 What are Tolerations?
- Tolerations are applied **on Pods**.  
- They allow pods to be scheduled on tainted nodes.  
- Think: **“I can live with this taint.”**

Defined in Pod spec:
```yaml
tolerations:
- key: "key"
  operator: "Equal"
  value: "value"
  effect: "NoSchedule"
```

---

## 🔹 Demo: Taints and Tolerations

### Step 1: Taint a node
```bash
kubectl taint nodes worker1 dedicated=experiment:NoSchedule
```
This means: only Pods with a toleration for `dedicated=experiment` can run on `worker1`.

### Step 2: Pod **without toleration**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: no-toleration-pod
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
```
- This Pod will **not** schedule on `worker1` (tainted).  
- It will go to another node, or stay `Pending` if no other node is free.

### Step 3: Pod **with matching toleration**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: toleration-pod
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "experiment"
    effect: "NoSchedule"
```
- This Pod **can** run on `worker1`.

### Step 4: Using `NoExecute` effect
```bash
kubectl taint nodes worker1 env=prod:NoExecute
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: tolerate-noexecute-pod
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
  tolerations:
  - key: "env"
    operator: "Equal"
    value: "prod"
    effect: "NoExecute"
    tolerationSeconds: 30
```
- Pod can run on the tainted node but will be **evicted after 30s** if tolerationSeconds is set.

---

## 🔹 Common Use-Cases
- **Dedicated nodes**: Ensure only certain workloads (e.g., GPU jobs, monitoring agents) run on specific nodes.  
- **Special hardware**: Nodes with SSD, GPU, or large memory.  
- **Evict workloads**: Force non-tolerant pods to leave a node.  

---

## 🔹 Quick Reference

| Command | Purpose |
|---------|---------|
| `kubectl taint nodes <node> key=value:NoSchedule` | Prevent scheduling unless tolerated |
| `kubectl taint nodes <node> key=value:PreferNoSchedule` | Avoid scheduling unless necessary |
| `kubectl taint nodes <node> key=value:NoExecute` | Evict pods unless tolerated |
| `kubectl taint nodes <node> key-` | Remove taint |

---
