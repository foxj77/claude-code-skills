# Claude Code Skills Collection

A curated collection of Claude Code skills for Kubernetes platform engineering, GitOps operations, and Helm chart management.

## Overview

These skills provide expert knowledge that Claude Code can leverage when assisting with:

- **Flux CD GitOps** - Troubleshooting, operations, and patterns
- **Kubernetes Platform Engineering** - Multi-tenancy, operations, security, and continual improvement
- **Helm Charts** - Development, maintenance, and review

## Skills Index

### Flux CD GitOps

| Skill | Description |
|-------|-------------|
| [flux-troubleshooting](./flux-troubleshooting/) | Diagnosing and resolving Flux issues |
| [flux-operations](./flux-operations/) | Day-to-day Flux management and operations |
| [flux-gitops-patterns](./flux-gitops-patterns/) | Repository structures and deployment patterns |

### Kubernetes Platform

| Skill | Description |
|-------|-------------|
| [k8s-platform-tenancy](./k8s-platform-tenancy/) | Multi-tenant namespace management |
| [k8s-platform-operations](./k8s-platform-operations/) | Day-2 operations and incident response |
| [k8s-security-hardening](./k8s-security-hardening/) | Security controls and compliance |
| [k8s-security-redteam](./k8s-security-redteam/) | Penetration testing and security assessment |
| [k8s-continual-improvement](./k8s-continual-improvement/) | SLOs, FinOps, and platform maturity |

### Helm Charts

| Skill | Description |
|-------|-------------|
| [helm-chart-development](./helm-chart-development/) | Creating and structuring Helm charts |
| [helm-chart-testing](./helm-chart-testing/) | Writing and analyzing Helm test manifests |
| [helm-chart-maintenance](./helm-chart-maintenance/) | Versioning, testing, and releases |
| [helm-chart-review](./helm-chart-review/) | Security and quality review |

### Shared Resources

| Resource | Description |
|----------|-------------|
| [_shared/references/](/_shared/references/) | Common patterns referenced by multiple skills |

## Installation

### Option 1: Clone to Claude Code Skills Directory

```bash
git clone https://github.com/foxj77/claude-code-skills.git ~/.claude/skills/
```

### Option 2: Add as Additional Working Directory

Add to your Claude Code configuration:
```json
{
  "additionalWorkingDirectories": [
    "/path/to/claude-code-skills"
  ]
}
```

### Option 3: Symlink Individual Skills

```bash
ln -s /path/to/claude-code-skills/flux-troubleshooting ~/.claude/skills/flux-troubleshooting
```

## Skill Structure

Each skill follows a standard structure:

```
skill-name/
├── SKILL.md           # Main skill documentation
└── references/        # Supporting reference materials (optional)
    └── *.md
```

### SKILL.md Format

Every skill includes:
- **Keywords** - Terms that trigger skill activation
- **When to Use** - Scenarios where this skill applies
- **Related Skills** - Cross-references to complementary skills
- **Quick Reference** - Common commands/patterns table
- **Detailed Content** - In-depth knowledge and examples

## Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md) for guidelines on:
- Adding new skills
- Updating existing skills
- Submitting pull requests

## License

This project is licensed under the MIT License - see [LICENSE](./LICENSE) for details.

## Acknowledgments

- [Flux CD Documentation](https://fluxcd.io/docs/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Helm Documentation](https://helm.sh/docs/)
- [Claude Code Documentation](https://docs.anthropic.com/claude-code)
