# Security Lab

This lab covers Role‑Based Access Control (RBAC) and Pod Security admission.

## 1. Service Accounts and RBAC

Create a service account:

```bash
kubectl create serviceaccount viewer
```

Define a Role (`role.yaml`) that allows listing Pods:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

Bind the role to the service account:

```bash
kubectl create rolebinding read-pods-binding --role=pod-reader --serviceaccount=default:viewer --namespace=default
```

Test permissions:

```bash
kubectl auth can-i list pods --as system:serviceaccount:default:viewer
```

## 2. Pod Security Admission

As of Kubernetes 1.25, PodSecurityPolicy is deprecated. Pod Security Admission replaces it by enforcing preset policies at the namespace level.

Label the namespace to enforce the **baseline** policy:

```bash
kubectl label --overwrite ns default pod-security.kubernetes.io/enforce=baseline
```

Audit a namespace with the **restricted** policy:

```bash
kubectl label --overwrite ns default pod-security.kubernetes.io/audit=restricted
```

Warn on violations:

```bash
kubectl label --overwrite ns default pod-security.kubernetes.io/warn=baseline
```

Check the namespace:

```bash
kubectl describe ns default
```

## Next Steps

Continue to the `observability/` lab to set up monitoring and logging.
