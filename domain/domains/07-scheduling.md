# ⚙️ Scheduling & Affinity

Learn about taints, tolerations, affinity, and priority classes.

## Example
```yaml
tolerations:
- key: "dedicated"
  operator: "Equal"
  value: "gpu"
  effect: "NoSchedule"
```