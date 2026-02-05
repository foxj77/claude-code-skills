---
name: external-dns-troubleshooting
description: Use when DNS records are not being created or updated for Kubernetes Services or Ingresses, when ExternalDNS logs show errors, when records are stale or orphaned, or when provider-specific sync failures occur
---

# ExternalDNS Troubleshooting

Diagnose and resolve failures in ExternalDNS — the controller that synchronises Kubernetes Service/Ingress annotations to DNS providers (Route53, Azure DNS, CloudFlare, Google Cloud DNS, etc.).

## Keywords

external-dns, externaldns, dns, records, route53, azure-dns, cloudflare, google-cloud-dns, dns-sync, dns-records, a-record, cname, txt-record, annotation, hostname, domain, zone, provider, registry, ownership

## When to Use This Skill

- DNS records are not being created for new Services or Ingresses
- Existing DNS records are stale, pointing to old IPs
- ExternalDNS logs show authentication, permission, or zone errors
- TXT ownership records are missing or conflicting
- Records appear in the log as "planned" but never created
- DNS propagation seems broken after a provider migration
- ExternalDNS is running but doing nothing (no changes detected)

### When NOT to Use

- DNS records exist but TLS certs are failing → use [cert-manager-troubleshooting](../cert-manager-troubleshooting)
- DNS is working but the Service has no endpoints → use [k8s-namespace-troubleshooting](../k8s-namespace-troubleshooting)
- ExternalDNS config is delivered by Flux and not appearing → use [flux-troubleshooting](../flux-troubleshooting)

## Related Skills

- [cert-manager-troubleshooting](../cert-manager-troubleshooting) - TLS certificates depend on DNS
- [kyverno-troubleshooting](../kyverno-troubleshooting) - Policies may block annotations
- [external-secrets-troubleshooting](../external-secrets-troubleshooting) - Provider credentials from external stores
- [k8s-namespace-troubleshooting](../k8s-namespace-troubleshooting) - General namespace diagnosis
- [k8s-platform-operations](../k8s-platform-operations) - Cluster-wide operations
- [flux-troubleshooting](../flux-troubleshooting) - GitOps delivery of ExternalDNS config

## Quick Reference

| Task | Command |
|------|---------|
| Check ExternalDNS pod | `kubectl get pods -n external-dns` |
| View logs | `kubectl logs -n external-dns deploy/external-dns --tail=200` |
| List managed records (dry-run) | `kubectl logs -n external-dns deploy/external-dns \| grep "Desired"` |
| Check source annotations | `kubectl get ingress -A -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}\t{.metadata.annotations.external-dns\.alpha\.kubernetes\.io/hostname}\n{end}'` |
| Find TXT ownership records | `kubectl logs -n external-dns deploy/external-dns \| grep "txt"` |
| Force sync | `kubectl rollout restart deploy/external-dns -n external-dns` |

---

## Diagnostic Workflow

```
DNS record not appearing?
├─ ExternalDNS pod running?
│   ├─ No → Check deployment, image, RBAC (Section 1)
│   └─ Yes → Check logs for errors
│       ├─ Authentication/permission error → Fix provider credentials (Section 2)
│       ├─ Zone not found / filtered out → Fix domain filter config (Section 3)
│       ├─ Source yielded 0 endpoints → Fix annotations/sources (Section 4)
│       ├─ Record planned but not created → Provider API issue (Section 5)
│       └─ No log activity at all → Check interval and RBAC (Section 6)
├─ Record exists but wrong value?
│   ├─ TXT ownership record missing → Ownership conflict (Section 7)
│   └─ Multiple ExternalDNS instances → Registry conflict (Section 7)
└─ Record was deleted unexpectedly?
    └─ Orphaned record cleanup or ownership loss (Section 7)
```

---

## Section 1: Controller Health

```bash
# Pod status
kubectl get pods -n external-dns -o wide
kubectl describe deploy/external-dns -n external-dns

# RBAC — ExternalDNS needs read access to Services, Ingresses, Nodes
kubectl auth can-i list services --as=system:serviceaccount:external-dns:external-dns
kubectl auth can-i list ingresses --as=system:serviceaccount:external-dns:external-dns

# Check arguments and configuration
kubectl get deploy/external-dns -n external-dns -o jsonpath='{.spec.template.spec.containers[0].args}' | tr ',' '\n'
```

### Key Arguments to Verify

| Argument | Purpose | Common Mistake |
|----------|---------|----------------|
| `--source=service,ingress` | What resources to watch | Missing a source type |
| `--domain-filter=example.com` | Restrict to specific domains | Typo in domain or too restrictive |
| `--provider=aws` | DNS provider | Wrong provider name |
| `--policy=upsert-only` | Create/update but never delete | Using `sync` deletes unmanaged records |
| `--registry=txt` | Ownership tracking | Missing registry causes conflicts |
| `--txt-owner-id=my-cluster` | Unique owner per cluster | Duplicate owner IDs across clusters |
| `--interval=1m` | Sync interval | Too long hides issues |

---

## Section 2: Provider Authentication

> **Full provider auth reference:** See [provider-authentication.md](../_shared/references/provider-authentication.md) for generic diagnostic steps, auth method tables, and provider-specific issue matrices for AWS, Azure, GCP, Vault, and CloudFlare.

The commands below are ExternalDNS-specific. For general auth debugging patterns, use the shared reference.

```bash
# Quick auth check — grep ExternalDNS logs for auth failures
kubectl logs -n external-dns deploy/external-dns --tail=200 | grep -iE 'credential|auth|forbidden|access denied|assume role|sts|azure|authorization|cloudflare|google|permission'
```

### Required Provider Permissions (ExternalDNS-Specific)

| Provider | Required Permissions |
|----------|---------------------|
| AWS Route53 | `route53:ChangeResourceRecordSets`, `route53:ListResourceRecordSets`, `route53:ListHostedZones`, `route53:ListHostedZonesByName` |
| Azure DNS | `DNS Zone Contributor` on the zone resource group (or custom role with `Microsoft.Network/dnsZones/*`) |
| CloudFlare | API token with Zone:DNS:Edit on target zones |
| Google Cloud DNS | `roles/dns.admin` on the project (or `dns.changes.create`, `dns.resourceRecordSets.*`, `dns.managedZones.list`) |

---

## Section 3: Domain Filter and Zone Issues

```bash
# Check which domains ExternalDNS is configured to manage
kubectl get deploy/external-dns -n external-dns -o jsonpath='{.spec.template.spec.containers[0].args}' | tr ',' '\n' | grep -i domain

# Check if the target zone exists at the provider
kubectl logs -n external-dns deploy/external-dns --tail=200 | grep -iE 'zone|domain|filter'

# List all zones ExternalDNS can see
kubectl logs -n external-dns deploy/external-dns --tail=200 | grep -i "all hosted zones"
```

| Problem | Symptom in Logs | Fix |
|---------|-----------------|-----|
| Domain filter too strict | "0 zones match" or skipped records | Widen `--domain-filter` or add `--exclude-domains` |
| Zone doesn't exist at provider | "zone not found" | Create the DNS zone at the provider first |
| Zone ID filter wrong | Records in wrong zone | Fix `--zone-id-filter` argument |
| Sub-domain not covered | Parent zone exists but child not matched | Add sub-domain to domain filter or use wildcard |

---

## Section 4: Source and Annotation Issues

```bash
# Check which sources ExternalDNS watches
kubectl get deploy/external-dns -n external-dns -o jsonpath='{.spec.template.spec.containers[0].args}' | tr ',' '\n' | grep source

# Find Services with external-dns annotations
kubectl get svc -A -o json | jq -r '.items[] | select(.metadata.annotations["external-dns.alpha.kubernetes.io/hostname"] != null) | "\(.metadata.namespace)/\(.metadata.name)\t\(.metadata.annotations["external-dns.alpha.kubernetes.io/hostname"])"'

# Find Ingresses with hostnames
kubectl get ingress -A -o json | jq -r '.items[] | select(.spec.rules) | .metadata.namespace + "/" + .metadata.name + "\t" + (.spec.rules[].host // "no-host")'

# Check if source yields endpoints
kubectl logs -n external-dns deploy/external-dns --tail=200 | grep -i "endpoint"
```

### Required Annotations (Services)

| Annotation | Purpose | Example |
|------------|---------|---------|
| `external-dns.alpha.kubernetes.io/hostname` | Target DNS name | `app.example.com` |
| `external-dns.alpha.kubernetes.io/ttl` | Record TTL | `"300"` |
| `external-dns.alpha.kubernetes.io/target` | Override target IP/hostname | `lb.example.com` |

### Service Type Requirements

| Service Type | ExternalDNS Behaviour |
|--------------|----------------------|
| LoadBalancer | Uses `.status.loadBalancer.ingress` for target IP/hostname |
| ClusterIP | Only works if `--publish-internal-services` is set |
| NodePort | Only works if `--service-type-filter=NodePort` and node IPs available |
| ExternalName | Uses `.spec.externalName` as CNAME target |

**Common issue:** LoadBalancer Service has no `.status.loadBalancer.ingress` yet — ExternalDNS cannot determine the target IP. Check if the cloud load balancer provisioned successfully.

---

## Section 5: Record Creation Failures

```bash
# Check for planned vs applied changes
kubectl logs -n external-dns deploy/external-dns --tail=300 | grep -iE 'planned|create|update|delete|change|apply'

# Check for rate limiting
kubectl logs -n external-dns deploy/external-dns --tail=300 | grep -iE 'throttl|rate limit|429|too many'

# Check for provider API errors
kubectl logs -n external-dns deploy/external-dns --tail=300 | grep -iE 'error|fail|invalid|conflict'
```

| Problem | Symptom | Fix |
|---------|---------|-----|
| Planned but not applied | "Creating" in logs, record missing at provider | Check `--dry-run` flag — remove it |
| Rate limited | 429 errors or "rate limit" messages | Increase `--interval`, reduce record count |
| Invalid record | "invalid" or "validation" errors | Check record name/value format for the provider |
| Conflicting record | "conflict" or "already exists" | TXT ownership mismatch — see Section 7 |

---

## Section 6: No Activity

```bash
# Check sync interval
kubectl get deploy/external-dns -n external-dns -o jsonpath='{.spec.template.spec.containers[0].args}' | tr ',' '\n' | grep interval

# Check if watching the right namespace
kubectl get deploy/external-dns -n external-dns -o jsonpath='{.spec.template.spec.containers[0].args}' | tr ',' '\n' | grep namespace

# Check RBAC can list resources
kubectl auth can-i list services -A --as=system:serviceaccount:external-dns:external-dns
kubectl auth can-i list ingresses -A --as=system:serviceaccount:external-dns:external-dns

# Recent log entries (any activity at all?)
kubectl logs -n external-dns deploy/external-dns --tail=50 --since=10m
```

| Cause | Check | Fix |
|-------|-------|-----|
| `--namespace` flag restricts scope | Args show `--namespace=X` | Remove flag for cluster-wide, or add target namespaces |
| RBAC too restrictive | `auth can-i` returns "no" | Update ClusterRole to list services/ingresses/nodes |
| No annotated resources exist | Services/Ingresses lack annotations | Add `external-dns.alpha.kubernetes.io/hostname` annotation |
| Interval too long | Args show `--interval=60m` | Reduce to `1m` or `5m` |

---

## Section 7: Ownership and Registry

ExternalDNS uses TXT records as an ownership registry to track which records it manages. Problems here cause records to be ignored, duplicated, or accidentally deleted.

```bash
# Check registry type and owner ID
kubectl get deploy/external-dns -n external-dns -o jsonpath='{.spec.template.spec.containers[0].args}' | tr ',' '\n' | grep -iE 'registry|owner'

# Look for ownership conflicts in logs
kubectl logs -n external-dns deploy/external-dns --tail=300 | grep -iE 'ownership|registry|txt|skip|ignore|conflict'
```

| Problem | Cause | Fix |
|---------|-------|-----|
| Record exists but ExternalDNS ignores it | TXT record has different owner ID | Update `--txt-owner-id` to match, or delete stale TXT record |
| Multiple clusters fighting over records | Same `--txt-owner-id` on different clusters | Give each cluster a unique owner ID |
| Records deleted unexpectedly | `--policy=sync` removes unmanaged records | Switch to `--policy=upsert-only` unless full sync is intended |
| Orphaned records after cluster deletion | No cleanup ran | Manually delete DNS records and TXT ownership records |
| TXT prefix collision | Default prefix `a-` or `cname-` collides | Set `--txt-prefix` to something unique |

---

## Provider-Specific Gotchas

| Provider | Gotcha | Detail |
|----------|--------|--------|
| **Route53** | Private hosted zones | Need `--aws-zone-type=private` and VPC association |
| **Route53** | Alias records | ExternalDNS creates A records, not aliases — use `--aws-prefer-cname` for CNAME |
| **Azure DNS** | Resource group required | Must specify `--azure-resource-group` |
| **Azure DNS** | Subscription filter | Multi-subscription needs `--azure-subscription-id` |
| **CloudFlare** | Proxy mode | `external-dns.alpha.kubernetes.io/cloudflare-proxied: "true"` for orange cloud |
| **CloudFlare** | Zone ID vs name | API token must have access to the specific zone |
| **Google Cloud DNS** | Project required | Must specify `--google-project` |
| **Google Cloud DNS** | Managed zone name | Zone names differ from domain names |

---

## MCP Tools Available

When the appropriate MCP servers are connected, prefer these over raw kubectl where available:

- `mcp__flux-operator-mcp__get_kubernetes_resources` - Query ExternalDNS deployment, pods, services, ingresses
- `mcp__flux-operator-mcp__get_kubernetes_logs` - Retrieve ExternalDNS pod logs
- `mcp__flux-operator-mcp__get_kubernetes_metrics` - Check ExternalDNS resource consumption

---

## Common Mistakes

| Mistake | Why It Fails | Instead |
|---------|--------------|---------|
| Running `--policy=sync` without understanding it | Deletes DNS records not managed by ExternalDNS, causing outages | Start with `upsert-only`; switch to `sync` only after auditing all existing records |
| Same `--txt-owner-id` across multiple clusters | Clusters overwrite each other's records | Use a unique owner ID per cluster (e.g., `cluster-name-region`) |
| Forgetting `--source=ingress` when using Ingresses | ExternalDNS only watches Services by default in some configs | Explicitly list all source types: `--source=service --source=ingress` |
| Setting `--dry-run` and forgetting to remove it | Records are planned but never created — looks like a provider issue | Check args for `--dry-run` before investigating provider auth |
| Not checking LoadBalancer provisioning | ExternalDNS has no target IP to use | Verify `.status.loadBalancer.ingress` is populated on the Service |
| Debugging DNS propagation before checking ExternalDNS logs | Wastes time on DNS caching when the record was never created | Always check ExternalDNS logs first, then provider console, then DNS propagation |

## Behavioural Guidelines

1. **Check logs first** — ExternalDNS is heavily log-driven; the answer is almost always in the logs.
2. **Verify annotations** — Most "ExternalDNS isn't working" issues are missing or malformed annotations.
3. **Check the provider console** — Confirm whether the record exists at the provider, not just via DNS lookup (caching).
4. **Never expose provider credentials** — List secret names, never decode values.
5. **Understand the policy** — `upsert-only` vs `sync` has dramatically different behaviour. Confirm which is set before troubleshooting deletions.
6. **Check owner IDs in multi-cluster setups** — Ownership conflicts are silent; records just stop updating.
7. **Restart as a last resort** — A rollout restart forces a full sync cycle, but diagnose the root cause first.
