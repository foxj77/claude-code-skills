# Provider Authentication Patterns

Canonical reference for cloud provider authentication in Kubernetes controllers. Referenced by multiple skills including external-dns-troubleshooting and external-secrets-troubleshooting.

## Overview

Most Kubernetes controllers that integrate with cloud providers (ExternalDNS, External Secrets Operator, cert-manager, etc.) use the same authentication mechanisms. This reference covers the common patterns and diagnostic steps.

## Authentication Methods by Provider

| Provider | Preferred Method | Fallback Method | Key Indicator |
|----------|-----------------|-----------------|---------------|
| AWS | IRSA (IAM Roles for Service Accounts) | Static credentials Secret | `eks.amazonaws.com/role-arn` annotation on SA |
| Azure | Workload Identity | Service principal secret (`azure.json`) | `azure.workload.identity/client-id` annotation on SA |
| GCP | GKE Workload Identity | Service account key Secret | `iam.gke.io/gcp-service-account` annotation on SA |
| HashiCorp Vault | Kubernetes auth | Token or AppRole | `auth.kubernetes` in SecretStore spec |
| CloudFlare | API token Secret | API key Secret | `cloudflare-api-token` Secret name |

---

## AWS (IRSA)

```bash
# Check IRSA annotation on service account
kubectl get sa ${SA_NAME} -n ${NS} -o jsonpath='{.metadata.annotations.eks\.amazonaws\.com/role-arn}'

# Check for static credentials secret
kubectl get secret ${CRED_SECRET} -n ${NS} 2>/dev/null

# Verify projected token volume (IRSA mounts this automatically)
kubectl get pods -n ${NS} -o jsonpath='{.items[0].spec.volumes[*].name}' | tr ' ' '\n' | grep token

# Log grep for AWS auth errors
kubectl logs -n ${NS} deploy/${DEPLOY_NAME} --tail=200 | grep -iE 'credential|auth|forbidden|access denied|assume role|sts'
```

### Common AWS IAM Issues

| Problem | Symptom | Fix |
|---------|---------|-----|
| IRSA not configured | "no credentials" or uses node role | Add `eks.amazonaws.com/role-arn` annotation to SA |
| Trust policy wrong | "not authorized to perform sts:AssumeRoleWithWebIdentity" | Fix OIDC provider and conditions in IAM trust policy |
| Missing permissions | "AccessDenied" on specific API calls | Add required permissions to the IAM role policy |
| Wrong region | "endpoint not found" or timeout | Set `AWS_DEFAULT_REGION` or controller's region flag |
| Cross-account without assume | "AccessDenied" | Configure cross-account role trust and `sts:AssumeRole` |

---

## Azure (Workload Identity)

```bash
# Check Workload Identity annotation on service account
kubectl get sa ${SA_NAME} -n ${NS} -o jsonpath='{.metadata.annotations.azure\.workload\.identity/client-id}'

# Check for azure.json config (used by some controllers)
kubectl get deploy/${DEPLOY_NAME} -n ${NS} -o json | jq '.spec.template.spec.volumes'

# Check pod labels for Workload Identity injection
kubectl get pods -n ${NS} -o jsonpath='{.items[0].metadata.labels}' | jq .

# Log grep for Azure auth errors
kubectl logs -n ${NS} deploy/${DEPLOY_NAME} --tail=200 | grep -iE 'azure|authorization|subscription|resource group|tenant|client'
```

### Common Azure Issues

| Problem | Symptom | Fix |
|---------|---------|-----|
| Workload Identity not configured | "no credential" or "DefaultAzureCredential failed" | Add `azure.workload.identity/client-id` annotation and federated credential |
| Missing role assignment | "AuthorizationFailed" | Assign required role (e.g., `DNS Zone Contributor`) on the target resource |
| Wrong subscription | Resource not found | Set `--azure-subscription-id` or controller config for correct subscription |
| Tenant ID mismatch | "AADSTS700016" | Verify tenant ID in controller config matches the Azure AD tenant |

---

## GCP (Workload Identity)

```bash
# Check GKE Workload Identity annotation
kubectl get sa ${SA_NAME} -n ${NS} -o jsonpath='{.metadata.annotations.iam\.gke\.io/gcp-service-account}'

# Check for service account key secret
kubectl get deploy/${DEPLOY_NAME} -n ${NS} -o json | jq '.spec.template.spec.volumes[] | select(.secret)'

# Log grep for GCP auth errors
kubectl logs -n ${NS} deploy/${DEPLOY_NAME} --tail=200 | grep -iE 'google|gcp|project|permission|iam|credential'
```

### Common GCP Issues

| Problem | Symptom | Fix |
|---------|---------|-----|
| Workload Identity not bound | "iam.gke.io annotation missing" or uses node SA | Create IAM policy binding between KSA and GSA |
| Missing IAM role | "PERMISSION_DENIED" | Grant required role on the target resource (e.g., `roles/dns.admin`) |
| Wrong project | Resource not found | Set `--google-project` or project ID in controller config |

---

## HashiCorp Vault

```bash
# Check Vault address in config
kubectl get secretstore -n ${NS} -o json | jq '.items[].spec.provider.vault.server'

# Check auth method
kubectl get secretstore -n ${NS} -o json | jq '.items[].spec.provider.vault.auth'

# Log grep for Vault errors
kubectl logs -n ${NS} deploy/${DEPLOY_NAME} --tail=200 | grep -iE 'vault|token|auth|permission|seal|lease'
```

### Common Vault Issues

| Problem | Symptom | Fix |
|---------|---------|-----|
| Vault sealed | "Vault is sealed" | Unseal Vault or check auto-unseal mechanism |
| Token expired | "permission denied" | Rotate token or check Kubernetes auth TTL |
| Wrong mount path | "no handler for route" | Fix `mountPath` in provider spec |
| Wrong KV version | Empty data or wrong structure | Set `version: "v2"` or `version: "v1"` explicitly |
| K8s auth role wrong | "invalid role" | Verify Vault role allows the service account and namespace |

---

## CloudFlare

```bash
# Check for API token secret
kubectl get secret ${TOKEN_SECRET} -n ${NS} 2>/dev/null

# Log grep for CloudFlare errors
kubectl logs -n ${NS} deploy/${DEPLOY_NAME} --tail=200 | grep -iE 'cloudflare|api token|forbidden|zone'
```

### Common CloudFlare Issues

| Problem | Symptom | Fix |
|---------|---------|-----|
| Token missing or invalid | "Authentication error" or 403 | Verify secret exists and contains valid API token |
| Token lacks zone permissions | "forbidden" on specific zone | Edit token permissions in CloudFlare dashboard to include target zone |
| Zone ID vs zone name confusion | Wrong zone targeted | Use zone ID filter or verify zone name matches |

---

## General Debugging Steps

Regardless of provider, follow this order:

1. **Confirm the controller pod is running** — `kubectl get pods -n ${NS}`
2. **Check service account annotations** — Most auth issues stem from missing annotations
3. **Grep controller logs for auth keywords** — `credential|auth|forbidden|denied|permission`
4. **Verify the secret exists** (if using static credentials) — List the secret, never decode
5. **Check provider-side permissions** — Confirm the IAM role, policy, or token has the required access
6. **Test connectivity** — Ensure the pod can reach the provider endpoint (network policies, firewalls)

> **Never decode or display credential secrets.** List secret names and key names only.
