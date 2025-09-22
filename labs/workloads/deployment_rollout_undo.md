# kubectl rollout undo & Revisions — Hands‑On Guide

This guide walks you through creating revisions for a Deployment, viewing rollout history, and reverting to the previous or a specific revision. It also covers watch status, pause/resume, change‑cause annotations, and common troubleshooting.

---

## Prerequisites
- A working Kubernetes cluster and `kubectl` configured.
- A namespace to play in (optional):
  ```bash
  kubectl create ns demo-rollout
  kubectl config set-context --current --namespace=demo-rollout
  ```

---

## 1) Create a Sample Deployment (Revision 1)
Create `deploy.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels:
    app: web
spec:
  replicas: 2
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.25.3
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 2
            periodSeconds: 5
```
Apply it (this creates **Revision 1**):
```bash
kubectl apply -f deploy.yaml
kubectl rollout status deploy/web
```

> Tip: `revisionHistoryLimit` controls how many past ReplicaSets (revisions) are retained. If it’s too low, older revisions may be garbage‑collected and you can’t roll back to them.

---

## 2) Record a Change‑Cause for Clear History (Optional but Recommended)
Set a human‑readable reason for the next change using an annotation:
```bash
kubectl annotate deploy/web kubectl.kubernetes.io/change-cause="Initial deploy nginx:1.25.3"
```
You can also bake this annotation into your manifest’s metadata.

---

## 3) Make a Change to Create **Revision 2**
Update the image to create a new rollout (and add a change cause):
```bash
kubectl set image deploy/web nginx=nginx:1.26.0   --record=false
kubectl annotate deploy/web kubectl.kubernetes.io/change-cause="Upgrade to nginx:1.26.0"
kubectl rollout status deploy/web
```

Check history:
```bash
kubectl rollout history deploy/web
kubectl rollout history deploy/web --revision=1
kubectl rollout history deploy/web --revision=2
```

Expected output shape:
```
deployments "web"
REVISION  CHANGE-CAUSE
1         Initial deploy nginx:1.25.3
2         Upgrade to nginx:1.26.0
```

---

## 4) Create a **Broken Revision 3** (to practice rollback)
Intentionally set a bad image tag:
```bash
kubectl set image deploy/web nginx=nginx:does-not-exist
kubectl annotate deploy/web kubectl.kubernetes.io/change-cause="Broken image for demo"

# Watch rollout fail
kubectl rollout status deploy/web --timeout=30s || true

# Inspect events (note ImagePullBackOff, etc.)
kubectl describe deploy/web | sed -n '/Events/,$p'
```

Now you should see the latest ReplicaSet failing to become Ready.

---

## 5) Roll Back
### A) Undo the **last** rollout
```bash
kubectl rollout undo deploy/web
kubectl rollout status deploy/web
```
This reverts from the broken **Revision 3** back to **Revision 2**.

### B) Undo to a **specific** revision
```bash
kubectl rollout undo deploy/web --to-revision=1
kubectl rollout status deploy/web
```
This resets Pods to the template from **Revision 1**.

Re‑check history (note the new top revision reflects the rollback action):
```bash
kubectl rollout history deploy/web
```

---

## 6) Pause, Batch Changes, Resume
Useful when you want to stage multiple spec changes before triggering a rollout.
```bash
# Pause to stop new rollouts while you edit
kubectl rollout pause deploy/web

# Make multiple edits (image, env, resources, etc.)
kubectl set image deploy/web nginx=nginx:1.27.0
kubectl annotate deploy/web kubectl.kubernetes.io/change-cause="Batch update to nginx:1.27.0"

# Resume to trigger rollout
kubectl rollout resume deploy/web
kubectl rollout status deploy/web
```

---

## 7) Watch and Inspect
```bash
# Live watch
kubectl get rs,pods -l app=web -w

# See ReplicaSets and which one is current
kubectl get rs -l app=web -o wide

# See which Pods belong to which RS
kubectl get pods -l app=web --show-labels
```

---

## 8) DaemonSets & StatefulSets
`kubectl rollout undo` also works for **DaemonSet** and **StatefulSet** kinds:
```bash
kubectl rollout history ds/<name>
kubectl rollout undo ds/<name> --to-revision=<n>

kubectl rollout history statefulset/<name>
kubectl rollout undo statefulset/<name> --to-revision=<n>
```
> For StatefulSets, updates roll out ordinally (pod-0, pod-1, …). Rollbacks likewise proceed in order.

---

## 9) Common Issues & Fixes
- **No revisions found**: Ensure you changed the Pod template (e.g., image/env/resources). Scaling `replicas` alone doesn’t create a new revision.
- **Desired revision missing**: It may have been garbage‑collected due to `revisionHistoryLimit`. Increase the limit for critical apps.
- **Rollout stuck**: Check `kubectl describe deploy/<name>` events, failing probes, budget constraints (PDB), or quota limits. Use:
  ```bash
  kubectl get events --sort-by=.lastTimestamp | tail -n 20
  kubectl describe rs -l app=web
  kubectl describe pod -l app=web
  ```
- **Health checks failing after rollback**: Re‑validate `readinessProbe` and `livenessProbe` against the image you rolled back to.

---

## 10) Clean Up
```bash
kubectl delete ns demo-rollout
```

---

## Quick Command Reference
```bash
# History
kubectl rollout history deploy/<name>
kubectl rollout history deploy/<name> --revision=<n>

# Undo
kubectl rollout undo deploy/<name>
kubectl rollout undo deploy/<name> --to-revision=<n>

# Status / Flow control
kubectl rollout status deploy/<name>
kubectl rollout pause deploy/<name>
kubectl rollout resume deploy/<name>
```

---

### Appendix: Verify Revisions via ReplicaSet Annotations
Each revision corresponds to a ReplicaSet with `deployment.kubernetes.io/revision`:
```bash
kubectl get rs -l app=web -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.annotations.deployment\\.kubernetes\\.io/revision}{"\n"}{end}'
```
This is handy when correlating Pods ↔ RS ↔ Deployment revisions.
