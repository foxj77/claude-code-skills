# Flux CLI Command Reference

## Status Commands

```bash
# Overall Flux health
flux check

# All resources status
flux get all -A

# Specific resource types
flux get kustomizations -A
flux get helmreleases -A
flux get sources git -A
flux get sources helm -A
flux get sources oci -A

# Watch mode
flux get kustomizations -A --watch
```

## Reconciliation Commands

```bash
# Force Kustomization reconcile
flux reconcile kustomization <name> -n <namespace>
flux reconcile kustomization <name> -n <namespace> --with-source

# Force HelmRelease reconcile
flux reconcile helmrelease <name> -n <namespace>
flux reconcile helmrelease <name> -n <namespace> --with-source

# Force source reconcile
flux reconcile source git <name> -n <namespace>
flux reconcile source helm <name> -n <namespace>
```

## Suspend/Resume Commands

```bash
# Suspend
flux suspend kustomization <name> -n <namespace>
flux suspend helmrelease <name> -n <namespace>
flux suspend source git <name> -n <namespace>

# Resume
flux resume kustomization <name> -n <namespace>
flux resume helmrelease <name> -n <namespace>
flux resume source git <name> -n <namespace>
```

## Log Commands

```bash
# All logs
flux logs -A

# Error level only
flux logs -A --level=error

# Follow mode
flux logs -A --follow

# Specific resource
flux logs --kind=Kustomization --name=<name> -n <namespace>
flux logs --kind=HelmRelease --name=<name> -n <namespace>
```

## Export Commands

```bash
# Export resource YAML
flux export kustomization <name> -n <namespace>
flux export helmrelease <name> -n <namespace>
flux export source git <name> -n <namespace>
```

## Diff Commands

```bash
# Preview changes
flux diff kustomization <name> --path <local-path>
```

## Bootstrap Commands

```bash
# GitHub bootstrap
flux bootstrap github \
  --owner=<owner> \
  --repository=<repo> \
  --branch=main \
  --path=./clusters/homelab \
  --personal

# Check prerequisites
flux check --pre
```
