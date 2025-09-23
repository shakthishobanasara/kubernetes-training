# Kubernetes Scheduling — Phases, Priority & Preemption (Quick Field Guide)

This guide explains **how the scheduler places Pods**, phase-by-phase, and includes demos for **PriorityClass** and **preemption**.

---

## Phase 0 — Setup (what the scheduler sees on each Pod)
The scheduler uses these fields/rules on a Pod to decide where it can go:
- **Resources**: `resources.requests` (CPU/memory)
- **Node rules**: `nodeName`, `nodeSelector`, **Node Affinity** (required/preferred)
- **Pod rules**: **Pod Affinity/Anti-Affinity** (required/preferred)
- **Taints/Tolerations**
- **Priority**: `priorityClassName` (affects *which* Pod is scheduled first)
- Cluster state: node health, capacity, images present, topology, etc.

---

## Phase 1 — Scheduling Queue
- New Pods enter the **scheduling queue**.
- Pods are dequeued in order of **priority** (higher first), then fairness/FIFO for equal priorities.
- Backoff applies to Pods that failed to schedule repeatedly.

**Inspect:**  
```bash
kubectl get pods -A --field-selector=status.phase=Pending
```

---

## Phase 2 — Filtering (aka Predicates)
Eliminate nodes that **cannot** run the Pod. Reasons include:
- Not enough free **CPU/Memory** for the Pod’s **requests**
- Node has **taints** not tolerated by the Pod
- Node fails **nodeSelector / required Node Affinity**
- Pod-level **(anti)affinity** rules unmet
- Node conditions (NotReady, DiskPressure, etc.)

**Inspect failures:**  
```bash
kubectl describe pod <pod> | sed -n '/Events:/,$p'
```

---

## Phase 3 — Scoring (aka Priorities)
Score each feasible node (0–100). Higher is better.
Common scoring plugins/factors:
- **LeastRequested / BalancedResourceAllocation** — spread/balance resource usage
- **ImageLocality** — prefer nodes that already have the image
- **TopologySpread** — spread across zones / nodes
- **Preferred Node Affinity** — add weight if a node matches preferred terms
- **Preferred Pod (Anti)Affinity** — reward/penalize co-location patterns

> Result is a ranked list of nodes for the Pod.

---

## Phase 4 — Node Selection
- Pick the **highest-scoring** node.
- On ties, choose pseudo-randomly among the top nodes.

**Observe scheduling result:**  
```bash
kubectl get pod <pod> -o jsonpath='{.spec.nodeName}'
```

---

## Phase 5 — Binding
- Scheduler **binds** Pod → updates `.spec.nodeName`.
- Then the **kubelet** on that node pulls images, creates containers, runs probes, etc.

**Inspect bindings (events):**  
```bash
kubectl get events --sort-by=.lastTimestamp | grep -i scheduled
```

---

## Phase 6 — Preemption (only if needed and enabled)
If **no node is feasible**, scheduler may **preempt** (evict) lower-priority Pods to make room for a higher-priority Pod.

- Only considers **lower-priority** victims.
- Tries to **minimize disruption** (fewest evictions).
- Evicted Pods are rescheduled elsewhere (if possible).

**Watch for preemption:** look for events `Preempted` / `Evicted` on victims.

---

# Demo: Priority & Preemption

> This demo creates a **high-priority Pod** that can preempt lower-priority ones if resources are tight.

## 1) Create PriorityClasses
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 1000
globalDefault: false
description: "Low priority workloads"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "High priority, may preempt"
```
Apply:
```bash
kubectl apply -f priorityclasses.yaml
```

## 2) Saturate a node with low-priority Pods
> Adjust replicas/requests to fit your node capacity so it gets full.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: low-load
spec:
  replicas: 6
  selector:
    matchLabels: { app: low-load }
  template:
    metadata:
      labels: { app: low-load }
    spec:
      priorityClassName: low-priority
      containers:
      - name: cpu
        image: nginx
        resources:
          requests:
            cpu: "200m"
            memory: "128Mi"
```
Apply and confirm many Pods land on the **same node**:
```bash
kubectl apply -f low-load.yaml
kubectl get pod -l app=low-load -o wide
```

## 3) Submit a high-priority Pod that needs room
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: must-run
spec:
  priorityClassName: high-priority
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        cpu: "1000m"
        memory: "256Mi"
```
Apply:
```bash
kubectl apply -f must-run.yaml
```

## 4) Observe preemption
If no node can fit `must-run`, the scheduler **preempts** lower-priority Pods on a chosen node.
```bash
# Watch scheduling / preemption events
kubectl get events --sort-by=.lastTimestamp | egrep -i 'preempt|evict|schedule|must-run'

# See which Pods got evicted/victimized
kubectl get pods -o wide
```

> Tip: If preemption doesn’t happen, increase `must-run` requests or increase `low-load` replicas until the chosen node has to evict.

---

# Quick Reference (Cheatsheet)

**Phases:**  
1) **Queue** → 2) **Filter** → 3) **Score** → 4) **Select** → 5) **Bind** → 6) **Preempt** (if needed)

**Priority:** Higher `PriorityClass.value` = scheduled earlier; can **preempt** lower-priority Pods when resources are tight.

**Common Gotchas:**  
- Pod stuck Pending? Check **Events** (usually resource, taint, or (anti)affinity).  
- No preemption? Requests too small; cluster has spare capacity; or preemption disabled by policy.  
- Resource pressure? Right-size requests/limits; use PodDisruptionBudgets for safer evictions (note: PDBs don’t prevent preemption, but influence victim selection).

---

# Useful Commands

```bash
# Show node capacity/allocations
kubectl describe node <node> | sed -n '/Allocatable/,/Events/p'

# Show scheduler-related events (cluster-wide)
kubectl get events -A --sort-by=.lastTimestamp | tail -n 50

# See Pod’s node after scheduling
kubectl get pod <pod> -o jsonpath='{.spec.nodeName}'; echo

# Show why a Pod can’t schedule
kubectl describe pod <pod> | sed -n '/Events:/,$p'
```

---

✅ You now have a clean mental model of scheduling + a reproducible **Priority/Preemption** lab.
