# Storage Lab

In this lab you will provision storage using PersistentVolumes (PV), PersistentVolumeClaims (PVC), ConfigMaps and Secrets.

## 1. PersistentVolume and PersistentVolumeClaim

Create a static PersistentVolume (`pv.yaml`):

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
    path: /tmp/data
```

Create a PersistentVolumeClaim (`pvc.yaml`):

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

Apply both resources:

```bash
kubectl apply -f pv.yaml
kubectl apply -f pvc.yaml
kubectl get pv,pvc
```

## 2. Mount PVC in a Pod

You can mount the claim in a Pod or Deployment:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox-storage
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["sleep","3600"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: pvc-demo
```

Apply and verify the mount exists:

```bash
kubectl apply -f busybox-storage.yaml
kubectl exec -it busybox-storage -- ls /data
```

## 3. ConfigMaps

Create a ConfigMap from literal values:

```bash
kubectl create configmap app-config --from-literal=APP_MODE=production --from-literal=LOG_LEVEL=info
kubectl get configmap app-config -o yaml
```

Consume the ConfigMap in a Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cm-demo
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["sh","-c","echo $APP_MODE && sleep 3600"]
    env:
    - name: APP_MODE
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_MODE
```

Apply and observe:

```bash
kubectl apply -f cm-demo.yaml
kubectl logs cm-demo
```

## 4. Secrets

Create a Secret:

```bash
kubectl create secret generic db-secret --from-literal=username=admin --from-literal=password=Pa$$w0rd
kubectl get secrets db-secret -o yaml
```

Mount the Secret as environment variables:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-demo
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["sh","-c","echo $DB_USERNAME && echo $DB_PASSWORD && sleep 3600"]
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
```

Apply and inspect logs:

```bash
kubectl apply -f secret-demo.yaml
kubectl logs secret-demo
```

## Next Steps

Proceed to the `security/` lab to work with RBAC and Pod security.
