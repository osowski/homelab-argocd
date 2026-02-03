# Architecture Decision Records (ADRs)

This directory contains Architecture Decision Records (ADRs) for the homelab-argocd project.

## What are ADRs?

An Architecture Decision Record (ADR) is a document that captures an important architectural decision made along with its context and consequences. ADRs help us:

- **Document the "why"** behind architectural choices
- **Preserve context** for future maintainers
- **Facilitate discussion** about proposed changes
- **Track evolution** of the system over time

## ADR Format

We follow the format from [adr.github.io](https://adr.github.io/), which includes:

1. **Title and Number**: Sequential numbering with descriptive title
2. **Date**: When the decision was made
3. **Status**: Proposed, Accepted, Deprecated, or Superseded
4. **Context**: What issue motivated this decision
5. **Decision**: What we decided to do
6. **Consequences**: What becomes easier or harder

See [0000-template.md](0000-template.md) for the full template.

## When to Create an ADR

Create an ADR when making decisions that:

- Impact system architecture or design patterns
- Affect future feature implementation or refactoring
- Choose between multiple viable technical approaches
- Introduce new tools, frameworks, or dependencies
- Change integration patterns or data flows

## Current ADRs

| Number | Title | Status |
|--------|-------|--------|
| [0000](0000-template.md) | Template | - |
| [0001](0001-app-of-apps-pattern.md) | Use ArgoCD App of Apps Pattern | Accepted |

## Creating a New ADR

1. Copy the template:
   ```bash
   cp adrs/0000-template.md adrs/NNNN-short-title.md
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

- [Architecture Decision Records (adr.github.io)](https://adr.github.io/)
- [ADR Tools and Examples](https://github.com/joelparkerhenderson/architecture-decision-record)
- [CLAUDE.md - ADR Guidelines](../CLAUDE.md#universal-work-rules)
