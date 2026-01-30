# Kubernetes Security Hardening

Expert knowledge for securing Kubernetes platforms including Pod Security Standards, network policies, secrets management, admission control, and compliance.

## Keywords

kubernetes, security, hardening, pod security, pss, psa, network policy, rbac, secrets, encryption, audit, compliance, cis benchmark, admission control, kyverno, opa, gatekeeper

## When to Use This Skill

- Implementing Pod Security Standards
- Hardening cluster security configuration
- Setting up network policies for zero-trust
- Configuring secrets management
- Implementing admission control policies
- Conducting security audits
- Ensuring CIS benchmark compliance

## Related Skills

- [k8s-platform-tenancy](../k8s-platform-tenancy) - Multi-tenant security
- [k8s-security-redteam](../k8s-security-redteam) - Test your defenses
- [helm-chart-review](../helm-chart-review) - Chart security review
- [Shared: Pod Security Context](../_shared/references/pod-security-context.md)
- [Shared: Network Policies](../_shared/references/network-policies.md)
- [Shared: RBAC Patterns](../_shared/references/rbac-patterns.md)

## Quick Reference

| Task | Command |
|------|---------|
| Check PSS violations | `kubectl get pods -A -o json \| jq '.items[] \| select(.spec.securityContext.runAsNonRoot != true)'` |
| Audit cluster-admin | `kubectl get clusterrolebindings -o json \| jq '.items[] \| select(.roleRef.name=="cluster-admin")'` |
| List network policies | `kubectl get networkpolicies -A` |
| Run CIS benchmark | `kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml` |

## Pod Security Standards

For detailed security context configuration, see [Shared: Pod Security Context](../_shared/references/pod-security-context.md).

### Namespace Enforcement
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: secure-namespace
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### Profile Summary

| Profile | Use Case | Key Restrictions |
|---------|----------|------------------|
| **Privileged** | System/infra | None |
| **Baseline** | General | No privileged, no hostPath |
| **Restricted** | Security-sensitive | Non-root, drop caps, seccomp |

## Admission Control

### Kyverno Policy Example
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-run-as-nonroot
spec:
  validationFailureAction: Enforce
  rules:
  - name: run-as-non-root
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "Containers must run as non-root"
      pattern:
        spec:
          containers:
          - securityContext:
              runAsNonRoot: true
```

### Image Verification
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-images
spec:
  validationFailureAction: Enforce
  rules:
  - name: verify-signature
    match:
      any:
      - resources:
          kinds:
          - Pod
    verifyImages:
    - imageReferences:
      - "registry.company.com/*"
      attestors:
      - entries:
        - keyless:
            rekor:
              url: https://rekor.sigstore.dev
```

### OPA Gatekeeper Constraint
```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-team-label
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Namespace"]
  parameters:
    labels:
    - key: "team"
```

## Network Security

For detailed NetworkPolicy patterns, see [Shared: Network Policies](../_shared/references/network-policies.md).

### Zero-Trust Implementation
1. Apply default deny all
2. Allow DNS egress
3. Allow specific required traffic only
4. Audit with network policy logging

## RBAC Security

For detailed RBAC patterns, see [Shared: RBAC Patterns](../_shared/references/rbac-patterns.md).

### Audit Commands
```bash
# Find cluster-admin bindings
kubectl get clusterrolebindings -o json | \
  jq '.items[] | select(.roleRef.name=="cluster-admin") | .subjects'

# Find wildcard permissions
kubectl get roles,clusterroles -A -o json | \
  jq '.items[] | select(.rules[].verbs[] | contains("*")) | .metadata.name'

# Service account permissions
kubectl auth can-i --list --as=system:serviceaccount:${NS}:${SA}
```

## Secrets Management

### External Secrets
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-secrets
spec:
  refreshInterval: 1h
  secretStoreRef:
    kind: ClusterSecretStore
    name: vault
  target:
    name: app-secrets
    creationPolicy: Owner
  data:
  - secretKey: password
    remoteRef:
      key: secret/data/app
      property: password
```

### Encryption at Rest
```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-key>
  - identity: {}
```

## Runtime Security

### Falco Rules
```yaml
- rule: Shell Spawned in Container
  desc: Detect shell spawned in container
  condition: >
    spawned_process and
    container and
    proc.name in (shell_binaries)
  output: >
    Shell spawned in container
    (user=%user.name container=%container.name shell=%proc.name)
  priority: WARNING
  tags: [container, shell]
```

### Audit Policy
```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata
  resources:
  - group: ""
    resources: ["secrets", "configmaps"]
- level: RequestResponse
  users: ["system:anonymous"]
  verbs: ["*"]
- level: RequestResponse
  resources:
  - group: "rbac.authorization.k8s.io"
```

## Supply Chain Security

### SLSA Requirements
| Level | Requirements |
|-------|--------------|
| SLSA 1 | Build process documented |
| SLSA 2 | Version control, hosted build |
| SLSA 3 | Verified source, isolated build |
| SLSA 4 | Two-party review, hermetic builds |

### Image Signing (Cosign)
```bash
# Sign image
cosign sign --key cosign.key registry.example.com/app:v1.0.0

# Verify image
cosign verify --key cosign.pub registry.example.com/app:v1.0.0
```

## Security Scanning

| Tool | Target | Frequency |
|------|--------|-----------|
| Trivy | Container images | Every build |
| Kubescape | Cluster config | Daily |
| Falco | Runtime behavior | Continuous |
| kube-bench | CIS benchmark | Weekly |
| Polaris | Best practices | On change |

### Run kube-bench
```bash
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
kubectl logs -l app=kube-bench
```

## Security Checklist

### Cluster Level
- [ ] API server private network only
- [ ] etcd encrypted, access restricted
- [ ] Audit logging enabled
- [ ] PSS enforced cluster-wide
- [ ] Network policies default deny
- [ ] RBAC least privilege
- [ ] Secrets encrypted at rest

### Workload Level
- [ ] Non-root containers
- [ ] Read-only root filesystem
- [ ] No privilege escalation
- [ ] Capabilities dropped
- [ ] Resource limits set
- [ ] Signed images only
- [ ] No hostPath mounts

### Tenant Level
- [ ] Namespace isolation
- [ ] Network policies enforced
- [ ] Resource quotas applied
- [ ] RBAC scoped to namespace
- [ ] SA tokens disabled by default

## MCP Tools

- `mcp__flux-operator-mcp__get_kubernetes_resources` - Query resources
- `mcp__flux-operator-mcp__apply_kubernetes_manifest` - Apply policies
