# Flux CD Troubleshooting

Expert knowledge for diagnosing and resolving Flux CD reconciliation failures and errors in GitOps environments.

## Keywords

flux, fluxcd, troubleshooting, debug, error, failed, reconciliation, kustomization, helmrelease, source, gitrepository, ocirepository, artifact, health check

## When to Use This Skill

- Flux resources show `Ready: False` status
- Reconciliation errors appear in logs
- Deployments are not syncing from Git
- HelmRelease installations are failing
- Source artifacts are not being fetched
- Image automation is not updating tags

## Related Skills

- [flux-operations](../flux-operations) - Suspend/resume/reconcile operations
- [flux-gitops-patterns](../flux-gitops-patterns) - Architecture and patterns
- [k8s-platform-operations](../k8s-platform-operations) - Incident response procedures

## Quick Reference

| Task | Command |
|------|---------|
| Check Flux health | `flux check` |
| View all errors | `flux logs -A --level=error` |
| Get all status | `flux get all -A` |
| Warning events | `kubectl get events -n flux-system --field-selector type=Warning` |

## Diagnostic Workflow

```
Is the resource Ready?
├─ Yes → Check if correct version/revision deployed
└─ No → Is the source Ready?
    ├─ Yes → Check Kustomization/HelmRelease logs
    │   ├─ dry-run failed → Fix YAML syntax in Git
    │   ├─ health check timeout → Check pod logs/resources
    │   └─ dependency not ready → Fix parent first
    └─ No → Check source credentials/connectivity
        ├─ authentication failed → Update deploy key/secret
        ├─ checkout failed → Verify branch/tag exists
        └─ artifact not found → Check repository URL
```

## Diagnostic Commands

### 1. Check Controller Health
```bash
flux check
```

### 2. View Error Logs
```bash
flux logs --all-namespaces --level=error
```

### 3. Get Warning Events
```bash
kubectl get events -n flux-system --field-selector type=Warning
```

### 4. Inspect Resource Status
```bash
flux get kustomizations -A
flux get helmreleases -A
flux get sources all -A
```

### 5. Controller-Specific Logs
```bash
kubectl logs -n flux-system deploy/source-controller
kubectl logs -n flux-system deploy/kustomize-controller
kubectl logs -n flux-system deploy/helm-controller
kubectl logs -n flux-system deploy/notification-controller
kubectl logs -n flux-system deploy/image-reflector-controller
kubectl logs -n flux-system deploy/image-automation-controller
```

## Error Pattern Reference

| Error Pattern | Cause | GitOps Solution |
|---------------|-------|-----------------|
| `failed to checkout` | Git authentication or network | Verify deploy keys/credentials |
| `dry-run failed: Invalid` | Invalid manifest YAML | Fix syntax in Git repository |
| `health check timeout` | Pods not becoming ready | Check resource limits, images |
| `dependency not ready` | Parent Kustomization failed | Fix upstream dependency first |
| `artifact not found` | Source not synced | Check source status, reconcile |
| `Unsupported value` | Invalid field value | Correct the value in Git |
| `UNAUTHORIZED` | Registry auth failed | Check imagePullSecrets |
| `MANIFEST_UNKNOWN` | OCI tag doesn't exist | Verify tag in registry |

## OCI Repository Troubleshooting

### Common OCI Issues
```bash
# Check OCI repository status
flux get sources oci -A

# View OCI-specific errors
kubectl logs -n flux-system deploy/source-controller | grep -i oci

# Verify registry authentication
kubectl get secret -n flux-system flux-system -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d
```

### OCI Error Patterns
| Error | Cause | Fix |
|-------|-------|-----|
| `UNAUTHORIZED` | Invalid credentials | Update docker-registry secret |
| `MANIFEST_UNKNOWN` | Tag not found | Check tag exists in registry |
| `DENIED` | Permission denied | Check registry permissions |
| `NAME_UNKNOWN` | Repository not found | Verify repository path |

## Image Automation Troubleshooting

### Check Image Policies
```bash
# View image policies
flux get images policy -A

# View image repositories
flux get images repository -A

# Check update status
flux get images update -A
```

### Image Automation Errors
| Error | Cause | Fix |
|-------|-------|-----|
| `no new images found` | Filter too restrictive | Adjust semver/regex pattern |
| `access denied` | Registry auth | Check image pull secrets |
| `failed to push` | Git write access | Check deploy key has write permission |

## Webhook/Notification Debugging

```bash
# Check notification controller
kubectl logs -n flux-system deploy/notification-controller

# View alerts
flux get alerts -A

# View providers
flux get alert-providers -A

# Test webhook receiver
kubectl logs -n flux-system deploy/notification-controller | grep -i receiver
```

## Structured Log Analysis

```json
{
  "level": "error",
  "ts": "2024-01-15T09:36:41.286Z",
  "controllerGroup": "kustomize.toolkit.fluxcd.io",
  "controllerKind": "Kustomization",
  "name": "resource-name",
  "namespace": "namespace",
  "msg": "Reconciliation failed after 2s, next try in 5m0s",
  "revision": "main@sha1:abc123",
  "error": "specific error message"
}
```

## GitOps Principles for Troubleshooting

1. **Never fix directly in cluster** - Identify root cause, fix in Git
2. **Suspend before debugging** - Prevent Flux from reverting test changes
3. **Check the full dependency chain** - Infrastructure before apps
4. **Verify source availability** - Git repos and registries must be accessible
5. **Review recent commits** - Issues often correlate with recent changes

## MCP Tools Available

When the Flux MCP server is connected:
- `mcp__flux-operator-mcp__get_flux_instance` - Get Flux installation status
- `mcp__flux-operator-mcp__get_kubernetes_logs` - Get pod logs
- `mcp__flux-operator-mcp__get_kubernetes_resources` - Query resources
- `mcp__flux-operator-mcp__search_flux_docs` - Search Flux documentation
