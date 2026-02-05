---
name: kyverno-troubleshooting
description: Use when Kyverno policies are blocking resource creation, when ClusterPolicy or Policy resources show errors, when audit results are unexpected, when mutations are not applying, when generate rules fail, or when admission webhook errors occur
---

# Kyverno Troubleshooting

Diagnose and resolve failures in Kyverno — the Kubernetes-native policy engine that validates, mutates, generates, and cleans up resources using policies written as Kubernetes resources (no new language to learn).

## Keywords

kyverno, policy, clusterpolicy, validate, mutate, generate, cleanup, admission, webhook, deny, block, enforce, audit, policyreport, clusterpolicyreport, rule, match, exclude, preconditions, background, policy-exception

## When to Use This Skill

- Resource creation is blocked with "admission webhook denied the request"
- ClusterPolicy or Policy resources show errors or are not applied
- Mutations are not being applied to resources as expected
- Generate rules are not creating expected resources
- Policy reports show unexpected violations
- Webhook timeouts or failures are disrupting cluster operations
- Policies work in `Audit` mode but fail in `Enforce` mode
- Background scan results are missing or incomplete

## Related Skills

- [kro-troubleshooting](../kro-troubleshooting) - Policies may block kro-created resources
- [cert-manager-troubleshooting](../cert-manager-troubleshooting) - Policies may affect certificate resources
- [external-secrets-troubleshooting](../external-secrets-troubleshooting) - Policies may block secret creation
- [external-dns-troubleshooting](../external-dns-troubleshooting) - Policies may restrict annotations
- [k8s-namespace-troubleshooting](../k8s-namespace-troubleshooting) - General namespace diagnosis
- [k8s-security-hardening](../k8s-security-hardening) - Security policies and Pod Security Standards
- [k8s-platform-tenancy](../k8s-platform-tenancy) - Tenant isolation policies
- [flux-troubleshooting](../flux-troubleshooting) - GitOps delivery of Kyverno policies

## Quick Reference

| Task | Command |
|------|---------|
| Check Kyverno pods | `kubectl get pods -n kyverno` |
| List ClusterPolicies | `kubectl get clusterpolicy` |
| List Policies | `kubectl get policy -A` |
| Check policy status | `kubectl get clusterpolicy -o wide` |
| View policy reports | `kubectl get policyreport -A` |
| View cluster policy reports | `kubectl get clusterpolicyreport` |
| Check admission logs | `kubectl logs -n kyverno deploy/kyverno-admission-controller --tail=200` |
| Check background logs | `kubectl logs -n kyverno deploy/kyverno-background-controller --tail=200` |
| Test policy dry-run | `kubectl apply -f resource.yaml --dry-run=server` |

---

## Diagnostic Workflow

```
Resource blocked by Kyverno?
├─ Identify which policy blocked it
│   ├─ Error message contains policy name → Check that policy (Section 2)
│   └─ Generic webhook error → Check webhook health (Section 1)
├─ Policy exists but not enforcing?
│   ├─ Policy status Ready: False → Spec error (Section 2)
│   ├─ validationFailureAction: Audit → Only reports, doesn't block (Section 3)
│   └─ match/exclude misconfigured → Wrong resource selection (Section 3)
├─ Mutation not applying?
│   ├─ Mutate rule exists → Check match criteria and patch (Section 4)
│   └─ Order matters → Check policy ordering (Section 4)
├─ Generate rule not creating resources?
│   ├─ Trigger resource matched? → Check match/preconditions (Section 5)
│   └─ RBAC sufficient? → Kyverno SA needs create permission (Section 5)
└─ Webhook causing cluster issues?
    └─ Timeouts or failOpen config (Section 7)
```

---

## Section 1: Controller Health

```bash
# Check all Kyverno components
kubectl get pods -n kyverno -o wide

# Kyverno v1.10+ has multiple controllers:
# - kyverno-admission-controller (validates/mutates on admission)
# - kyverno-background-controller (background scans, generate, cleanup)
# - kyverno-reports-controller (policy reports)
# - kyverno-cleanup-controller (cleanup policies)

# Controller logs
kubectl logs -n kyverno deploy/kyverno-admission-controller --tail=200
kubectl logs -n kyverno deploy/kyverno-background-controller --tail=200
kubectl logs -n kyverno deploy/kyverno-reports-controller --tail=100

# Check CRDs
kubectl get crd | grep kyverno

# Check webhook configurations
kubectl get validatingwebhookconfigurations | grep kyverno
kubectl get mutatingwebhookconfigurations | grep kyverno

# Check webhook endpoints
kubectl get endpoints -n kyverno
```

### Component Issues

| Component | Symptom | Fix |
|-----------|---------|-----|
| Admission controller down | All resource creates/updates fail or bypass policy | Scale up; check OOM/CrashLoop |
| Background controller down | Reports not generated, generate rules don't fire | Restart; check RBAC for background operations |
| Reports controller down | Missing PolicyReport resources | Restart; may need significant memory for large clusters |
| Webhook cert expired | "x509: certificate has expired" on admission | Delete webhook secret, restart admission controller to regenerate |
| All Kyverno pods OOM | Large policy set or high cluster activity | Increase memory limits; reduce policy scope with match/exclude |

---

## Section 2: Policy Status and Validation

```bash
# Check ClusterPolicy status
kubectl get clusterpolicy -o wide
kubectl describe clusterpolicy ${POLICY_NAME}

# Check Policy status (namespace-scoped)
kubectl get policy -n ${NS} -o wide
kubectl describe policy ${POLICY_NAME} -n ${NS}

# Check for policy spec errors
kubectl get clusterpolicy ${POLICY_NAME} -o json | jq '.status'

# List all rules in a policy
kubectl get clusterpolicy ${POLICY_NAME} -o json | jq '.spec.rules[].name'
```

### Policy Validation Errors

| Error | Cause | Fix |
|-------|-------|-----|
| "rule is not valid" | Invalid rule spec | Check YAML syntax, pattern structure, and field names |
| "path not found" | `match` or `mutate` references a field that doesn't exist | Verify the resource schema has the referenced path |
| "invalid pattern" | Pattern value doesn't match expected type | Field types must match (string where string expected) |
| "conflict with existing policy" | Two policies with conflicting mutations | Resolve conflict or set policy ordering |
| Policy `Ready: False` | Compilation or validation failure | Check `describe` for the specific error message |
| "unknown field" | Kyverno version doesn't support the field | Check Kyverno version compatibility |

---

## Section 3: Validate Rules — Audit vs Enforce

```bash
# Check validation failure action
kubectl get clusterpolicy ${POLICY_NAME} -o jsonpath='{.spec.validationFailureAction}'

# Check per-rule overrides
kubectl get clusterpolicy ${POLICY_NAME} -o json | jq '.spec.rules[] | {name: .name, action: .validate.validationFailureAction}'

# Check policy reports for audit violations
kubectl get policyreport -n ${NS} -o json | jq '.items[].results[] | select(.result == "fail") | {policy: .policy, rule: .rule, resource: .resources[0].name, message: .message}'

# Check cluster-level violations
kubectl get clusterpolicyreport -o json | jq '.items[].results[] | select(.result == "fail") | {policy: .policy, rule: .rule, resource: .resources[0].name}'
```

### Audit vs Enforce

| Mode | Behaviour | Use Case |
|------|-----------|----------|
| `Audit` | Allows the resource but reports violation | Testing policies before enforcing |
| `Enforce` | Blocks the resource if it violates the policy | Production enforcement |

| Problem | Cause | Fix |
|---------|-------|-----|
| Policy not blocking anything | `validationFailureAction: Audit` | Change to `Enforce` when ready |
| Policy blocks everything | Match criteria too broad | Add `exclude` rules for system namespaces, kyverno, kube-system |
| Policy blocks some resources unexpectedly | `match` includes resources you didn't intend | Add `exclude` for the exempted resources or namespaces |
| Violations in report but resource exists | Audit mode — resource was allowed | Switch to Enforce to actually block |

### Match and Exclude Patterns

```yaml
# Common pattern: enforce on all namespaces except system ones
spec:
  rules:
    - name: check-labels
      match:
        any:
          - resources:
              kinds:
                - Pod
      exclude:
        any:
          - resources:
              namespaces:
                - kube-system
                - kyverno
                - cert-manager
```

---

## Section 4: Mutate Rules

```bash
# Check if mutation is configured
kubectl get clusterpolicy ${POLICY_NAME} -o json | jq '.spec.rules[] | select(.mutate) | {name: .name, mutate: .mutate}'

# Check if mutation was applied (compare resource against expected)
kubectl get ${RESOURCE_TYPE} ${NAME} -n ${NS} -o yaml | grep -A5 "expected-field"

# Check mutation webhook
kubectl get mutatingwebhookconfigurations kyverno-resource-mutating-webhook-cfg -o yaml

# Logs for mutation errors
kubectl logs -n kyverno deploy/kyverno-admission-controller --tail=200 | grep -iE 'mutate|patch'
```

### Mutation Types

| Type | Use Case | Common Issue |
|------|----------|--------------|
| `patchStrategicMerge` | Add/modify fields using K8s strategic merge | Target path doesn't exist at admission time |
| `patchesJson6902` | JSON patch operations (add, remove, replace) | Wrong path or operation type |
| `foreach` | Mutate items in arrays (containers, volumes) | Iterator variable scoping |

### Mutation Problems

| Problem | Cause | Fix |
|---------|-------|-----|
| Mutation not applied | Match criteria don't select the resource | Check `match.resources.kinds`, namespaces, selectors |
| Mutation partially applied | Multiple rules, one failed | Check each rule independently |
| Mutation conflicts with user values | Strategic merge overwrites user fields | Use `patchesJson6902` with `add` (not `replace`) for conditional patches |
| Order-dependent mutations | Policy A's mutation needed before Policy B validates | Set `webhookConfiguration` ordering or combine into one policy |
| Mutation applied but reverted | Another controller (e.g., Flux) reconciles the original | Apply mutation at source (Git) instead of admission |

---

## Section 5: Generate Rules

```bash
# Check generate rules
kubectl get clusterpolicy ${POLICY_NAME} -o json | jq '.spec.rules[] | select(.generate) | {name: .name, kind: .generate.kind}'

# Check background controller (handles generate)
kubectl logs -n kyverno deploy/kyverno-background-controller --tail=200 | grep -iE 'generate|trigger'

# Check if generated resources exist
kubectl get ${GENERATED_KIND} -n ${TARGET_NS}

# Check RBAC for generated resource type
kubectl auth can-i create ${GENERATED_KIND} --as=system:serviceaccount:kyverno:kyverno-background-controller -n ${TARGET_NS}
```

### Generate Rule Patterns

```yaml
# Generate NetworkPolicy when namespace is created
spec:
  rules:
    - name: default-deny
      match:
        any:
          - resources:
              kinds:
                - Namespace
      generate:
        synchronize: true       # Keep in sync (update if policy changes)
        apiVersion: networking.k8s.io/v1
        kind: NetworkPolicy
        name: default-deny
        namespace: "{{request.object.metadata.name}}"
        data:
          spec:
            podSelector: {}
            policyTypes:
              - Ingress
              - Egress
```

### Generate Problems

| Problem | Cause | Fix |
|---------|-------|-----|
| Generated resource not created | Background controller RBAC insufficient | Grant create/update permissions for the generated resource type |
| Trigger matched but nothing generated | Preconditions not met | Check `preconditions` in the rule |
| Generated resource deleted | `synchronize: true` and policy was removed | Resources are cleaned up when policy/trigger is removed if sync is on |
| Re-creation loop | Generated resource conflicts with another controller | Disable `synchronize` or ensure no other controller manages the resource |
| Works for new resources but not existing | Generate only triggers on admission events | Run a background scan or use a CronJob policy |

---

## Section 6: Policy Exceptions

```bash
# List policy exceptions
kubectl get policyexception -A

# Check if an exception applies
kubectl get policyexception -A -o json | jq '.items[] | {name: .metadata.name, namespace: .metadata.namespace, policies: [.spec.exceptions[].policyName], match: .spec.match}'
```

| Problem | Cause | Fix |
|---------|-------|-----|
| Exception not working | Feature not enabled | Set `--enablePolicyException=true` on admission controller |
| Exception too broad | Exempts more resources than intended | Narrow `match` criteria in the PolicyException |
| Exception ignored | PolicyException namespace doesn't match | Ensure exception is in the correct namespace |

---

## Section 7: Webhook Issues

```bash
# Check webhook configuration
kubectl get validatingwebhookconfigurations kyverno-resource-validating-webhook-cfg -o yaml
kubectl get mutatingwebhookconfigurations kyverno-resource-mutating-webhook-cfg -o yaml

# Check webhook failure policy
kubectl get validatingwebhookconfigurations kyverno-resource-validating-webhook-cfg -o jsonpath='{.webhooks[*].failurePolicy}'

# Check webhook timeouts
kubectl get validatingwebhookconfigurations kyverno-resource-validating-webhook-cfg -o jsonpath='{.webhooks[*].timeoutSeconds}'

# Check namespace exclusions
kubectl get validatingwebhookconfigurations kyverno-resource-validating-webhook-cfg -o json | jq '.webhooks[].namespaceSelector'
```

### Webhook Failure Modes

| `failurePolicy` | Behaviour | Risk |
|------------------|-----------|------|
| `Fail` (default) | Reject resource if webhook unreachable | Cluster operations blocked if Kyverno is down |
| `Ignore` | Allow resource if webhook unreachable | Policies bypassed during Kyverno outage |

### Webhook Problems

| Problem | Cause | Fix |
|---------|-------|-----|
| Webhook timeout (30s default) | Complex policies or high load | Increase timeout, optimise policies, scale Kyverno replicas |
| "connection refused" | Admission controller pod down | Restart pod; check resource limits |
| All creates blocked | Webhook `Fail` + Kyverno down | Scale up Kyverno or temporarily set `Ignore` (security risk) |
| Webhook blocks Kyverno itself | No exclusion for kyverno namespace | Add namespace exclusion: `kyverno` to webhook |
| Webhook blocks kube-system | No exclusion for system namespaces | Exclude `kube-system`, `kube-node-lease`, `kube-public` |
| cert error on webhook | TLS certificate expired | Delete webhook secret in kyverno namespace, restart admission controller |

### Emergency: Disable Kyverno Webhooks

```bash
# If Kyverno is blocking all cluster operations and can't recover:
# WARNING: This disables ALL Kyverno policy enforcement

# Option 1: Delete webhook configs (Kyverno recreates on restart)
kubectl delete validatingwebhookconfigurations kyverno-resource-validating-webhook-cfg
kubectl delete mutatingwebhookconfigurations kyverno-resource-mutating-webhook-cfg

# Option 2: Scale down Kyverno (webhooks timeout, failurePolicy takes effect)
kubectl scale deploy -n kyverno --all --replicas=0

# Then fix the underlying issue and scale back up
kubectl scale deploy -n kyverno --all --replicas=1
```

---

## Policy Testing

```bash
# Dry-run a resource against admission policies
kubectl apply -f resource.yaml --dry-run=server

# Use Kyverno CLI for offline testing
kyverno apply policy.yaml --resource resource.yaml

# Test all policies against a resource
kyverno apply /path/to/policies/ --resource resource.yaml

# Generate policy report for existing resources
kyverno apply policy.yaml --cluster
```

---

## Common Mistakes

| Mistake | Why It Fails | Instead |
|---------|--------------|---------|
| Enforcing policies without testing in Audit first | Blocks legitimate workloads in production | Deploy with `Audit`, review PolicyReports, then switch to `Enforce` |
| Not excluding system namespaces | Blocks kube-system, kyverno, or cert-manager operations | Always exclude `kube-system`, `kyverno`, and infrastructure namespaces |
| Broad match with no exclude | Every resource in the cluster matches | Use specific `kinds`, `namespaces`, or `selector` in `match` |
| Debugging by reading policy YAML alone | Policy may be correct but match criteria wrong | Use `--dry-run=server` or Kyverno CLI to test actual behaviour |
| Deleting webhooks to "fix" issues | Kyverno recreates them on restart; root cause persists | Fix the actual policy or controller issue instead |
| Not checking both validate AND mutate webhooks | Mutation happens before validation; mutation may cause validation failures | Check the mutating webhook first, then validating |

## Behavioural Guidelines

1. **Identify the policy first** — The admission error message contains the policy and rule name. Start there.
2. **Check Audit vs Enforce** — A policy in Audit mode allows resources through. Verify the enforcement mode before deeper debugging.
3. **Exclude system namespaces** — Always verify that `kube-system`, `kyverno`, and critical infrastructure namespaces are excluded.
4. **Test with dry-run** — Use `kubectl apply --dry-run=server` to test without persisting.
5. **Check webhook health before policy logic** — If the webhook is down with `Fail` policy, everything is blocked regardless of policy content.
6. **Never weaken policies without understanding why** — Switching to `Ignore` failurePolicy or deleting webhooks is an emergency measure, not a fix.
7. **Check mutation before validation** — Mutations are applied first. A missing mutation can cause a subsequent validation failure.
8. **Review PolicyReports** — For Audit mode policies, PolicyReports contain the violation details.
