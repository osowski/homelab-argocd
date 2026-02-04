# Argo CD Self-Management Guide

This document describes the transition from manual Argo CD installation to Helm-based self-management for homelab/demo environments.

**Context:** This procedure addresses the specific challenge of migrating a manually-installed Argo CD instance to GitOps self-management while handling immutable Kubernetes field constraints.

## Table of Contents

1. [Problem Statement](#problem-statement)
2. [Solution Overview](#solution-overview)
3. [Prerequisites](#prerequisites)
4. [Transition Procedure](#transition-procedure)
5. [Troubleshooting](#troubleshooting)
6. [Post-Migration](#post-migration)
7. [References](#references)

---

## Problem Statement

Kubernetes Deployment selectors are **immutable** - once created, they cannot be changed.

**Manual installation** creates Deployments with selector:
```yaml
selector:
  matchLabels:
    app.kubernetes.io/name: argocd-server
```

**Helm chart** requires selector:
```yaml
selector:
  matchLabels:
    app.kubernetes.io/name: argocd-server
    app.kubernetes.io/instance: argocd
```

Since selectors cannot be changed, existing Deployments must be deleted and recreated.

---

## Solution Overview

### Automated via Force=true

The Application manifest uses `Force=true` sync option to automatically:
1. Detect immutable field conflicts
2. Delete existing resources
3. Create new resources with correct configuration

### Sync Options Configuration

For this self-managed demo environment:

| Option | Enabled | Purpose |
|--------|---------|---------|
| `CreateNamespace=true` | ✅ | Ensures argocd namespace exists |
| `Replace=true` | ✅ | Handles large resource specifications |
| `Force=true` | ✅ | Deletes/recreates resources with immutable conflicts |
| `ServerSideApply` | ❌ | **Disabled** - conflicts with Force and causes schema validation errors |

**Why ServerSideApply is disabled:**
- Performs strict schema validation that fails on runtime status fields (e.g., `.status.terminatingReplicas`)
- Force=true uses delete/recreate strategy, making ServerSideApply's incremental patching unnecessary
- These strategies are mutually exclusive - using both causes validation errors
- **For production:** Re-enable ServerSideApply after transition and remove Force=true

---

## Prerequisites

- ✅ Manual Argo CD installation already running
- ✅ Application manifest in Git with `Force=true` sync option
- ✅ Values configured with `configs.secret.createSecret: false` to preserve secrets
- ✅ Git repository access configured

---

## Transition Procedure

### Step 1: Verify Current State

```bash
# Verify Argo CD is running
kubectl get deployments -n argocd

# Check current Deployment selectors (should NOT have instance label)
kubectl get deployment argocd-server -n argocd -o jsonpath='{.spec.selector}'
```

Expected: Selector WITHOUT `app.kubernetes.io/instance`.

### Step 2: Apply Self-Management Application

The `Force=true` sync option automatically handles deletion and recreation.

```bash
# Apply the Application manifest
kubectl apply -f clusters/portcullis/infrastructure/argocd.yaml

# Watch the sync process
kubectl get applications -n argocd argocd -w
```

**Expected behavior:**
1. Argo CD detects selector mismatch
2. Deletes existing Deployments (brief downtime)
3. Creates new Deployments with correct selectors
4. Pods start with correct labels
5. Services select pods successfully

### Step 3: Verify Recreation

```bash
# Watch Deployments being recreated
kubectl get deployments -n argocd -w

# Verify new selectors include instance label
kubectl get deployment argocd-server -n argocd -o jsonpath='{.spec.selector}' | jq
```

Expected output:
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
# Check Services can now select pods
kubectl get endpoints -n argocd

# All services should have endpoints
```

### Step 5: Test Access

```bash
# Port forward
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Or test via Ingress
curl -k https://argocd.portcullis.osow.ski
```

---

## Troubleshooting

### Issue: Force=true Not Working

Based on [GitHub Issue #14910](https://github.com/argoproj/argo-cd/issues/14910), `Force=true` at the Application level may fail for several reasons:

#### Cause 1: Diff Not Detected

Argo CD's diff algorithm may skip immutable fields, preventing OutOfSync status.

**Diagnostic:**
```bash
# Check sync status
kubectl get application argocd -n argocd -o jsonpath='{.status.sync.status}'

# Force hard refresh
argocd app diff argocd --hard-refresh
```

#### Cause 2: Helm Merge Conflicts

Helm's three-way merge may skip immutable fields rather than triggering deletion.

#### Cause 3: Self-Management Protection

Argo CD may protect against deleting its own critical components.

### Solution 1: Resource-Level Annotations

Add sync annotations directly to Deployments via Helm values (more explicit than Application-level).

**Already configured** in `infrastructure/argocd/base/values.yaml`:
```yaml
server:
  deploymentAnnotations:
    argocd.argoproj.io/sync-options: "Force=true,Replace=true"

controller:
  statefulsetAnnotations:
    argocd.argoproj.io/sync-options: "Force=true,Replace=true"

repoServer:
  deploymentAnnotations:
    argocd.argoproj.io/sync-options: "Force=true,Replace=true"
```

**Apply:**
```bash
git add infrastructure/argocd/base/values.yaml
git commit -m "Add resource-level Force annotations"
git push

kubectl patch application argocd -n argocd --type merge -p '{"operation":{"sync":{}}}'
```

### Solution 2: PreSync Hook

Use a Sync Hook to explicitly delete Deployments before sync.

**Available at:** `infrastructure/argocd/base/presync-delete-deployments.yaml`

**One-time manual application:**
```bash
# Apply hook
kubectl apply -f infrastructure/argocd/base/presync-delete-deployments.yaml

# Wait for completion
kubectl wait --for=condition=complete job/argocd-presync-delete-deployments -n argocd --timeout=60s

# Sync
kubectl patch application argocd -n argocd --type merge -p '{"operation":{"sync":{}}}'
```

### Solution 3: Manual Deletion

If automation fails completely:

```bash
# Delete Deployments
kubectl delete deployment -n argocd \
  argocd-server \
  argocd-repo-server \
  argocd-redis \
  argocd-applicationset-controller \
  argocd-notifications-controller \
  argocd-dex-server

# Delete StatefulSet
kubectl delete statefulset argocd-application-controller -n argocd

# Trigger sync
kubectl patch application argocd -n argocd --type merge -p '{"operation":{"sync":{}}}'
```

**Downtime:** 1-3 minutes (Argo CD UI/API only)

### Common Error: ServerSideApply Schema Validation

**Error message:**
```
ComparisonError: Failed to compare desired state to live state: failed to calculate diff:
error calculating structured merge diff: error building typed value from live resource:
.status.terminatingReplicas: field not declared in schema
```

**Cause:**
- `ServerSideApply=true` validates against OpenAPI schema
- Runtime status fields exist in live resources but not in schema
- Conflicts with `Force=true` strategy

**Solution:**
Remove `ServerSideApply=true` from sync options (already done in `clusters/portcullis/infrastructure/argocd.yaml`).

---

## Post-Migration

### What Was Preserved

- ✅ Secrets (argocd-secret, argocd-redis, argocd-notifications-secret)
- ✅ ConfigMaps (now managed by Helm)
- ✅ Services (updated with new selectors)
- ✅ Admin password and authentication

### What Was Recreated

- ✅ Deployments (with new immutable selectors)
- ✅ Pods (with instance labels)
- ✅ StatefulSet for application-controller

### Verification

```bash
# Verify all Deployments have correct selectors
for deploy in argocd-server argocd-repo-server argocd-redis; do
  echo "=== $deploy ==="
  kubectl get deployment $deploy -n argocd -o jsonpath='{.spec.selector.matchLabels}' | jq
done

# Expected: Both app.kubernetes.io/name AND app.kubernetes.io/instance

# Verify Service endpoints
kubectl get endpoints -n argocd | grep argocd-server
```

### Remove Force Annotations

Once migration is complete, **remove Force=true annotations** to prevent unnecessary deletions:

```bash
# Edit infrastructure/argocd/base/values.yaml
# Remove deploymentAnnotations and statefulsetAnnotations with Force=true
git commit -m "Remove Force annotations post-migration"
git push
```

**Why?** Force=true causes deletion/recreation on EVERY sync, wasting resources and causing unnecessary downtime.

### Rollback (if needed)

```bash
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

---

## References

- [Argo CD Sync Options Documentation](https://argo-cd.readthedocs.io/en/latest/user-guide/sync-options/)
- [GitHub Issue #14910 - Replace=true and immutable fields](https://github.com/argoproj/argo-cd/issues/14910)
- [Handle immutable fields in Kubernetes with ArgoCD](https://medium.com/@paolocarta_it/handle-immutable-fields-in-kubernetes-with-argocd-0910253d566e)
- [ArgoCD Self-Management Guide](https://www.teracloud.io/single-post/self-managed-argocd-wait-argocd-can-manage-itself)
