# Kubernetes Pod Patterns — Practical Examples (No external images)

This document covers common **Pod design patterns** used in Kubernetes with clear explanations and **copy‑paste YAML examples** you can run in a lab. No external images are included — everything is text + YAML + commands.

---

## Table of Contents
1. [Sidecar Pattern](#1-sidecar-pattern)  
2. [Ambassador / Proxy Pattern](#2-ambassador--proxy-pattern)  
3. [Adapter Pattern (Log/Metric Adapter)](#3-adapter-pattern-logmetric-adapter)  
4. [Init Container Pattern](#4-init-container-pattern)  
5. [Probe Pattern (Liveness / Readiness / Startup)](#5-probe-pattern-liveness--readiness--startup)  
6. [Ephemeral Containers (Debugging)](#6-ephemeral-containers-debugging)  
7. [Leader Election Pattern](#7-leader-election-pattern)  
8. [Sidecar + Shared EmptyDir (for caching or data sharing)](#8-sidecar--shared-emptydir-for-caching-or-data-sharing)  
9. [Anti-affinity Pattern (Pod Placement)](#9-anti-affinity-pattern-pod-placement)  
10. [Pod Disruption Budget (Availability during maintenance)](#10-pod-disruption-budget-availability-during-maintenance)  
11. [Quick Test & Cleanup Commands](#11-quick-test--cleanup-commands)

---

## 1. Sidecar Pattern

**Idea:** run a helper container alongside the main application container in the same Pod. Sidecars are useful for logging, proxying, config reloaders, or synchronization.

**Example:** Nginx main container + small file-synchronizer sidecar (busybox `tail -f` simulating work).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-demo
  labels:
    app: sidecar-demo
spec:
  containers:
  - name: nginx
    image: nginx:1.23
    volumeMounts:
    - name: shared
      mountPath: /usr/share/nginx/html
  - name: file-writer-sidecar
    image: busybox
    command: ["sh", "-c", "echo 'Hello from sidecar' > /shared/index.html && sleep 3600"]
    volumeMounts:
    - name: shared
      mountPath: /shared
  volumes:
  - name: shared
    emptyDir: {}
```

**Test:**
```bash
kubectl apply -f sidecar-pod.yaml
kubectl exec -it sidecar-demo -- cat /usr/share/nginx/html/index.html
```

**Why use it:** Sidecars add features without changing the main app image.

---

## 2. Ambassador / Proxy Pattern

**Idea:** a proxy container (ambassador) forwards traffic between the app container and external services. Useful when the app cannot be changed to talk to new endpoints directly.

**Example:** An ambassador proxy using `socat` to forward traffic (demo).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ambassador-demo
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "echo 'app listening' && sleep 3600"]
  - name: ambassador
    image: alpine/socat
    args: ["TCP-LISTEN:8080,fork", "TCP:example.com:80"]
```

**Note:** `alpine/socat` may be large; this is a demo. In real setups use stable proxy images (Envoy, Linkerd sidecar, or tiny socat builds).

**Test:** Port-forward and curl the ambassador port.

---

## 3. Adapter Pattern (Log/Metric Adapter)

**Idea:** convert one format to another (e.g., container logs -> central collector). Adapter runs alongside app and tails logs, transforms them, and forwards to a backend.

**Example:** An adapter tailing a file and printing to stdout (simulated):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: adapter-demo
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "while true; do echo $(date) hello >> /var/log/app.log; sleep 2; done"]
    volumeMounts:
    - name: logs
      mountPath: /var/log
  - name: adapter
    image: busybox
    command: ["sh", "-c", "tail -F /var/log/app.log | sed 's/hello/ADAPTER: &/'"]
    volumeMounts:
    - name: logs
      mountPath: /var/log
  volumes:
  - name: logs
    emptyDir: {}
```

**Why use it:** Decouples app from logging/metrics pipeline.

---

## 4. Init Container Pattern

**Idea:** run one-time setup tasks before application containers start (migrations, fetching secrets, creating directories).

**Example:** Init container that populates a config file before main container starts.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  initContainers:
  - name: init-config
    image: busybox
    command: ["sh", "-c", "echo 'configured=true' > /config/app.conf"]
    volumeMounts:
    - name: config
      mountPath: /config
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "cat /config/app.conf && sleep 3600"]
    volumeMounts:
    - name: config
      mountPath: /config
  volumes:
  - name: config
    emptyDir: {}
```

**Test:** Apply and check logs; the app prints the file created by init container.

---

## 5. Probe Pattern (Liveness / Readiness / Startup)

**Idea:** Use health probes to tell Kubernetes when an app is ready or needs restarting.

**Example:** Container with HTTP readiness/liveness using a simple shell HTTP server (using `python3 -m http.server` for demo).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probes-demo
spec:
  containers:
  - name: web
    image: python:3.11-alpine
    command: ["sh", "-c", "mkdir -p /tmp/www && echo OK > /tmp/www/index.html && python3 -m http.server 8080 --directory /tmp/www"]
    ports:
    - containerPort: 8080
    readinessProbe:
      httpGet:
        path: /
        port: 8080
      initialDelaySeconds: 1
      periodSeconds: 3
    livenessProbe:
      httpGet:
        path: /
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
```

**Test:** `kubectl describe pod probes-demo` to see probe results.

---

## 6. Ephemeral Containers (Debugging)

**Idea:** Attach a temporary container to a running Pod for debugging without changing Pod spec.

**How to use:** (cluster must allow ephemeral containers)
```bash
kubectl apply -f sidecar-pod.yaml   # use an existing pod
kubectl debug -it <pod-name> --image=busybox --target=nginx
# or create with kubectl debug for new pod
```

**Note:** Ephemeral containers are not for production features — only debugging.

---

## 7. Leader Election Pattern

**Idea:** For HA applications that must have a single active leader, use leader election (Coordination API `Lease` objects or built-in libraries like client-go leader election).

**Simple demo using `kubectl` and a tiny leader script** (conceptual, not production-ready). Use `kubectl exec` to illustrate only.

A typical production approach: applications use Kubernetes API to create/update `Lease` objects; client-go handles lock/renewal.

**Reference pattern (pseudo steps):**
- Each replica attempts to acquire a lease.
- The one that acquires becomes leader and performs leader-only work.
- On lease expiration, others attempt to take over.

---

## 8. Sidecar + Shared emptyDir (for caching or data sharing)

**Idea:** use a sidecar with a shared `emptyDir` volume for cache warming or file syncing.

Example is similar to the sidecar example in #1 — the sidecar writes files into `emptyDir` and the main container serves them.

---

## 9. Anti-affinity Pattern (Pod Placement)

**Idea:** Spread replicas across nodes to improve availability.

**Example:** A deployment that uses `podAntiAffinity` to avoid co-locating pods on the same node.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: anti-affinity-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: anti-demo
  template:
    metadata:
      labels:
        app: anti-demo
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values: ["anti-demo"]
              topologyKey: "kubernetes.io/hostname"
      containers:
      - name: nginx
        image: nginx:1.23
```

**Test:** `kubectl get pods -o wide` to see distribution across nodes.

---

## 10. Pod Disruption Budget (Availability during maintenance)

**Idea:** Prevent too many replicas from being evicted at once during voluntary disruptions (e.g., node drain).

**Example:**
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: anti-demo-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: anti-demo
```

**Uses:** Ensures kubectl drain or node upgrades do not evict more than allowed.

---

## 11. Quick Test & Cleanup Commands

Apply an example:
```bash
kubectl apply -f sidecar-pod.yaml
kubectl apply -f init-demo.yaml
kubectl apply -f probes-demo.yaml
kubectl apply -f anti-affinity.yaml
```

Inspect:
```bash
kubectl get pods --show-labels
kubectl describe pod sidecar-demo
kubectl logs -c adapter adapter-demo
```

Delete:
```bash
kubectl delete pod sidecar-demo init-demo probes-demo || true
kubectl delete deployment anti-affinity-demo || true
kubectl delete pdb anti-demo-pdb || true
```

---

## Final notes

- These patterns are building blocks — combine them (e.g., Sidecar + Probes + Leader Election) for production-ready designs.
- For demos we used lightweight images (busybox, python) — replace with production images in real clusters.
- Always test on staging before applying patterns in production.

