# Argo CD Self-Management Transition Procedure

This document describes the automated transition from manual Argo CD installation to Helm-based self-management.

**Note:** This procedure is specific to homelab/demo environments where Argo CD was initially installed manually and is now transitioning to self-management via Helm chart.

## Problem Statement

Kubernetes Deployment selectors are immutable. The manual installation creates Deployments with selector:
```yaml
selector:
  matchLabels:
    app.kubernetes.io/name: argocd-server
```

But the Helm chart requires:
```yaml
selector:
  matchLabels:
    app.kubernetes.io/name: argocd-server
    app.kubernetes.io/instance: argocd
```

Since selectors cannot be changed, the existing Deployments must be deleted and recreated.

## Solution: Automated via Force=true

The Application manifest includes `Force=true` sync option, which automatically handles this by:
1. Detecting immutable field conflicts
2. Deleting the existing resource
3. Creating the new resource with correct configuration

**No manual intervention required!**

### Sync Options Configuration

For this self-managed demo environment, the following sync options are used:
- ✅ `CreateNamespace=true` - Ensures the argocd namespace exists
- ✅ `Replace=true` - Handles large resource specifications
- ✅ `Force=true` - Deletes and recreates resources with immutable field conflicts
- ❌ `ServerSideApply=false` - **Removed to avoid schema validation conflicts**

**Why ServerSideApply is disabled:**
- ServerSideApply performs strict schema validation that fails on runtime status fields (e.g., `.status.terminatingReplicas`)
- Force=true uses delete/recreate strategy, making ServerSideApply's incremental patching unnecessary
- These two strategies are mutually exclusive - Force takes precedence but causes validation errors
- **For production environments**, you may want to enable ServerSideApply after the initial transition is complete and remove Force=true annotations

## Prerequisites

- Manual Argo CD installation already running
- Argo CD Application manifest committed to Git with `Force=true` sync option
- Values configured with `configs.secret.createSecret: false` to preserve secrets

## Transition Steps

### Step 1: Verify Current State

```bash
# Verify Argo CD is running
kubectl get deployments -n argocd

# Check current Deployment selectors
kubectl get deployment argocd-server -n argocd -o jsonpath='{.spec.selector}'
```

Expected output shows selector WITHOUT `app.kubernetes.io/instance`.

### Step 2: Apply Argo CD Self-Management Application

**The Force=true sync option will automatically delete and recreate Deployments during sync.**

```bash
# Apply the Application manifest
kubectl apply -f clusters/portcullis/infrastructure/argocd.yaml

# Watch the sync process (Deployments will be deleted and recreated)
kubectl get applications -n argocd argocd -w
```

**Expected behavior:**
1. Argo CD detects selector mismatch
2. Deletes existing Deployments (brief downtime)
3. Creates new Deployments with correct selectors
4. Pods come up with correct labels
5. Services can now select pods

### Step 3: Verify Helm Recreation

```bash
# Watch Deployments being recreated
kubectl get deployments -n argocd -w

# Verify new selectors include instance label
kubectl get deployment argocd-server -n argocd -o jsonpath='{.spec.selector}' | jq
```

Expected output should show:
```json
{
  "matchLabels": {
    "app.kubernetes.io/name": "argocd-server",
    "app.kubernetes.io/instance": "argocd"
  }
}
```

### Step 4: Verify Service Endpoints

```bash
# Check that Services can now select pods
kubectl get endpoints -n argocd

# All services should have endpoints listed
```

### Step 5: Test Argo CD Access

```bash
# Port forward to test
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Or test via Ingress
curl -k https://argocd.portcullis.osow.ski
```

## Rollback Procedure

If issues occur, reapply the manual installation:

```bash
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

## What Gets Preserved

- ✅ Secrets (argocd-secret, argocd-redis, argocd-notifications-secret)
- ✅ ConfigMaps (configurations will be managed by Helm going forward)
- ✅ Services (will be updated with new selectors)
- ✅ Admin password and authentication

## What Gets Recreated

- Deployments (with new immutable selectors)
- Pods (with new labels including instance)
- StatefulSet for application-controller

## Expected Downtime

- **Estimated**: 1-3 minutes
- **Affected**: Argo CD UI and API (application deployments continue running)
- **Recovery**: Automatic as Helm recreates resources

## Post-Transition

After successful transition:
- Argo CD manages itself via GitOps
- Changes to Argo CD configuration go through Git
- Helm chart upgrades handled via Application manifest version updates
