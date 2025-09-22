
# 🔧 Connecting to a Kubernetes Cluster using `kubectl`

This guide walks you through everything you need to know about configuring `kubectl` to connect to your Kubernetes cluster, using `kubeconfig` files, setting context, and managing multiple clusters.

---

## 📁 1. What is `kubeconfig`?

`kubeconfig` is the configuration file used by `kubectl` to access Kubernetes clusters.

- Default location: `$HOME/.kube/config` (Linux/macOS) or `%USERPROFILE%\.kube\config` (Windows)
- Contains details about:
  - **clusters**: API server endpoints
  - **users**: Authentication details (token, certs, etc.)
  - **contexts**: Cluster + user + namespace combo

---

## 🛠 2. Basic Syntax

### View current context:
```bash
kubectl config current-context
```

### View all contexts:
```bash
kubectl config get-contexts
```

### Set the current context:
```bash
kubectl config use-context <context-name>
```

### View full config:
```bash
kubectl config view --minify
```

---

## 📌 3. Setting Up Access to a Cluster

### Option 1: Use `kubeconfig` file directly
```bash
export KUBECONFIG=/path/to/your/kubeconfig.yaml
kubectl get pods
```

You can also combine multiple kubeconfig files:
```bash
export KUBECONFIG=~/.kube/config:/path/to/another/kubeconfig
```

### Option 2: Copy to default location
```bash
cp my-kubeconfig.yaml ~/.kube/config
```

---

## 🔄 4. Working with `kubectl config`

### Set a cluster manually:
```bash
kubectl config set-cluster my-cluster --server=https://1.2.3.4 --insecure-skip-tls-verify=true
```

### Set credentials for a user:
```bash
kubectl config set-credentials my-user --token=abcd1234
```

### Set a context:
```bash
kubectl config set-context my-context --cluster=my-cluster --user=my-user
```

### Use that context:
```bash
kubectl config use-context my-context
```

---

## 🌐 5. Multiple Cluster Configs

### Merge kubeconfigs:
```bash
KUBECONFIG=config1.yaml:config2.yaml kubectl config view --merge --flatten > merged-config.yaml
```

---

## 💡 6. Tips

- Always back up your `~/.kube/config` before making changes.
- Use tools like `kubectx` and `kubens` for switching between contexts and namespaces easily.
- Cloud providers (e.g., AWS, GCP, Azure) have their own commands to update kubeconfig:
  - AWS: `aws eks update-kubeconfig --name cluster-name`
  - GCP: `gcloud container clusters get-credentials cluster-name`
  - Azure: `az aks get-credentials --name cluster-name`

---

## ✅ 7. Test Your Connection

```bash
kubectl cluster-info
kubectl get nodes
kubectl get pods --all-namespaces
```

If these return results, your `kubectl` is properly connected.

---

🔐 *Be mindful of storing sensitive credentials in kubeconfig files. Use encryption and proper access controls.*

