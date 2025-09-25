# What happens if the **container runtime** (e.g., containerd/CRI-O) goes down

The runtime creates and manages **pod sandboxes** and containers via CRI. kubelet talks to it over a Unix socket.

---

## Effects
- **New Pods** cannot start; pod sandboxes fail to create.
- **Liveness/readiness** restarts and image pulls fail.
- **Existing containers** may keep running (they’re normal processes), but kubelet can’t manage them until runtime returns.
- After runtime restarts, it will usually **re-discover** running containers.

## Detect
```bash
# Runtime status
sudo systemctl status containerd
sudo ctr --version || crictl info

# Kubelet logs will show CRI errors
journalctl -u kubelet -f | grep -i 'cri'
```

---

## Simulate / Restore
```bash
sudo systemctl stop containerd   # or crio
# ...
sudo systemctl start containerd
```

---

## Notes
- Socket path must match kubelet’s `--container-runtime-endpoint` (e.g., `unix:///run/containerd/containerd.sock`).
- If runtime is up but **CNI** is broken, pods still fail at the networking step.
