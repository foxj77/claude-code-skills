---
name: kro-troubleshooting
description: Use when kro ResourceGroup resources are not reconciling, when instances fail to create child resources, when DAG dependency ordering is wrong, when CEL expressions in kro produce errors, or when kro controller logs show failures
---

# kro Troubleshooting

Diagnose and resolve failures in kro (Kubernetes Resource Orchestrator) — the controller that lets you define ResourceGroups to compose multiple Kubernetes resources into a single custom API, with dependency ordering via a directed acyclic graph (DAG) and CEL expressions for dynamic configuration.

## Keywords

kro, resource-group, resourcegroup, instance, dag, dependency, cel, composition, custom-resource, custom-api, reconcile, controller, schema, status, child-resources, resource-orchestrator, azure

## When to Use This Skill

- ResourceGroup status is not `Active` or shows errors
- Instances of a ResourceGroup are not creating expected child resources
- CEL expressions in ResourceGroup spec are producing errors
- Child resources are created in the wrong order or have dependency failures
- ResourceGroup CRD is not being generated
- kro controller logs show reconciliation failures
- Status fields are not being populated on instances

## Related Skills

- [kyverno-troubleshooting](../kyverno-troubleshooting) - Policies may block kro-created resources
- [cert-manager-troubleshooting](../cert-manager-troubleshooting) - Certificates in ResourceGroups
- [external-secrets-troubleshooting](../external-secrets-troubleshooting) - Secrets in ResourceGroups
- [external-dns-troubleshooting](../external-dns-troubleshooting) - DNS records in ResourceGroups
- [k8s-namespace-troubleshooting](../k8s-namespace-troubleshooting) - General namespace diagnosis
- [k8s-platform-operations](../k8s-platform-operations) - Cluster-wide operations
- [flux-troubleshooting](../flux-troubleshooting) - GitOps delivery of kro resources

## Quick Reference

| Task | Command |
|------|---------|
| Check kro controller | `kubectl get pods -n kro-system` |
| List ResourceGroups | `kubectl get resourcegroup -A` |
| Describe ResourceGroup | `kubectl describe resourcegroup NAME` |
| Check ResourceGroup status | `kubectl get resourcegroup NAME -o jsonpath='{.status}'` |
| View controller logs | `kubectl logs -n kro-system deploy/kro-controller-manager --tail=200` |
| List instances | `kubectl get NAME-PLURAL -A` (uses the CRD name from ResourceGroup) |
| Check generated CRD | `kubectl get crd \| grep NAME` |

---

## Diagnostic Workflow

```
ResourceGroup not working?
├─ kro controller running?
│   ├─ No → Check deployment, image, RBAC (Section 1)
│   └─ Yes → Check ResourceGroup status
│       ├─ Status missing → Controller not processing (Section 1)
│       ├─ State: Inactive → Spec validation error (Section 2)
│       └─ State: Active → Check instances (Section 3)
├─ Instance not creating resources?
│   ├─ Instance exists?
│   │   ├─ No → CRD not generated from ResourceGroup (Section 2)
│   │   └─ Yes → Check instance status
│   │       ├─ CEL error → Expression issue (Section 4)
│   │       ├─ Dependency failed → DAG ordering (Section 5)
│   │       └─ Child resource error → Underlying resource issue (Section 6)
└─ Resources created but status not populated?
    └─ Status CEL expression issue (Section 4)
```

---

## Section 1: Controller Health

```bash
# Check kro controller pods
kubectl get pods -n kro-system -o wide
kubectl describe deploy/kro-controller-manager -n kro-system

# Controller logs
kubectl logs -n kro-system deploy/kro-controller-manager --tail=200

# Check CRDs are installed
kubectl get crd | grep kro
# Expected: resourcegroups.kro.run

# Check RBAC
kubectl get clusterrole | grep kro
kubectl get clusterrolebinding | grep kro

# Check leader election
kubectl logs -n kro-system deploy/kro-controller-manager --tail=50 | grep -i "leader"
```

| Problem | Symptom | Fix |
|---------|---------|-----|
| Controller CrashLooping | OOMKill or panic in logs | Increase memory limits; check for recursive ResourceGroups |
| CRD missing | `resourcegroups.kro.run` not found | Reinstall kro (Helm chart or manifests) |
| RBAC insufficient | "forbidden" in controller logs | kro needs broad permissions to create child resources |
| Webhook failure | Resource creation rejected | Check webhook pod and certificates |

---

## Section 2: ResourceGroup Validation

```bash
# Check ResourceGroup status
kubectl get resourcegroup ${RG_NAME} -o yaml

# Check if CRD was generated
kubectl get crd | grep ${EXPECTED_CRD_NAME}

# Describe for events and conditions
kubectl describe resourcegroup ${RG_NAME}

# Check spec for obvious issues
kubectl get resourcegroup ${RG_NAME} -o json | jq '.spec'
```

### ResourceGroup Spec Structure

A ResourceGroup defines:
1. **Schema** — The custom API (spec and status fields for instances)
2. **Resources** — Child resources to create, with CEL expressions for dynamic values
3. **Dependencies** — Implicit from CEL references between resources (DAG)

### Validation Errors

| Error | Cause | Fix |
|-------|-------|-----|
| "invalid CEL expression" | Syntax error in a CEL expression | Check CEL syntax — see Section 4 |
| "cyclic dependency detected" | Resources reference each other circularly | Break the cycle; DAG must be acyclic |
| "duplicate resource ID" | Two resources have the same `id` | Give each resource a unique `id` |
| "unknown field in schema" | Schema references a field not defined | Ensure `spec` schema matches CEL usage |
| "invalid GVK" | `group`, `version`, or `kind` wrong for a resource | Check the child resource API group/version/kind |
| State stays empty | Controller not reconciling | Check controller logs for this ResourceGroup name |

### Common Spec Mistakes

```yaml
# ResourceGroup spec structure reminder
apiVersion: kro.run/v1alpha1
kind: ResourceGroup
metadata:
  name: my-app
spec:
  schema:
    apiVersion: v1alpha1          # Version for the generated CRD
    kind: MyApp                   # Kind for the generated CRD
    spec:                         # Fields users provide on instances
      name:
        type: string
        required: true
      replicas:
        type: integer
        default: 1
    status:                       # Fields populated from child resources
      deploymentReady:
        type: boolean
  resources:
    - id: deployment              # Unique ID for dependency tracking
      template:                   # Standard K8s resource template
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: ${schema.spec.name}  # CEL expression referencing schema
        spec:
          replicas: ${schema.spec.replicas}
          # ... rest of deployment spec
```

---

## Section 3: Instance Troubleshooting

```bash
# List instances of the ResourceGroup's generated CRD
# The CRD name is derived from the ResourceGroup's schema.kind
kubectl get ${KIND_PLURAL} -A

# Describe a specific instance
kubectl describe ${KIND} ${INSTANCE_NAME} -n ${NS}

# Check instance status
kubectl get ${KIND} ${INSTANCE_NAME} -n ${NS} -o yaml

# Check child resources created by the instance
kubectl get all -n ${NS} -l "kro.run/instance=${INSTANCE_NAME}"
```

| Problem | Symptom | Fix |
|---------|---------|-----|
| Instance created but no child resources | Status shows no conditions | Check controller logs for this instance |
| Some child resources missing | Partial creation | Check DAG order — earlier dependency may have failed |
| Instance stuck "Reconciling" | Never reaches Ready | Check which child resource is blocking |
| Instance deletion stuck | Finalizer blocking | Child resources may have their own finalizers; check them |

---

## Section 4: CEL Expression Issues

kro uses CEL (Common Expression Language) for dynamic values in resource templates and status mappings.

```bash
# Check controller logs for CEL errors
kubectl logs -n kro-system deploy/kro-controller-manager --tail=300 | grep -iE 'cel|expression|eval'

# Check ResourceGroup for CEL expressions
kubectl get resourcegroup ${RG_NAME} -o json | jq -r '.. | strings' | grep '\${'
```

### CEL Expression Reference

| Expression | Purpose | Example |
|------------|---------|---------|
| `${schema.spec.FIELD}` | Reference instance spec field | `${schema.spec.name}` |
| `${schema.spec.FIELD > 0 ? "yes" : "no"}` | Conditional | Ternary for conditional values |
| `${resources.RESOURCE_ID.status.FIELD}` | Reference another resource's status | `${resources.deployment.status.readyReplicas}` |
| `${schema.spec.FIELD + "-suffix"}` | String concatenation | `${schema.spec.name + "-svc"}` |

### Common CEL Errors

| Error | Cause | Fix |
|-------|-------|-----|
| "undeclared reference" | Field name doesn't exist in schema or resource | Check schema field names and resource IDs exactly |
| "type mismatch" | CEL expects string but got int (or vice versa) | Use explicit conversion: `string(schema.spec.replicas)` |
| "no such field" | Wrong path in resource status reference | Verify the child resource actually has that status field |
| "missing key" | Optional field not provided and no default | Add `default` in schema or use CEL `has()` / ternary |
| Expression not evaluated | Missing `${}` wrapper | Wrap CEL expressions in `${...}` |

### CEL Debugging Tips

1. **Start simple** — Test with hardcoded values first, then add CEL expressions one at a time.
2. **Check types** — CEL is strongly typed. Use `int()`, `string()`, `bool()` for conversions.
3. **Reference by ID** — Cross-resource references use the `id` field, not the resource name.
4. **Status references** — You can only reference status from resources that are earlier in the DAG.

---

## Section 5: DAG and Dependency Issues

kro automatically builds a dependency graph from CEL references between resources. Resources that reference others are created after their dependencies.

```bash
# Check ResourceGroup for dependency structure
kubectl get resourcegroup ${RG_NAME} -o json | jq '.status.topologicalOrder // .status'

# Controller logs for DAG-related errors
kubectl logs -n kro-system deploy/kro-controller-manager --tail=200 | grep -iE 'dag|depend|order|cycle|topolog'
```

| Problem | Cause | Fix |
|---------|-------|-----|
| Cyclic dependency | Resource A references B and B references A | Refactor to break the cycle; use a third resource or restructure |
| Wrong creation order | Implicit dependency not detected | Add an explicit CEL reference to establish ordering |
| Resource waits forever | Dependency never becomes ready | Check the blocking dependency resource separately |
| Unnecessary sequencing | All resources serialised when some are independent | Ensure only necessary CEL cross-references exist |

### Understanding the DAG

```
Example ResourceGroup with 3 resources:

  namespace (no dependencies)
       ↓
  deployment (references namespace)
       ↓
  service (references deployment)

kro creates: namespace first → then deployment → then service
Independent resources (no cross-references) are created in parallel.
```

---

## Section 6: Child Resource Failures

When kro creates child resources, they may fail for reasons unrelated to kro itself.

```bash
# List child resources managed by kro
kubectl get all -n ${NS} -l "kro.run/managed-by=kro"

# Check if child resources have errors
kubectl get events -n ${NS} --field-selector type=Warning --sort-by='.lastTimestamp'

# Describe specific failing child resource
kubectl describe ${RESOURCE_TYPE} ${RESOURCE_NAME} -n ${NS}
```

| Problem | Cause | Fix |
|---------|-------|-----|
| Deployment child can't pull image | ImagePullBackOff | Fix image name/tag or imagePullSecret |
| Service child has no endpoints | Selector mismatch | Verify selector labels match pod labels in CEL template |
| PVC child pending | No StorageClass or capacity | Check StorageClass availability |
| Admission webhook rejects child | Kyverno or other policy blocks | Check policy violations — see kyverno-troubleshooting |
| RBAC denied on child creation | kro SA lacks permission | Add RBAC rules for the child resource's API group |

---

## Upgrading kro

kro is in active development (currently v1alpha1). Upgrades may include breaking changes.

```bash
# Check current version
kubectl get deploy kro-controller-manager -n kro-system -o jsonpath='{.spec.template.spec.containers[0].image}'

# Check CRD version
kubectl get crd resourcegroups.kro.run -o jsonpath='{.spec.versions[*].name}'
```

| Upgrade Issue | Symptom | Fix |
|---------------|---------|-----|
| CRD schema changed | ResourceGroups fail validation after upgrade | Update ResourceGroup specs to match new schema |
| API version bump | Old resources not recognized | Apply conversion webhook or recreate resources |
| New required fields | Existing ResourceGroups invalid | Add newly required fields to existing specs |

---

## Common Mistakes

| Mistake | Why It Fails | Instead |
|---------|--------------|---------|
| Circular CEL references between resources | kro cannot build a valid DAG | Map out dependencies before writing; ensure the graph is acyclic |
| Referencing status of a resource that isn't a dependency | The referenced resource may not exist yet | Only reference status from resources earlier in the dependency chain |
| Forgetting `${}` around CEL expressions | Value is treated as a literal string, not evaluated | Wrap all dynamic values in `${...}` |
| Not giving kro RBAC for child resource types | Controller cannot create the child resources | Grant kro's service account permissions for every resource type used in ResourceGroups |
| Hardcoding namespaces in child resources | Resources created in wrong namespace or conflict across instances | Use `${schema.spec.namespace}` or let kro inherit the instance's namespace |
| Testing with production resources | Child resource creation may affect production | Test ResourceGroups in a separate namespace or cluster first |

## Behavioural Guidelines

1. **Check ResourceGroup state first** — If the ResourceGroup is not `Active`, no instances can work.
2. **Follow the DAG** — When child resources fail, identify which resource in the dependency chain failed first.
3. **Validate CEL expressions incrementally** — Start with simple expressions and add complexity gradually.
4. **Check kro controller RBAC** — kro needs permissions for every resource type it manages. Missing RBAC is a silent failure.
5. **Treat child resource failures separately** — Once kro creates a child resource, diagnose it using the appropriate skill (e.g., use k8s-namespace-troubleshooting for pod failures).
6. **Remember kro is alpha** — API and behaviour may change between versions. Check release notes after upgrades.
7. **Never expose secrets** — ResourceGroups may reference secrets; list names only, never decode.
