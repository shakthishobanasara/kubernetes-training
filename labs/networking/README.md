# Networking Lab

This lab explores Services and Ingress resources to expose your applications.

## 1. ClusterIP Service

Expose the `web` Deployment internally:

```bash
kubectl expose deployment web --port=80 --target-port=80 --type=ClusterIP
kubectl get svc
```

## 2. NodePort Service

Expose the Deployment on each node's IP and a random high port:

```bash
kubectl expose deployment web --port=80 --target-port=80 --type=NodePort
kubectl get svc
```

Find the node's IP address and the allocated nodePort to access the application from your browser.

## 3. LoadBalancer Service (Cloud)

If you are running on a cloud provider that supports external load balancers, you can create a LoadBalancer service:

```bash
kubectl expose deployment web --port=80 --target-port=80 --type=LoadBalancer
```

## 4. Ingress

Ingress resources allow you to define HTTP(S) routing rules for multiple services behind a single endpoint.

First ensure your cluster has an ingress controller installed (e.g., Nginx). Then create `ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
spec:
  rules:
  - host: web.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web
            port:
              number: 80
```

Apply and check:

```bash
kubectl apply -f ingress.yaml
kubectl get ingress
kubectl describe ingress web-ingress
```

Update your `/etc/hosts` or DNS to point `web.example.com` to the ingress controller's IP.

## Next Steps

Proceed to the `storage/` lab to learn about persistent storage and configuration.
