# homelab-argocd

GitOps repository for homelab Kubernetes clusters using ArgoCD.

## Overview

This repository contains the declarative configuration for all applications and infrastructure components deployed to homelab Kubernetes clusters. It implements the **App of Apps** pattern for managing ArgoCD applications.

## Repository Structure

```
homelab-argocd/
├── bootstrap/              # Helm chart for bootstrapping ArgoCD App of Apps
│   ├── Chart.yaml
│   ├── values.yaml         # Default values
│   ├── values-*.yaml       # Cluster-specific values
│   └── templates/
│       ├── argocd-projects.yaml
│       ├── infrastructure.yaml
│       └── workloads.yaml
├── argocd-projects/        # ArgoCD Project CRD definitions
│   ├── infrastructure-project.yaml
│   └── workloads-project.yaml
├── infrastructure/         # Platform infrastructure applications
│   └── (future: cert-manager, longhorn, monitoring)
├── workloads/              # User-facing applications
│   └── http-echo/
│       ├── base/           # Base Kubernetes manifests
│       └── overlays/       # Cluster-specific overlays
│           └── portcullis/
├── clusters/               # Cluster-specific application instances
│   └── portcullis/
│       ├── infrastructure/
│       └── workloads/
│           └── http-echo.yaml
└── docs/                   # Documentation
    ├── architecture.md
    ├── adding-applications.md
    ├── bootstrap-procedure.md
    └── cluster-onboarding.md
```

## Quick Start

### Prerequisites

- Kubernetes cluster with ArgoCD installed
- `kubectl` configured with cluster access
- `helm` CLI installed

### Bootstrap a Cluster

1. Clone this repository:
   ```bash
   git clone https://github.com/osowski/homelab-argocd.git
   cd homelab-argocd
   ```

2. Deploy the bootstrap application:
   ```bash
   helm template bootstrap ./bootstrap/ \
     --values ./bootstrap/values-<cluster-name>.yaml \
     | kubectl apply -f -
   ```

3. Watch ArgoCD sync the applications:
   ```bash
   kubectl get applications -n argocd -w
   ```

### Access ArgoCD UI

```bash
# Port-forward to ArgoCD server
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Navigate to https://localhost:8080 and login with username `admin` and the password from above.

## How It Works

1. **Bootstrap**: The bootstrap Helm chart creates:
   - ArgoCD Project definitions (infrastructure, workloads)
   - Parent Applications (infrastructure-apps, workloads-apps)

2. **Parent Applications**: These App of Apps watch the `clusters/<cluster-name>/` directories and automatically create child Applications.

3. **Child Applications**: Each child Application deploys actual workloads or infrastructure components using Kustomize overlays or Helm charts.

4. **GitOps Flow**: Changes pushed to this repository are automatically synced to the cluster by ArgoCD.

## Documentation

- [Architecture](docs/architecture.md) - Detailed system design and GitOps flow
- [Adding Applications](docs/adding-applications.md) - How to add new applications (Kustomize and Helm overview)
- [Adding Helm Workloads](docs/adding-helm-workloads.md) - Comprehensive guide for Helm-based deployments
- [Bootstrap Procedure](docs/bootstrap-procedure.md) - Detailed bootstrap deployment steps
- [Cluster Onboarding](docs/cluster-onboarding.md) - How to onboard new clusters
- [Architecture Decision Records](adrs/) - Record of architectural decisions

## Current Clusters

- **portcullis** - Primary homelab cluster

## Current Applications

### Workloads
- **http-echo** - Validation service (echo.portcullis.osow.ski)

### Infrastructure
- (Future: cert-manager, longhorn, kube-prometheus-stack)

## Security

- Secrets are managed externally (not committed to this repository)
- ArgoCD Projects enforce RBAC boundaries:
  - `infrastructure` project: Can create cluster-scoped resources
  - `workloads` project: Namespace-scoped resources only

## Related Repositories

- [homelab-ansible](https://github.com/osowski/homelab-argocd) - Infrastructure provisioning and cluster lifecycle management

## License

Private homelab repository
