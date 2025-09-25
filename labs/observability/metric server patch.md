---

## 0. Prerequisites: Metrics Server
HPA requires Metrics Server.

```bash
kubectl top nodes || kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Patch if kubelet TLS blocks metrics (common in labs/playgrounds):
kubectl -n kube-system patch deploy metrics-server --type='json' -p='[
  {"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"},
  {"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname"}
]'

# Verify
kubectl top nodes
kubectl top pods -A
```

---
