# What happens if **kube-scheduler** goes down

kube-scheduler binds **unscheduled** Pods to Nodes.

---

## Effects
- Any newly created Pods (without `nodeName`) remain **Pending**.
- Rollouts **stall** at the Pod-creation step (Deployments/StatefulSets create Pods that never bind).
- **DaemonSets** still work because the controller sets `nodeName` directly (no scheduler needed).

## What still works
- Existing Pods continue running.
- Controllers can *create* Pods (they just won’t bind).

---

## Detect
```bash
kubectl -n kube-system get pods -l component=kube-scheduler -o wide
kubectl get pods -A --field-selector=status.phase=Pending
kubectl describe pod <pending-pod> | grep -i 'unschedul'
```

---

## Simulate / Restore (kubeadm)
```bash
sudo mv /etc/kubernetes/manifests/kube-scheduler.yaml /etc/kubernetes/manifests/ks.yaml.bak
# Observe Pending pods increase
sudo mv /etc/kubernetes/manifests/ks.yaml.bak /etc/kubernetes/manifests/kube-scheduler.yaml
```

---

## Tip
If you **manually set** `spec.nodeName` on a Pod, kubelet will run it even with the scheduler down (hand-binding). Use with caution.
