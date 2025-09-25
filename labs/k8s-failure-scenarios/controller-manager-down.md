# What happens if **kube-controller-manager** goes down

kube-controller-manager runs core **reconciliation loops** (Deployment/RS/StatefulSet, Job/CronJob, Endpoints/EndpointSlices, Node lifecycle, PV binder, GC, etc.).

---

## Effects
- **No reconciliation**:
  - Deleting a Pod from a Deployment won’t be replaced.
  - **CronJobs** won’t create Jobs.
  - **PVC** binding/provisioning/attach-detach halts.
  - **Endpoints/EndpointSlices** stop updating (Services get stale backends).
  - **Node NotReady** → evictions **don’t** occur.
  - **Garbage Collection & finalizers** stall; namespaces can hang `Terminating`.
- **HPA/VPA** backed by controllers stop acting (HPA also depends on metrics-server).

## What still works
- Scheduler keeps scheduling **already-created, unscheduled Pods** (if any).
- Existing Pods keep running.

---

## Detect
```bash
kubectl -n kube-system get pods -l component=kube-controller-manager -o wide
kubectl get deploy -A | grep -E 'DESIRED|AVAILABLE'
kubectl get pvc -A            # look for Pending
kubectl get endpointslices -A # look for staleness
```

---

## Simulate / Restore (kubeadm)
```bash
sudo mv /etc/kubernetes/manifests/kube-controller-manager.yaml /etc/kubernetes/manifests/kcm.yaml.bak
# ...observe stalling behavior...
sudo mv /etc/kubernetes/manifests/kcm.yaml.bak /etc/kubernetes/manifests/kube-controller-manager.yaml
```

---

## Takeaway
Without controller-manager, desired ↔ actual state **stops converging**.
