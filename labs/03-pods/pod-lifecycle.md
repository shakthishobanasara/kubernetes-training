
# 🌀 Kubernetes Pod Lifecycle

Understanding the **Pod Lifecycle** is essential for managing workloads effectively in Kubernetes. A Pod’s lifecycle reflects its **current state** from creation to deletion.

---

## 📚 Lifecycle Overview

A Pod typically goes through these **phases**:

### 1. **Pending**
- The Pod is accepted by the Kubernetes system but not yet running.
- Might be waiting for the scheduler to assign a node or waiting for image pulls.

### 2. **Running**
- The Pod has been bound to a node and all containers have been created.
- At least one container is still running or starting.

### 3. **Succeeded**
- All containers in the Pod have terminated **successfully** (exit code 0).
- Happens with Pods meant to run short-lived jobs (e.g., `Job` controller).

### 4. **Failed**
- All containers in the Pod have terminated, **but at least one** has failed (non-zero exit code).

### 5. **Unknown**
- The state of the Pod could not be determined (e.g., due to node communication issues).

---

## 🔄 Container States Inside a Pod

Each container in a Pod goes through its own **lifecycle states**:

### 🔸 `Waiting`
- The container is waiting to start (e.g., waiting for image to download).

### 🔸 `Running`
- The container is executing without issues.

### 🔸 `Terminated`
- The container has stopped, either successfully or due to an error.

Use:
```bash
kubectl describe pod <pod-name>
```
to view these states.

---

## 💥 Pod Lifecycle Events

| Event              | Description |
|--------------------|-------------|
| `Scheduled`        | Pod is assigned to a node |
| `Pulling`          | Image is being pulled |
| `Created`          | Pod object is created |
| `Started`          | Containers have started |
| `Killing`          | Pod is being deleted or evicted |

---

## 📑 Example: Check Pod Status

```bash
kubectl get pods
kubectl describe pod <pod-name>
```

Look at `Status`, `Events`, and `State` fields.

---

## 🔄 Restart Policies

- **Always** (default): Restart container on failure (used in Deployments)
- **OnFailure**: Restart only on failure (used in Jobs)
- **Never**: Don't restart

Defined in PodSpec under `restartPolicy`.

---

## 📌 Important Concepts

- Pods are **ephemeral** – use Controllers like Deployments, StatefulSets, or Jobs to manage them.
- Logs from terminated Pods may be lost unless persisted.
- Use `livenessProbe` and `readinessProbe` to control Pod health.

---

✅ *Mastering Pod lifecycle helps in debugging, tuning restart behavior, and writing production-ready manifests.*
