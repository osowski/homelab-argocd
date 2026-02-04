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
- kube-prometheus-stack infrastructure implementation
  - Complete monitoring stack with Prometheus, Grafana, Alertmanager
  - Base values with conservative resource limits
  - Portcullis cluster overlay with ingress configuration
  - ServiceMonitor and PrometheusRule CRDs enabled
  - Persistent storage for Prometheus, Grafana, and Alertmanager

### Changed
- Updated `README.md` to reference new Helm deployment guide
- Updated `docs/adding-applications.md` to link to detailed Helm guide

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
