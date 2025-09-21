# Pods Lab

This lab introduces the basic building block of Kubernetes: the **Pod**. A Pod encapsulates one or more containers that share network and storage resources.

## Objectives

- Create a simple Pod from the command line (imperative)
- Define a Pod using a declarative YAML manifest
- Inspect Pod status and logs
- Explore basic probing configurations

## 1. Imperative Pod Creation

Create a Pod running the `nginx` image:

```bash
kubectl run nginx --image=nginx --restart=Never
```

List pods:

```bash
kubectl get pods
```

Describe the Pod and view events:

```bash
kubectl describe pod nginx
```

## 2. Declarative Pod Manifest

Create a file `pod.yaml` with the following contents:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-declarative
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 30
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
```

Apply the manifest:

```bash
kubectl apply -f pod.yaml
```

View Pod logs:

```bash
kubectl logs nginx-declarative
```

Delete both Pods when finished:

```bash
kubectl delete pod nginx nginx-declarative
```

## Next Steps

Proceed to the `workloads/` lab to learn about Deployments and StatefulSets.
