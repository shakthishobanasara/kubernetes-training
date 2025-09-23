# Kubernetes Secrets — Creation, Security, and Usage

Secrets store **confidential** data (tokens, passwords, certs). They are base64-encoded in etcd (not encrypted by default).

---

## 1) Types of Secrets

- `Opaque` (default) — generic key–values
- `kubernetes.io/tls` — TLS cert/key pair
- `kubernetes.io/dockerconfigjson` — image pull credentials
- `kubernetes.io/basic-auth`, `kubernetes.io/ssh-auth` — convenience types
- `kubernetes.io/service-account-token` — auto-managed by Kubernetes

---

## 2) Ways to Create a Secret

### A. From literals (imperative)
```bash
# Use single quotes or escape $ to avoid shell expansion:
kubectl create secret generic db-secret   --from-literal='username=admin'   --from-literal='password=Pa$$w0rd'
# or: password=Pa\$\$w0rd
```

### B. From files
```bash
# Files become values; filenames become keys
echo -n 'admin' > username.txt
echo -n 'Pa$$w0rd' > password.txt
kubectl create secret generic db-secret --from-file=username.txt --from-file=password.txt
# Override keys:
kubectl create secret generic db-secret --from-file=user=username.txt --from-file=pwd=password.txt
```

### C. From an env-file
```bash
# .env (KEY=VALUE per line). Values are treated *literally* (no shell expansion).
kubectl create secret generic db-secret --from-env-file=.env
```

### D. TLS secret
```bash
kubectl create secret tls web-tls   --cert=server.crt   --key=server.key
```

### E. Docker registry secret (image pull)
```bash
kubectl create secret docker-registry regcred   --docker-server=registry.example.com   --docker-username=myuser   --docker-password='MyP@ss!'   --docker-email=dev@example.com
```

### F. Declaratively (YAML) — **recommended for GitOps**
> Store **encrypted** values in Git using SOPS/SealedSecrets; avoid committing raw base64!

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  username: admin         # plain text; API server encodes to data[]
  password: Pa$$w0rd
# data:                    # alternatively provide base64 directly
#   username: YWRtaW4=
#   password: UGEkJHcwcmQ=
```
Apply:
```bash
kubectl apply -f db-secret.yaml
```

---

## 3) Using Secrets in Pods

### A. As environment variables
```yaml
apiVersion: v1
kind: Pod
metadata: { name: secret-env-demo }
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh","-c","env | grep -E 'DB_' ; sleep 3600"]
    env:
      - name: DB_USER
        valueFrom:
          secretKeyRef:
            name: db-secret
            key: username
      - name: DB_PASS
        valueFrom:
          secretKeyRef:
            name: db-secret
            key: password
```

### B. As mounted files (volume)
```yaml
apiVersion: v1
kind: Pod
metadata: { name: secret-volume-demo }
spec:
  volumes:
  - name: creds
    secret:
      secretName: db-secret
      items:
      - key: username
        path: user
      - key: password
        path: pass
  containers:
  - name: app
    image: busybox
    command: ["sh","-c","ls -l /etc/creds; sleep 3600"]
    volumeMounts:
    - name: creds
      mountPath: /etc/creds
      readOnly: true
```

### C. Image pull secrets (for private images)
```yaml
apiVersion: v1
kind: Pod
metadata: { name: private-image-demo }
spec:
  imagePullSecrets:
  - name: regcred
  containers:
  - name: app
    image: registry.example.com/team/app:1.0.0
```

### D. Projected volumes (combine with ConfigMaps)
```yaml
volumes:
- name: settings
  projected:
    sources:
    - configMap: { name: app-config }
    - secret: { name: db-secret }
```

---

## 4) Updating Secrets (Preferred, Declarative)

> Many orgs **disallow direct editing** of `Secret` objects (RBAC/admission). Even when allowed, editing shows **base64** values and is error‑prone. Prefer generating YAML and applying it.

```bash
# Safest pattern
kubectl create secret generic db-secret   --from-literal='username=admin'   --from-literal='password=NewP@ss'   --dry-run=client -o yaml | kubectl apply -f -

# Or maintain db-secret.yaml and:
kubectl apply -f db-secret.yaml
kubectl rollout restart deploy/web   # refresh consumers
```

> Note: `kubectl edit secret db-secret` may work **only if** RBAC permits and the Secret is not `immutable: true`; however, it’s generally discouraged in production.

---

## 5) Security Notes & Limits

- Base64 ≠ encryption. **Enable encryption at rest** for etcd (api-server config) and use RBAC.
- Avoid printing secrets in logs. `kubectl describe` omits values by design.
- Consider **Sealed Secrets** (Bitnami) or **SOPS** (Mozilla) to keep secrets encrypted in Git.
- Mark secrets **immutable** to prevent accidental edits:
  ```yaml
  metadata: { name: db-secret }
  immutable: true
  ```
- Size limit ~**1MiB** per secret object.
- On multi-tenant clusters, segregate by **namespace** and restrict `get/list` via RBAC.

---

## 6) Verifying & Troubleshooting

```bash
kubectl get secret db-secret -o jsonpath='{.data.username}' | base64 -d; echo
kubectl get secret db-secret -o yaml          # values appear base64-encoded
kubectl describe secret db-secret             # does NOT show secret values
kubectl exec -it <pod> -- printenv | grep DB_
kubectl exec -it <pod> -- ls -l /etc/creds
```

**Common issues**
- **Password looks corrupted (e.g., `Pa6345w0rd`)** → you likely had `$` expansion in your shell. Use single quotes or escape dollars.
- Pod can’t pull image → set `imagePullSecrets` or create a Secret of type `dockerconfigjson`.
- Updates not reflected → restart workload; some apps only read at start.

---

## 7) Clean-up

```bash
kubectl delete secret db-secret
kubectl delete secret web-tls regcred --ignore-not-found
```
