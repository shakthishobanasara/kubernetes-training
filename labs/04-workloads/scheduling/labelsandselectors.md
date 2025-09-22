# Labels & Selectors in Kubernetes — Full Lab (with YAML files embedded)



---

## Overview

Labels and selectors are the foundation of how Kubernetes groups and identifies objects. This guide explains **what labels and selectors are**, shows **common selector types**, and demonstrates **why they are critical for Deployments, Services, ReplicaSets, and more** — with copy-paste YAML and commands.

---

## 1. Quick definitions

- **Label**: Key/value metadata attached to objects (Pods, Services, Deployments, Nodes, etc.). Example: `app=web`, `env=prod`, `tier=frontend`
- **Selector**: A query over labels that identifies a set of objects. Used by controllers (Deployment, ReplicaSet), Services, and other APIs to find matching Pods.

**Core idea:** Labels *tag* objects. Selectors *pick* objects by those tags.

---

## 2. Label syntax and examples

Labels are arbitrary key/value pairs:
```yaml
metadata:
  labels:
    app: web
    env: staging
    team.k8s.io/owner: alice
```

---

## 3. Selector types

### Equality-based (simple)
- `key=value` or `key!=value`

Example in a Service:
```yaml
selector:
  app: web
  tier: frontend
```

### Set-based (advanced)
- Operators: `in`, `notin`, `exists`, `!exists`

Example:
```yaml
selector:
  matchLabels:
    app: web
  matchExpressions:
  - { key: env, operator: In, values: ["staging", "prod"] }
  - { key: tier, operator: Exists }
```

---

## 4. Why selector must match the Pod template in a Deployment

A **Deployment** defines:
- A `.spec.selector` — how the Deployment finds pods it manages.
- A `.spec.template.metadata.labels` — labels applied to Pods it creates.

They must match; otherwise the Deployment cannot manage its Pods correctly.

---

## 5. Hands-on lab — files & expected flow

You will find three manifests below:

1. `deploy-correct.yaml` — correct Deployment (labels/selectors aligned)
2. `svc.yaml` — Service that selects those pods
3. `deploy-broken.yaml` — Deployment example that can illustrate issues when selectors/labels are misused

You can apply them directly by copying the YAML blocks into local files or by using here-docs.

---

## 6. YAML manifests (copy these into files)

### deploy-correct.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-correct
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo-web
  template:
    metadata:
      labels:
        app: demo-web
    spec:
      containers:
      - name: nginx
        image: nginx:1.23
        ports:
        - containerPort: 80
```

### svc.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-svc
spec:
  selector:
    app: demo-web
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```

### deploy-broken.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-broken
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-broken
  template:
    metadata:
      labels:
        app: web-broken
        extra: me
    spec:
      containers:
      - name: nginx
        image: nginx:1.23
```

---

## 7. Commands — step by step demo

### Apply the correct deployment and service
```bash
# Option A: Save files locally, then apply
# Save the YAML blocks above into deploy-correct.yaml and svc.yaml
kubectl apply -f deploy-correct.yaml
kubectl apply -f svc.yaml

# Option B: Apply via here-docs (single-shot)
cat <<'EOF' | kubectl apply -f -
# (paste deploy-correct.yaml content here)
EOF

cat <<'EOF' | kubectl apply -f -
# (paste svc.yaml content here)
EOF
```

Check status:
```bash
kubectl get deploy web-correct
kubectl get pods -l app=demo-web -o wide
kubectl get svc web-svc
kubectl get endpoints web-svc
kubectl describe svc web-svc
```

You should see endpoints populated with pod IPs.

### Demonstrate label-based selection
```bash
# Show pods and labels
kubectl get pods --show-labels

# Query by label
kubectl get pods -l app=demo-web
```

### Apply the broken deployment and observe differences
```bash
kubectl apply -f deploy-broken.yaml
kubectl get deploy web-broken
kubectl get pods -l app=web-broken --show-labels
```

Try creating a Service that expects a different label and observe empty endpoints:
```yaml
# bad-svc.yaml (example)
apiVersion: v1
kind: Service
metadata:
  name: bad-svc
spec:
  selector:
    app: not-present
  ports:
    - port: 80
      targetPort: 80
```
```bash
kubectl apply -f bad-svc.yaml
kubectl get endpoints bad-svc   # should be empty
kubectl describe svc bad-svc
```

### Label operations
```bash
# Add/modify a label
kubectl label pod <pod-name> env=staging --overwrite

# Remove a label
kubectl label pod <pod-name> env-

# Select with expressions
kubectl get pods -l 'env in (staging,prod)'
```

---

## 8. Troubleshooting & tips

- Ensure `.spec.selector.matchLabels` exactly matches `.spec.template.metadata.labels` in Deployments.
- Use `kubectl describe` to debug Services and Deployments.
- Avoid multiple controllers claiming the same label set.
- Prefer `app.kubernetes.io/*` recommended labels for standardization.

---

## 9. Cleanup
```bash
kubectl delete -f deploy-broken.yaml
kubectl delete -f svc.yaml
kubectl delete -f deploy-correct.yaml
kubectl delete svc bad-svc || true
```

---

## 10. Copy-ready: Downloadable YAMLs via here-docs

If you prefer, copy these three blocks into files locally. Example (Linux/macOS):
```bash
cat > deploy-correct.yaml <<'EOF'
# (paste the deploy-correct.yaml block from this doc)
EOF
cat > svc.yaml <<'EOF'
# (paste the svc.yaml block from this doc)
EOF
cat > deploy-broken.yaml <<'EOF'
# (paste the deploy-broken.yaml block from this doc)
EOF
```

---

### Final notes
This single MD contains both explanation and runnable artifacts for teaching/demo. Use the **correct** deployment to demonstrate happy-path, then show how services rely on labels to discover endpoints and how a **broken** deployment or misaligned selector causes services to have *no endpoints* or controllers to misbehave.
