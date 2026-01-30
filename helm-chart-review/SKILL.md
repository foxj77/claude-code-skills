---
name: helm-chart-review
description: Use when performing code reviews of Helm charts, conducting security audits of charts, assessing chart quality and best practices, running automated chart analysis tools, or validating chart structure and organization
---

# Helm Chart Review

Review Helm charts including security, quality, best practices, and code review guidelines.

## Keywords

helm, chart, review, reviewing, security, quality, auditing, audit, best practices, code review, trivy, kubescape, polaris, pluto, lint, linting, validating, validation, analyzing, analysis

## When to Use This Skill

- Performing code reviews of Helm charts
- Conducting security audits of charts
- Assessing chart quality and best practices
- Running automated chart analysis tools
- Validating chart structure and organization

## Related Skills

- [helm-chart-development](../helm-chart-development) - Creating charts
- [helm-chart-maintenance](../helm-chart-maintenance) - Maintaining charts
- [k8s-security-hardening](../k8s-security-hardening) - Security best practices
- [Shared: Pod Security Context](../_shared/references/pod-security-context.md)
- [Shared: RBAC Patterns](../_shared/references/rbac-patterns.md)

## Quick Reference

| Task | Command |
|------|---------|
| Lint chart | `helm lint mychart/ --strict` |
| Security scan | `trivy config mychart/` |
| Best practices | `helm template myrelease mychart/ \| polaris audit --audit-path -` |
| Deprecated APIs | `helm template myrelease mychart/ \| pluto detect -` |
| NSA framework | `helm template myrelease mychart/ \| kubescape scan framework nsa -` |

## Review Checklist

### Structure & Organization
- [ ] Standard directory structure followed
- [ ] Chart.yaml has required fields (apiVersion, name, version)
- [ ] README.md exists and is complete
- [ ] NOTES.txt provides useful post-install information
- [ ] .helmignore excludes unnecessary files
- [ ] Templates organized logically

### Values Design
- [ ] values.yaml has sensible defaults
- [ ] All values documented with comments
- [ ] values.schema.json validates inputs
- [ ] No hardcoded values in templates
- [ ] Sensitive values use secrets, not configmaps

### Security
- [ ] Pod security context defined
- [ ] Container security context defined
- [ ] Service account with minimal permissions
- [ ] Network policies included (if applicable)
- [ ] No privileged containers by default
- [ ] Resource limits defined

### Quality
- [ ] Templates pass `helm lint`
- [ ] Unit tests exist and pass
- [ ] Labels follow Kubernetes conventions
- [ ] Proper use of helpers (_helpers.tpl)
- [ ] Consistent naming conventions

## Security Review

### Pod Security Checklist

```yaml
# REQUIRED security settings
podSecurityContext:
  runAsNonRoot: true       # ✓ Never run as root
  fsGroup: 1000            # ✓ Set filesystem group
  seccompProfile:
    type: RuntimeDefault   # ✓ Use seccomp

securityContext:
  allowPrivilegeEscalation: false  # ✓ Block privilege escalation
  readOnlyRootFilesystem: true     # ✓ Immutable container
  runAsNonRoot: true               # ✓ Non-root user
  runAsUser: 1000                  # ✓ Specific UID
  capabilities:
    drop:
      - ALL                        # ✓ Drop all capabilities
```

### Security Anti-Patterns

```yaml
# ❌ BAD: Privileged container
securityContext:
  privileged: true

# ❌ BAD: Running as root
securityContext:
  runAsUser: 0

# ❌ BAD: Writable root filesystem
securityContext:
  readOnlyRootFilesystem: false

# ❌ BAD: Host namespaces
hostNetwork: true
hostPID: true
hostIPC: true

# ❌ BAD: Dangerous volume mounts
volumes:
  - name: host
    hostPath:
      path: /

# ❌ BAD: Secrets in environment variables (prefer mounted secrets)
env:
  - name: DB_PASSWORD
    value: "hardcoded-password"

# ❌ BAD: No resource limits
resources: {}
```

### RBAC Review

```yaml
# ✓ GOOD: Minimal permissions
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch"]
  resourceNames: ["my-config"]  # Even better: specific resources

# ❌ BAD: Overly permissive
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]

# ❌ BAD: Cluster-wide access when namespace-scoped sufficient
kind: ClusterRole  # Should be Role if namespace-scoped
```

### Image Security

```yaml
# ✓ GOOD: Specific tag
image:
  repository: myapp
  tag: "v1.2.3"  # Specific version

# ❌ BAD: Latest tag
image:
  repository: myapp
  tag: "latest"  # Mutable, unpredictable

# ✓ GOOD: Digest pinning for critical apps
image:
  repository: myapp@sha256:abc123...
```

## Quality Review

### Template Quality

```yaml
# ✓ GOOD: Use helpers for common patterns
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}

# ❌ BAD: Duplicated logic
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
  labels:
    app: {{ .Chart.Name }}
    release: {{ .Release.Name }}
```

### Whitespace & Formatting

```yaml
# ✓ GOOD: Proper indentation with nindent
spec:
  {{- with .Values.nodeSelector }}
  nodeSelector:
    {{- toYaml . | nindent 4 }}
  {{- end }}

# ❌ BAD: Manual indentation (error-prone)
spec:
  {{- if .Values.nodeSelector }}
  nodeSelector:
{{ toYaml .Values.nodeSelector | indent 4 }}
  {{- end }}
```

### Conditional Resources

```yaml
# ✓ GOOD: Clean conditional
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
...
{{- end }}

# ✓ GOOD: Required values
image: {{ required "image.repository is required" .Values.image.repository }}

# ❌ BAD: Silent failures
image: {{ .Values.image.repository }}  # Empty if not set
```

### Labels & Annotations

```yaml
# ✓ GOOD: Standard Kubernetes labels
metadata:
  labels:
    app.kubernetes.io/name: {{ include "myapp.name" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
    app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
    helm.sh/chart: {{ include "myapp.chart" . }}

# ❌ BAD: Non-standard labels
metadata:
  labels:
    app: myapp
    version: v1
```

## Values Review

### Documentation

```yaml
# ✓ GOOD: Documented values
# -- Number of pod replicas
# @default -- 1
replicaCount: 1

# -- Image configuration
image:
  # -- Repository for the container image
  repository: myapp
  # -- Image pull policy
  # @default -- IfNotPresent
  pullPolicy: IfNotPresent

# ❌ BAD: Undocumented values
replicaCount: 1
image:
  repository: myapp
  pullPolicy: IfNotPresent
```

### Defaults

```yaml
# ✓ GOOD: Sensible, secure defaults
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 128Mi

securityContext:
  runAsNonRoot: true
  readOnlyRootFilesystem: true

# ❌ BAD: No defaults for critical settings
resources: {}
securityContext: {}
```

### Schema Validation

```json
// ✓ GOOD: Comprehensive schema
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["image"],
  "properties": {
    "replicaCount": {
      "type": "integer",
      "minimum": 1,
      "maximum": 100
    },
    "image": {
      "type": "object",
      "required": ["repository"],
      "properties": {
        "repository": {
          "type": "string",
          "minLength": 1
        },
        "pullPolicy": {
          "type": "string",
          "enum": ["Always", "IfNotPresent", "Never"]
        }
      }
    }
  }
}
```

## Code Review Comments

### Severity Levels

| Level | Description | Action |
|-------|-------------|--------|
| 🔴 Critical | Security vulnerability, data loss risk | Must fix before merge |
| 🟠 Major | Best practice violation, significant issue | Should fix |
| 🟡 Minor | Style, minor improvement | Nice to have |
| 🔵 Suggestion | Alternative approach | Consider |

### Example Review Comments

```markdown
🔴 **Critical: Security - Privileged Container**
The container is running as privileged which grants full host access.
```yaml
# Current
securityContext:
  privileged: true

# Suggested
securityContext:
  privileged: false
  allowPrivilegeEscalation: false
```

---

🟠 **Major: Missing Resource Limits**
No resource limits defined. This can lead to resource starvation.
```yaml
# Add to values.yaml
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 128Mi
```

---

🟡 **Minor: Use nindent instead of indent**
`nindent` handles newlines automatically and is more reliable.
```yaml
# Current
{{ toYaml .Values.labels | indent 4 }}

# Suggested
{{- toYaml .Values.labels | nindent 4 }}
```

---

🔵 **Suggestion: Consider using a helper**
This pattern is repeated in multiple templates. Consider extracting to _helpers.tpl.
```

## Automated Review Tools

### helm lint
```bash
# Basic linting
helm lint mychart/

# Strict mode
helm lint mychart/ --strict

# With values
helm lint mychart/ -f values-production.yaml
```

### Trivy (Security)
```bash
# Scan chart for misconfigurations
trivy config mychart/

# JSON output
trivy config mychart/ -f json -o results.json
```

### Kubescape
```bash
# Scan rendered templates
helm template myrelease mychart/ | kubescape scan -

# With specific framework
helm template myrelease mychart/ | kubescape scan framework nsa -
```

### Polaris
```bash
# Scan for best practices
helm template myrelease mychart/ | polaris audit --audit-path -
```

### Pluto (Deprecations)
```bash
# Check for deprecated APIs
helm template myrelease mychart/ | pluto detect -
```

## Review Process

### Before Review
1. Run `helm lint` locally
2. Run `helm template` to verify rendering
3. Run security scans (Trivy, Kubescape)
4. Run unit tests

### During Review
1. Check structure and organization
2. Review values.yaml design
3. Audit security settings
4. Verify template quality
5. Check documentation

### After Review
1. Verify all critical issues addressed
2. Re-run automated checks
3. Test deployment in dev environment
4. Approve or request changes

## Review Template

```markdown
## Helm Chart Review: [chart-name] v[version]

### Summary
[Brief description of changes]

### Automated Checks
- [ ] `helm lint` passes
- [ ] `helm template` renders correctly
- [ ] Unit tests pass
- [ ] Security scan clean (Trivy/Kubescape)
- [ ] No deprecated APIs (Pluto)

### Manual Review
- [ ] Chart structure follows conventions
- [ ] Values documented and have sensible defaults
- [ ] Security contexts properly configured
- [ ] Resource limits defined
- [ ] RBAC follows least privilege
- [ ] Labels follow Kubernetes conventions
- [ ] README accurate and complete

### Issues Found
| Severity | Issue | Location |
|----------|-------|----------|
| 🔴 | ... | ... |
| 🟠 | ... | ... |

### Recommendation
[ ] ✅ Approve
[ ] 🔄 Request changes
[ ] ❌ Reject
```
