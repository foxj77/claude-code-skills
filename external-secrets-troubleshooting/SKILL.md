---
name: external-secrets-troubleshooting
description: Use when ExternalSecret or SecretStore resources show errors, when secrets are not syncing from external providers like AWS Secrets Manager, Azure Key Vault, HashiCorp Vault, or GCP Secret Manager, or when secret data is stale or missing
---

# External Secrets Operator Troubleshooting

Diagnose and resolve failures in the External Secrets Operator (ESO) — the controller that synchronises secrets from external stores (AWS Secrets Manager, Azure Key Vault, HashiCorp Vault, GCP Secret Manager, etc.) into Kubernetes Secrets.

## Keywords

external-secrets, externalsecret, secretstore, clustersecretstore, eso, vault, aws-secrets-manager, azure-key-vault, gcp-secret-manager, secret-sync, secret-rotation, provider, authentication, secret-data, pushsecret

## When to Use This Skill

- ExternalSecret resources show `SecretSyncedError` or `Ready: False`
- SecretStore or ClusterSecretStore shows authentication errors
- Kubernetes Secrets are not being created or contain stale data
- Secret rotation is not triggering updates
- PushSecret is failing to write back to external stores
- Application pods fail because expected secrets are missing
- Provider-specific connectivity or permission issues

### When NOT to Use

- Secrets exist but TLS certs are not being issued → use [cert-manager-troubleshooting](../cert-manager-troubleshooting)
- Secret data is correct but the consuming pod fails → use [k8s-namespace-troubleshooting](../k8s-namespace-troubleshooting)
- ESO resources are delivered by Flux and not appearing → use [flux-troubleshooting](../flux-troubleshooting)

## Related Skills

- [kyverno-troubleshooting](../kyverno-troubleshooting) - Policies may block Secret creation
- [cert-manager-troubleshooting](../cert-manager-troubleshooting) - TLS certs stored as secrets
- [external-dns-troubleshooting](../external-dns-troubleshooting) - Shared provider auth patterns
- [kro-troubleshooting](../kro-troubleshooting) - Resource groups may include ExternalSecrets
- [k8s-namespace-troubleshooting](../k8s-namespace-troubleshooting) - General namespace diagnosis
- [k8s-security-hardening](../k8s-security-hardening) - Secrets management and access control
- [flux-troubleshooting](../flux-troubleshooting) - GitOps delivery of ESO resources

## Quick Reference

| Task | Command |
|------|---------|
| Check operator pods | `kubectl get pods -n external-secrets` |
| List ExternalSecrets | `kubectl get externalsecret -A` |
| List SecretStores | `kubectl get secretstore -A && kubectl get clustersecretstore` |
| Check sync status | `kubectl get externalsecret -A -o wide` |
| View operator logs | `kubectl logs -n external-secrets deploy/external-secrets --tail=200` |
| Describe failing secret | `kubectl describe externalsecret NAME -n NS` |

---

## Diagnostic Workflow

```
Secret not appearing in cluster?
├─ ExternalSecret exists?
│   ├─ No → Create the ExternalSecret resource
│   └─ Yes → Check ExternalSecret status
│       ├─ SecretSyncedError → Check SecretStore (Section 2)
│       ├─ InvalidStoreRef → Store name/namespace mismatch (Section 2)
│       └─ Ready: True but Secret wrong → Data mapping issue (Section 4)
├─ SecretStore/ClusterSecretStore healthy?
│   ├─ No → Provider auth issue (Section 3)
│   └─ Yes → Check ExternalSecret spec
│       ├─ Key not found at provider → Wrong path/key (Section 4)
│       ├─ Permission denied → IAM/RBAC at provider (Section 3)
│       └─ Sync interval too long → Adjust refreshInterval
└─ Secret exists but data is stale?
    ├─ refreshInterval too long → Reduce interval
    ├─ Provider value unchanged → Check provider-side value
    └─ Rotation not triggered → Check rotation config (Section 5)
```

---

## Section 1: Operator Health

```bash
# Check ESO controller pods
kubectl get pods -n external-secrets -o wide
kubectl describe deploy/external-secrets -n external-secrets

# Check CRDs are installed
kubectl get crd | grep external-secrets

# Expected CRDs
# clustersecretstores.external-secrets.io
# externalsecrets.external-secrets.io
# secretstores.external-secrets.io
# pushsecrets.external-secrets.io

# Operator logs
kubectl logs -n external-secrets deploy/external-secrets --tail=200
kubectl logs -n external-secrets deploy/external-secrets-webhook --tail=100 2>/dev/null
kubectl logs -n external-secrets deploy/external-secrets-cert-controller --tail=100 2>/dev/null

# Check webhook (common failure point)
kubectl get validatingwebhookconfigurations | grep external-secrets
kubectl get mutatingwebhookconfigurations | grep external-secrets
```

### Webhook Issues

| Symptom | Cause | Fix |
|---------|-------|-----|
| ExternalSecret creation rejected | Webhook cert expired or webhook pod down | Restart cert-controller pod, check webhook TLS |
| Timeout on create/update | Webhook pod unreachable | Check network policies, webhook Service endpoints |
| "connection refused" on admission | Webhook deployment scaled to 0 | Scale up webhook deployment |

---

## Section 2: SecretStore and ClusterSecretStore

```bash
# Check SecretStore status
kubectl get secretstore -n ${NS} -o wide
kubectl describe secretstore ${STORE_NAME} -n ${NS}

# Check ClusterSecretStore status
kubectl get clustersecretstore -o wide
kubectl describe clustersecretstore ${STORE_NAME}

# Check the ExternalSecret's store reference
kubectl get externalsecret ${ES_NAME} -n ${NS} -o jsonpath='{.spec.secretStoreRef}'
```

### Store Reference Problems

| Error | Cause | Fix |
|-------|-------|-----|
| `SecretStore "X" not found` | Store doesn't exist in the namespace | Create SecretStore or use ClusterSecretStore |
| `InvalidStoreRef` | Name or kind mismatch | Verify `secretStoreRef.name` and `secretStoreRef.kind` (SecretStore vs ClusterSecretStore) |
| Store `Ready: False` | Provider auth or connectivity failure | See Section 3 |

---

## Section 3: Provider Authentication

> **Full provider auth reference:** See [provider-authentication.md](../_shared/references/provider-authentication.md) for generic diagnostic steps, auth method tables, and provider-specific issue matrices for AWS, Azure, GCP, Vault, and CloudFlare.

The commands and tables below are ESO-specific. For general auth debugging patterns, use the shared reference.

```bash
# Quick auth check — grep ESO logs for auth failures
kubectl logs -n external-secrets deploy/external-secrets --tail=200 | grep -iE 'credential|auth|forbidden|access denied|sts|azure|keyvault|vault|google|permission|seal'
```

### Required Provider Permissions (ESO-Specific)

| Provider | Required Permissions |
|----------|---------------------|
| AWS Secrets Manager | `secretsmanager:GetSecretValue`, `secretsmanager:ListSecrets` (for `find`), `sts:AssumeRole` (cross-account) |
| Azure Key Vault | Key Vault Secrets User (or Key Vault Reader); Key Vault Certificates User for certs |
| HashiCorp Vault | Policy granting `read` on the secret path; K8s auth role allowing the SA and namespace |
| GCP Secret Manager | `roles/secretmanager.secretAccessor` on the project or secret |

### Vault-Specific Checks (ESO)

ESO's Vault integration has unique failure modes beyond standard auth:

```bash
# Check Vault address and auth method in SecretStore
kubectl get secretstore -n ${NS} -o json | jq '.items[].spec.provider.vault | {server, auth: (.auth | keys)}'
```

| Vault Issue | Symptom | Fix |
|-------------|---------|-----|
| Vault sealed | "Vault is sealed" | Unseal Vault or check auto-unseal |
| Token expired | "permission denied" | Rotate token or check Kubernetes auth config |
| Wrong mount path | "no handler for route" | Fix `mountPath` in SecretStore spec |
| Wrong KV version | Empty data or wrong structure | Set `version: "v2"` or `version: "v1"` in provider spec |
| K8s auth role wrong | "invalid role" | Verify role exists in Vault and allows the service account |

---

## Section 4: ExternalSecret Spec and Data Mapping

```bash
# Check ExternalSecret status and conditions
kubectl get externalsecret ${ES_NAME} -n ${NS} -o yaml

# Check the target Secret exists
kubectl get secret ${TARGET_SECRET} -n ${NS}

# Compare ExternalSecret keys with provider keys
kubectl get externalsecret ${ES_NAME} -n ${NS} -o jsonpath='{.spec.data[*].remoteRef.key}'
```

### Data Mapping Problems

| Error | Cause | Fix |
|-------|-------|-----|
| `key "X" not found` | Remote key/path doesn't exist at provider | Verify exact path at provider console |
| `property "X" not found` | JSON property extraction failed | Check `.remoteRef.property` matches provider JSON structure |
| Secret exists but empty | `decodingStrategy` mismatch | Set `decodingStrategy: Auto` or match provider encoding |
| Wrong encoding in Secret | Base64 double-encoding | Check `data` vs `dataFrom` and encoding settings |
| Template rendering failed | Bad `target.template` syntax | Fix Go template syntax in ExternalSecret spec |

### ExternalSecret Spec Patterns

```yaml
# Single key
spec:
  data:
    - secretKey: password          # Key in the K8s Secret
      remoteRef:
        key: /app/database         # Path at the provider
        property: password         # JSON property (if value is JSON)

# Multiple keys from one provider secret (dataFrom)
spec:
  dataFrom:
    - extract:
        key: /app/database         # All JSON keys become Secret keys

# Template for custom formatting
spec:
  target:
    template:
      data:
        connection: "host={{ .host }} port={{ .port }} user={{ .user }}"
```

---

## Section 5: Refresh and Rotation

```bash
# Check refresh interval
kubectl get externalsecret ${ES_NAME} -n ${NS} -o jsonpath='{.spec.refreshInterval}'

# Check last sync time
kubectl get externalsecret ${ES_NAME} -n ${NS} -o jsonpath='{.status.refreshTime}'

# Check sync conditions
kubectl get externalsecret ${ES_NAME} -n ${NS} -o json | jq '.status.conditions'
```

| Problem | Cause | Fix |
|---------|-------|-----|
| Secret not updating after provider change | `refreshInterval` too long | Reduce to `1m` or `5m` for active development |
| Sync successful but value unchanged | Cached at provider (e.g., Vault lease) | Check provider-side caching/leasing |
| `refreshInterval: 0` | Sync disabled — only runs once | Set a positive interval for continuous sync |

---

## Section 6: PushSecret Issues

```bash
# Check PushSecret status
kubectl get pushsecret -n ${NS} -o wide
kubectl describe pushsecret ${PS_NAME} -n ${NS}

# Logs for PushSecret errors
kubectl logs -n external-secrets deploy/external-secrets --tail=200 | grep -iE 'push|write|put'
```

| Problem | Cause | Fix |
|---------|-------|-----|
| PushSecret `SyncError` | Write permissions missing at provider | Grant write access (e.g., `secretsmanager:PutSecretValue`) |
| "secret already exists" | Conflict with existing provider secret | Set `updatePolicy: Replace` or use a different key |
| PushSecret ignored | CRD not installed or feature not enabled | Verify `pushsecrets.external-secrets.io` CRD exists |

---

## Common Root Cause Patterns

| Symptom | Likely Root Cause | Evidence to Confirm |
|---------|-------------------|---------------------|
| All ExternalSecrets failing | SecretStore auth broken | `kubectl get secretstore -n NS` shows `Ready: False` |
| One ExternalSecret failing | Wrong remote key path | `describe externalsecret` shows "key not found" |
| Secret created but empty | Data mapping or encoding issue | Check `decodingStrategy` and `remoteRef.property` |
| Intermittent sync failures | Provider rate limiting | Operator logs show 429 or throttle errors |
| Secrets stale after provider update | Refresh interval too long | `status.refreshTime` is old |
| Webhook rejection on create | Webhook cert expired | Restart cert-controller, check webhook config |
| All secrets in one namespace failing | Namespace-scoped SecretStore issue | Other namespaces using ClusterSecretStore work |

---

## MCP Tools Available

When the appropriate MCP servers are connected, prefer these over raw kubectl where available:

- `mcp__flux-operator-mcp__get_kubernetes_resources` - Query ExternalSecrets, SecretStores, and Kubernetes Secrets
- `mcp__flux-operator-mcp__get_kubernetes_logs` - Retrieve ESO controller and webhook logs
- `mcp__flux-operator-mcp__get_kubernetes_metrics` - Check ESO resource consumption

---

## Common Mistakes

| Mistake | Why It Fails | Instead |
|---------|--------------|---------|
| Using SecretStore when ClusterSecretStore is needed | Each namespace needs its own SecretStore copy | Use ClusterSecretStore for shared provider access across namespaces |
| Setting `refreshInterval: 0` expecting real-time sync | Disables refresh entirely — secret syncs once | Use a short interval (`1m`) for near-real-time |
| Confusing `data` and `dataFrom` | `data` maps individual keys; `dataFrom` extracts all keys from a remote secret | Use `dataFrom.extract` for JSON secrets with multiple keys |
| Not checking the webhook when ExternalSecrets fail to create | Webhook validation rejects malformed specs silently | Check webhook pod health and admission webhook configs |
| Decoding secrets to verify values | Exposes sensitive data in terminal and agent context | Compare secret key names and lengths only — never decode |
| Missing `property` for JSON secrets | Gets the entire JSON blob as a single value | Set `.remoteRef.property` to extract specific JSON fields |

## Behavioural Guidelines

1. **Check SecretStore health first** — Most ExternalSecret failures trace back to a broken store.
2. **Never decode or display secret values** — Compare key names, data lengths, and sync timestamps instead.
3. **Verify the exact remote path** — Provider paths are case-sensitive and format-specific (e.g., `/secret/data/app` for Vault KV v2 vs `/secret/app` for v1).
4. **Check provider console** — Confirm the secret exists at the provider before debugging ESO.
5. **Watch the operator logs** — ESO logs are detailed and include the exact provider error message.
6. **Namespace matters** — SecretStore is namespace-scoped; ClusterSecretStore is cluster-scoped. Wrong reference kind silently fails.
7. **Restart as a last resort** — Diagnose first, restart the operator only after confirming the config is correct.
