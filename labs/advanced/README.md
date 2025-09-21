# Advanced Lab

This lab covers Vertical Pod Autoscaler (VPA), node autoscaling and custom operators.

## 1. Vertical Pod Autoscaler (optional)

VPA automatically adjusts container CPU and memory requests based on usage. It is a separate project and not included in vanilla Kubernetes. To install the VPA in a cluster, follow the instructions in the official repository.

Create a `vpa.yaml`:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  updatePolicy:
    updateMode: "Auto"
```

Apply the VPA and watch recommendations:

```bash
kubectl apply -f vpa.yaml
kubectl describe vpa web-vpa
```

## 2. Cluster Autoscaler and Karpenter

Cluster Autoscaler automatically adds or removes nodes based on pending Pods. Karpenter is an alternative that provisions nodes on demand. Installation is provider specific—follow your cloud provider's guide.

## 3. Custom Resources and Operators

Operators extend Kubernetes with custom resources and controllers. To deploy an operator:

1. Install the CRDs:

   ```bash
   kubectl apply -f crd.yaml
   ```
2. Deploy the operator controller:

   ```bash
   kubectl apply -f operator.yaml
   ```
3. Create an instance of the custom resource:

   ```bash
   kubectl apply -f cr-instance.yaml
   ```

Observe the operator reconciling the resource and creating managed objects.

## 4. Debugging Pods

Sometimes you need to troubleshoot running containers. Useful commands include:

```bash
kubectl describe pod <pod>
kubectl logs <pod>
kubectl exec -it <pod> -- /bin/sh
kubectl debug <pod> --image=busybox --share-processes
```

## Completion

You have completed the advanced lab. Feel free to experiment with scaling, custom resources and troubleshooting tools.
