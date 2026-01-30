# Flux CD Operations

Expert knowledge for managing Flux CD reconciliation operations including suspend, resume, force sync, rollback, and multi-cluster management.

## Keywords

flux, fluxcd, reconcile, suspend, resume, sync, rollback, drift, helmrelease, kustomization, gitops, operations, management, upgrade

## When to Use This Skill

- Need to force immediate reconciliation
- Suspending reconciliation for maintenance
- Rolling back a failed deployment
- Upgrading Flux components
- Managing drift detection
- Multi-cluster Flux operations

## Related Skills

- [flux-troubleshooting](../flux-troubleshooting) - Diagnosing issues
- [flux-gitops-patterns](../flux-gitops-patterns) - Architecture patterns
- [k8s-platform-operations](../k8s-platform-operations) - General cluster operations

## Quick Reference

| Task | Command |
|------|---------|
| Force sync all | `flux reconcile ks flux-system --with-source` |
| Suspend resource | `flux suspend ks <name> -n <ns>` |
| Resume resource | `flux resume ks <name> -n <ns>` |
| Check status | `flux get all -A` |
| View diff | `flux diff ks <name> --path <local>` |

## Reconciliation Management

### Force Immediate Reconciliation
```bash
# Kustomization with source refresh
flux reconcile kustomization <name> -n <namespace> --with-source

# HelmRelease with source refresh
flux reconcile helmrelease <name> -n <namespace> --with-source

# Source only
flux reconcile source git <name> -n <namespace>
flux reconcile source oci <name> -n <namespace>
flux reconcile source helm <name> -n <namespace>
```

### Suspend Reconciliation
```bash
# CLI method (immediate, not persisted in Git)
flux suspend kustomization <name> -n <namespace>
flux suspend helmrelease <name> -n <namespace>
flux suspend source git <name> -n <namespace>
```

**GitOps method (persisted, recommended):**
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: <name>
spec:
  suspend: true
```

### Resume Reconciliation
```bash
flux resume kustomization <name> -n <namespace>
flux resume helmrelease <name> -n <namespace>
```

## Rollback Strategies

### Strategy 1: Git Revert (Recommended)
```bash
# Identify the bad commit
git log --oneline -10

# Revert it
git revert <commit-sha>
git push origin main

# Force reconciliation
flux reconcile kustomization flux-system --with-source
```

### Strategy 2: Version Pin
```yaml
# Pin HelmRelease to specific version
spec:
  chart:
    spec:
      version: "1.2.3"  # Exact working version
```

### Strategy 3: Emergency Helm Rollback
```bash
# 1. Suspend Flux first!
flux suspend helmrelease <name> -n <namespace>

# 2. List Helm history
helm history <release-name> -n <namespace>

# 3. Rollback to previous revision
helm rollback <release-name> <revision> -n <namespace>

# 4. Fix in Git, then resume
flux resume helmrelease <name> -n <namespace>
```

## Drift Detection

### Enable Drift Detection
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
spec:
  driftDetection:
    mode: enabled  # or 'warn'
    ignore:
      - paths: ["/spec/replicas"]
        target:
          kind: Deployment
      - paths: ["/metadata/annotations"]
        target:
          kind: Service
```

### Check for Drift
```bash
# View drift in HelmRelease status
kubectl get helmrelease <name> -n <ns> -o jsonpath='{.status.drift}'

# Manual diff
flux diff kustomization <name> --path ./path/to/local
```

## Flux Upgrade Procedures

### Check Current Version
```bash
flux version
flux check
```

### Upgrade Flux
```bash
# Update CLI
brew upgrade fluxcd/tap/flux

# Upgrade cluster components
flux install --export > flux-system/gotk-components.yaml
git add -A && git commit -m "Upgrade Flux to $(flux version --client)"
git push
```

### Upgrade with Bootstrap
```bash
flux bootstrap github \
  --owner=<org> \
  --repository=<repo> \
  --branch=main \
  --path=./clusters/production \
  --personal
```

## Multi-Cluster Operations

### Switch Context
```bash
# List contexts
kubectl config get-contexts

# Switch context
kubectl config use-context <cluster-name>

# Verify Flux in new context
flux check
```

### Cross-Cluster Source Reference
```yaml
# In target cluster, reference source from management cluster
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: shared-config
spec:
  url: https://github.com/org/shared-config
  ref:
    branch: main
```

## HelmRelease Configuration

### Remediation Settings
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
spec:
  install:
    remediation:
      retries: 3
  upgrade:
    remediation:
      retries: 5
      remediateLastFailure: true
    cleanupOnFail: true
  rollback:
    timeout: 5m
    recreate: false
    force: false
    cleanupOnFail: false
  test:
    enable: true
```

### Disable Health Checks (Debugging)
```yaml
spec:
  install:
    disableWait: true
  upgrade:
    disableWait: true
```

## GitOps Best Practices

1. **Prefer Git operations** - Rollbacks via git revert maintain audit trail
2. **Always resume suspended resources** - Don't leave resources suspended indefinitely
3. **Use --with-source** - Ensures source is refreshed during reconciliation
4. **Check dependsOn chains** - Child resources need parents reconciled first
5. **Document suspensions** - Note why and when in commit messages

## MCP Tools Available

- `mcp__flux-operator-mcp__reconcile_flux_kustomization`
- `mcp__flux-operator-mcp__reconcile_flux_helmrelease`
- `mcp__flux-operator-mcp__reconcile_flux_source`
- `mcp__flux-operator-mcp__suspend_flux_reconciliation`
- `mcp__flux-operator-mcp__resume_flux_reconciliation`
