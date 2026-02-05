---
name: k8s-platform-tenancy
description: Use when provisioning new tenant namespaces, configuring tenant RBAC roles and bindings, setting up resource quotas and limits, implementing network isolation between tenants, managing tenant lifecycle (onboarding/offboarding), or designing self-service provisioning
---

# Kubernetes Platform Tenancy

Manage multi-tenant Kubernetes platforms, namespace provisioning, RBAC, resource isolation, and tenant lifecycle management.

## Keywords

kubernetes, multi-tenant, namespace, tenancy, rbac, resource quota, limit range, network policy, isolation, onboarding, offboarding, self-service, platform engineering, provisioning, configuring, setting up, implementing, managing, designing

## When to Use This Skill

- Provisioning new tenant namespaces
- Configuring tenant RBAC roles and bindings
- Setting up resource quotas and limits
- Implementing network isolation between tenants
- Managing tenant lifecycle (onboarding/offboarding)
- Designing self-service provisioning

## Related Skills

- [k8s-security-hardening](../k8s-security-hardening) - Security controls for tenants
- [k8s-platform-operations](../k8s-platform-operations) - Day-to-day operations
- [k8s-continual-improvement](../k8s-continual-improvement) - SLOs and cost allocation
- [k8s-security-redteam](../k8s-security-redteam) - Test tenant isolation
- [k8s-namespace-troubleshooting](../k8s-namespace-troubleshooting) - Namespace-scoped diagnosis
- [Shared: Pod Security Context](../_shared/references/pod-security-context.md)
- [Shared: Network Policies](../_shared/references/network-policies.md)
- [Shared: RBAC Patterns](../_shared/references/rbac-patterns.md)

## Quick Reference

| Task | Command |
|------|---------|
| List tenants | `kubectl get ns -l platform.io/tenant` |
| Check quota | `kubectl describe resourcequota -n tenant-NAME` |
| Audit RBAC | `kubectl auth can-i --list --as=user -n tenant-NAME` |
| View policies | `kubectl get networkpolicies -n tenant-NAME` |

## Tenant Isolation Model

### Namespace-per-Tenant Pattern
```
cluster/
├── platform-system/          # Platform team only
│   ├── monitoring/
│   ├── ingress/
│   └── cert-manager/
├── tenant-alpha/             # Tenant workloads
├── tenant-beta/
└── tenant-gamma/
```

### Isolation Layers
1. **Namespace** - Logical boundary
2. **RBAC** - Access control
3. **NetworkPolicy** - Network isolation
4. **ResourceQuota** - Resource limits
5. **LimitRange** - Default constraints
6. **PodSecurityStandard** - Security baseline

## Namespace Provisioning

### Standard Tenant Namespace
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-${TENANT_NAME}
  labels:
    platform.io/tenant: ${TENANT_NAME}
    platform.io/environment: ${ENV}
    platform.io/cost-center: ${COST_CENTER}
    platform.io/tier: ${TIER}
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
  annotations:
    platform.io/owner: ${OWNER_EMAIL}
    platform.io/created: ${DATE}
    platform.io/expires: ${EXPIRY_DATE}  # Optional for temp namespaces
```

### Resource Quota Templates

**Bronze Tier:**
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-quota
spec:
  hard:
    requests.cpu: "5"
    requests.memory: 10Gi
    limits.cpu: "10"
    limits.memory: 20Gi
    persistentvolumeclaims: "5"
    services.loadbalancers: "1"
    count/pods: "25"
```

**Silver Tier:**
```yaml
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    persistentvolumeclaims: "10"
    services.loadbalancers: "2"
    count/pods: "50"
```

**Gold Tier:**
```yaml
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    persistentvolumeclaims: "20"
    services.loadbalancers: "5"
    count/pods: "100"
```

### Limit Range
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: tenant-limits
spec:
  limits:
  - type: Container
    default:
      cpu: 500m
      memory: 512Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    max:
      cpu: "4"
      memory: 8Gi
    min:
      cpu: 50m
      memory: 64Mi
  - type: PersistentVolumeClaim
    max:
      storage: 100Gi
    min:
      storage: 1Gi
```

## RBAC Configuration

For detailed RBAC patterns, see [Shared: RBAC Patterns](../_shared/references/rbac-patterns.md).

### Tenant Admin Role
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: tenant-admin
rules:
- apiGroups: ["", "apps", "batch"]
  resources: ["*"]
  verbs: ["*"]
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses", "networkpolicies"]
  verbs: ["*"]
- apiGroups: ["autoscaling"]
  resources: ["horizontalpodautoscalers"]
  verbs: ["*"]
```

### Tenant Developer Role
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: tenant-developer
rules:
- apiGroups: ["", "apps", "batch"]
  resources: ["pods", "deployments", "services", "configmaps", "secrets"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["pods/log", "pods/exec"]
  verbs: ["get", "create"]
```

## Network Isolation

For detailed NetworkPolicy patterns, see [Shared: Network Policies](../_shared/references/network-policies.md).

### Standard Policy Set
Apply these in order to each tenant namespace:
1. `default-deny-all` - Zero trust baseline
2. `allow-dns` - DNS resolution
3. `allow-same-namespace` - Intra-namespace communication
4. `allow-ingress-controller` - External traffic
5. `allow-prometheus-scrape` - Monitoring

## Tenant Lifecycle

### Onboarding Checklist
1. **Create namespace** with standard labels
2. **Apply Pod Security Standard** (restricted)
3. **Configure ResourceQuota** based on tier
4. **Apply LimitRange** with sensible defaults
5. **Create RBAC** roles and bindings
6. **Deploy NetworkPolicies** for isolation
7. **Configure monitoring** (ServiceMonitor, alerts)
8. **Set up logging** forwarding
9. **Document** in tenant registry
10. **Notify** tenant with access instructions

### Offboarding Checklist
1. **Notify tenant** of decommission date
2. **Backup** any required data
3. **Revoke RBAC** bindings
4. **Delete workloads** (deployments, services)
5. **Delete PVCs** and data
6. **Remove monitoring** configuration
7. **Delete namespace**
8. **Update tenant registry**
9. **Archive documentation**

### Quota Modification Process
1. Tenant submits request (ticket/form)
2. Platform team reviews capacity
3. If approved, update ResourceQuota in Git
4. Flux applies changes
5. Notify tenant of new limits

## Self-Service Provisioning

### Namespace Request CRD Pattern
```yaml
apiVersion: platform.io/v1
kind: TenantRequest
metadata:
  name: new-tenant-request
spec:
  tenantName: alpha
  tier: silver
  owner: team-alpha@company.com
  costCenter: CC-1234
  environments:
    - dev
    - staging
    - prod
```

### Controller Automation
- Watch TenantRequest CRs
- Validate against policies
- Create namespace with standard resources
- Notify requestor

## Service Tiers

| Tier | CPU | Memory | Storage | Support | SLA |
|------|-----|--------|---------|---------|-----|
| Bronze | 5 | 10Gi | 50Gi | Best effort | None |
| Silver | 10 | 20Gi | 200Gi | Business hours | 99% |
| Gold | 20 | 40Gi | 500Gi | 24/7 | 99.5% |
| Platinum | Custom | Custom | Custom | Dedicated | 99.9% |

## Platform Services for Tenants

- **Ingress** - NGINX/Traefik with TLS
- **Certificates** - cert-manager with Let's Encrypt
- **Secrets** - External Secrets Operator
- **Monitoring** - Prometheus/Grafana (read-only)
- **Logging** - Loki with namespace filtering
- **Service Mesh** - Optional Istio/Linkerd

## Common Mistakes

| Mistake | Why It Fails | Instead |
|---------|--------------|---------|
| Granting `*` verbs on all resources in tenant role | Tenants can modify NetworkPolicies or ResourceQuotas, breaking isolation | Enumerate specific resources and verbs per role |
| Forgetting default LimitRange on new namespaces | Pods without resource requests get best-effort QoS and are evicted first | Always pair ResourceQuota with a LimitRange |
| Using `platform.io/tenant` label without enforcement | Anyone can relabel a namespace and bypass tenant scoping | Enforce labels with admission control (Kyverno/OPA) |
| Skipping NetworkPolicy on "internal-only" namespaces | Compromised pod in one tenant can reach all others | Apply default-deny + explicit allow to every tenant namespace |
| Deleting namespace before revoking RBAC | Tenant users see confusing errors; orphaned ClusterRoleBindings remain | Revoke bindings first, then delete namespace (follow offboarding checklist) |

## MCP Tools

```bash
# Using kubectl via MCP
mcp__flux-operator-mcp__get_kubernetes_resources
mcp__flux-operator-mcp__apply_kubernetes_manifest
```
