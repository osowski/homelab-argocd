# Code Review Checklist

This checklist helps catch common issues during the planning and implementation phases. Review these items **before** creating a pull request.

## Security

### Secrets and Credentials
- [ ] No secrets, API keys, or tokens are hardcoded in any files
- [ ] All secrets are provided via environment variables
- [ ] Environment variables are documented in README.md prerequisites
- [ ] No `.env`, `.env.local`, or credential files are committed

### File Permissions
- [ ] Scripts containing secrets use restrictive permissions (`0700` owner-only, not `0755`)
- [ ] Cloud-init files or temporary scripts with credentials are deleted after use
- [ ] Use secure deletion (`shred -u`) for files containing sensitive data
- [ ] Secrets in temporary files are cleaned up in error paths too

### Input Validation
- [ ] User input is validated and sanitized
- [ ] No command injection vulnerabilities (especially in shell commands)
- [ ] No SQL injection vectors
- [ ] File paths are validated before use

## Code Quality

### Defensive Programming
- [ ] Array/list access checks length before indexing (e.g., `ipv4[0]` requires `| length > 0` check)
- [ ] JSON parsing has error handling for malformed data
- [ ] External command results are validated before use
- [ ] Race conditions are considered (e.g., VM state vs IP assignment)

### Feature Flags and Conditionals
- [ ] When features can be disabled (e.g., `tailscale_enabled: false`), code handles both states
- [ ] Conditional dependencies are checked before use (e.g., cloud-init file exists before using `--cloud-init`)
- [ ] Default values are provided for optional configurations
- [ ] Feature toggle combinations are tested

### Idempotency
- [ ] Playbooks can be run multiple times without creating duplicates
- [ ] Resource existence is checked before creation
- [ ] Operations are reversible via destroy playbooks

## Documentation

### Required Documentation Updates
When implementing features, update these files in `/docs`:

- [ ] **`docs/architecture.md`** - If changing system design, adding roles, or modifying workflows
- [ ] **`docs/project_spec.md`** - If implementing requirements or changing specifications
- [ ] **`docs/changelog.md`** - Always update with new features, fixes, and changes
- [ ] **`README.md`** - If adding new prerequisites, environment variables, or usage instructions

### Architecture Decision Records
- [ ] Create ADR in `/adrs/` for architectural decisions that impact future development
- [ ] ADRs follow the format from https://adr.github.io/
- [ ] ADRs are referenced in relevant documentation files

## Git and GitHub

### Branch Naming
- [ ] Branch follows format: `feature-<issue-id>/<description>` or `fix-<issue-id>/<description>`
- [ ] Branch is associated with a GitHub Issue
- [ ] Branch is NOT directly on `main`

### Pull Request Description
- [ ] Description accurately reflects what changed (no outdated specs like "20GB" when it's "80GB")
- [ ] Description explains **what** changed and **why**
- [ ] Includes explicit markdown link to GitHub Issue: `[#123](https://github.com/osowski/homelab-ansible/issues/123)`
- [ ] Not just "Closes #123" - use proper markdown link format

### Commits
- [ ] Commit messages clearly state intent and outcome
- [ ] Commits are focused on single logical changes
- [ ] No bleeding of multiple unrelated changes into one commit

## Testing and Verification

### Before Creating PR
- [ ] Run relevant playbooks to verify functionality
- [ ] Check for syntax errors (`ansible-playbook --syntax-check`)
- [ ] Test both success and failure paths
- [ ] Verify idempotency (run playbook twice, second run should show no changes)

### Edge Cases
- [ ] Test with features disabled (e.g., `tailscale_enabled: false`)
- [ ] Test with minimal and maximal configurations
- [ ] Test error handling and cleanup on failure
- [ ] Verify cleanup scripts and destroy playbooks

## Common Pitfalls (from Past Reviews)

These specific issues have been caught in previous code reviews:

1. **Tailscale auth key in world-readable files** - Always use `0700` permissions and secure deletion
2. **Accessing array elements without bounds checking** - Use `| length > 0` before `[0]`
3. **Cloud-init file missing when feature disabled** - Conditionally include flags based on feature state
4. **Documentation not updated** - Always update `/docs` for major features
5. **Branch naming wrong** - Must follow `feature-<id>/` or `fix-<id>/` pattern
6. **PR description inaccurate** - Ensure specs match actual implementation
7. **Race conditions** - VM "Running" state doesn't guarantee IP assignment
8. **MicroK8s ingress controller is Traefik, not NGINX** - Since MicroK8s v1.28+, the default ingress controller is Traefik (service: `traefik` in namespace `ingress`), not NGINX
