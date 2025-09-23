# Kubernetes ConfigMaps — Creation & Usage Guide

ConfigMaps store **non‑confidential** key–value configuration for your apps. Use them to decouple config from images.

---

## 1) Ways to Create a ConfigMap

### A. From literal key–values (imperative)
```bash
kubectl create configmap app-config   --from-literal=APP_MODE=prod   --from-literal=TIMEOUT_SECONDS=30
```

### B. From a file
```bash
# file: settings.properties
SPRING_PROFILES_ACTIVE=dev
LOG_LEVEL=INFO

kubectl create configmap app-config --from-file=settings.properties
# Keys become the filename by default; override with:
kubectl create configmap app-config --from-file=app.properties=settings.properties
```

### C. From multiple files / a directory
```bash
# All files in ./config/ become entries; filenames are keys
kubectl create configmap app-config --from-file=./config/
```

### D. Declaratively (YAML) — recommended for GitOps
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
data:
  APP_MODE: "prod"
  TIMEOUT_SECONDS: "30"
  app.properties: |
    SPRING_PROFILES_ACTIVE=dev
    LOG_LEVEL=INFO
```
Apply:
```bash
kubectl apply -f app-configmap.yaml
```

### E. From env-file
```bash
# .env format (KEY=VALUE per line)
kubectl create configmap app-config --from-env-file=.env
```

> **Tip:** Use `--from-file` for multiline files; use `--from-literal` for simple key–values. Prefer **declarative YAML** for review and versioning.

---

## 2) Using ConfigMaps in Pods

### A. As environment variables
```yaml
apiVersion: v1
kind: Pod
metadata: { name: cm-env-demo }
spec:
  containers:
  - name: web
    image: nginx:1.27
    envFrom:
      - configMapRef:
          name: app-config
    env:
      - name: SINGLE_KEY
        valueFrom:
          configMapKeyRef:
            name: app-config
            key: APP_MODE
```

### B. As mounted files (volume)
```yaml
apiVersion: v1
kind: Pod
metadata: { name: cm-volume-demo }
spec:
  volumes:
    - name: cfg
      configMap:
        name: app-config
        # Optional: only select some keys, or set fileMode
        items:
          - key: app.properties
            path: app.properties
  containers:
  - name: web
    image: busybox
    command: ["sh","-c","cat /etc/cfg/app.properties; sleep 3600"]
    volumeMounts:
      - name: cfg
        mountPath: /etc/cfg
        readOnly: true
```

### C. Projected with other sources
```yaml
volumes:
- name: app-configs
  projected:
    sources:
    - configMap: { name: app-config }
    - secret: { name: db-secret }
```

---

## 3) Updates & Rollouts

- **Mutable (default):** Updating a ConfigMap **does not** automatically restart Pods. Trigger a rollout:
  ```bash
  kubectl rollout restart deploy/web
  ```
- **Immutable:** Set `immutable: true` to prevent accidental edits (requires re-creation to change).
  ```yaml
  metadata: { name: app-config }
  immutable: true
  ```

> **Watch out:** Pods mount ConfigMaps as files; the kubelet may refresh content, but most apps don’t hot‑reload. Prefer explicit restarts.

---

## 4) Best Practices

- Treat ConfigMaps as **application config**, not secrets.
- Keep keys small; total size limit is ~**1MiB** per object.
- Use **namespaces** and clear naming: `appname-env-config`.
- Store YAML in Git; use Kustomize/Helm for env overlays.
- For Windows containers, mount paths must be Windows‑style (e.g., `C:\config`).

---

## 5) Troubleshooting

```bash
kubectl describe configmap app-config
kubectl get configmap app-config -o yaml
kubectl exec -it <pod> -- env | grep APP_MODE
kubectl exec -it <pod> -- ls -l /etc/cfg
```

**Common issues**
- Key not found → check `items.key` spelling.
- Missing refresh → restart workload.
- Large files → use a volume (e.g., CSI, PVC) instead of stuffing huge data in ConfigMap.

---

## 6) Clean-up

```bash
kubectl delete configmap app-config
```
