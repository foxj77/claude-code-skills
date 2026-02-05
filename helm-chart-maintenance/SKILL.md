---
name: helm-chart-maintenance
description: Use when managing chart versions (semantic versioning), updating chart dependencies, implementing upgrade strategies, writing and running chart tests, setting up CI/CD pipelines for charts, handling deprecation and breaking changes, or publishing charts to OCI registries
---

# Helm Chart Maintenance

Maintain Helm charts including versioning, dependency management, upgrades, testing, and release processes.

## Keywords

helm, chart, maintenance, versioning, semver, semantic versioning, dependencies, upgrading, upgrade, testing, tests, ci/cd, release, releasing, changelog, deprecation, helm-unittest, chart-releaser, managing, updating, publishing

## When to Use This Skill

- Managing chart versions (semantic versioning)
- Updating chart dependencies
- Implementing upgrade strategies
- Writing and running chart tests
- Setting up CI/CD pipelines for charts
- Handling deprecation and breaking changes
- Publishing charts to OCI registries

## Related Skills

- [helm-chart-development](../helm-chart-development) - Creating new charts
- [helm-chart-review](../helm-chart-review) - Quality and security review
- [flux-gitops-patterns](../flux-gitops-patterns) - GitOps deployment patterns

## Quick Reference

| Task | Command |
|------|---------|
| Update dependencies | `helm dependency update mychart/` |
| Run tests | `helm test myrelease` |
| Unit tests | `helm unittest mychart/` |
| Package chart | `helm package mychart/` |
| Push to OCI | `helm push mychart-1.0.0.tgz oci://registry.example.com/charts` |
| Generate docs | `helm-docs --chart-search-root=charts/` |

## Versioning Strategy

### Semantic Versioning for Charts
```
MAJOR.MINOR.PATCH

MAJOR: Breaking changes (incompatible values schema changes)
MINOR: New features (backward compatible)
PATCH: Bug fixes (backward compatible)
```

### Version Bump Guidelines

| Change Type | Version Bump | Example |
|-------------|--------------|---------|
| Breaking values change | MAJOR | `image.name` → `image.repository` |
| Remove deprecated field | MAJOR | Remove `legacyMode` |
| New optional feature | MINOR | Add `metrics.enabled` |
| New template | MINOR | Add `servicemonitor.yaml` |
| Bug fix | PATCH | Fix label selector |
| Documentation | PATCH | Update README |
| Dependency update (minor) | PATCH | PostgreSQL 12.1.0 → 12.1.5 |
| Dependency update (major) | MINOR+ | PostgreSQL 11.x → 12.x |

### Chart.yaml Version Management
```yaml
# Chart version (your release)
version: 2.1.0

# App version (upstream application)
appVersion: "3.5.2"
```

## Dependency Management

### Adding Dependencies
```yaml
# Chart.yaml
dependencies:
  - name: postgresql
    version: "12.x.x"      # Use range for flexibility
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
```

### Dependency Commands
```bash
# Update dependencies
helm dependency update mychart/

# Build dependencies (download to charts/)
helm dependency build mychart/

# List dependencies
helm dependency list mychart/
```

### Chart.lock Management
```yaml
# Chart.lock (auto-generated, commit to git)
dependencies:
- name: postgresql
  repository: https://charts.bitnami.com/bitnami
  version: 12.1.6
digest: sha256:abc123...
generated: "2024-01-15T10:30:00Z"
```

### Dependency Version Ranges
```yaml
# Exact version
version: "12.1.6"

# Patch range (12.1.x)
version: "~12.1.0"

# Minor range (12.x.x)
version: "^12.0.0"

# Greater than
version: ">=12.0.0"

# Range
version: ">=12.0.0 <13.0.0"
```

### Import Values from Dependencies
```yaml
dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: https://charts.bitnami.com/bitnami
    import-values:
      - child: primary.service
        parent: database
      # Or import all
      - child: null
        parent: postgresql
```

## Upgrade Strategies

### Non-Breaking Upgrades
```yaml
# Add new fields with defaults
newFeature:
  enabled: false  # Default to off for existing users
```

### Breaking Changes
```yaml
# 1. Deprecate in MINOR release
# values.yaml
legacyField: ""  # @deprecated Use newField instead

# 2. Add migration helper
{{- if .Values.legacyField }}
{{- fail "legacyField is deprecated, please use newField" }}
{{- end }}

# 3. Remove in MAJOR release
```

### Upgrade Testing
```bash
# Test upgrade from previous version
helm upgrade myrelease mychart/ \
  --dry-run \
  --debug \
  -f old-values.yaml

# Diff changes
helm diff upgrade myrelease mychart/ -f values.yaml
```

## Testing

### Test Connection Template
```yaml
# templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "myapp.fullname" . }}-test-connection"
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  containers:
    - name: wget
      image: busybox
      command: ['wget']
      args: ['{{ include "myapp.fullname" . }}:{{ .Values.service.port }}']
  restartPolicy: Never
```

### Test with Assertions (using helm-unittest)
```yaml
# tests/deployment_test.yaml
suite: deployment tests
templates:
  - deployment.yaml
tests:
  - it: should create deployment with correct replicas
    set:
      replicaCount: 3
    asserts:
      - equal:
          path: spec.replicas
          value: 3

  - it: should use correct image
    set:
      image:
        repository: myapp
        tag: v1.0.0
    asserts:
      - equal:
          path: spec.template.spec.containers[0].image
          value: myapp:v1.0.0

  - it: should have security context
    asserts:
      - isNotNull:
          path: spec.template.spec.securityContext
      - equal:
          path: spec.template.spec.containers[0].securityContext.runAsNonRoot
          value: true

  - it: should fail without required value
    set:
      image.repository: null
    asserts:
      - failedTemplate: {}
```

### Running Tests
```bash
# Helm built-in test
helm test myrelease

# helm-unittest plugin
helm unittest mychart/

# With coverage
helm unittest mychart/ --output-file results.xml --output-type JUnit
```

## CI/CD Integration

### GitHub Actions Workflow
```yaml
name: Helm Chart CI

on:
  push:
    paths:
      - 'charts/**'
  pull_request:
    paths:
      - 'charts/**'

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: azure/setup-helm@v3
      - name: Lint charts
        run: helm lint charts/*

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: azure/setup-helm@v3
      - name: Install helm-unittest
        run: helm plugin install https://github.com/helm-unittest/helm-unittest
      - name: Run tests
        run: helm unittest charts/*

  template:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: azure/setup-helm@v3
      - name: Template charts
        run: |
          for chart in charts/*; do
            helm template test $chart --debug
          done

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Trivy
        uses: aquasecurity/trivy-action@0.28.0
        with:
          scan-type: 'config'
          scan-ref: 'charts/'

  release:
    needs: [lint, test, template, security]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Package and push
        run: |
          helm package charts/*
          # Push to registry
```

### Chart Releaser (cr)
```bash
# Package and upload to GitHub releases
cr package charts/mychart
cr upload --owner org --git-repo charts
cr index --owner org --git-repo charts --push
```

## Documentation

### README Template
```markdown
# MyApp Helm Chart

![Version: 1.0.0](https://img.shields.io/badge/Version-1.0.0-informational)
![AppVersion: 2.3.1](https://img.shields.io/badge/AppVersion-2.3.1-informational)

## Description

A Helm chart for deploying MyApp on Kubernetes.

## Prerequisites

- Kubernetes 1.23+
- Helm 3.10+
- PV provisioner (if persistence enabled)

## Installing

```bash
helm repo add myrepo https://charts.example.com
helm install myrelease myrepo/myapp
```

## Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `replicaCount` | Number of replicas | `1` |
| `image.repository` | Image repository | `myapp` |
| `image.tag` | Image tag | `""` (uses appVersion) |

## Upgrading

### From 1.x to 2.x

Breaking changes:
- `image.name` renamed to `image.repository`
- Minimum Kubernetes version is now 1.23

Migration:
```yaml
# Old (1.x)
image:
  name: myapp

# New (2.x)
image:
  repository: myapp
```
```

### Auto-Generate Docs (helm-docs)
```bash
# Install helm-docs
brew install norwoodj/tap/helm-docs

# Generate README from template
helm-docs --chart-search-root=charts/
```

## Deprecation Process

### 1. Mark Deprecated
```yaml
# values.yaml - add deprecation notice
# @deprecated legacyField - use newField instead
legacyField: ""
```

### 2. Add Warning in Templates
```yaml
{{- if .Values.legacyField }}
{{- $_ := printf "WARNING: legacyField is deprecated and will be removed in v3.0.0. Use newField instead." | fail }}
{{- end }}
```

### 3. Document in CHANGELOG
```markdown
## [2.5.0] - 2024-01-15
### Deprecated
- `legacyField` is deprecated, use `newField` instead (removal in v3.0.0)
```

### 4. Remove in Major Version
```markdown
## [3.0.0] - 2024-06-01
### Removed
- Removed deprecated `legacyField` (use `newField`)
```

## OCI Registry Publishing

### Push to OCI Registry
```bash
# Login to registry
helm registry login registry.example.com

# Package chart
helm package mychart/

# Push to OCI registry
helm push mychart-1.0.0.tgz oci://registry.example.com/charts

# Push with specific tag
helm push mychart-1.0.0.tgz oci://registry.example.com/charts/mychart:1.0.0
```

### Pull from OCI Registry
```bash
# Pull chart
helm pull oci://registry.example.com/charts/mychart --version 1.0.0

# Install directly from OCI
helm install myrelease oci://registry.example.com/charts/mychart --version 1.0.0
```

### GitHub Container Registry (GHCR)
```bash
# Login to GHCR
echo $GITHUB_TOKEN | helm registry login ghcr.io -u USERNAME --password-stdin

# Push chart
helm push mychart-1.0.0.tgz oci://ghcr.io/org/charts

# Reference in Flux
# apiVersion: source.toolkit.fluxcd.io/v1beta2
# kind: OCIRepository
# spec:
#   url: oci://ghcr.io/org/charts/mychart
#   ref:
#     tag: 1.0.0
```

### CI/CD OCI Publishing
```yaml
# GitHub Actions
- name: Push to GHCR
  run: |
    echo "${{ secrets.GITHUB_TOKEN }}" | helm registry login ghcr.io -u ${{ github.actor }} --password-stdin
    helm package charts/mychart
    helm push mychart-*.tgz oci://ghcr.io/${{ github.repository_owner }}/charts
```

## Helm Secrets Plugin

### Installation
```bash
helm plugin install https://github.com/jkroepke/helm-secrets
```

### Encrypt Values
```bash
# Create SOPS config
cat > .sops.yaml <<EOF
creation_rules:
  - path_regex: .*secrets\.yaml$
    age: age1...
EOF

# Encrypt secrets file
helm secrets encrypt values-secrets.yaml

# Decrypt for editing
helm secrets edit values-secrets.yaml
```

### Using Encrypted Values
```bash
# Install with encrypted values
helm secrets install myrelease mychart/ -f values.yaml -f values-secrets.yaml

# Template with secrets
helm secrets template myrelease mychart/ -f values-secrets.yaml
```

## Maintenance Checklist

### Monthly
- [ ] Check for dependency updates
- [ ] Review security advisories
- [ ] Update appVersion if needed
- [ ] Run test suite

### Quarterly
- [ ] Review deprecated features for removal
- [ ] Update minimum Kubernetes version
- [ ] Audit values schema completeness
- [ ] Review and update documentation

### Per Release
- [ ] Update Chart.yaml version
- [ ] Update CHANGELOG.md
- [ ] Run full test suite
- [ ] Test upgrade from previous version
- [ ] Update README if values changed
