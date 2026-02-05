---
name: cert-manager-troubleshooting
description: Use when TLS certificates are not being issued or renewed, when Certificate or CertificateRequest resources show errors, when ACME challenges fail, or when Issuer or ClusterIssuer resources are not ready
---

# cert-manager Troubleshooting

Diagnose and resolve failures in cert-manager — the controller that automates TLS certificate issuance and renewal from sources like Let's Encrypt (ACME), HashiCorp Vault, Venafi, and self-signed CAs.

## Keywords

cert-manager, certificate, tls, ssl, https, letsencrypt, acme, issuer, clusterissuer, certificaterequest, order, challenge, http01, dns01, self-signed, ca, renew, renewal, expired, not-ready, ingress, annotation

## When to Use This Skill

- Certificate resources show `Ready: False` or `Issuing` for too long
- CertificateRequest is stuck or shows errors
- ACME challenges (HTTP-01 or DNS-01) are failing
- Issuer or ClusterIssuer shows `Ready: False`
- TLS secrets are not being created or contain expired certificates
- Ingress TLS annotations are not triggering certificate creation
- Certificate renewal is not happening before expiry

### When NOT to Use

- DNS records are not being created (even if needed for DNS-01 challenges) → use [external-dns-troubleshooting](../external-dns-troubleshooting)
- Issuer credentials come from an external store and the store is failing → use [external-secrets-troubleshooting](../external-secrets-troubleshooting)
- cert-manager resources are delivered by Flux and not appearing → use [flux-troubleshooting](../flux-troubleshooting)

## Related Skills

- [external-dns-troubleshooting](../external-dns-troubleshooting) - DNS records required for DNS-01 challenges
- [external-secrets-troubleshooting](../external-secrets-troubleshooting) - Issuer credentials from external stores
- [kyverno-troubleshooting](../kyverno-troubleshooting) - Policies may block cert-manager resources
- [k8s-namespace-troubleshooting](../k8s-namespace-troubleshooting) - General namespace diagnosis
- [k8s-platform-operations](../k8s-platform-operations) - Cluster-wide operations
- [flux-troubleshooting](../flux-troubleshooting) - GitOps delivery of cert-manager resources

## Quick Reference

| Task | Command |
|------|---------|
| Check cert-manager pods | `kubectl get pods -n cert-manager` |
| List Certificates | `kubectl get certificate -A` |
| List Issuers | `kubectl get issuer -A && kubectl get clusterissuer` |
| Check CertificateRequests | `kubectl get certificaterequest -A` |
| Check Orders (ACME) | `kubectl get order -A` |
| Check Challenges (ACME) | `kubectl get challenge -A` |
| View cert-manager logs | `kubectl logs -n cert-manager deploy/cert-manager --tail=200` |
| Check cert expiry | `kubectl get secret ${SECRET} -n ${NS} -o jsonpath='{.data.tls\.crt}' \| base64 -d \| openssl x509 -noout -dates` |

---

## Diagnostic Workflow

```
TLS certificate not working?
├─ Certificate resource exists?
│   ├─ No → Check Ingress annotations or create Certificate (Section 4)
│   └─ Yes → Check Certificate status
│       ├─ Ready: True → Certificate issued, check if correct (Section 6)
│       └─ Ready: False → Check CertificateRequest
│           ├─ No CertificateRequest → Issuer reference broken (Section 2)
│           ├─ CertificateRequest pending → Check Issuer health (Section 2)
│           └─ CertificateRequest denied → Check approval/RBAC
├─ Using ACME (Let's Encrypt)?
│   ├─ Order created? → Check Order status
│   │   ├─ Order pending → Check Challenge status (Section 3)
│   │   ├─ Order invalid → All challenges failed
│   │   └─ Order errored → ACME account issue (Section 2)
│   └─ Challenge status?
│       ├─ HTTP-01 pending → Solver pod/ingress issues (Section 3)
│       └─ DNS-01 pending → DNS provider issues (Section 3)
└─ Certificate expired?
    └─ Renewal not triggered → Check renewal config (Section 5)
```

---

## Section 1: Controller Health

```bash
# Check all cert-manager components
kubectl get pods -n cert-manager -o wide

# Controller logs (main component)
kubectl logs -n cert-manager deploy/cert-manager --tail=200

# Webhook logs (validates/converts resources)
kubectl logs -n cert-manager deploy/cert-manager-webhook --tail=100

# CA injector logs (injects CA bundles)
kubectl logs -n cert-manager deploy/cert-manager-cainjector --tail=100

# Check CRDs
kubectl get crd | grep cert-manager

# Check webhook connectivity
kubectl get validatingwebhookconfigurations | grep cert-manager
kubectl get mutatingwebhookconfigurations | grep cert-manager
```

### Controller Issues

| Symptom | Cause | Fix |
|---------|-------|-----|
| Certificate stuck at "Issuing" | Controller not processing | Check controller logs for errors, restart if needed |
| Any cert-manager resource creation fails | Webhook unreachable | Check webhook pod, Service, and network policies |
| "conversion webhook" errors | CRD version mismatch | Ensure cert-manager version matches CRD version |
| Leader election errors | Multiple controllers competing | Check for duplicate installations |

---

## Section 2: Issuer and ClusterIssuer

```bash
# Check Issuer status
kubectl get issuer -n ${NS} -o wide
kubectl describe issuer ${ISSUER_NAME} -n ${NS}

# Check ClusterIssuer status
kubectl get clusterissuer -o wide
kubectl describe clusterissuer ${ISSUER_NAME}

# Check the Certificate's issuer reference
kubectl get certificate ${CERT_NAME} -n ${NS} -o jsonpath='{.spec.issuerRef}'
```

### ACME Issuer (Let's Encrypt)

```bash
# Check ACME account registration
kubectl describe issuer ${ISSUER_NAME} -n ${NS} | grep -A10 "Status:"

# Check ACME account secret
kubectl get secret ${ISSUER_NAME}-account-key -n ${NS} 2>/dev/null

# Verify ACME server URL
kubectl get issuer ${ISSUER_NAME} -n ${NS} -o jsonpath='{.spec.acme.server}'
```

| ACME Server | URL | Use Case |
|-------------|-----|----------|
| Let's Encrypt Production | `https://acme-v02.api.letsencrypt.org/directory` | Production certs |
| Let's Encrypt Staging | `https://acme-staging-v02.api.letsencrypt.org/directory` | Testing (fake certs, higher rate limits) |

| Error | Cause | Fix |
|-------|-------|-----|
| "account not registered" | ACME registration failed | Check email, delete account secret, let cert-manager re-register |
| "invalid account" | Account key mismatch | Delete the account key secret and restart |
| Issuer `Ready: False` | Can't reach ACME server | Check network/proxy, verify server URL |

### CA Issuer

```bash
# Check CA secret exists and has required keys
kubectl get secret ${CA_SECRET} -n ${NS}
kubectl get secret ${CA_SECRET} -n ${NS} -o jsonpath='{.data}' | jq 'keys'
# Must contain: tls.crt, tls.key
```

### Vault Issuer

```bash
# Check Vault connection
kubectl describe issuer ${ISSUER_NAME} -n ${NS} | grep -A20 "Spec:"

# Logs for Vault errors
kubectl logs -n cert-manager deploy/cert-manager --tail=200 | grep -iE 'vault|pki|sign|auth'
```

---

## Section 3: ACME Challenges (HTTP-01 and DNS-01)

```bash
# List all challenges
kubectl get challenge -A -o wide

# Describe failing challenge
kubectl describe challenge ${CHALLENGE_NAME} -n ${NS}

# Check challenge state
kubectl get challenge -A -o json | jq -r '.items[] | "\(.metadata.namespace)/\(.metadata.name)\t\(.spec.type)\t\(.status.state)\t\(.status.reason // "no reason")"'
```

### HTTP-01 Challenge Troubleshooting

```bash
# Check solver pod
kubectl get pods -n ${NS} -l acme.cert-manager.io/http01-solver=true

# Check solver Ingress/Service
kubectl get ingress -n ${NS} -l acme.cert-manager.io/http01-solver=true
kubectl get svc -n ${NS} -l acme.cert-manager.io/http01-solver=true

# Test challenge URL locally
# The URL is: http://<domain>/.well-known/acme-challenge/<token>
kubectl get challenge ${CHALLENGE_NAME} -n ${NS} -o jsonpath='{.spec.token}'
```

| HTTP-01 Problem | Cause | Fix |
|-----------------|-------|-----|
| Solver pod not created | RBAC or class annotation issue | Check cert-manager logs, verify ingress class |
| Solver pod pending | Resource constraints | Check node capacity and resource quotas |
| Challenge URL returns 404 | Ingress not routing to solver | Check ingress class, annotations, and controller |
| Challenge URL unreachable | Firewall blocks port 80 | Open port 80 from internet to ingress controller |
| "wrong status code: 503" | Load balancer health check failing | Solver pod needs time; check LB health check config |
| Challenge stays pending | Let's Encrypt can't reach cluster | Verify domain resolves to cluster ingress IP |

### DNS-01 Challenge Troubleshooting

```bash
# Check which DNS provider is configured
kubectl get issuer ${ISSUER_NAME} -n ${NS} -o json | jq '.spec.acme.solvers[].dns01'

# Check for DNS provider credentials
kubectl get challenge ${CHALLENGE_NAME} -n ${NS} -o json | jq '.spec.solver.dns01'

# Verify TXT record was created
dig TXT _acme-challenge.${DOMAIN} @8.8.8.8

# Check cert-manager DNS provider logs
kubectl logs -n cert-manager deploy/cert-manager --tail=300 | grep -iE 'dns|challenge|present|cleanup|txt'
```

| DNS-01 Problem | Cause | Fix |
|----------------|-------|-----|
| TXT record not created | Provider auth failed | Check DNS provider credentials in Issuer |
| TXT record exists but challenge fails | DNS propagation delay | Increase `--dns01-recursive-nameservers-only` and check nameservers |
| Wrong zone | DNS split-horizon or sub-domain issue | Set `--dns01-recursive-nameservers=8.8.8.8:53` |
| "zone not found" | Provider can't find the DNS zone | Verify zone exists and credentials have access |
| Cleanup failed (TXT not removed) | Provider API error on delete | Manually remove stale `_acme-challenge` TXT records |

### Common DNS-01 Providers

| Provider | Secret Type | Key Fields |
|----------|-------------|------------|
| Route53 | AWS credentials or IRSA | `accessKeyID`, `secretAccessKeySecretRef`, or IRSA annotation |
| Azure DNS | Service principal or Workload Identity | `clientID`, `clientSecretSecretRef`, `subscriptionID`, `resourceGroupName` |
| CloudFlare | API token | `apiTokenSecretRef` |
| Google Cloud DNS | Service account key | `serviceAccountSecretRef`, `project` |

---

## Section 4: Certificate and Ingress Configuration

```bash
# Check Certificate spec
kubectl get certificate ${CERT_NAME} -n ${NS} -o yaml

# Check CertificateRequest chain
kubectl get certificaterequest -n ${NS} -o wide
kubectl describe certificaterequest ${CR_NAME} -n ${NS}

# Check if Ingress annotations trigger cert creation
kubectl get ingress -n ${NS} -o json | jq '.items[] | {name: .metadata.name, annotations: .metadata.annotations, tls: .spec.tls}'
```

### Ingress Annotation Issues

| Annotation | Purpose | Common Mistake |
|------------|---------|----------------|
| `cert-manager.io/issuer` | Namespace Issuer | Using ClusterIssuer name with this annotation |
| `cert-manager.io/cluster-issuer` | ClusterIssuer | Typo in name or using Issuer name |
| `cert-manager.io/issue-temporary-certificate` | Temp cert during issuance | Forgetting this causes TLS errors during initial issuance |
| `cert-manager.io/common-name` | Override CN | Ignored if dnsNames are set |

### Certificate Spec Problems

| Problem | Cause | Fix |
|---------|-------|-----|
| "issuer not found" | Wrong `issuerRef.name` or `issuerRef.kind` | Verify issuer exists and kind matches (Issuer vs ClusterIssuer) |
| "dnsNames required" | No domains specified | Add `dnsNames` to Certificate spec |
| "secret already exists" | Conflict with manually created Secret | Delete existing Secret or change `secretName` |
| CertificateRequest not created | Spec validation failed | Check cert-manager webhook logs |

---

## Section 5: Renewal and Expiry

```bash
# Check certificate dates
kubectl get certificate -A -o json | jq -r '.items[] | "\(.metadata.namespace)/\(.metadata.name)\t\(.status.notAfter)\t\(.status.renewalTime)\tReady=\(.status.conditions[0].status)"'

# Check from the actual TLS secret
kubectl get secret ${SECRET} -n ${NS} -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -dates -subject

# Force renewal
kubectl cert-manager renew ${CERT_NAME} -n ${NS}
# Or annotate to trigger renewal
kubectl annotate certificate ${CERT_NAME} -n ${NS} cert-manager.io/renew-before-expiry=now --overwrite
```

| Problem | Cause | Fix |
|---------|-------|-----|
| Not renewing before expiry | `renewBefore` too short or controller stopped | Check `renewBefore` (default: 2/3 of cert lifetime) |
| Renewed but old cert in Secret | Secret not updated | Delete Secret; cert-manager will recreate |
| Let's Encrypt rate limited | Too many issuances for same domain | Wait (production: 5 certs/week per domain), use staging for testing |
| Wildcard cert not renewing | DNS-01 solver broken | Check DNS-01 challenge flow specifically |

### Let's Encrypt Rate Limits

| Limit | Value | Scope |
|-------|-------|-------|
| Certificates per domain | 50/week | Registered domain (e.g., example.com) |
| Duplicate certificates | 5/week | Same set of domain names |
| Failed validations | 5/hour | Per account, per hostname |
| New registrations | 10/3 hours | Per IP |

---

## Section 6: Verifying Issued Certificates

```bash
# Check certificate details from the K8s Secret
kubectl get secret ${SECRET} -n ${NS} -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -text | head -30

# Verify cert chain
kubectl get secret ${SECRET} -n ${NS} -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl verify -show_chain

# Check cert matches key
CERT_MOD=$(kubectl get secret ${SECRET} -n ${NS} -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -modulus | md5)
KEY_MOD=$(kubectl get secret ${SECRET} -n ${NS} -o jsonpath='{.data.tls\.key}' | base64 -d | openssl rsa -noout -modulus | md5)
echo "Match: $([ "$CERT_MOD" = "$KEY_MOD" ] && echo YES || echo NO)"

# Check if staging cert (not trusted in browsers)
kubectl get secret ${SECRET} -n ${NS} -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -issuer
# Staging issuer contains "(STAGING)" or "Fake LE"
```

| Problem | Cause | Fix |
|---------|-------|-----|
| Browser shows "Not Trusted" | Using Let's Encrypt staging issuer | Switch to production issuer server URL |
| Wrong SANs on cert | `dnsNames` in Certificate don't match Ingress hosts | Update Certificate `dnsNames` to match |
| Cert issued but Ingress uses old one | Ingress controller cached old cert | Restart ingress controller or wait for cache TTL |
| "certificate signed by unknown authority" | CA cert not in client trust store | Add CA cert to trust store or use `ca.crt` from Secret |

---

## MCP Tools Available

When the appropriate MCP servers are connected, prefer these over raw kubectl where available:

- `mcp__flux-operator-mcp__get_kubernetes_resources` - Query Certificates, CertificateRequests, Orders, Challenges, Issuers
- `mcp__flux-operator-mcp__get_kubernetes_logs` - Retrieve cert-manager controller, webhook, and cainjector logs
- `mcp__flux-operator-mcp__get_kubernetes_metrics` - Check cert-manager resource consumption

---

## Common Mistakes

| Mistake | Why It Fails | Instead |
|---------|--------------|---------|
| Testing with Let's Encrypt production | Hits rate limits quickly during debugging | Use `acme-staging-v02.api.letsencrypt.org` until everything works, then switch |
| Wildcard cert with HTTP-01 | HTTP-01 doesn't support wildcards, only specific domains | Use DNS-01 solver for wildcard certificates (`*.example.com`) |
| Forgetting port 80 for HTTP-01 | Let's Encrypt must reach `/.well-known/acme-challenge/` over HTTP | Ensure port 80 is open from the internet to the ingress controller |
| Using `cert-manager.io/issuer` for a ClusterIssuer | Ingress annotation looks for a namespace-scoped Issuer | Use `cert-manager.io/cluster-issuer` annotation instead |
| Deleting and recreating Certificate for renewal | Causes a new ACME order, consuming rate limits | Use `kubectl cert-manager renew` to trigger renewal of existing cert |
| Not checking Order and Challenge resources | Debugging the Certificate when the problem is in the ACME challenge | Follow the chain: Certificate → CertificateRequest → Order → Challenge |

## Behavioural Guidelines

1. **Follow the resource chain** — Certificate → CertificateRequest → Order → Challenge. Diagnose at the deepest failing layer.
2. **Never display raw private key contents** — Examining certificate metadata (dates, SANs, issuer) is safe. Modulus comparison (`openssl rsa -noout -modulus`) for cert/key matching is acceptable, but never print full `tls.key` data.
3. **Check the Issuer first** — If the Issuer is `Ready: False`, no Certificate can be issued.
4. **Distinguish staging from production** — Staging certs are not browser-trusted but have much higher rate limits for testing.
5. **Watch for rate limits** — Let's Encrypt production has strict limits. Check `kubectl describe order` for rate limit messages before retrying.
6. **Verify DNS resolution** — For both HTTP-01 and DNS-01, the domain must resolve correctly. Use `dig` to verify.
7. **Check all cert-manager components** — The controller, webhook, and cainjector all play different roles. A webhook failure blocks resource creation entirely.
