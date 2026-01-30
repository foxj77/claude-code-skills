# Shared References

This directory contains canonical reference documentation shared across multiple skills to reduce redundancy and ensure consistency.

## Contents

| File | Description | Used By |
|------|-------------|---------|
| [pod-security-context.md](references/pod-security-context.md) | Pod and container security context patterns | k8s-security-hardening, k8s-platform-tenancy, helm-chart-development, helm-chart-review |
| [network-policies.md](references/network-policies.md) | NetworkPolicy patterns for zero-trust | k8s-security-hardening, k8s-platform-tenancy, k8s-security-redteam |
| [rbac-patterns.md](references/rbac-patterns.md) | RBAC roles, bindings, and audit patterns | k8s-security-hardening, k8s-platform-tenancy, k8s-security-redteam |

## Usage

Skills should reference these shared documents rather than duplicating content:

```markdown
## Security Context

For detailed security context configuration, see [Shared: Pod Security Context](../_shared/references/pod-security-context.md).
```

## Contributing

When adding new shared content:
1. Ensure it's referenced by 2+ skills
2. Add to this README
3. Update referencing skills to link here
