# Bootstrap Procedure

This document describes the detailed procedure for bootstrapping ArgoCD on a new or existing Kubernetes cluster.

## Prerequisites

### Cluster Requirements

- Kubernetes cluster (1.25+)
- ArgoCD installed in `argocd` namespace
- `kubectl` configured with admin access
- `helm` CLI installed (3.x)

### Installation Verification

Verify ArgoCD is installed:

```bash
kubectl get pods -n argocd
```

Expected output: All ArgoCD pods in `Running` state.

### Repository Access

Clone this repository:

```bash
git clone https://github.com/osowski/homelab-argocd.git
cd homelab-argocd
```

## Bootstrap Steps

### Step 1: Prepare Cluster-Specific Values

For a new cluster, create a values file:

**`bootstrap/values-<cluster-name>.yaml`**
```yaml
cluster:
  name: "<cluster-name>"
  domain: "osow.ski"
```

### Step 2: Validate Bootstrap Chart

Test the Helm template rendering:

```bash
helm template bootstrap ./bootstrap/ \
  --values ./bootstrap/values-<cluster-name>.yaml
```

Review the output for correctness:
- ArgoCD Projects created (infrastructure, workloads)
- Parent Applications created (infrastructure-apps, workloads-apps)
- Correct cluster name and repository URL

### Step 3: Apply Bootstrap

Deploy the bootstrap chart:

```bash
helm template bootstrap ./bootstrap/ \
  --values ./bootstrap/values-<cluster-name>.yaml \
  | kubectl apply -f -
```

Expected output:
```
appproject.argoproj.io/infrastructure created
appproject.argoproj.io/workloads created
application.argoproj.io/infrastructure-apps created
application.argoproj.io/workloads-apps created
```

### Step 4: Verify Parent Applications

Check that parent Applications are created:

```bash
kubectl get applications -n argocd
```

Expected output:
```
NAME                  SYNC STATUS   HEALTH STATUS
infrastructure-apps   Synced        Healthy
workloads-apps        Synced        Healthy
```

### Step 5: Wait for Child Applications

Parent Applications will create child Applications. Monitor:

```bash
kubectl get applications -n argocd -w
```

You should see child Applications appear as they're discovered in `clusters/<cluster-name>/` directories.

### Step 6: Verify Application Sync

Check that all applications are syncing:

```bash
kubectl get applications -n argocd -o wide
```

Look for:
- `SYNC STATUS`: Should be `Synced` or `Syncing`
- `HEALTH STATUS`: Should be `Healthy` or `Progressing`

### Step 7: Check Application Workloads

Verify that actual resources are deployed:

```bash
# List all namespaces created by applications
kubectl get namespaces

# Check pods in a specific application namespace
kubectl get pods -n <app-namespace>
```

## Accessing ArgoCD UI

### Port Forward (Development)

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Access at: https://localhost:8080

### Get Admin Password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

Login with username `admin` and the password from above.

### Create Ingress (Production)

For production access, create an ingress:

**`clusters/<cluster>/infrastructure/argocd-ingress.yaml`** (future enhancement)
```yaml
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argocd-ingress
  namespace: argocd
spec:
  project: infrastructure
  source:
    repoURL: https://github.com/osowski/homelab-argocd.git
    targetRevision: HEAD
    path: infrastructure/argocd-ingress/overlays/<cluster>
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## Troubleshooting

### Bootstrap Application Not Created

**Symptom**: `helm template` succeeds but `kubectl apply` fails

**Solutions**:
1. Verify ArgoCD CRDs are installed:
   ```bash
   kubectl get crd applications.argoproj.io
   kubectl get crd appprojects.argoproj.io
   ```

2. Check ArgoCD is running:
   ```bash
   kubectl get pods -n argocd
   ```

3. Review ArgoCD logs:
   ```bash
   kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller
   ```

### Parent Applications Not Syncing

**Symptom**: `infrastructure-apps` or `workloads-apps` shows `OutOfSync`

**Solutions**:
1. Check repository access:
   ```bash
   kubectl describe application infrastructure-apps -n argocd
   ```

2. Verify path exists in repository:
   ```bash
   ls -la clusters/<cluster-name>/infrastructure/
   ls -la clusters/<cluster-name>/workloads/
   ```

3. Manually sync from CLI:
   ```bash
   kubectl patch application infrastructure-apps -n argocd \
     --type merge -p '{"operation":{"sync":{}}}'
   ```

### Child Applications Not Created

**Symptom**: Parent apps are synced but child apps don't appear

**Solutions**:
1. Verify directory contains valid Application manifests:
   ```bash
   kubectl apply --dry-run=client -f clusters/<cluster>/workloads/<app>.yaml
   ```

2. Check parent Application logs:
   ```bash
   kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller \
     | grep infrastructure-apps
   ```

3. Force parent Application to refresh:
   ```bash
   kubectl delete application infrastructure-apps -n argocd
   helm template bootstrap ./bootstrap/ \
     --values ./bootstrap/values-<cluster-name>.yaml \
     | kubectl apply -f -
   ```

### Application Stuck in Progressing

**Symptom**: Application shows `Progressing` but never becomes `Healthy`

**Solutions**:
1. Check pod status:
   ```bash
   kubectl get pods -n <app-namespace>
   kubectl describe pod <pod-name> -n <app-namespace>
   ```

2. Review pod logs:
   ```bash
   kubectl logs <pod-name> -n <app-namespace>
   ```

3. Check ArgoCD Application health:
   ```bash
   kubectl describe application <app-name> -n argocd
   ```

### Repository Authentication Failed

**Symptom**: Applications show "failed to fetch repo" errors

**Solutions**:
1. Verify repository URL is correct in values file
2. Check ArgoCD can reach GitHub:
   ```bash
   kubectl exec -n argocd <argocd-server-pod> -- \
     curl -I https://github.com/osowski/homelab-argocd.git
   ```

3. If using private repo, configure credentials:
   ```bash
   kubectl create secret generic repo-credentials -n argocd \
     --from-literal=url=https://github.com/osowski/homelab-argocd.git \
     --from-literal=password=<token> \
     --from-literal=username=<username>
   ```

## Re-Bootstrapping

To re-apply the bootstrap (safe, idempotent):

```bash
helm template bootstrap ./bootstrap/ \
  --values ./bootstrap/values-<cluster-name>.yaml \
  | kubectl apply -f -
```

This is safe to run multiple times. Existing Applications will be updated, not recreated.

## Removing Bootstrap

To remove all ArgoCD Applications (WARNING: destructive):

```bash
# Delete parent Applications (will cascade to children)
kubectl delete application infrastructure-apps -n argocd
kubectl delete application workloads-apps -n argocd

# Delete Projects
kubectl delete appproject infrastructure -n argocd
kubectl delete appproject workloads -n argocd
```

**Note**: This removes ArgoCD Applications but does NOT remove the deployed workloads. To remove workloads, delete them before removing Applications, or set `prune: true` in sync policy.

## Upgrading Bootstrap

To update the bootstrap chart:

1. Make changes to `bootstrap/` files
2. Test locally:
   ```bash
   helm template bootstrap ./bootstrap/ \
     --values ./bootstrap/values-<cluster-name>.yaml
   ```

3. Apply changes:
   ```bash
   helm template bootstrap ./bootstrap/ \
     --values ./bootstrap/values-<cluster-name>.yaml \
     | kubectl apply -f -
   ```

4. Verify Applications updated:
   ```bash
   kubectl get applications -n argocd
   ```

## Post-Bootstrap Tasks

After successful bootstrap:

1. **Verify all applications synced**:
   ```bash
   kubectl get applications -n argocd
   ```

2. **Check application health**:
   ```bash
   kubectl get pods --all-namespaces
   ```

3. **Test application endpoints**:
   ```bash
   # Get LoadBalancer IP
   kubectl get svc -n ingress traefik

   # Update /etc/hosts with application FQDNs
   # Test with curl
   ```

4. **Configure ArgoCD notifications** (optional):
   - Slack, email, or webhook notifications for sync events

5. **Set up monitoring** (optional):
   - Monitor ArgoCD Application sync status
   - Alert on OutOfSync or Degraded states

## Next Steps

- Add new applications: [Adding Applications](adding-applications.md)
- Onboard new clusters: [Cluster Onboarding](cluster-onboarding.md)
- Understand architecture: [Architecture](architecture.md)
