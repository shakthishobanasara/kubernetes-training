# Observability Lab

This lab sets up metrics collection, logging, and visualization using the Prometheus stack and explores the Horizontal Pod Autoscaler (HPA).

## 1. Metrics Server

Install the Kubernetes metrics server to enable resource metrics:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Check that metrics are available:

```bash
kubectl top nodes
kubectl top pods
```

## 2. Horizontal Pod Autoscaler

Create an HPA for the `web` Deployment:

```bash
kubectl autoscale deployment web --min=2 --max=5 --cpu-percent=50
kubectl get hpa
```

Generate load to observe scaling:

```bash
kubectl run -i --tty load-gen --image=busybox -- /bin/sh
# Inside the shell, run a loop to generate CPU load
while true; do wget -q -O- http://web; done
```

## 3. Prometheus & Grafana via Helm

Add the repository and install the Prometheus community chart:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kube-prom prometheus-community/kube-prometheus-stack
```

Check the pods:

```bash
kubectl get pods -n kube-prometheus-stack
```

Port‑forward Grafana and log in (default user `admin`, password printed in secrets):

```bash
kubectl port-forward svc/kube-prometheus-stack-grafana -n kube-prometheus-stack 3000:80
```

Access Grafana at `http://localhost:3000` and explore dashboards.

## 4. Logging with kubectl

View logs of a Pod:

```bash
kubectl logs <pod-name>
```

Follow logs in real time:

```bash
kubectl logs -f <pod-name>
```

## Next Steps

Go to the `advanced/` lab to explore autoscaling nodes and custom resources.
