---
name: flux-gitops-patterns
description: Use when designing GitOps repository structure, setting up dependency chains between resources, implementing multi-environment deployments, configuring secrets management with Flux, setting up notifications and alerts, or optimizing reconciliation intervals
---

# Flux CD GitOps Patterns

Implement GitOps best practices with Flux CD, including repository structure, dependency management, and deployment patterns.

## Keywords

flux, fluxcd, gitops, patterns, architecture, repository, structure, kustomization, helmrelease, dependencies, multi-tenant, multi-cluster, secrets, sops, designing, creating, deployment, deployments, notification, alerts, alerting, reconciliation, interval

## When to Use This Skill

- Designing GitOps repository structure
- Setting up dependency chains between resources
- Implementing multi-environment deployments
- Configuring secrets management with Flux
- Setting up notifications and alerts
- Optimizing reconciliation intervals

## Related Skills

- [flux-troubleshooting](../flux-troubleshooting) - Diagnosing issues
- [flux-operations](../flux-operations) - Day-to-day operations
- [k8s-platform-tenancy](../k8s-platform-tenancy) - Multi-tenant patterns

## Quick Reference

| Task | Pattern/Command |
|------|----------------|
| Monorepo layout | `clusters/`, `infrastructure/`, `apps/` directories |
| Multi-repo layout | Separate repos per team/concern |
| Component layout | `namespace.yaml`, `repository.yaml`, `release.yaml` |
| Force sync | `flux reconcile ks flux-system --with-source` |
| Variable substitution | `spec.postBuild.substitute` / `substituteFrom` |
| Encrypt secrets | SOPS with `spec.decryption.provider: sops` |
| Set up alerts | `Alert` + `Provider` in `notification.toolkit.fluxcd.io` |

## Repository Patterns

### Monorepo (Recommended for Small Teams)
```
homelab/
├── clusters/
│   ├── production/
│   │   ├── flux-system/
│   │   ├── infrastructure.yaml
│   │   └── apps.yaml
│   └── staging/
│       ├── flux-system/
│       └── ...
├── infrastructure/
│   ├── kustomization.yaml
│   ├── cert-manager/
│   ├── external-dns/
│   └── external-secrets/
├── apps/
│   ├── kustomization.yaml
│   ├── grafana/
│   └── loki/
└── helm/
    └── local-charts/
```

### Multi-Repo (Enterprise)
```
fleet-infra/          # Flux bootstrap, cluster configs
├── clusters/
│   ├── production/
│   └── staging/

platform-components/  # Shared infrastructure
├── cert-manager/
├── ingress-nginx/
└── monitoring/

team-alpha-apps/      # Team-specific apps
├── app1/
└── app2/
```

### Component Structure
```
component-name/
├── namespace.yaml       # Namespace isolation
├── repository.yaml      # HelmRepository source
├── release.yaml         # HelmRelease deployment
└── kustomization.yaml   # Resource orchestration
```

## Dependency Management

### Kustomization Dependencies
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  dependsOn:
    - name: infrastructure
    - name: configs
  interval: 10m
  path: ./apps
  prune: true
  wait: true
  sourceRef:
    kind: GitRepository
    name: flux-system
```

### Deployment Order
```
Flux System (bootstrapped)
  └─> Infrastructure (CRDs, operators)
      ├─> Configs (ConfigMaps, Secrets)
      └─> Monitoring (Prometheus, Grafana)
          └─> Apps (depends on all above)
```

## Source Configuration

### GitRepository
```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 10m
  url: https://github.com/org/repo
  ref:
    branch: main
  secretRef:
    name: flux-system
  ignore: |
    /*
    !/clusters/
    !/infrastructure/
    !/apps/
```

### OCI Repository
```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: OCIRepository
metadata:
  name: podinfo
  namespace: flux-system
spec:
  interval: 10m
  url: oci://ghcr.io/stefanprodan/manifests/podinfo
  ref:
    tag: latest
  provider: generic  # or aws, azure, gcp
```

### HelmRepository
```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: bitnami
  namespace: flux-system
spec:
  interval: 30m
  url: https://charts.bitnami.com/bitnami
```

## Variable Substitution

### Environment Variables
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
spec:
  postBuild:
    substitute:
      CLUSTER_NAME: production
      DOMAIN: example.com
      ENVIRONMENT: prod
    substituteFrom:
      - kind: ConfigMap
        name: cluster-vars
      - kind: Secret
        name: cluster-secrets
```

### Usage in Manifests
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
spec:
  rules:
    - host: app.${DOMAIN}  # Substituted
```

## Secrets Management

### SOPS Integration
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
spec:
  decryption:
    provider: sops
    secretRef:
      name: sops-age  # Contains age private key
```

### External Secrets Pattern
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-secrets
spec:
  refreshInterval: 1h
  secretStoreRef:
    kind: ClusterSecretStore
    name: vault
  target:
    name: app-secrets
    creationPolicy: Owner
  data:
    - secretKey: db-password
      remoteRef:
        key: secret/data/app
        property: password
```

## Notification Patterns

### Provider Configuration
```yaml
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Provider
metadata:
  name: discord
  namespace: flux-system
spec:
  type: discord
  secretRef:
    name: discord-webhook
```

### Alert Configuration
```yaml
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Alert
metadata:
  name: on-call-alerts
  namespace: flux-system
spec:
  providerRef:
    name: discord
  eventSeverity: error
  eventSources:
    - kind: GitRepository
      name: '*'
    - kind: Kustomization
      name: '*'
    - kind: HelmRelease
      name: '*'
      namespace: '*'
```

## Interval Guidelines

| Resource Type | Recommended Interval | Rationale |
|---------------|---------------------|-----------|
| GitRepository | 10m | Balance freshness vs API calls |
| HelmRepository | 30m | Charts update less frequently |
| Kustomization | 10m | Sync with Git interval |
| HelmRelease | 15m | Allow chart fetch first |
| OCIRepository | 10m | Similar to Git |

## GitOps Principles

1. **Declarative** - Desired state in Git, not imperative commands
2. **Versioned** - All changes tracked in Git history
3. **Automated** - Flux applies changes automatically
4. **Auditable** - Git log provides complete audit trail
5. **Self-healing** - Drift automatically corrected

## Anti-Patterns to Avoid

| Anti-Pattern | Problem | Solution |
|--------------|---------|----------|
| Direct kubectl apply | Bypasses GitOps | Always commit to Git |
| Forgotten suspensions | Resources don't update | Document, set reminders |
| Missing dependsOn | Race conditions | Define explicit dependencies |
| Secrets in Git (plaintext) | Security breach | Use SOPS or External Secrets |
| Very short intervals | API throttling | Use 10m+ for most resources |
| Wildcard sources | Security risk | Explicit resource names |
