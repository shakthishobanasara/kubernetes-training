# Environment Setup for Kubernetes CKA Training

This guide helps you set up a local environment suitable for following the labs in this CKA training. Two popular options are **Minikube** and **kind** (Kubernetes-in-Docker). Both allow you to run a single‑node cluster on your local machine.

## Prerequisites

- A machine with at least 8 GB RAM and 4 CPU cores
- Docker installed and running
- kubectl (Kubernetes CLI) installed
- Optional: Helm (package manager) installed

### Install kubectl

Follow the instructions for your operating system from the [official documentation](https://kubernetes.io/docs/tasks/tools/) to install `kubectl`. Verify the installation:

```bash
kubectl version --client
```

### Option 1: Minikube

Minikube runs a lightweight VM or container to host a single Kubernetes node.

1. [Install Minikube](https://minikube.sigs.k8s.io/docs/start/) for your OS.
2. Start a cluster:

   ```bash
   minikube start --cpus=4 --memory=8192 --driver=docker
   ```
3. Verify the cluster is running:

   ```bash
   kubectl get nodes
   ```
4. To stop the cluster when you're done:

   ```bash
   minikube stop
   ```

### Option 2: kind

kind creates a Kubernetes cluster in Docker containers. It is useful when you need multiple nodes.

1. Install kind:

   ```bash
   go install sigs.k8s.io/kind@v0.20.0
   ```
   Ensure `$GOPATH/bin` is in your `PATH`.
2. Create a cluster:

   ```bash
   kind create cluster --name cka-demo
   ```
3. Use kubectl to check the nodes:

   ```bash
   kubectl get nodes
   ```
4. To delete the cluster:

   ```bash
   kind delete cluster --name cka-demo
   ```

## Switching Contexts

If you have multiple clusters configured, use the `kubectl config` command to switch contexts:

```bash
kubectl config get-contexts
kubectl config use-context <context-name>
```

## Next Steps

Continue with the labs under the `pods/` directory to start creating your first Kubernetes resources.
