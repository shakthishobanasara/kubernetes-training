
# Deployment vs StatefulSet with PV + Service (Side-by-Side Demo)

This demo shows the **practical differences** between a **Deployment** and a **StatefulSet**, both attaching storage and services.

---

## 1. Shared PVC with Deployment

In this setup:
- All replicas share the same PVC (`shared-pvc`).
- Pods are interchangeable (random names).
- Service load-balances across replicas.

### Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: busybox
        command: ["sh", "-c", "echo $(hostname) >> /data/out.txt; sleep 3600"]
        volumeMounts:
        - name: shared-storage
          mountPath: /data
      volumes:
      - name: shared-storage
        persistentVolumeClaim:
          claimName: shared-pvc
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Service
metadata:
  name: web-deployment-svc
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
```

### What happens?
- Pods get random names (`web-deployment-7d9f4d9f6b-abc12`).
- Both pods write into the same file `/data/out.txt` on the same PVC.
- Service distributes traffic randomly across pods.

---

## 2. Per-Pod PVCs with StatefulSet

In this setup:
- Each replica gets its own PVC (`data-web-0`, `data-web-1`, …).
- Pods have stable names (`web-0`, `web-1`).
- Headless service gives DNS like `web-0.web-stateful-svc.default.svc.cluster.local`.

### StatefulSet YAML

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "web-stateful-svc"
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: busybox
        command: ["sh", "-c", "echo $(hostname) >> /data/out.txt; sleep 3600"]
        volumeMounts:
        - name: data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
---
apiVersion: v1
kind: Service
metadata:
  name: web-stateful-svc
spec:
  clusterIP: None   # Headless service
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
```

### What happens?
- Pods are named `web-0` and `web-1`.
- Each pod writes into **its own PVC** (`data-web-0`, `data-web-1`).
- You can address pods individually:
  - `web-0.web-stateful-svc`
  - `web-1.web-stateful-svc`

---

## 3. Key Differences You’ll Observe

| Feature | Deployment | StatefulSet |
|---------|------------|-------------|
| **Pod names** | Random | Stable (`-0`, `-1`, …) |
| **PVCs** | One shared PVC | One PVC per pod |
| **Service** | LoadBalancer/ClusterIP | Headless Service (per-pod DNS) |
| **Scaling** | Arbitrary order | Ordered, one by one |
| **Use case** | Stateless apps (web, API) | Stateful apps (DBs, Kafka, Zookeeper) |

---

## 4. How to Test

1. Apply Deployment + PVC + Service:
   ```bash
   kubectl apply -f deployment-demo.yaml
   kubectl get pods,pvc,svc
   ```

2. Apply StatefulSet + Service:
   ```bash
   kubectl apply -f statefulset-demo.yaml
   kubectl get pods,pvc,svc
   ```

3. Check data:
   ```bash
   # For Deployment (all pods share one PVC)
   kubectl exec -it <any-deployment-pod> -- cat /data/out.txt

   # For StatefulSet (each pod has its own PVC)
   kubectl exec -it web-0 -- cat /data/out.txt
   kubectl exec -it web-1 -- cat /data/out.txt
   ```

You’ll notice:
- **Deployment** → all pods write into the **same file**.  
- **StatefulSet** → each pod writes into its **own file**.

---

✅ With this demo, you’ll clearly see why StatefulSets exist in addition to Deployments.
