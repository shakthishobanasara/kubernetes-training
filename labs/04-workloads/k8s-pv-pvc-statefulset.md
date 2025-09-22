# From Zero to StatefulSet: PV → PVC → StatefulSet (Kubernetes-the-Hard-Way Friendly)

This hands-on guide assumes a **bare cluster** (like *Kubernetes the Hard Way*) with **no StorageClasses** and **no PersistentVolumes** yet.  
You’ll learn what **PV** and **PVC** are, and then create a **StatefulSet** that uses them—via **two paths**:

- **Path A (Recommended for labs): Dynamic provisioning** with a tiny local provisioner.
- **Path B (No add-ons): Static PVs** that you create yourself.

> If you just want the shortest path in a single-node/small lab, use **Path A**. If you want to stay 100% “Kubernetes the Hard Way,” use **Path B**.

---

## Table of Contents
- [1. Concepts (PV, PVC, StorageClass)](#1-concepts-pv-pvc-storageclass)
- [2. Cluster Baseline Checks](#2-cluster-baseline-checks)
- [3. Path A — Dynamic Provisioning (Local Path Provisioner)](#3-path-a--dynamic-provisioning-local-path-provisioner)
  - [3.1 Install a dynamic provisioner](#31-install-a-dynamic-provisioner)
  - [3.2 Create a headless Service](#32-create-a-headless-service)
  - [3.3 Create the StatefulSet (dynamic)](#33-create-the-statefulset-dynamic)
  - [3.4 Verify](#34-verify)
- [4. Path B — Static PVs (No Provisioner)](#4-path-b--static-pvs-no-provisioner)
  - [4.1 Prepare host paths on nodes](#41-prepare-host-paths-on-nodes)
  - [4.2 Create one PV per ordinal](#42-create-one-pv-per-ordinal)
  - [4.3 Create a headless Service](#43-create-a-headless-service)
  - [4.4 Create the StatefulSet (static)](#44-create-the-statefulset-static)
  - [4.5 Verify](#45-verify)
- [5. Troubleshooting Cheatsheet](#5-troubleshooting-cheatsheet)
- [6. Cleanup](#6-cleanup)

---

## 1. Concepts (PV, PVC, StorageClass)

- **PersistentVolume (PV)**: A piece of storage in the cluster. Created by an admin (static) or dynamically by a provisioner. Lives at **cluster scope**.
- **PersistentVolumeClaim (PVC)**: A request for storage by a user/workload. Lives **in a namespace**. If the cluster can satisfy the claim (size, mode, class), it **binds** to a PV.
- **StorageClass (SC)**: A template that tells Kubernetes **how to dynamically create PVs** (via a CSI driver/provisioner). If you **don’t** have any SCs, **dynamic provisioning won’t happen**; you must create PVs yourself.

**Flow:** Pod → PVC → binds to PV (created statically or dynamically via SC).

---

## 2. Cluster Baseline Checks

```bash
# Nodes
kubectl get nodes -o wide

# Any StorageClasses? (likely none in KTHW)
kubectl get sc

# Any PVs/PVCs already?
kubectl get pv
kubectl get pvc -A
```

If `kubectl get sc` returns nothing and `kubectl get pv` returns nothing, you’re truly starting from zero.

---

## 3. Path A — Dynamic Provisioning (Local Path Provisioner)

> Easiest for labs/single-node/small clusters. Creates hostPath-backed PVs automatically.

### 3.1 Install a dynamic provisioner

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.25/deploy/local-path-storage.yaml

# Make it the default StorageClass so PVCs without an explicit SC will bind
kubectl patch storageclass local-path \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

Verify:
```bash
kubectl get sc
```

You should see `local-path` with the default annotation.

### 3.2 Create a headless Service

StatefulSets need a governing service for stable network IDs. Your spec uses `serviceName: "web"`.

```yaml
# web-headless-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: web
  labels:
    app: web
spec:
  clusterIP: None    # headless
  selector:
    app: web
  ports:
    - name: http
      port: 80
```

```bash
kubectl apply -f web-headless-svc.yaml
```

### 3.3 Create the StatefulSet (dynamic)

> `storageClassName` is **optional** if you marked `local-path` as default. Including it makes intent explicit.

```yaml
# web-statefulset-dynamic.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web-stateful
spec:
  serviceName: "web"
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.23
        ports:
        - containerPort: 80
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
      storageClassName: local-path   # omit if you prefer to rely on default
```

Apply:
```bash
kubectl apply -f web-statefulset-dynamic.yaml
```

### 3.4 Verify

```bash
# Watch PVCs bind and Pods start
kubectl get pvc,pv,pods -w

# When Running, test service DNS names (each Pod gets a stable A record)
kubectl run curl --image=radial/busyboxplus:curl -it --rm --restart=Never -- \
  sh -lc 'for i in 0 1 2; do echo === web-stateful-$i ===; curl -s web-stateful-$i.web.default.svc.cluster.local | head; done'
```

---

## 4. Path B — Static PVs (No Provisioner)

> Pure “Kubernetes the Hard Way” style: you create PVs yourself. Each PVC (from the StatefulSet) will bind to a matching PV.

### 4.1 Prepare host paths on nodes

Pick a node (or nodes) that will host the data. Create directories for each ordinal:
```bash
# On the node(s) (via SSH), e.g. /data/pv/...
sudo mkdir -p /data/pv/www-web-stateful-0
sudo mkdir -p /data/pv/www-web-stateful-1
sudo mkdir -p /data/pv/www-web-stateful-2
sudo chmod -R 777 /data/pv       # demo only; tighten perms in real setups
```

Get your node hostname label values:
```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'
```

### 4.2 Create one PV per ordinal

> Keep `storageClassName` **empty** (unset) on both PV **and** PVC for matching without a class. Pin each PV to a node using `nodeAffinity` so the Pod schedules where the data lives.

```yaml
# pv-www-web-stateful.yaml (creates 3 PVs)
apiVersion: v1
kind: List
items:
- apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: pv-www-web-stateful-0
  spec:
    capacity:
      storage: 1Gi
    accessModes: ["ReadWriteOnce"]
    persistentVolumeReclaimPolicy: Retain
    # storageClassName intentionally omitted
    nodeAffinity:
      required:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values: ["<node-name>"]   # replace with actual
    hostPath:
      path: /data/pv/www-web-stateful-0
- apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: pv-www-web-stateful-1
  spec:
    capacity:
      storage: 1Gi
    accessModes: ["ReadWriteOnce"]
    persistentVolumeReclaimPolicy: Retain
    nodeAffinity:
      required:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values: ["<node-name>"]
    hostPath:
      path: /data/pv/www-web-stateful-1
- apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: pv-www-web-stateful-2
  spec:
    capacity:
      storage: 1Gi
    accessModes: ["ReadWriteOnce"]
    persistentVolumeReclaimPolicy: Retain
    nodeAffinity:
      required:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values: ["<node-name>"]
    hostPath:
      path: /data/pv/www-web-stateful-2
```

Apply:
```bash
kubectl apply -f pv-www-web-stateful.yaml
kubectl get pv
```

You should see 3 PVs in `Available` state.

### 4.3 Create a headless Service

Same as Path A (repeat here for completeness).

```yaml
# web-headless-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: web
  labels:
    app: web
spec:
  clusterIP: None
  selector:
    app: web
  ports:
    - name: http
      port: 80
```

```bash
kubectl apply -f web-headless-svc.yaml
```

### 4.4 Create the StatefulSet (static)

> **Do not set** `storageClassName` in the claim template (leave it empty) so it can bind to your storageClass-less PVs.

```yaml
# web-statefulset-static.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web-stateful
spec:
  serviceName: "web"
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.23
        ports:
        - containerPort: 80
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
      # storageClassName: ""   # leave unset entirely
```

Apply:
```bash
kubectl apply -f web-statefulset-static.yaml
```

### 4.5 Verify

```bash
kubectl get pvc,pv,pods -w
```

Expected:
- PVCs `www-web-stateful-{0,1,2}` become **Bound** to your PVs.
- Pods `web-stateful-{0,1,2}` become **Running** on the node(s) specified by PV `nodeAffinity`.

Test DNS & serving:
```bash
kubectl run curl --image=radial/busyboxplus:curl -it --rm --restart=Never -- \
  sh -lc 'for i in 0 1 2; do echo === web-stateful-$i ===; wget -qO- web-stateful-$i.web.default.svc.cluster.local | head; done'
```

> If you need content, drop a quick file into each volume:
```bash
# Write a small index.html into ordinal 0's volume
kubectl exec -it web-stateful-0 -- sh -lc 'echo "Hello from pod 0" > /usr/share/nginx/html/index.html && cat /usr/share/nginx/html/index.html'
```

---

## 5. Troubleshooting Cheatsheet

- **PVC Pending / “no persistent volumes available for this claim”**  
  Create a PV that matches size & `accessModes`, or install a StorageClass.
- **PVC Pending / “waiting for a volume to be created by external provisioner”**  
  You referenced a StorageClass but no provisioner is installed → install CSI or switch to static PVs.
- **Node Affinity conflict**  
  PV bound to a node where the Pod can’t schedule (taints/labels/zone). Fix node availability or PV `nodeAffinity`.
- **StatefulSet won’t start (no headless Service)**  
  Ensure `serviceName` exists and is **headless** (`clusterIP: None`).
- **Changed storageClassName after creating PVC**  
  You can’t mutate `storageClassName` on an existing PVC. Delete & recreate the claim (data loss) or create a PV that matches the existing claim.
- **Multi-node RW access needed**  
  HostPath PVs are **RWO** per node. For RWX across nodes, use a network filesystem CSI (NFS, CephFS, etc.).

---

## 6. Cleanup

```bash
# StatefulSet & Pods
kubectl delete sts web-stateful

# PVCs (per-ordinal claims)
kubectl delete pvc -l app=web  # if labels set; otherwise delete by name

# PVs (if you created static ones)
kubectl delete -f pv-www-web-stateful.yaml

# Headless service
kubectl delete -f web-headless-svc.yaml

# If you installed local-path provisioner (Path A) and want to remove it:
kubectl delete -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.25/deploy/local-path-storage.yaml
```

---

### Quick Decision Flow

- Want the **easiest** path in labs? → **Path A (dynamic)**, set `local-path` as default, apply the StatefulSet.
- Want **pure KTHW** with zero add-ons? → **Path B (static PVs)**, create PVs first, then the StatefulSet.
