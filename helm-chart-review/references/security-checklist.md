# Helm Chart Security Checklist

## Container Security

### Pod Security Context
```yaml
# ✅ Required
podSecurityContext:
  runAsNonRoot: true
  fsGroup: 1000
  seccompProfile:
    type: RuntimeDefault
```

| Setting | Required | Reason |
|---------|----------|--------|
| `runAsNonRoot: true` | ✅ | Prevent root execution |
| `fsGroup` | ✅ | Proper file permissions |
| `seccompProfile` | ✅ | Restrict syscalls |

### Container Security Context
```yaml
# ✅ Required
securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  runAsNonRoot: true
  runAsUser: 1000
  capabilities:
    drop:
      - ALL
```

| Setting | Required | Reason |
|---------|----------|--------|
| `allowPrivilegeEscalation: false` | ✅ | Prevent privilege escalation |
| `readOnlyRootFilesystem: true` | ✅ | Immutable container |
| `runAsNonRoot: true` | ✅ | Non-root user |
| `runAsUser` | ✅ | Explicit UID |
| `capabilities.drop: [ALL]` | ✅ | Minimal capabilities |

### Prohibited Settings
```yaml
# ❌ Never allow by default
securityContext:
  privileged: true           # ❌

hostNetwork: true            # ❌
hostPID: true                # ❌
hostIPC: true                # ❌

volumes:
  - name: host
    hostPath:                # ❌
      path: /
```

## Image Security

### Image Configuration
```yaml
# ✅ Good
image:
  repository: registry.example.com/myapp
  tag: "v1.2.3"  # Specific version
  pullPolicy: IfNotPresent

# ❌ Bad
image:
  repository: myapp
  tag: "latest"  # Mutable
```

| Check | Status |
|-------|--------|
| Specific image tag (not `latest`) | ✅ |
| Private registry preferred | ✅ |
| Image pull policy appropriate | ✅ |
| Image pull secrets configured | ✅ |

## RBAC Security

### Service Account
```yaml
# ✅ Minimal service account
serviceAccount:
  create: true
  automount: false  # Don't mount token unless needed
```

### Role Permissions
```yaml
# ✅ Least privilege
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]
  resourceNames: ["my-config"]

# ❌ Overly permissive
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
```

| Check | Status |
|-------|--------|
| No wildcard permissions | ✅ |
| Namespace-scoped when possible | ✅ |
| Specific resource names when possible | ✅ |
| Token automount disabled by default | ✅ |

## Secrets Management

### Secret Handling
```yaml
# ✅ External secret reference
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-credentials
        key: password

# ❌ Hardcoded secret
env:
  - name: DB_PASSWORD
    value: "hardcoded-password"
```

| Check | Status |
|-------|--------|
| No hardcoded secrets | ✅ |
| Secrets from secretKeyRef | ✅ |
| External secrets integration | ✅ |
| Secrets not logged in NOTES.txt | ✅ |

## Network Security

### Network Policy
```yaml
# ✅ Include network policy option
networkPolicy:
  enabled: true
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - port: 8080
  egress:
    - to:
        - namespaceSelector: {}
      ports:
        - port: 53
          protocol: UDP
```

| Check | Status |
|-------|--------|
| Network policy template included | ✅ |
| Default deny option available | ✅ |
| Egress restricted when possible | ✅ |

## Resource Limits

### Resource Configuration
```yaml
# ✅ Always set limits
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 128Mi
```

| Check | Status |
|-------|--------|
| CPU limits defined | ✅ |
| Memory limits defined | ✅ |
| Requests defined | ✅ |
| Sensible defaults | ✅ |

## Ingress Security

### Ingress Configuration
```yaml
# ✅ Secure ingress
ingress:
  enabled: false  # Disabled by default
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
  tls:
    - secretName: app-tls
      hosts:
        - app.example.com
```

| Check | Status |
|-------|--------|
| TLS enabled option | ✅ |
| SSL redirect annotation | ✅ |
| Ingress disabled by default | ✅ |

## Scanning Commands

```bash
# Trivy config scan
trivy config mychart/

# Kubescape security scan
helm template myrelease mychart/ | kubescape scan framework nsa -

# Checkov scan
checkov -d mychart/

# Polaris audit
helm template myrelease mychart/ | polaris audit --audit-path -
```

## Review Summary Template

```markdown
## Security Review: [chart-name]

### Critical Issues
- [ ] No privileged containers
- [ ] No host namespaces
- [ ] No hostPath volumes

### High Priority
- [ ] runAsNonRoot: true
- [ ] allowPrivilegeEscalation: false
- [ ] readOnlyRootFilesystem: true
- [ ] Capabilities dropped

### Medium Priority
- [ ] Resource limits defined
- [ ] Service account minimal
- [ ] Network policy option
- [ ] No hardcoded secrets

### Low Priority
- [ ] Image tag not latest
- [ ] Ingress TLS option
- [ ] Pod disruption budget

### Scan Results
- Trivy: X issues
- Kubescape: X% compliant
- Polaris: X/100

### Recommendation
[ ] ✅ Approved
[ ] ⚠️ Approved with conditions
[ ] ❌ Requires remediation
```
