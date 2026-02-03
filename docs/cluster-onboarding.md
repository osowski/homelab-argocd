# Cluster Onboarding

This guide describes how to onboard a new Kubernetes cluster to this GitOps repository.

## Overview

Onboarding a cluster involves:
1. Preparing the cluster (installing ArgoCD)
2. Creating cluster-specific configuration
3. Deploying the bootstrap
4. Adding applications

## Prerequisites

### Cluster Setup

The cluster must have:
- Kubernetes 1.25 or later
- Network connectivity to GitHub
- LoadBalancer or NodePort access for ingress
- Sufficient resources for workloads

### Required Tools

On your local machine:
- `kubectl` configured for the cluster
- `helm` CLI (3.x or later)
- `git` CLI

## Step 1: Install ArgoCD

If ArgoCD is not already installed, deploy it using the standard installation:

```bash
# Create argocd namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for all ArgoCD pods to be running:

```bash
kubectl get pods -n argocd -w
```

## Step 2: Create Cluster Directory Structure

In this repository, create the cluster directory structure:

```bash
mkdir -p clusters/<cluster-name>/infrastructure
mkdir -p clusters/<cluster-name>/workloads
```

Create a `.gitkeep` file to preserve empty directories:

```bash
touch clusters/<cluster-name>/infrastructure/.gitkeep
touch clusters/<cluster-name>/workloads/.gitkeep
```

## Step 3: Create Cluster-Specific Values

Create a values file for the cluster:

**`bootstrap/values-<cluster-name>.yaml`**
```yaml
cluster:
  name: "<cluster-name>"
  domain: "osow.ski"
```

Adjust the domain if different for this cluster.

## Step 4: Test Bootstrap Rendering

Validate the bootstrap configuration:

```bash
helm template bootstrap ./bootstrap/ \
  --values ./bootstrap/values-<cluster-name>.yaml
```

Review the output:
- Verify cluster name is correct
- Check repository URL is correct
- Ensure paths reference the new cluster directories

## Step 5: Commit Cluster Configuration

Commit the new cluster configuration:

```bash
git add bootstrap/values-<cluster-name>.yaml
git add clusters/<cluster-name>/
git commit -m "Add <cluster-name> cluster configuration"
git push
```

## Step 6: Deploy Bootstrap to Cluster

Deploy the bootstrap to the new cluster:

```bash
helm template bootstrap ./bootstrap/ \
  --values ./bootstrap/values-<cluster-name>.yaml \
  | kubectl apply -f -
```

Verify the bootstrap was created:

```bash
kubectl get applications -n argocd
```

Expected output:
```
NAME                  SYNC STATUS   HEALTH STATUS
infrastructure-apps   Synced        Healthy
workloads-apps        Synced        Healthy
```

## Step 7: Add Applications to Cluster

Add applications by creating ArgoCD Application manifests in the cluster directories.

### Example: Add http-echo workload

**`clusters/<cluster-name>/workloads/http-echo.yaml`**
```yaml
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: http-echo
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: workloads
  source:
    repoURL: https://github.com/osowski/homelab-argocd.git
    targetRevision: HEAD
    path: workloads/http-echo/overlays/<cluster-name>
  destination:
    server: https://kubernetes.default.svc
    namespace: http-echo
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Create Overlay for the Application

**`workloads/http-echo/overlays/<cluster-name>/kustomization.yaml`**
```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

patches:
  - path: ingress-patch.yaml
    target:
      kind: Ingress
      name: http-echo
```

**`workloads/http-echo/overlays/<cluster-name>/ingress-patch.yaml`**
```yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: http-echo
  namespace: http-echo
spec:
  rules:
    - host: echo.<cluster-name>.osow.ski
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: http-echo
                port:
                  name: http
```

Commit and push:

```bash
git add workloads/http-echo/overlays/<cluster-name>/
git add clusters/<cluster-name>/workloads/http-echo.yaml
git commit -m "Add http-echo to <cluster-name> cluster"
git push
```

## Step 8: Verify Application Deployment

ArgoCD will automatically detect the new Application and sync it:

```bash
# Watch applications
kubectl get applications -n argocd -w

# Check pods
kubectl get pods -n http-echo

# Check ingress
kubectl get ingress -n http-echo
```

## Step 9: Configure DNS and Ingress

### Get LoadBalancer IP

```bash
kubectl get svc -n ingress traefik -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

### Update DNS

Add DNS records for the cluster applications:
- `*.{cluster-name}.osow.ski` → LoadBalancer IP

Or update `/etc/hosts` for local testing:
```
<LoadBalancer-IP>  echo.<cluster-name>.osow.ski
```

### Test Application

```bash
curl http://echo.<cluster-name>.osow.ski
```

## Cluster-Specific Considerations

### Resource Constraints

For resource-constrained clusters, adjust:
- Replica counts in overlays
- Resource requests/limits
- Number of applications deployed

### Network Policies

If the cluster uses NetworkPolicies:
1. Allow ArgoCD to pull from GitHub
2. Allow ingress traffic to workloads
3. Allow inter-namespace communication if needed

### Storage Classes

Verify storage classes exist for persistent workloads:

```bash
kubectl get storageclass
```

Create cluster-specific storage configurations if needed.

### Ingress Controller

Ensure ingress controller is installed:
- Traefik (default for MicroK8s)
- Nginx Ingress
- Other

Adjust ingress annotations in overlays accordingly.

## Multi-Cluster Patterns

### Shared Applications

For applications deployed to all clusters:
1. Create base manifests in `workloads/<app>/base/`
2. Create overlay per cluster in `workloads/<app>/overlays/<cluster>/`
3. Reference overlay in each cluster's Application manifest

### Cluster-Specific Applications

For applications unique to one cluster:
1. Create manifests in `workloads/<app>/overlays/<cluster>/` only
2. Skip creating a base (or create minimal base)
3. Reference in that cluster's Application manifest only

### Environment Separation

Use clusters to separate environments:
- `dev` cluster - Development environment
- `staging` cluster - Staging environment
- `prod` cluster - Production environment

Use overlays to adjust:
- Replica counts
- Resource limits
- External service endpoints

## Removing a Cluster

To decommission a cluster:

### Step 1: Remove Applications

Delete all Application manifests:

```bash
rm -rf clusters/<cluster-name>/
git commit -m "Remove <cluster-name> cluster"
git push
```

### Step 2: Wait for Cleanup

ArgoCD will delete the Applications from the cluster. Monitor:

```bash
kubectl get applications -n argocd -w
```

### Step 3: Remove Bootstrap

Delete parent Applications:

```bash
kubectl delete application infrastructure-apps -n argocd
kubectl delete application workloads-apps -n argocd
```

### Step 4: Uninstall ArgoCD (Optional)

```bash
kubectl delete namespace argocd
```

### Step 5: Clean Up Repository

Remove cluster-specific files:

```bash
rm bootstrap/values-<cluster-name>.yaml
git commit -m "Remove <cluster-name> bootstrap configuration"
git push
```

## Troubleshooting

### Bootstrap Fails to Deploy

**Symptom**: `kubectl apply` fails with errors

**Solutions**:
1. Verify ArgoCD is installed:
   ```bash
   kubectl get pods -n argocd
   ```

2. Check CRDs are present:
   ```bash
   kubectl get crd applications.argoproj.io
   ```

3. Review error messages in bootstrap output

### Applications Not Syncing

**Symptom**: Applications created but not syncing

**Solutions**:
1. Verify repository access from cluster
2. Check Application path is correct
3. Ensure overlay directory exists for cluster
4. Review ArgoCD logs for errors

### Ingress Not Working

**Symptom**: Applications deployed but not accessible

**Solutions**:
1. Verify ingress controller is running:
   ```bash
   kubectl get pods -n ingress
   ```

2. Check LoadBalancer has external IP:
   ```bash
   kubectl get svc -n ingress
   ```

3. Verify DNS or `/etc/hosts` is configured
4. Test service directly:
   ```bash
   kubectl port-forward -n <namespace> svc/<service> 8080:8080
   curl http://localhost:8080
   ```

### Resource Exhaustion

**Symptom**: Pods stuck in `Pending` state

**Solutions**:
1. Check node resources:
   ```bash
   kubectl top nodes
   ```

2. Check pod resource requests:
   ```bash
   kubectl describe pod <pod-name> -n <namespace>
   ```

3. Adjust resource requests in overlays
4. Scale down or remove applications

## Best Practices

1. **Use consistent naming**: `<cluster-name>` should match across all files
2. **Test bootstrap locally**: Always run `helm template` before applying
3. **Start with minimal applications**: Add one application at a time
4. **Document cluster-specific notes**: Add README in `clusters/<cluster-name>/`
5. **Monitor resource usage**: Track CPU, memory, and storage
6. **Version control everything**: Commit all changes to Git
7. **Use automated sync**: Enable auto-sync for faster feedback

## Next Steps

- Add applications: [Adding Applications](adding-applications.md)
- Review architecture: [Architecture](architecture.md)
- Bootstrap procedure: [Bootstrap Procedure](bootstrap-procedure.md)
