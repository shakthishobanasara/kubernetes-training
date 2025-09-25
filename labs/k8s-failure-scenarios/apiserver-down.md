# What happens if **kube-apiserver** goes down

kube-apiserver is the **front door** to the control plane. No clients or components can read/write cluster state without it.

---

## Effects
- `kubectl` and controllers **cannot talk** to the control plane → API calls fail.
- **New Pods** cannot be created; **scheduling stops** (scheduler can’t list/watch unscheduled pods).
- **Controllers** (Deployments, StatefulSets, HPA, PV binder, etc.) **stall** (they all use the API).
- **Existing Pods** keep running on nodes; kubelet continues local restarts.

## What still works
- Node workloads continue; Services route with existing iptables/nft rules.
- kubelet manages container restarts locally (no new desired state pulled).

---

## Detect
```bash
kubectl version --short       # will hang/fail
kubectl get --raw=/healthz    # no response
# From control-plane host:
sudo crictl ps | grep kube-apiserver
journalctl -u kubelet -f     # watch static-pod restarts
```

---

## Simulate (kubeadm)
```bash
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /etc/kubernetes/manifests/kas.yaml.bak
kubectl get componentstatuses  # deprecated, but many labs still use
```
Restore:
```bash
sudo mv /etc/kubernetes/manifests/kas.yaml.bak /etc/kubernetes/manifests/kube-apiserver.yaml
```

---

## Recovery
- Fix crashing config (flags, certs, etc.).
- Check **etcd** availability (apiserver depends on it).
- Verify **serving certs** & **client certs** have not expired.
