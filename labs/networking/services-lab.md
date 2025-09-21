# Kubernetes Services Lab

This lab explores the different **Service types** and **Ingress** in Kubernetes, how to create them, and when to use each.

---

## 1. Service Types Comparison

| Service Type | Command (example) | How to Access | When to Use (Real-world Scenario) |
|--------------|------------------|---------------|-----------------------------------|
| **ClusterIP** (default) | ```kubectl expose deployment web --port=80 --target-port=80 --type=ClusterIP``` | Inside cluster only: `http://web.default.svc.cluster.local:80` | Backend APIs, databases, or any **internal-only service**. |
| **NodePort** | ```kubectl expose deployment web --port=80 --target-port=80 --type=NodePort``` | Outside: `http://<NodeIP>:<NodePort>` (e.g., `http://192.168.99.100:30080`) | For **testing/demo** or when node IPs are reachable. Rarely prod. |
| **LoadBalancer** | ```kubectl expose deployment web --port=80 --target-port=80 --type=LoadBalancer``` | Cloud external IP: `http://<EXTERNAL-IP>:80` | Public-facing apps in **cloud providers** (AWS ELB, Azure LB, GCP LB). |
| **ExternalName** | ```kubectl create service externalname my-svc --external-name=db.example.com``` | Inside cluster: `http://my-svc.default.svc.cluster.local` → resolves to `db.example.com` | For connecting in-cluster apps to **external services**. |
| **Ingress** | ```kubectl apply -f ingress.yaml``` | Outside: `http://app.example.com` (after updating DNS/hosts) | For **routing multiple apps** behind a single IP/hostname with TLS termination. |

---

## 2. YAML Examples

### 2.1 ClusterIP Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-clusterip
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
```

---

### 2.2 NodePort Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
spec:
  type: NodePort
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080   # optional; if omitted, K8s assigns a random port (30000–32767)
```

---

### 2.3 LoadBalancer Service
*(works if your cluster runs in AWS, GCP, Azure, etc.)*
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-loadbalancer
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
```

---

### 2.4 ExternalName Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-db
spec:
  type: ExternalName
  externalName: db.example.com
  ports:
    - port: 3306   # logical port for clients (not used to open connections directly)
```

---

### 2.5 Ingress
*(requires an ingress controller like NGINX or Traefik)*  
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: web.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-clusterip
            port:
              number: 80
```

After applying, update `/etc/hosts` (for demo) with:
```
<ingress-controller-ip>   web.example.com
```

Then browse:
```
http://web.example.com
```

---

## 3. Tips & Tricks

- For quick local testing (e.g. Minikube):
  ```bash
  minikube service web-clusterip --url
  minikube service web-nodeport --url
  minikube service web-loadbalancer --url
  ```

- **ClusterIP** = internal-only (default).  
- **NodePort** = dev/demo, not usually prod.  
- **LoadBalancer** = cloud production use.  
- **ExternalName** = bridge to external DNS.  
- **Ingress** = one IP, many apps, HTTP routing + TLS.  

---
