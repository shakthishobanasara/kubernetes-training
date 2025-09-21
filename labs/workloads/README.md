# Workloads Lab

Kubernetes provides higher‑level abstractions to manage pods at scale. In this lab you will work with Deployments, ReplicaSets, StatefulSets and DaemonSets.

## 1. Deployments

A Deployment manages a ReplicaSet to ensure a desired number of pods are running and provides rolling updates.

**Create a Deployment**:

```bash
kubectl create deployment web --image=nginx --replicas=2
```

Scale the Deployment:

```bash
kubectl scale deployment/web --replicas=5
```

Update the container image:

```bash
kubectl set image deployment/web nginx=nginx:1.23
```

Rollback if necessary:

```bash
kubectl rollout undo deployment/web
```

**Declarative Manifest** (`deployment.yaml`):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
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
        image: nginx:1.23
        ports:
        - containerPort: 80
```

Apply it with:

```bash
kubectl apply -f deployment.yaml
```

## 2. StatefulSets

StatefulSets provide stable network identities and persistent storage for stateful applications.

**Manifest** (`statefulset.yaml`):

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web-stateful
spec:
  serviceName: "web"
  replicas: 3
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
        image: nginx:1.23
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

Apply and inspect:

```bash
kubectl apply -f statefulset.yaml
kubectl get pods -l app=web
```

## 3. DaemonSets

DaemonSets ensure that all (or some) nodes run a copy of a Pod. Useful for logging or monitoring agents.

Example manifest:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-agent
spec:
  selector:
    matchLabels:
      name: log-agent
  template:
    metadata:
      labels:
        name: log-agent
    spec:
      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:v1.15-debian-1
```

Apply with:

```bash
kubectl apply -f daemonset.yaml
```

## Next Steps

Proceed to the `networking/` lab to expose your applications to internal and external clients.
