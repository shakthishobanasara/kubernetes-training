# What happens if **etcd** goes down (or loses quorum)

etcd is the **source of truth** for the Kubernetes control plane. The API server reads/writes cluster state to etcd. In HA, etcd requires **quorum** (`> N/2` members) to serve **writes**.

---

## Effects

### If **single etcd** (or etcd cluster **loses quorum**)
- **Writes fail**: `kubectl apply`, controller updates, status updates → **errors/timeouts**.
- **Reads**:
  - May be served from the API server’s **watch cache** for a while (stale), but eventually fail if uncached.
- **Controllers** (deployments, statefulsets, PV binding, HPA, etc.) **stall** because API writes can’t be persisted.
- **New Pods**: can be created in the API (if cache allows) but **won’t reconcile reliably**; scheduling and status updates will fail.
- **Existing Pods**: **continue running** on nodes (kubelet/container runtime keep them alive).

### If etcd is **completely down**
- API server **cannot persist or fetch** state; most operations return 5xx/timeout.
- Cluster appears “frozen” from the control plane perspective.

---

## What still works
- Running Pods keep running.
- kubelet continues restarting containers on the **same node** if they crash.
- Node-local processes keep serving traffic; Services keep routing **with existing endpoints** (no updates).

---

## How to detect (symptoms)
```bash
kubectl get --raw='/healthz/etcd'        # often 500 when etcd is down
kubectl -v=9 get pods                     # apiserver errors to etcd visible in logs
# API server logs:
kubectl -n kube-system logs deploy/kube-apiserver -c kube-apiserver | grep -i etcd
```

etcd logs show elections/failures; check members/quorum:
```bash
ETCDCTL_API=3 etcdctl --endpoints=<list> endpoint status -w table
ETCDCTL_API=3 etcdctl member list -w table
```

---

## How to simulate (kubeadm, single control plane)
> **Lab only**. Static Pod lives in `/etc/kubernetes/manifests/etcd.yaml`.
```bash
sudo mv /etc/kubernetes/manifests/etcd.yaml /etc/kubernetes/manifests/etcd.yaml.bak
watch kubectl -n kube-system get pods -o wide | grep etcd
```
Restore:
```bash
sudo mv /etc/kubernetes/manifests/etcd.yaml.bak /etc/kubernetes/manifests/etcd.yaml
```

---

## Recovery checklist
- For HA: restore **quorum** (bring up members / replace failed ones).
- Disk full / I/O latency / defrag issues → check `etcdctl defrag` windowing.
- TLS/cert expiry between API server and etcd.
- Last resort: restore from a **snapshot**:
  ```bash
  ETCDCTL_API=3 etcdctl snapshot restore backup.db --data-dir=/var/lib/etcd-new
  ```

---

## Takeaways
- etcd down = **no reliable control-plane state changes**.
- Design for **HA + regular snapshots + defrag**; isolate etcd I/O from other noisy neighbors.
