
# 📄 Understanding Kubernetes Manifest Structure

Kubernetes manifests are YAML or JSON files that describe the **desired state** of Kubernetes objects such as Pods, Deployments, Services, etc.

---

## 🧱 Basic Structure

Every manifest follows a similar format:

```yaml
apiVersion: <group/version>
kind: <ObjectType>
metadata:
  name: <name>
  labels:
    key: value
spec:
  # Object-specific configuration goes here
```

---

## 🧩 Key Fields Breakdown

### 1. `apiVersion`
- Defines the API version to use for the object.
- Examples:
  - `v1` for core objects like Pods, Services
  - `apps/v1` for Deployments, StatefulSets

### 2. `kind`
- Specifies the type of Kubernetes object.
- Examples: `Pod`, `Deployment`, `Service`, `ConfigMap`, `Secret`

### 3. `metadata`
- Contains identifying information:
  - `name`: Unique name of the object
  - `namespace`: (Optional) Namespace for scoping
  - `labels` and `annotations`

### 4. `spec`
- Describes the **desired state** of the object.
- Different for each `kind`.

---

## ✅ Example: Pod Manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: demo
spec:
  containers:
  - name: my-container
    image: nginx
    ports:
    - containerPort: 80
```

---

## ✅ Example: Deployment Manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
      - name: my-container
        image: nginx
        ports:
        - containerPort: 80
```

### Note:
- `template` describes the Pod definition inside the Deployment.
- `selector.matchLabels` must match `template.metadata.labels`.

---

## 🧠 Tips

- Use `kubectl apply -f <file>.yaml` to apply manifests.
- Use `kubectl explain <resource>` to see schema details.
- Manifests can be managed declaratively with GitOps tools.

---

🎯 *Understanding the manifest structure is key to mastering Kubernetes object management.*
