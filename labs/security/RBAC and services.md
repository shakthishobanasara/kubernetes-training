# Kubernetes Authentication & Authorization (Step by Step)

This guide explains **who you are** (authentication) and **what you can do** (authorization) in Kubernetes—with copy-paste demos.

---

## 1. Mental Model
1. **Client → API server**: You send a request (kubectl, curl, app).
2. **Authentication**: API server verifies your identity (certs, tokens, OIDC, etc.).
3. **Authorization (RBAC)**: API server checks if that identity is allowed to do the verb on the resource.
4. **Admission**: Mutating/Validating webhooks may modify/validate the request before it’s stored.

---

## 2. Identities in Kubernetes
- **Users**: external identities (humans, CI, etc.). Not objects in the cluster.
- **ServiceAccounts (SAs)**: in-cluster identities (for Pods). They live in a namespace and are used by your workloads.

---

## 3. RBAC Building Blocks
- **Role** (namespaced) & **ClusterRole** (cluster-wide): *What* actions are allowed (verbs) on *which* resources.
- **RoleBinding** (namespaced) & **ClusterRoleBinding** (cluster-wide): *Who* (subjects: users, groups, SAs) gets those permissions.
- **Subjects** examples:
  - User: `alice`
  - Group: `devs`
  - ServiceAccount: `system:serviceaccount:<namespace>:<name>`

---

## 4. End-to-End Lab (Namespaced Access)

**Goal**: make a ServiceAccount that can list/get pods in a `demo` namespace—nothing else.

### A. Setup
```bash
kubectl create ns demo
kubectl -n demo create serviceaccount robot
```

### Option 1: Imperative RBAC
```bash
kubectl -n demo create role pod-reader \
  --verb=get --verb=list --verb=watch \
  --resource=pods

kubectl -n demo create rolebinding pod-reader-binding \
  --role=pod-reader \
  --serviceaccount=demo:robot
```

### Option 2: YAML
```yaml
# role-pod-reader.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: demo
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get","list","watch"]
---
# rb-pod-reader.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: demo
subjects:
- kind: ServiceAccount
  name: robot
  namespace: demo
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f role-pod-reader.yaml
kubectl apply -f rb-pod-reader.yaml
```

### B. Test Authorization
```bash
# Should be allowed
kubectl auth can-i list pods -n demo \
  --as=system:serviceaccount:demo:robot

# Should be forbidden
kubectl auth can-i get secrets -n demo \
  --as=system:serviceaccount:demo:robot
```

### C. Get a Short-Lived Token (1.24+)
```bash
TOKEN=$(kubectl -n demo create token robot --duration=10m)
echo "$TOKEN" | head -c 40 && echo "..."
```

Test with kubectl:
```bash
kubectl --token="$TOKEN" -n demo get pods
kubectl --token="$TOKEN" -n demo get secrets   # should be forbidden
```

### D. Use from Inside a Pod
```bash
kubectl -n demo run tester --image=bitnami/kubectl:latest \
  --restart=Never --serviceaccount=robot --command -- sleep 3600

kubectl -n demo exec -it tester -- sh -lc '
API="https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT"
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
CACERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
curl -s --cacert "$CACERT" -H "Authorization: Bearer $TOKEN" \
  "$API/api/v1/namespaces/demo/pods" | head
'
```

---

## 5. Cluster-Wide Example (ClusterRole)

Allow the same SA to list **nodes**:
```bash
kubectl create clusterrole node-reader \
  --verb=get --verb=list --verb=watch \
  --resource=nodes

kubectl create clusterrolebinding node-reader-binding \
  --clusterrole=node-reader \
  --serviceaccount=demo:robot
```

Test:
```bash
kubectl --token="$TOKEN" get nodes
```

---

## 6. Common Patterns & Tips
- **Default SA**: Pods without `serviceAccountName` use `default`. Avoid giving it broad rights.
- **Lock down tokens**:
  - Pods that don’t need API: `automountServiceAccountToken: false`.
  - Use short-lived tokens (`kubectl create token`) for CI.
- **Groups**: Bind ClusterRoles to IdP groups (e.g., `devs`).
- **Predefined roles**: `view`, `edit`, `admin` exist—bind instead of reinventing.
- **Impersonation**: `kubectl auth can-i ... --as=<user>` to test without real creds.
- **Authorization modes**: Usually `Node,RBAC`. Others: `Webhook, ABAC` (rare).
- **Admission**: After RBAC, webhooks enforce org policies (Pod Security, OPA, Kyverno).

---

## 7. Debug Checklist
- Wrong subject in RoleBinding? For SAs, `kind: ServiceAccount` + correct namespace.
- Cluster resource (like nodes)? Needs **ClusterRole**.
- Cross-namespace access? Roles are namespace-bound.
- Old long-lived tokens? Since 1.24, use TokenRequest or Pod-projected tokens.
