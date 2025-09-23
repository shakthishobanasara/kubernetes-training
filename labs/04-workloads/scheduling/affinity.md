# Kubernetes Scheduling: Manual, nodeSelector, Node Affinity, Pod Affinity & Anti-Affinity

A compact field guide with **ready-to-run YAMLs**, commands, and all common options.

---

## 0) Pre-req: Label your nodes for demos
```bash
kubectl label nodes worker1 disktype=ssd --overwrite
kubectl label nodes worker2 disktype=hdd --overwrite
kubectl label nodes worker1 zone=us-east-1a --overwrite
kubectl label nodes worker2 zone=us-east-1b --overwrite
```
Also ensure you have a deployment to match against for Pod (anti)affinity demos:
```bash
kubectl create deploy frontend --image=nginx --replicas=1   --dry-run=client -o yaml | kubectl apply -f -
kubectl label deploy/frontend app=frontend
```

---

## 1) Manual Scheduling with `nodeName`
Forces the pod onto an exact node. Bypasses the scheduler.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: manual-schedule
spec:
  nodeName: worker1
  containers:
  - name: bb
    image: busybox
    command: ["sh","-c","sleep 3600"]
```
**Notes**
- If `worker1` is unschedulable / lacks resources, Pod stays `Pending`.
- No fallback to other nodes.

---

## 2) `nodeSelector` (basic, exact key=value)
```bash
kubectl label nodes worker1 gpu=true --overwrite
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nodeselector-demo
spec:
  nodeSelector:
    gpu: "true"
  containers:
  - name: app
    image: nginx
```
**Notes**
- Simple equality matching only.
- For expressions, use **Node Affinity**.

---

## 3) Node Affinity (advanced, expressions & weights)

### 3.1 Hard requirement — `requiredDuringSchedulingIgnoredDuringExecution`
Pod **must** match these expressions at scheduling time.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-affinity-required
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In            # In | NotIn | Exists | DoesNotExist | Gt | Lt
            values: ["ssd"]
          - key: zone
            operator: In
            values: ["us-east-1a","us-east-1b"]
  containers:
  - name: app
    image: nginx
```

### 3.2 Soft preference — `preferredDuringSchedulingIgnoredDuringExecution`
Scheduler tries to honor these with a weight (1–100); can still place elsewhere.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-affinity-preferred
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values: ["us-east-1a"]
      - weight: 20
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values: ["ssd"]
  containers:
  - name: app
    image: nginx
```
**Notes**
- `IgnoredDuringExecution`: if node labels change after scheduling, pod stays put.
- The not-yet-implemented `requiredDuringSchedulingRequiredDuringExecution` is **not** available in most versions.

---

## 4) Pod Affinity (co-locate with other Pods)
Use when pods benefit from low-latency or shared caches.

### 4.1 Hard requirement (required)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-affinity-required
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values: ["frontend"]
        topologyKey: "kubernetes.io/hostname"  # same node
  containers:
  - name: app
    image: nginx
```
### 4.2 Soft preference (preferred)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-affinity-preferred
spec:
  affinity:
    podAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 50
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: frontend
          topologyKey: "topology.kubernetes.io/zone"  # same zone if possible
  containers:
  - name: app
    image: nginx
```

### 4.3 Matching across namespaces
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-affinity-ns
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: frontend
        topologyKey: "kubernetes.io/hostname"
        namespaces: ["default","team-a"]            # explicit list
        # or use namespaceSelector to match namespaces by labels:
        # namespaceSelector:
        #   matchExpressions:
        #   - key: team
        #     operator: In
        #     values: ["platform"]
  containers:
  - name: app
    image: nginx
```
**Label selector operators for pod (anti)affinity**: `In`, `NotIn`, `Exists`, `DoesNotExist`

---

## 5) Pod Anti-Affinity (spread away from other Pods)
Use for HA: avoid placing replicas on the same node / zone.

### 5.1 Hard requirement (required)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-antiaffinity-required
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: frontend
        topologyKey: "kubernetes.io/hostname"  # must be on a different node
  containers:
  - name: app
    image: nginx
```

### 5.2 Soft preference (preferred)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-antiaffinity-preferred
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 90
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: frontend
          topologyKey: "topology.kubernetes.io/zone" # prefer different zone
  containers:
  - name: app
    image: nginx
```
**Tips**
- For Deployments/StatefulSets, prefer **pod anti-affinity** to spread replicas.
- You can combine with **topology spread constraints** for even distribution.

---

## 6) Combine Node Affinity + Pod (Anti)Affinity
All *required* terms must be satisfied; *preferred* terms influence scoring.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: combined-affinity-demo
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values: ["ssd"]
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: frontend
        topologyKey: "kubernetes.io/hostname"
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 50
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: backend
          topologyKey: "topology.kubernetes.io/zone"
  containers:
  - name: app
    image: nginx
```

---

## 7) Debugging & Tips
```bash
# Why is my pod pending?
kubectl describe pod <name> | sed -n '/Events:/,$p'

# See node labels to debug matching
kubectl get nodes --show-labels

# Taints can also block scheduling; list them:
kubectl get nodes -o json | jq -r '.items[].spec.taints // []'
```
**Common pitfalls**
- Using `nodeSelector` when you need expressions (use Node Affinity).
- Missing/typo in `topologyKey` (must be a valid node label key).
- Forgetting to label nodes or label mismatch types.
- Combining strict requirements that have no feasible nodes → pod stays `Pending`.

---

## 8) Clean-up
```bash
kubectl delete pod manual-schedule nodeselector-demo   node-affinity-required node-affinity-preferred   pod-affinity-required pod-affinity-preferred pod-affinity-ns   pod-antiaffinity-required pod-antiaffinity-preferred   combined-affinity-demo --ignore-not-found
kubectl delete deploy frontend --ignore-not-found
```

---

### Cheatsheet Summary

| Feature        | Field Location            | Strength  | Best for |
|----------------|---------------------------|-----------|---------|
| `nodeName`     | `spec.nodeName`           | Hard      | Pin to exact node (dev/test) |
| `nodeSelector` | `spec.nodeSelector`       | Hard      | Simple key=value |
| Node Affinity  | `spec.affinity.nodeAffinity` | Hard/Soft | Expressions with operators & weights |
| Pod Affinity   | `spec.affinity.podAffinity`  | Hard/Soft | Co-locate with labeled pods |
| Pod Anti-Aff.  | `spec.affinity.podAntiAffinity` | Hard/Soft | Spread away for HA |
