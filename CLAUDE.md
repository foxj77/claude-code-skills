# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Markdown-based documentation collection** for Claude Code AI skills. It contains curated knowledge for Kubernetes platform engineering, Flux CD GitOps operations, and Helm chart management. Each "skill" is a self-contained documentation package that Claude Code instances can leverage when assisting with DevOps, platform engineering, and GitOps operations.

**This is not a software project** - it is a documentation repository with no build process, tests, or compiled artifacts.

## Repository Structure

```
claude-code-skills/
├── README.md                    # Main project overview and skills index
├── CONTRIBUTING.md              # Contribution guidelines
├── .github/workflows/           # CI/CD for security scanning
│   └── security-scan.yml
├── _shared/                     # Shared reference materials
│   └── references/              # Common patterns referenced by multiple skills
└── [skill-name]/                # Individual skill directories
    ├── SKILL.md                 # Main skill documentation with frontmatter
    └── references/              # Supporting materials (optional)
```

## Skill Structure

Every skill directory follows this standardized format:

**SKILL.md** - The main skill file with:
- **YAML frontmatter** containing `name` and `description` (used by Claude Code for skill activation)
- **Keywords** section - Terms that trigger skill activation
- **When to Use This Skill** - Scenarios where the skill applies
- **Related Skills** - Cross-references to complementary skills
- **Quick Reference** - Common commands/patterns table
- **Detailed Content** - In-depth knowledge and examples

**references/** - Optional supporting materials for additional context

## Development Workflow

There is no build process. Development consists of:

1. **Adding/modifying skills** - Create or edit skill directories following the standard structure
2. **Updating README.md** - Add new skills to the appropriate section in the skills index
3. **Testing content** - Manual verification by using skills with Claude Code
4. **Submitting PRs** - Pull requests trigger automated security scanning

## Key Conventions

### SKILL.md Frontmatter
Every SKILL.md must begin with YAML frontmatter:
```yaml
---
name: skill-name
description: Use when [scenario 1], [scenario 2], or [scenario 3]
---
```

The `description` field is critical - it's what Claude Code uses to determine when to activate the skill.

### Cross-Referencing
- Link to related skills using relative paths: `[skill-name](../skill-name)`
- Link to shared references: `[Shared: Topic](../_shared/references/topic.md)`
- Keep cross-references synchronized when renaming skills

### Quick Reference Tables
Most skills include a "Quick Reference" section with common commands in a table format:
```markdown
| Task | Command |
|------|---------|
| Task description | `command` |
```

### Keywords Section
Include comma-separated terms that uniquely identify the skill's domain:
- Use common variations and synonyms
- Avoid overly generic terms that cause false matches

## Security Considerations

The repository has automated security scanning via GitHub Actions (`.github/workflows/security-scan.yml`) that runs on all PRs and pushes. It scans for:
- Secrets and credentials (TruffleHog)
- Personal information patterns
- API keys and tokens
- Internal/proprietary URLs

## Content Domains

The skills cover three primary domains:

1. **Flux CD GitOps** - Troubleshooting, operations, and architectural patterns
2. **Kubernetes Platform** - Tenancy, operations, security (hardening & redteam), and continual improvement
3. **Helm Charts** - Development, maintenance, and review

Related skills often cross-reference each other, as tasks in one domain frequently require knowledge from another.
