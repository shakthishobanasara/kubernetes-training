# Installing crictl on Ubuntu (for Kubernetes Nodes)



---

## 1) Why `crictl`?
- `crictl` = **Container Runtime Interface CLI**.  
- Lets you talk **directly to the container runtime** (containerd, CRI-O).  
- Useful for troubleshooting pods when `kubectl` is not enough (e.g., stuck in `ContainerCreating`).  
- Must be installed on **each node** (control plane + worker) where kubelet is running.

---

## 2) Check your runtime
```bash
ps aux | grep containerd
```
If you see `containerd` running → proceed.

---

## 3) Download and install crictl
```bash
# Choose version (match your Kubernetes minor version if possible)
VERSION="v1.31.1"   # works with Kubernetes 1.31+

# Download release
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz

# Extract binary to /usr/local/bin
sudo tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin

# Verify installation
crictl --version
```

---

## 4) Configure runtime endpoint
Create config file `/etc/crictl.yaml`:

```bash
sudo tee /etc/crictl.yaml <<EOF
runtime-endpoint: unix:///var/run/containerd/containerd.sock
image-endpoint: unix:///var/run/containerd/containerd.sock
timeout: 10
debug: false
EOF
```

> ⚠️ For CRI-O, replace the endpoint with:
> `unix:///var/run/crio/crio.sock`

---

## 5) Test crictl
```bash
# Show runtime info
crictl info

# List containers (similar to docker ps)
crictl ps -a

# List images
crictl images

# Logs of a container
crictl logs <container-id>
```

---

## 6) Cleanup
```bash
rm crictl-$VERSION-linux-amd64.tar.gz
```

---

## 7) Notes
- Install `crictl` on **every node** → because each kubelet only talks to its local runtime.  
- For multi-node clusters, automate installation (Ansible, parallel ssh, cloud-init).  
- Works seamlessly with both **containerd** and **CRI-O**.

---

✅ Done! You can now use `crictl` for low-level debugging on Kubernetes nodes.
