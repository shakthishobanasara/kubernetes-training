# Other major components going down — quick impacts

## CoreDNS
- **Symptom**: Service names don’t resolve; `nslookup kubernetes.default` fails.
- **Impact**: Apps that rely on DNS to reach Services will fail. Direct IPs may still work.
- **Detect**: `kubectl -n kube-system get deploy coredns`, check pods/logs.
- **Fix**: Restore deployment; check NodeLocal DNSCache if used; verify CNI path to CoreDNS.

## Ingress Controller (e.g., NGINX, Traefik, HAProxy)
- **Symptom**: External HTTP(S) 4xx/5xx or timeouts.
- **Impact**: Only Ingress-managed traffic is affected; internal ClusterIP still works.
- **Detect**: `kubectl -n ingress-nginx get pods`, check LB health probes.
- **Fix**: Restore controller; validate ConfigMap, TLS secrets, admission webhook.

## Cloud Controller Manager (CCM)
- **Symptom**: LoadBalancers don’t provision/update; Node addresses/Routes not managed.
- **Impact**: `type: LoadBalancer` Services stuck `EXTERNAL-IP: <pending>`, node routes outdated.
- **Detect**: `kubectl -n kube-system get pods -l k8s-app=cloud-controller-manager`
- **Fix**: Restart CCM; check cloud credentials/permissions and provider API status.

## CSI components (external-provisioner / attacher / node-driver)
- **Symptom**: PVCs stuck `Pending`; volumes not attaching/mounting.
- **Impact**: Workloads needing storage won’t start or will crashloop on mount.
- **Detect**: `kubectl -n <csi-ns> get pods`, look at PVC/PV events.
- **Fix**: Restore CSI pods; verify StorageClass & credentials.

## Metrics Server
- **Symptom**: `kubectl top` fails; HPA says “missing/incomplete metrics”.
- **Impact**: Autoscaling decisions stall (HPA holds last known replicas).
- **Detect**: `kubectl -n kube-system get deploy metrics-server`
- **Fix**: Restore metrics-server; verify API aggregation, certs, and flags.

## Admission Webhooks (mutating/validating)
- **Symptom**: Creates/updates **hang or fail** (`timeout`/`denied`) for resources subject to webhooks.
- **Impact**: Deployments might be blocked globally.
- **Detect**: `kubectl get validatingwebhookconfigurations -A`
- **Fix**: Fix the webhook service/certs or set `failurePolicy: Ignore` during recovery.

---

## Quick demo ideas (safe in lab)
- Scale `coredns` to 0 → show DNS failures.
- Stop Ingress controller → external traffic fails, ClusterIP works.
- Scale down metrics-server → HPA stops moving.

---

## General troubleshooting pattern
1. **Health checks**: `kubectl -n kube-system get pods`, component logs.
2. **Control-plane vs Node**: isolate where failure is.
3. **Dependencies**: etcd ↔ apiserver; CNIs for DNS; CSI for pods with PVCs.
4. **Rollback recent changes**; restore manifests; verify cert/cred expiry.
