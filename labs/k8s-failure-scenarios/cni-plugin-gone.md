# What happens if the **CNI plugin** is gone/broken

The CNI plugin config/binaries are used by the runtime to set up **Pod networking**.

---

## Effects
- New Pods get stuck in **ContainerCreating** or **Pending**:
  - Events show `failed to find plugin` or `CNI config not found`.
- Existing Pods typically **keep their networking** (until restarted).
- **Cross-node** connectivity or NetworkPolicy enforcement can break depending on the plugin failure mode.

## Detect
```bash
# On a node
ls -l /etc/cni/net.d/               # CNI configs
ls -l /opt/cni/bin/                 # CNI binaries
journalctl -u kubelet -f | grep -i cni
kubectl describe pod <pod> | grep -i 'cni\|network'
```

---

## Simulate / Restore
```bash
# Simulate by moving config away (lab only)
sudo mkdir -p /tmp/cni-backup && sudo mv /etc/cni/net.d/* /tmp/cni-backup/
# Restore
sudo mv /tmp/cni-backup/* /etc/cni/net.d/
```

---

## Notes
- Many CNIs ship as DaemonSets; ensure those pods are healthy (e.g., `calico-node`, `weave-net`, `cilium`).
- NetworkPolicy enforcement depends on the plugin; if the agent is down, policies might be ignored or default-deny might bite.
