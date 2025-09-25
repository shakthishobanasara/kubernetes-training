# What happens if **kube-proxy** is down (on a node)

kube-proxy programs **iptables/nftables** or **IPVS** rules for Service VIPs and NodePorts.

---

## Effects
- Existing routing rules continue to work (they live in the kernel).
- **Changes** to Services/Endpoints **won’t** be reflected on that node:
  - New Services might not be reachable.
  - Removed/added Pod backends won’t update → stale or blackholed traffic.
- ClusterIP/NodePort behavior differs by mode (iptables vs IPVS) but the theme is **stale dataplane**.

## Detect
```bash
kubectl -n kube-system get ds kube-proxy -o wide
# On the node:
sudo iptables -L -t nat | head
sudo ipvsadm -Ln         # if using IPVS
journalctl -u kube-proxy -f
```

---

## Simulate / Restore
```bash
# If run as a DaemonSet, delete or scale to 0 (lab only):
kubectl -n kube-system scale ds kube-proxy --replicas=0
# Or on a single node (managed by kubelet as static or ds):
sudo systemctl stop kube-proxy || true
```
Restore by scaling back or starting the service.

---

## Notes
- For **troubleshooting**, compare EndpointSlices vs the node’s dataplane:
  ```bash
  kubectl get endpointslices -n <ns> -l kubernetes.io/service-name=<svc> -o yaml
  ```
  If slices list backends that **don’t exist** in iptables/IPVS, kube-proxy isn’t reconciling.
