# Architecture

## Overview

This repository implements GitOps using ArgoCD's **App of Apps** pattern. The architecture separates infrastructure components from application workloads, with different RBAC policies for each.

## GitOps Flow

```
Developer commits to Git
         ↓
   GitHub Repository
         ↓
   ArgoCD detects change
         ↓
   ArgoCD syncs to cluster
         ↓
   Kubernetes applies manifests
```

## Directory Structure

### Bootstrap (`bootstrap/`)

The bootstrap Helm chart is the entry point. It creates:
- ArgoCD Project CRDs (infrastructure, workloads)
- Parent Applications (infrastructure-apps, workloads-apps)

**Key files:**
- `Chart.yaml` - Helm chart metadata
- `values.yaml` - Default configuration
- `values-<cluster>.yaml` - Cluster-specific overrides
- `templates/argocd-projects.yaml` - Project definitions
- `templates/infrastructure.yaml` - Infrastructure App of Apps
- `templates/workloads.yaml` - Workloads App of Apps

### ArgoCD Projects (`argocd-projects/`)

Standalone Project definitions for reference. These are also created by the bootstrap chart.

**Projects:**
- `infrastructure` - Can create cluster-scoped resources (CRDs, PVs, etc.)
- `workloads` - Namespace-scoped resources only (Deployments, Services, Ingress, etc.)

### Infrastructure (`infrastructure/`)

Platform infrastructure components deployed before workloads.

**Deployed components:**
- argocd - ArgoCD self-management (manages its own configuration)
- kube-prometheus-stack - Monitoring and alerting
- traefik - Ingress controller

**Future components:**
- cert-manager - TLS certificate management
- longhorn - Persistent storage
- external-dns - DNS automation

**Structure:**
```
infrastructure/<component>/
├── base/              # Base manifests or Helm values
└── overlays/          # Cluster-specific customization
    └── <cluster>/
```

### Workloads (`workloads/`)

User-facing applications and services.

**Structure:**
```
workloads/<app>/
├── base/              # Base Kubernetes manifests
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
└── overlays/          # Cluster-specific overlays
    └── <cluster>/
        ├── kustomization.yaml
        └── *-patch.yaml
```

### Clusters (`clusters/`)

Cluster-specific application instances. Parent Applications watch these directories.

**Structure:**
```
clusters/<cluster>/
├── infrastructure/
│   └── <app>.yaml     # ArgoCD Application CRD
└── workloads/
    └── <app>.yaml     # ArgoCD Application CRD
```

## Application Hierarchy

```
Bootstrap (Helm)
├── ArgoCD Projects
│   ├── infrastructure
│   └── workloads
├── infrastructure-apps (Parent Application)
│   └── clusters/<cluster>/infrastructure/*.yaml
│       └── Individual infrastructure Applications
└── workloads-apps (Parent Application)
    └── clusters/<cluster>/workloads/*.yaml
        └── Individual workload Applications
```

## Sync Policies

### Automated Sync

All applications use automated sync with:
- **Prune**: Remove resources not defined in Git
- **Self-Heal**: Revert manual changes to match Git state

### Sync Options

- `CreateNamespace=true` - Automatically create target namespaces

## RBAC Boundaries

### Infrastructure Project

- **Scope**: Cluster-wide
- **Allowed**: All cluster-scoped and namespace-scoped resources
- **Use case**: Platform components (storage, monitoring, ingress controllers)

### Workloads Project

- **Scope**: Namespace-scoped only
- **Allowed**: Deployments, Services, Ingress, ConfigMaps, Secrets, etc.
- **Denied**: ClusterRoles, CRDs, PersistentVolumes, etc.
- **Use case**: Application workloads

## Naming Conventions

### Hostnames

Pattern: `<service>.<cluster>.<domain>`

Examples:
- `echo.portcullis.osow.ski`
- `grafana.portcullis.osow.ski`

### Application Names

- Use lowercase hyphenated names
- Match the directory name in `workloads/` or `infrastructure/`
- Example: `http-echo`, `kube-prometheus-stack`

### Namespace Names

- Generally match the application name
- Infrastructure components may use standard names (e.g., `longhorn-system`, `monitoring`)

## Tool Choices

### Kustomize

Used for:
- Simple applications with minimal customization
- Applications without upstream Helm charts
- Example: http-echo

**Pros**: Simple, no templating, GitOps-friendly
**Cons**: Limited logic, verbose for complex apps

### Helm

Used for:
- Complex infrastructure components
- Applications with many configuration options
- Components with upstream Helm charts
- Example: kube-prometheus-stack, cert-manager

**Pros**: Rich templating, upstream support, values-based config
**Cons**: More complex, templating can be opaque

## Adding a New Application

See [Adding Applications](adding-applications.md) for detailed instructions.

**Quick steps:**
1. Create base manifests in `workloads/<app>/base/`
2. Create cluster overlay in `workloads/<app>/overlays/<cluster>/`
3. Create ArgoCD Application in `clusters/<cluster>/workloads/<app>.yaml`
4. Commit and push to Git
5. ArgoCD automatically creates and syncs the application

## Multi-Cluster Support

To add a new cluster:
1. Create `bootstrap/values-<cluster>.yaml`
2. Create `clusters/<cluster>/infrastructure/` directory
3. Create `clusters/<cluster>/workloads/` directory
4. Add applications to cluster directories
5. Deploy bootstrap to the new cluster

See [Cluster Onboarding](cluster-onboarding.md) for details.

## Security Considerations

### Secrets Management

**Current**: Secrets are managed manually outside this repository.

**Future**: Consider Sealed Secrets or External Secrets Operator for GitOps-native secret management.

### RBAC

ArgoCD Projects enforce RBAC boundaries:
- Infrastructure project can modify cluster-scoped resources
- Workloads project is restricted to namespace-scoped resources

### Repository Access

- This repository is private
- ArgoCD uses HTTPS with token/password authentication
- Consider using SSH keys or GitHub App for production

## Monitoring and Observability

### ArgoCD UI

Access via port-forward or ingress to view:
- Application sync status
- Resource health
- Sync history and diffs

### kubectl

```bash
# View all applications
kubectl get applications -n argocd

# View application details
kubectl describe application <app-name> -n argocd

# View application logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller
```

## Troubleshooting

### Application Not Syncing

1. Check Application status:
   ```bash
   kubectl get application <app-name> -n argocd
   ```

2. Describe Application for events:
   ```bash
   kubectl describe application <app-name> -n argocd
   ```

3. Check ArgoCD logs:
   ```bash
   kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller
   ```

### Sync Errors

1. View sync status in ArgoCD UI
2. Check resource events in target namespace
3. Verify Kustomize/Helm rendering:
   ```bash
   kubectl kustomize workloads/<app>/overlays/<cluster>/
   # or
   helm template <app> infrastructure/<app>/
   ```

### Parent Application Not Creating Children

1. Verify directory structure matches `path` in parent Application
2. Check that child Application manifests are valid YAML
3. Review parent Application logs for errors

## ArgoCD Self-Management

ArgoCD manages its own deployment and configuration through a dedicated Application manifest. This follows the GitOps principle where ArgoCD:

1. **Initial Bootstrap**: Manually installed via Helm or kubectl (chicken-and-egg requirement)
2. **Self-Management**: ArgoCD Application manifest deploys and manages ArgoCD via the official Helm chart
3. **Declarative Updates**: Configuration changes are made through Git commits, not manual kubectl commands

**Benefits:**
- Consistent GitOps workflow for all infrastructure
- Version-controlled ArgoCD configuration
- Automated updates and rollbacks
- Audit trail for all changes

**Implementation:**
- Helm chart: `argo-cd` from `https://argoproj.github.io/argo-helm`
- Base values: `infrastructure/argocd/base/values.yaml`
- Cluster overlays: `infrastructure/argocd/overlays/<cluster>/values.yaml`
- Application manifest: `clusters/<cluster>/infrastructure/argocd.yaml`
- Sync wave: `5` (early deployment, before other infrastructure)

## Future Enhancements

- ApplicationSets for multi-cluster templating
- Progressive delivery with Argo Rollouts
- Sealed Secrets or External Secrets Operator
- External DNS automation
- Monitoring and alerting for ArgoCD itself
