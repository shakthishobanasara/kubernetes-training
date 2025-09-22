# 🔧 Configuration & Secrets

Explore ConfigMaps, Secrets, and injecting environment variables into Pods.

## Example
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
data:
  APP_MODE: production
```