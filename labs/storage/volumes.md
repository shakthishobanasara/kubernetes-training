# Kubernetes Volumes — Notes & Demos

## 🔹 Why Volumes?
- Containers lose data if they crash/restart (ephemeral storage).  
- Pods with multiple containers need a **shared filesystem**.  
- Volumes solve both: **data persistence** + **shared storage**.

---

## 🔹 Volume Categories
1. **Ephemeral Volumes**  
   - Tied to Pod lifecycle.  
   - Examples: `emptyDir`, `configMap`, `secret`, `downwardAPI`.  

2. **Persistent Volumes (PVs)**  
   - Exist beyond Pod lifecycle.  
   - Examples: `hostPath`, `local`, `nfs`, cloud volumes (EBS, GCE, AzureDisk, etc).  

---

## 🔹 Core Volume Types with Demos

### 1. `emptyDir`
- **Temporary storage directory** on node.  
- Survives container restarts, but deleted when Pod is removed.  

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-demo
spec:
  containers:
  - name: writer
    image: busybox
    command: ["sh", "-c", "echo hello > /data/msg; sleep 3600"]
    volumeMounts:
    - name: temp-data
      mountPath: /data
  - name: reader
    image: busybox
    command: ["sh", "-c", "cat /data/msg; sleep 3600"]
    volumeMounts:
    - name: temp-data
      mountPath: /data
  volumes:
  - name: temp-data
    emptyDir: {}
```

---

### 2. `hostPath`
- Mounts a **host node’s path** into Pod.  
- Useful for node logs/configs.  
- ⚠️ Use with caution (can break node security).  

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-demo
spec:
  containers:
  - name: logger
    image: busybox
    command: ["sh", "-c", "tail -f /logs/syslog"]
    volumeMounts:
    - name: host-logs
      mountPath: /logs
  volumes:
  - name: host-logs
    hostPath:
      path: /var/log
      type: Directory
```

---

### 3. Persistent Volume (PV) + Persistent Volume Claim (PVC)
- **Cluster-wide resource** provisioned by admin (static) or automatically (dynamic with StorageClass).  
- PVC lets users request storage without worrying about backend.  

**PV:**
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-demo
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt/data
  persistentVolumeReclaimPolicy: Retain
```

**PVC:**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-demo
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

**Pod using PVC:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-demo-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "echo hello from pvc > /mnt/storage/msg; sleep 3600"]
    volumeMounts:
    - name: storage
      mountPath: /mnt/storage
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: pvc-demo
```

---

### 4. `nfs` (Network File System)
- Shared storage across nodes.  

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nfs-demo
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "echo shared > /mnt/nfs/msg; sleep 3600"]
    volumeMounts:
    - name: nfs-vol
      mountPath: /mnt/nfs
  volumes:
  - name: nfs-vol
    nfs:
      server: 10.0.0.5
      path: /exports
```

---

### 5. `local` (Local PVs)
- Binds to a specific node’s local disk.  

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: local-storage
  local:
    path: /mnt/disks/ssd1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node01
```

---

## 🔹 PV Provisioning

- **Static**: Admin pre-creates PVs. PVCs bind if they match.  
- **Dynamic**: PVC triggers PV creation using `StorageClass`.  

**StorageClass Example:**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: kubernetes.io/no-provisioner  # e.g., aws-ebs, gce-pd
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
```

---

## 🔹 Access Modes
- **RWO** – ReadWriteOnce (single node).  
- **ROX** – ReadOnlyMany (many nodes, read-only).  
- **RWX** – ReadWriteMany (many nodes, read-write).  
- **RWOP** – ReadWriteOncePod (only one pod).  

---

## 🔹 Reclaim Policies
- **Delete** – Delete storage after PVC deletion.  
- **Retain** – Keep PV + data after PVC deletion.  
- **Recycle** – (deprecated).  

---

## 🔹 Key Takeaways
- Use **emptyDir** for temp scratch space.  
- Use **hostPath** only for node-specific, non-production cases.  
- Use **PVs + PVCs** for real persistence.  
- Use **StorageClass** for automation.  
- Choose **AccessModes** + **ReclaimPolicy** wisely.  

---
