# Architecture Decision Records (ADRs)

This directory contains Architecture Decision Records (ADRs) for the homelab-argocd project.

> For ADR guidelines, format reference, and the canonical ADR template, see [homelab-docs ADRs](https://github.com/osowski/homelab-docs/blob/main/adrs/README.md). Cross-cutting ADRs that apply across multiple homelab repositories are also maintained there.

## Current ADRs

| Number | Title | Status |
|--------|-------|--------|
| [0001](0001-app-of-apps-pattern.md) | Use ArgoCD App of Apps Pattern | Accepted |

## Creating a New ADR

1. Copy the [canonical template from homelab-docs](https://github.com/osowski/homelab-docs/blob/main/adrs/0000-template.md):
   ```bash
   curl -o adrs/NNNN-short-title.md https://raw.githubusercontent.com/osowski/homelab-docs/main/adrs/0000-template.md
   ```

2. Update the number (increment from the last ADR)

3. Fill in:
   - Title
   - Current date
   - Status (usually "Proposed" initially)
   - Context, Decision, Consequences sections

4. Submit for review via pull request

5. Update status to "Accepted" when merged

## References

- [homelab-docs ADR Guidelines](https://github.com/osowski/homelab-docs/blob/main/adrs/README.md) - Canonical ADR format and guidelines
- [Architecture Decision Records (adr.github.io)](https://adr.github.io/)
- [CLAUDE.md - ADR Guidelines](../CLAUDE.md#universal-work-rules)
