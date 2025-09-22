# 🗃 Storage

Learn about Volumes, PV, PVC, and StorageClasses.

## Example
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mypvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```