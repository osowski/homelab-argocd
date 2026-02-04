# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- ArgoCD self-management application
  - Helm-based deployment using official argo-cd chart (v7.7.14)
  - Base configuration with resource limits for homelab environments
  - Portcullis cluster overlay with domain-specific settings
  - Ingress enabled with Traefik integration
  - Sync wave 5 for early deployment before other infrastructure
  - ignoreDifferences for webhook CA bundles and auto-generated secrets
  - ArgoCD now manages its own configuration through Git
- Comprehensive Helm deployment guide (`docs/adding-helm-workloads.md`)
  - Detailed walkthroughs for infrastructure and workload applications
  - Real-world examples with cert-manager and Grafana
  - Advanced patterns: multi-value files, sync waves, secrets management
  - Comprehensive troubleshooting section with 10+ common scenarios
  - Testing and validation procedures
  - Production-ready best practices
- **Traefik ingress controller** (sync-wave 10)
  - OCI Helm chart deployment with multi-source pattern
  - Base and cluster-specific values configuration
  - Deployed to monitoring namespace
- **kube-prometheus-stack monitoring** (sync-wave 20)
  - Complete monitoring stack with Prometheus, Grafana, Alertmanager
  - Base values with conservative resource limits
  - Portcullis cluster overlay with ingress configuration
  - ServiceMonitor and PrometheusRule CRDs enabled
  - Persistent storage for Prometheus, Grafana, and Alertmanager
- **Sync wave annotations** for deployment ordering
  - Bootstrap: wave 0
  - Parent applications: waves 1, 100
  - Infrastructure: waves 10-50
  - Workloads: waves 105+
- **Kustomization files** in cluster directories
  - `clusters/<cluster>/infrastructure/kustomization.yaml` lists infrastructure apps
  - `clusters/<cluster>/workloads/kustomization.yaml` lists workload apps
  - Parent Applications monitor these files for discovery

### Changed
- **BREAKING: Bootstrap pattern restructured**
  - Moved from `bootstrap/values-<cluster>.yaml` to `clusters/<cluster>/bootstrap.yaml`
  - Bootstrap now uses inline `valuesObject` for cluster configuration
  - Bootstrap deployed as ArgoCD Application with sync-wave 0
  - Enables per-cluster GitOps management of bootstrap configuration
- **Multi-source Application pattern for Helm charts**
  - Infrastructure applications use `sources` (plural) instead of single `source`
  - First source: upstream Helm chart (OCI or HTTP repository)
  - Second source: values files from Git repository using `ref: values`
  - Values referenced with `$values/infrastructure/<component>/...` pattern
  - Replaced raw GitHub URL pattern with multi-source approach
- **Parent Application naming**
  - Changed from `infrastructure-apps` and `workloads-apps` to `infrastructure` and `workloads`
  - Simplified naming convention for clarity
- **Infrastructure component deployment**
  - Moved from future/placeholder to implemented for traefik and kube-prometheus-stack
  - Added `ServerSideApply=true` sync option for CRDs
  - Added retry policies with exponential backoff
- Updated documentation:
  - `docs/architecture.md` - Bootstrap pattern, multi-source applications, sync waves, current state
  - `docs/adding-applications.md` - Multi-source Helm pattern, sync waves, kustomization updates
  - `docs/bootstrap-procedure.md` - New bootstrap Application deployment procedure
  - `docs/cluster-onboarding.md` - Updated for new bootstrap pattern and kustomization files
  - `README.md` - Reference to new Helm deployment guide

## [0.1.0] - 2025-01-XX

### Added
- Initial GitOps repository structure
- Bootstrap Helm chart implementing App of Apps pattern
- ArgoCD Project definitions (infrastructure, workloads)
- Parent Applications for automatic child Application discovery
- http-echo validation service deployment
- Comprehensive documentation:
  - Architecture overview
  - Application deployment guide
  - Bootstrap procedure
  - Cluster onboarding guide
- Architecture Decision Records (ADRs):
  - ADR-0001: App of Apps pattern selection
- Support for portcullis cluster
- Kustomize base + overlay pattern for applications
- GitOps automation with automated sync, prune, and self-heal

### Security
- ArgoCD RBAC Projects for permission boundaries
- Infrastructure project: Cluster-scoped resource access
- Workloads project: Namespace-scoped resources only
- Secrets excluded from repository (external management)

[Unreleased]: https://github.com/osowski/homelab-argocd/compare/v0.1.0...HEAD
[0.1.0]: https://github.com/osowski/homelab-argocd/releases/tag/v0.1.0
