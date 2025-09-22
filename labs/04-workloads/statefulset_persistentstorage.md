# Persistent Storage in Kubernetes StatefulSets

StatefulSets are designed for workloads that need **stable network IDs and stable storage**. Unlike Deployments, which create identical Pods, StatefulSets ensure that each Pod gets its **own identity and its own volume**.

---

## 1. The Role of Storage in StatefulSets

- **Deployment**: Pods are stateless, and all replicas typically share the same PVC (if you configure one).
- **StatefulSet**: Each Pod gets its **own PVC**, created automatically from a `volumeClaimTemplates`.

Why?  
Because StatefulSets are usually used for **stateful apps** like:
- Databases (MySQL, PostgreSQL)
- Distributed systems (Kafka, Cassandra, MongoDB)

These systems **cannot share the same disk** safely because:
- Each Pod needs its **own data files**.
- Concurrent writes from multiple Pods to the same volume can corrupt data.
- Each Pod must keep its data even if it is rescheduled.

---

## 2. How PVCs Are Created

When you define a `volumeClaimTemplates` in a StatefulSet, Kubernetes creates **one PVC per replica**.

Example template:
```yaml
volumeClaimTemplates:
- metadata:
    name: www
  spec:
    accessModes: ["ReadWriteOnce"]
    resources:
      requests:
        storage: 1Gi
```

If you set `replicas: 3`, Kubernetes creates:
```
www-web-stateful-0
www-web-stateful-1
www-web-stateful-2
```

Each Pod (`web-stateful-0`, `web-stateful-1`, `web-stateful-2`) mounts its **own PVC**.

---

## 3. Why They Are Not Shared

- **Isolation of Data**  
  Each Pod writes to its own storage, ensuring no data collisions.

- **Pod Identity**  
  Pods have stable names (`pod-0`, `pod-1`, etc.). Their PVCs match these identities.  
  - `web-stateful-0` → `www-web-stateful-0`  
  - `web-stateful-1` → `www-web-stateful-1`

- **Data Persistence**  
  If a Pod is deleted or rescheduled, its PVC is re-attached, preserving its data.

- **Safety**  
  Many storage systems allow only `ReadWriteOnce (RWO)` access mode, which means only one Pod at a time can mount the volume. This guarantees safe, isolated writes.

---

## 4. When Would You Share Volumes?

Normally you don’t, but in some cases you might:
- Use a **shared filesystem** (NFS, CephFS, EFS, etc.) with `ReadWriteMany (RWX)`.
- This allows multiple Pods to read/write the same volume.
- Use cases: shared static content, logs, or caches (not databases).

But this is **not typical for StatefulSets**, which are about keeping data per Pod.

---

## 5. Visualization

With 3 replicas:

```
Pod: web-stateful-0 ──> PVC: www-web-stateful-0 ──> PV (storage)
Pod: web-stateful-1 ──> PVC: www-web-stateful-1 ──> PV (storage)
Pod: web-stateful-2 ──> PVC: www-web-stateful-2 ──> PV (storage)
```

Each Pod has **its own PVC → PV → backend disk**.

---

## 6. Key Takeaways

- StatefulSets use **`volumeClaimTemplates`** to create one PVC per Pod.
- PVCs are **not shared** between replicas; each has its own.
- This is required for databases and stateful apps to avoid data corruption.
- To share data across Pods, you need a **shared filesystem + RWX volumes**, but this is not the common StatefulSet pattern.

---
