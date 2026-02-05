# Architecture Decision Records (ADRs)

This directory contains Architecture Decision Records (ADRs) specific to the homelab-argocd implementation.

> For the canonical ADR template and cross-cutting architectural decisions that apply across multiple homelab repositories, see [homelab-docs ADRs](https://github.com/osowski/homelab-docs/tree/main/adrs).

## Current ADRs

| Number | Title | Status |
|--------|-------|--------|
| [0001](0001-app-of-apps-pattern.md) | Use ArgoCD App of Apps Pattern | Accepted |

## Creating a New ADR

Copy the [canonical template from homelab-docs](https://github.com/osowski/homelab-docs/blob/main/adrs/0000-template.md):

```bash
curl -o adrs/NNNN-short-title.md https://raw.githubusercontent.com/osowski/homelab-docs/main/adrs/0000-template.md
```

Then fill in the template and submit via pull request. See [homelab-docs README](https://github.com/osowski/homelab-docs/blob/main/README.md#creating-adrs) for detailed guidance.

## References

- [homelab-docs ADRs](https://github.com/osowski/homelab-docs/tree/main/adrs) - Cross-cutting ADRs and canonical template
- [Architecture Decision Records (adr.github.io)](https://adr.github.io/)
- [CLAUDE.md - ADR Guidelines](../CLAUDE.md#universal-work-rules)
