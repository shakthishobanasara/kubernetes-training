# Static Pods vs DaemonSet — Quick Comparison & Tomorrow’s Demo Guide


---

## 1) Static Pod vs DaemonSet — Side‑by‑Side

| Aspect | **Static Pod** | **DaemonSet** |
|---|---|---|
| **Who manages it?** | Kubelet on each node watches a local path (`staticPodPath`) and runs pods from files | The DaemonSet controller in the API server schedules one Pod per node (or per selected nodes) |
| **Where is the source of truth?** | A file on the node’s filesystem (e.g., `/etc/kubernetes/manifests/*.yaml`) | Kubernetes API object stored in etcd |
| **Requires API server?** | No (runs even if API server is down) | Yes (controller/scheduler need API server) |
| **Visibility in `kubectl`** | Shown as a **mirror pod** with `-<nodeName>` suffix in `kube-system` | Regular pods owned by a DaemonSet |
| **How to update?** | Edit/replace the node-local file (SSH / config mgmt) | `kubectl apply -f ds.yaml` (declarative), rollout strategies available |
| **Placement** | Only on the node where the file exists | On all (or selected) nodes matching nodeSelector/tolerations |
| **Use cases** | **Bootstrapping** control plane (apiserver/scheduler/controller-manager/etcd), node‑critical agents that must survive API outages | Cluster-wide agents (log collectors, CNI, monitoring), node services managed declaratively |
| **Delete behavior** | `kubectl delete pod` only removes the **mirror**; kubelet recreates it if file remains | `kubectl delete pod` is transient; controller recreates; deleting the **DaemonSet** removes all pods |
| **Rollout controls** | Minimal (kubelet restarts pod when file changes) | Rich (max surge/unavailable via Deployment, or on/offline nodes via DS update strategy) |
| **Recommended for apps?** | No | Yes |

---

## 2) Demo Plan — “From Static Pod to DaemonSet”

> Works on KTHW, kubeadm, or any cluster with containerd. Replace paths if your distro differs.

### Prereqs (2–3 min)
- Confirm kubelet config path (often `/var/lib/kubelet/config.yaml` or `/etc/kubernetes/kubelet.conf` **for credentials** and `/var/lib/kubelet/config.yaml` **for configuration**).  
- Ensure `containerd` & `crictl` are present:
  ```bash
  crictl info | head -n 5
  kubectl get nodes -o wide
  ```

### Step 1 — Ensure `staticPodPath` is set (2–3 min)
- Open kubelet config and check/add:
  ```yaml
  # /var/lib/kubelet/config.yaml  (path may vary)
  kind: KubeletConfiguration
  apiVersion: kubelet.config.k8s.io/v1beta1
  staticPodPath: /etc/kubernetes/manifests
  ```
- Create path if missing & restart kubelet:
  ```bash
  sudo mkdir -p /etc/kubernetes/manifests
  sudo systemctl restart kubelet
  journalctl -u kubelet -f --no-pager &
  ```

> **Why**: This is the directory kubelet watches to auto‑create pods without the API server.

### Step 2 — Create a Static Pod (nginx) (3–4 min)
```yaml
# /etc/kubernetes/manifests/nginx-static.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-static
  namespace: kube-system
  labels:
    app: nginx-static
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
    readinessProbe:
      httpGet: { path: /, port: 80 }
      initialDelaySeconds: 2
      periodSeconds: 5
```
- Verify on the node and via API (mirror):
  ```bash
  # On the node runtime
  sudo crictl ps | grep nginx

  # Mirror pod (note the suffix -$(nodeName))
  kubectl get pods -n kube-system -o wide | grep nginx-static
  kubectl get pod -n kube-system -l app=nginx-static -o yaml | grep -E 'kubernetes.io/config|mirror'
  ```
**Key talking point**: Mirror pod annotations like `kubernetes.io/config.source=file` show it originated from the filesystem.

### Step 3 — Show “delete vs recreate” (2–3 min)
```bash
kubectl delete pod -n kube-system $(kubectl get pod -n kube-system -l app=nginx-static -o name)
# Observe: kubelet recreates the mirror because the file still exists
watch -n1 'kubectl get pods -n kube-system | grep nginx-static'
```
**Why**: Deleting the mirror doesn’t remove the source (file). Kubelet is the controller here.

### Step 4 — Live update by editing the file (2–3 min)
- Change the image tag to `nginx:1.27` and save. Kubelet will restart the pod automatically.
  ```bash
  sudo sed -i 's/nginx:1.25/nginx:1.27/' /etc/kubernetes/manifests/nginx-static.yaml
  watch -n1 'kubectl get pods -n kube-system -o wide | grep nginx-static'
  ```
**Why**: Kubelet detects file changes and restarts the container to match desired state.

### Step 5 — Remove the static pod (1–2 min)
```bash
sudo rm -f /etc/kubernetes/manifests/nginx-static.yaml
# kubelet terminates the pod; mirror disappears
watch -n1 'kubectl get pods -n kube-system | grep nginx-static || true'
```

### Step 6 — Recreate as a **DaemonSet** (5–6 min)
> Now show the declarative, API‑driven way to run one pod per node.

```yaml
# ds-nginx.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-ds
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: nginx-ds
  template:
    metadata:
      labels:
        app: nginx-ds
    spec:
      tolerations:
      - key: "node-role.kubernetes.io/control-plane"
        effect: "NoSchedule"
        operator: "Exists"
      containers:
      - name: nginx
        image: nginx:1.27
        ports:
        - containerPort: 80
```
- Apply and verify:
  ```bash
  kubectl apply -f ds-nginx.yaml
  kubectl get ds -n kube-system nginx-ds
  kubectl get pods -n kube-system -l app=nginx-ds -o wide
  ```
- Update strategy demo:
  ```bash
  kubectl set image ds/nginx-ds -n kube-system nginx=nginx:1.27.1
  kubectl rollout status ds/nginx-ds -n kube-system
  ```

**Why**: DaemonSet gives cluster‑level visibility, label‑driven placement, and rollout controls — preferred for ops agents and node daemons.

---

## 3) Cheat Sheet

1. **Problem**: Need critical components before API is up; need node‑local processes to survive API outages.  
2. **Solution A**: Static pods — kubelet reads files, runs pods, shows mirrors for visibility.  
3. **Trade‑offs**: Harder to manage at scale; not declarative; per‑node edits.  
4. **Solution B**: DaemonSet — API‑native, scalable, controllable rollouts, node selection via labels/tolerations.  
5. **Rule of thumb**: Static pods for **bootstrap/critical control plane**; DaemonSet for **everything else** (agents).

---

## 4) Common Gotchas & Tips

- **Namespace**: Use `kube-system` for static pods. Mirror pods live in `kube-system`.  
- **Name suffix**: Mirrors append `-<nodeName>`.  
- **Logs**: Use `journalctl -u kubelet -f` to watch file detection & pod lifecycle.  
- **Runtime tools**: `crictl` is your friend on nodes with containerd.  
- **Un-deleteable?** If `kubectl delete` keeps “coming back,” remove the file in `/etc/kubernetes/manifests`.  
- **API outage demo** (optional, advanced): If your control plane is static‑pod based, briefly stop the API server container; static pods keep running because kubelet owns them.

---

## 5) Cleanup

```bash
kubectl delete -f ds-nginx.yaml -n kube-system --ignore-not-found
sudo rm -f /etc/kubernetes/manifests/nginx-static.yaml
```

---

## 6) Quick FAQ (for Q&A)

- **Q:** Can I scale a static pod? **A:** Not via API. It’s just a local file; to run on multiple nodes, copy the file to each node.  
- **Q:** Can I use a URL instead of a file path? **A:** Yes (`staticPodURL` in KubeletConfiguration), but file path is most common.  
- **Q:** Will liveness/readiness probes work? **A:** Yes — kubelet enforces them locally.  
- **Q:** Where do events show? **A:** In the mirror pod’s events and kubelet logs.  
- **Q:** Which to choose for node agents? **A:** Prefer **DaemonSet** (centralized management & rollouts).

---
