# 📈 Monitoring, Logging & Observability

Use probes, metrics-server, Prometheus, and logging stacks.

## Example
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 3
  periodSeconds: 3
```