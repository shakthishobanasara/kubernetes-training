# 🌐 Networking

Dive into Kubernetes networking, Services, DNS, and Ingress.

## Example
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```