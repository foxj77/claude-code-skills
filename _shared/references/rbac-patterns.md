# RBAC Patterns Reference

Canonical reference for Kubernetes RBAC patterns. Referenced by multiple skills.

## Core Concepts

| Resource | Scope | Purpose |
|----------|-------|---------|
| Role | Namespace | Permissions within a namespace |
| ClusterRole | Cluster | Permissions cluster-wide or reusable template |
| RoleBinding | Namespace | Binds Role/ClusterRole to subjects in namespace |
| ClusterRoleBinding | Cluster | Binds ClusterRole to subjects cluster-wide |

## Principle of Least Privilege

```yaml
# ✅ GOOD: Specific, minimal permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-reader
  namespace: my-namespace
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch"]
  resourceNames: ["app-config"]  # Even more restrictive

# ❌ BAD: Wildcard permissions
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
```

## Standard Role Templates

### Read-Only Viewer
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: namespace-viewer
rules:
- apiGroups: ["", "apps", "batch", "networking.k8s.io"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
```

### Developer (Deploy Apps)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: namespace-developer
rules:
# Core resources
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "secrets", "persistentvolumeclaims"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
# Workloads
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets", "statefulsets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
# Batch
- apiGroups: ["batch"]
  resources: ["jobs", "cronjobs"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
# Debugging
- apiGroups: [""]
  resources: ["pods/log", "pods/exec", "pods/portforward"]
  verbs: ["get", "create"]
# Events (read-only)
- apiGroups: [""]
  resources: ["events"]
  verbs: ["get", "list", "watch"]
```

### Admin (Full Namespace Control)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: namespace-admin
rules:
- apiGroups: ["", "apps", "batch", "networking.k8s.io", "autoscaling"]
  resources: ["*"]
  verbs: ["*"]
# Explicitly deny dangerous resources
- apiGroups: [""]
  resources: ["nodes", "persistentvolumes", "namespaces"]
  verbs: []
```

## Service Account Patterns

### Minimal Service Account
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: my-namespace
automountServiceAccountToken: false  # Don't mount unless needed
```

### Service Account with Specific Permissions
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: config-reader
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: config-reader-role
rules:
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: config-reader-binding
subjects:
- kind: ServiceAccount
  name: config-reader
roleRef:
  kind: Role
  name: config-reader-role
  apiGroup: rbac.authorization.k8s.io
```

## Dangerous Permissions to Audit

```bash
# Find cluster-admin bindings
kubectl get clusterrolebindings -o json | \
  jq '.items[] | select(.roleRef.name=="cluster-admin") | {name: .metadata.name, subjects: .subjects}'

# Find wildcard permissions
kubectl get roles,clusterroles -A -o json | \
  jq '.items[] | select(.rules[]?.verbs[]? == "*") | .metadata.name'

# Find escalation permissions
kubectl get roles,clusterroles -A -o json | \
  jq '.items[] | select(.rules[]?.verbs[]? == "escalate" or .rules[]?.verbs[]? == "bind") | .metadata.name'

# Audit specific user/SA permissions
kubectl auth can-i --list --as=system:serviceaccount:${NS}:${SA}
kubectl auth can-i --list --as=user@example.com -n ${NS}
```

## Dangerous Verbs Matrix

| Verb | Risk | Why |
|------|------|-----|
| `*` | Critical | Grants all permissions |
| `create pods` | High | Can run arbitrary code |
| `create pods/exec` | High | Can exec into pods |
| `get secrets` | High | Access to sensitive data |
| `escalate` | Critical | Can grant higher permissions |
| `bind` | Critical | Can bind to any role |
| `impersonate` | Critical | Can act as other users |
| `delete` | Medium | Can disrupt services |

## ClusterRole Aggregation

```yaml
# Base role with aggregation label
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-view
  labels:
    rbac.example.com/aggregate-to-view: "true"
rules:
- apiGroups: ["monitoring.coreos.com"]
  resources: ["servicemonitors", "podmonitors"]
  verbs: ["get", "list", "watch"]
---
# Aggregating ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: view-extended
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.example.com/aggregate-to-view: "true"
rules: []  # Rules are auto-aggregated
```

## RoleBinding Examples

### Bind to User
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: my-namespace
subjects:
- kind: User
  name: developer@example.com
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: namespace-developer
  apiGroup: rbac.authorization.k8s.io
```

### Bind to Group (OIDC)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-alpha-binding
  namespace: team-alpha
subjects:
- kind: Group
  name: team-alpha-developers  # OIDC group claim
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: namespace-developer
  apiGroup: rbac.authorization.k8s.io
```

## Quick Reference

```bash
# Create role
kubectl create role pod-reader --verb=get,list,watch --resource=pods -n ${NS}

# Create rolebinding
kubectl create rolebinding pod-reader-binding --role=pod-reader --user=jane -n ${NS}

# Check permissions
kubectl auth can-i get pods -n ${NS} --as=jane

# Who can do X?
kubectl auth who-can get secrets -n ${NS}
```

## Related Skills
- k8s-security-hardening - RBAC security implementation
- k8s-platform-tenancy - Tenant RBAC configuration
- k8s-security-redteam - RBAC exploitation testing
