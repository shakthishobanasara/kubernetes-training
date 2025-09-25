# What happens if **kubelet** goes down (on a node)

kubelet is the node agent that **runs pods**, reports status, and performs liveness/readiness probes.

---

## Effects (on the affected node)
- Existing containers may **keep running** for a while, but:
  - No probes → unhealthy pods won’t be restarted.
  - No new Pods can start; sandboxes can’t be created.
- The node misses **heartbeats** → becomes **NotReady** after lease timeout.
- Controllers (NodeLifecycle) **may evict** Pods and reschedule them elsewhere (if controller-manager is up and policies allow).

## Control-plane view
- Pods on that node show **`Unknown`** or disappear after eviction.
- Services may remove backends on that node when endpoints update.

---

## Detect
```bash
kubectl get nodes
kubectl describe node <nodename> | grep -i 'Ready'
kubectl get pods -o wide | grep <nodename>
```

---

## Simulate / Restore
```bash
sudo systemctl stop kubelet
# ...observe NotReady...
sudo systemctl start kubelet
```

---

## Notes
- If **container runtime** is healthy, some workloads continue until they crash.
- Without kubelet, **no config drift** (DaemonSet changes, pod updates) can apply to that node.
