# Kubernetes Sealed Secrets — Secure Secret Management for GitOps

Sealed Secrets (by Bitnami) let you **encrypt Kubernetes Secrets** into a `SealedSecret` custom resource.  
These can be safely stored in Git (even public repos). Only the controller in your cluster can decrypt them.

---

## 1) Why Sealed Secrets?

- **Secure GitOps**: Commit encrypted secrets without leaking sensitive data.
- **One-way encryption**: Even cluster admins can’t decrypt the sealed secret outside the cluster.
- **Automated workflow**: `SealedSecret` is converted into a `Secret` by the controller inside Kubernetes.

---

## 2) Components

1. **kubeseal CLI** — client-side tool to encrypt secrets.
2. **Sealed Secrets Controller** — runs in-cluster; holds private key for decryption.
3. **CRD SealedSecret** — new resource type managed by the controller.

---

## 3) Installation

### A. Install Controller (Helm, recommended)
```bash
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets   --namespace kube-system
```

### B. Install CLI
- macOS: `brew install kubeseal`
- Linux:
  ```bash
  wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.27.0/kubeseal-0.27.0-linux-amd64.tar.gz
  tar -xvf kubeseal-*.tar.gz kubeseal
  sudo install -m 755 kubeseal /usr/local/bin/kubeseal
  ```
- Windows: download binary from releases.

Verify:
```bash
kubeseal --version
```

---

## 4) Creating a Sealed Secret

### Step 1: Create a normal Secret (locally, not applied!)
```bash
kubectl create secret generic db-secret   --from-literal=username=admin   --from-literal=password=Pa$$w0rd   --dry-run=client -o yaml > db-secret.yaml
```

### Step 2: Seal it
```bash
kubeseal --format=yaml < db-secret.yaml > db-sealedsecret.yaml

kubeseal --cert=pub-cert.pem --format=yaml < db-secret.yaml > db-sealedsecret.yaml
```
- Uses the cluster’s public key for encryption.
- Output is safe to commit into Git.

### Step 3: Apply SealedSecret
```bash
kubectl apply -f db-sealedsecret.yaml
```
The controller will create the matching `Secret` in the same namespace.

---

## 5) Example SealedSecret YAML

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-secret
  namespace: default
spec:
  encryptedData:
    username: AgB+Hkdf9z...
    password: AgC93kdjf0...
```

After applying, the controller generates:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: default
type: Opaque
data:
  username: YWRtaW4=
  password: UGEkJHcwcmQ=
```

---

## 6) Common Commands

- Export cluster public key (optional, for offline sealing):
  ```bash
  kubeseal --fetch-cert > pub-cert.pem
  ```

- Seal using cert:
  ```bash
  kubeseal --cert pub-cert.pem < secret.yaml > sealed.yaml
  ```

- View existing SealedSecret:
  ```bash
  kubectl get sealedsecret db-secret -o yaml
  ```

- Delete:
  ```bash
  kubectl delete sealedsecret db-secret
  ```

---

## 7) Best Practices

- **Always seal before committing**: never push raw Secret YAMLs.
- **Namespace-sensitive**: A sealed secret is valid only for the namespace and name it was created for.
- **RBAC**: Restrict who can create raw `Secret` objects; enforce `SealedSecret` usage.
- **Rotation**: Rotate controller keys periodically; supports multiple keys for transition.
- **Combine with GitOps tools**: ArgoCD/FluxCD automatically apply sealed secrets.

---

## 8) Alternatives

- **Mozilla SOPS** (file-level encryption, supports many backends).
- **External Secrets Operator** (syncs from AWS Secrets Manager, Vault, etc.).
- **HashiCorp Vault** (dynamic secrets).

---

## 9) Clean-up

```bash
kubectl delete sealedsecret db-secret
kubectl delete secret db-secret
```
