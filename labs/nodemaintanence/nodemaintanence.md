
# Kubernetes Node Maintenance + PDB Hands-On Lab

This lab walks you through **cordon → drain (evict) → maintenance → uncordon** on worker nodes, and how **PodDisruptionBudgets (PDBs)** protect availability during planned disruptions.

---

## 0) Prereqs
- Cluster with **1 control plane + ≥2 workers**  
- `kubectl` configured  

Deploy a sample app:
```bash
kubectl create ns node-maint
kubectl -n node-maint create deploy web \
  --image=ghcr.io/nginxinc/nginx-unprivileged:stable \
  --replicas=3 --port=8080
kubectl -n node-maint expose deploy web --port=80 --target-port=8080
```

---

## 1) Add a PDB
```bash
cat <<'EOF' | kubectl -n node-maint apply -f -
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: web
EOF
```

Check:
```bash
kubectl -n node-maint get pdb
```

---

## 2) Cordon a Node
```bash
kubectl cordon <node-name>
kubectl get nodes
```
➡️ Status should show `SchedulingDisabled`.

---

## 3) Drain (Evict Pods)
```bash
kubectl drain <node-name> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=60 \
  --timeout=10m
```

- PDB may **block eviction** if it would reduce pods below `minAvailable`.  
- Fix options:  
  - Scale replicas up (e.g. 4).  
  - Temporarily relax PDB:  
    ```bash
    kubectl -n node-maint patch pdb/web-pdb --type='merge' -p '{"spec":{"minAvailable":1}}'
    ```

---

## 4) Perform Maintenance
Do your upgrade, patch, or reboot.  
Verify node has no workload pods:
```bash
kubectl get pods -A -o wide | grep <node-name> || echo "No pods on node"
```

---

## 5) Uncordon
```bash
kubectl uncordon <node-name>
```
➡️ Pods will schedule again on this node.Uncordon = allows new scheduling, but no auto-rebalance.

To spread pods again, you need a rolling update, manual deletion, or the Kubernetes Descheduler.

---

## 6) Cleanup
```bash
kubectl delete ns node-maint
```

---

## Cheat Sheet
```bash
# See taints / cordon
kubectl describe node <node> | grep Taints

# Cordon / Uncordon
kubectl cordon <node>
kubectl uncordon <node>

# Drain with safety
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data

# PDB info
kubectl -n <ns> get pdb
kubectl -n <ns> describe pdb <name>
```
