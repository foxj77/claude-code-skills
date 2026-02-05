---
name: helm-chart-testing
description: Use when tests need to be added to a Helm chart, Helm tests are failing with unclear causes, or a chart needs test coverage analysis
---

# Helm Chart Testing

Write effective built-in Helm tests using pod annotations that run with `helm test`. Focus on designing tests that verify releases work correctly.

## Keywords

helm test, helm.sh/hook, test-success, test-failure, pod test, test annotation, smoke test, connectivity test, configuration validation, release verification, failing test, test coverage

## When to Use This Skill

- Adding test pods to a Helm chart
- Diagnosing why Helm tests are failing
- Determining what needs testing in a chart
- Writing connectivity, configuration, or readiness tests
- Designing test strategy for complex charts

## Related Skills

- [helm-chart-development](../helm-chart-development) - Chart structure and templating
- [helm-chart-maintenance](../helm-chart-maintenance) - Testing in CI/CD pipelines
- [k8s-platform-operations](../k8s-platform-operations) - Debugging test failures

## Quick Reference

| Task | Command/Annotation |
|------|-------------------|
| Run tests | `helm test RELEASE_NAME` |
| Run with logs | `helm test RELEASE_NAME --logs` |
| Mark pod as test | `helm.sh/hook: test-success` |
| Keep test pods | `helm.sh/hook-delete-policy: never` |
| Order execution | `helm.sh/hook-weight: "-5"` |

## How Helm Tests Work

1. Define test pods in `templates/` with `helm.sh/hook: test` annotation
2. Install chart with `helm install` or `helm upgrade`
3. Run tests with `helm test RELEASE_NAME`
4. Helm creates pods, executes them, reports results

**Test passes:** Pod exits with code 0
**Test fails:** Pod exits with non-zero code

## Test Annotations

| Annotation | Purpose |
|------------|---------|
| `helm.sh/hook: test` | Marks pod as a test (runs during `helm test`) |
| `helm.sh/hook: test-success` | Runs after successful release |
| `helm.sh/hook: test-failure` | Runs after failed release |
| `helm.sh/hook-weight:` | Controls execution order (lower = first) |
| `helm.sh/hook-delete-policy:` | Controls cleanup (`hook-succeeded`, `never`) |

## Test Analysis: What to Test

| Resource Type | Test Considerations |
|---------------|---------------------|
| Services | Endpoint reachability, DNS, correct ports |
| Deployments/StatefulSets | Pod readiness, replica count, rollout status |
| Ingress | Route reachability, TLS certificates |
| ConfigMaps/Secrets | Values present, mounted correctly |
| PVCs | Volume mounted, read/write access |
| CRDs | Custom resource creation, reconciliation |

**Analysis questions:**
- What is the minimum viable verification that the release worked?
- What are the critical failure modes?
- What external dependencies exist?
- Can tests run safely in production?

## Test Categories

| Category | Purpose | Examples |
|----------|---------|----------|
| Smoke | Quick "is it alive" | Service health, pod readiness |
| Functional | Verify specific behavior | API responses, database connectivity |
| Integration | Verify external interactions | Upstream services, third-party APIs |
| Data Validation | Verify deployed state | ConfigMap content, environment variables |

## Basic Test Pod Structure

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ .Release.Name }}-test-connectivity"
  annotations:
    helm.sh/hook: test-success
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
spec:
  containers:
  - name: test
    image: curlimages/curl:8.7.1
    command:
    - sh
    - -c
    - |
      set -e
      curl -f http://myapp-service:8080/health
  restartPolicy: Never
```

**Best practices:**
- Use `restartPolicy: Never`
- Use lightweight images (curlimages/curl, busybox, alpine)
- One test pod tests one thing well
- Exit code 0 = pass, non-zero = fail

**Common test images (always pin to a specific tag, not `latest`):**
- `curlimages/curl:8.7.1` - HTTP endpoint checks
- `busybox:1.36` - Basic shell utilities
- `bitnami/kubectl:1.29` - Kubernetes API queries
- `postgres:16-alpine`, `mysql:8.0-oracle` - Database connectivity

## Test Examples

### Service Connectivity Test

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ .Release.Name }}-api-test"
  annotations:
    helm.sh/hook: test-success
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
spec:
  containers:
  - name: api-test
    image: curlimages/curl:8.7.1
    command:
    - sh
    - -c
    - |
      set -e
      # Test main endpoint
      curl -f http://{{ .Release.Name }}-service:8080/health || exit 1
      # Verify response content
      curl -s http://{{ .Release.Name }}-service:8080/health | grep -q "status.*ok" || exit 1
      {{- if .Values.auth.enabled }}
      # Test authenticated endpoint
      curl -f http://{{ .Release.Name }}-service:8080/secure \
        -H "Authorization: Bearer {{ .Values.auth.testToken }}" || exit 1
      {{- end }}
  restartPolicy: Never
```

### Configuration Validation Test

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ .Release.Name }}-config-test"
  annotations:
    helm.sh/hook: test-success
spec:
  containers:
  - name: config-test
    image: busybox:1.36
    command:
    - sh
    - -c
    - |
      set -e
      test -f /app/config.yaml || exit 1
      grep -q "logLevel: {{ .Values.logLevel }}" /app/config.yaml || exit 1
      grep -q "database:" /app/config.yaml || exit 1
  volumeMounts:
  - name: config
    mountPath: /app/config.yaml
    subPath: config.yaml
  volumes:
  - name: config
    configMap:
      name: {{ include "myapp.fullname" . }}-config
  restartPolicy: Never
```

### Deployment Readiness Test

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ .Release.Name }}-readiness-test"
  annotations:
    helm.sh/hook: test-success
spec:
  serviceAccountName: {{ include "myapp.fullname" . }}-test-sa
  containers:
  - name: kubectl-test
    image: bitnami/kubectl:1.29
    command:
    - sh
    - -c
    - |
      set -e
      kubectl get deployment {{ .Release.Name }} -n {{ .Release.Namespace }} -o json | \
        jq -e '.status.readyReplicas == {{ .Values.replicaCount }}' || exit 1
  restartPolicy: Never
```

### Database Connection Test

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ .Release.Name }}-db-test"
  annotations:
    helm.sh/hook: test-success
spec:
  containers:
  - name: db-test
    image: postgres:16-alpine
    command:
    - sh
    - -c
    - |
      set -e
      nc -zv {{ .Values.database.host }} {{ .Values.database.port }} || exit 1
      PGPASSWORD={{ .Values.database.password }} \
      psql -h {{ .Values.database.host }} -p {{ .Values.database.port }} \
            -U {{ .Values.database.user }} -d {{ .Values.database.name }} \
            -c "SELECT 1;" || exit 1
  restartPolicy: Never
```

## Test Design Considerations

### Test Independence

Each test should verify one thing well:

```yaml
# Good: Single focused test
metadata:
  name: "{{ .Release.Name }}-test-health"

# Avoid: Tests multiple unrelated things
metadata:
  name: "{{ .Release.Name }}-test-everything"
```

### Resource Management

Set limits to prevent exhaustion:

```yaml
spec:
  containers:
  - name: test
    image: curlimages/curl:8.7.1
    resources:
      requests:
        cpu: 100m
        memory: 64Mi
      limits:
        cpu: 200m
        memory: 128Mi
```

### Cleanup Policies

| Policy | Behavior | Use Case |
|--------|----------|----------|
| `hook-succeeded` | Delete after passing | Normal operation |
| `never` | Never delete | Debugging |

For debugging:
```yaml
metadata:
  annotations:
    helm.sh/hook-delete-policy: never
```

### Error Handling

Provide clear failure messages:

```yaml
command:
- sh
- -c
- |
  set -e
  if ! curl -f http://service:8080/health; then
    echo "ERROR: Service health check failed"
    echo "Troubleshooting: kubectl get svc, kubectl logs -l app=myapp"
    exit 1
  fi
```

### Security

**Use least privilege RBAC:**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ include "myapp.fullname" . }}-test-sa
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: {{ include "myapp.fullname" . }}-test-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: {{ include "myapp.fullname" . }}-test-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: {{ include "myapp.fullname" . }}-test-role
subjects:
- kind: ServiceAccount
  name: {{ include "myapp.fullname" . }}-test-sa
```

**Don't embed secrets:**

```yaml
# Avoid
command: ["curl", "-H", "Authorization: Bearer super-secret-key", "http://service/"]

# Better
env:
- name: TEST_TOKEN
  valueFrom:
    secretKeyRef:
      name: test-credentials
      key: token
```

## Organizing Tests

```
mychart/
├── templates/
│   ├── tests/
│   │   ├── test-service-connectivity.yaml
│   │   ├── test-config-validation.yaml
│   │   └── test-readiness.yaml
│   ├── deployment.yaml
│   └── service.yaml
```

## Conditional Testing

Enable/disable tests globally:

```yaml
{{- if .Values.tests.enabled }}
apiVersion: v1
kind: Pod
metadata:
  name: "{{ .Release.Name }}-test"
  annotations:
    helm.sh/hook: test-success
spec:
  # ...
{{- end }}
```

In `values.yaml`:
```yaml
tests:
  enabled: true
```

Test specific configurations:

```yaml
{{- if .Values.metrics.enabled }}
# metrics test
{{- end }}
```

## Test Execution Order

Use `helm.sh/hook-weight` (lower runs first):

```yaml
# Test 1: Run first
metadata:
  annotations:
    helm.sh/hook-weight: "-5"

---
# Test 2: Run second
metadata:
  annotations:
    helm.sh/hook-weight: "0"
```

## Running and Debugging

### Running Tests

```bash
helm test my-release
helm test my-release --logs
helm test my-release --timeout 10m
helm test my-release -n my-namespace
```

### Debugging Failed Tests

```bash
kubectl get pods -n namespace -l helm.sh/hook=test
kubectl logs my-release-test-connectivity -n namespace
kubectl describe pod my-release-test-connectivity -n namespace
```

## Common Test Failures

| Symptom | Cause | Solution |
|---------|-------|----------|
| Image pull errors | Wrong image/registry | Verify image name and pull secrets |
| Connection refused | Service not ready | Add readiness test, increase timeout |
| Permission denied | Insufficient RBAC | Add service account and role |
| Command not found | Wrong base image | Use image with required tools |
| Timeout | Service startup too slow | Increase timeout or add retry logic |

## Anti-Patterns

### Don't Test During Installation

```yaml
# Wrong - runs during install
helm.sh/hook: post-install

# Correct - runs with helm test
helm.sh/hook: test-success
```

### Don't Over-Test Platform Guarantees

```yaml
# Wrong - Kubernetes guarantees this
command: ["kubectl", "get", "pod", "|", "grep", "myapp"]

# Correct - test application functionality
command: ["curl", "http://myapp-service/health"]
```

### Avoid Fragile Tests

```yaml
# Fragile - exact match
response=$(curl http://service/health)
[ "$response" == '{"status":"ok"}' ]

# Robust - content check
curl http://service/health | grep -q "status.*ok"
```

### Avoid Long-Running Tests

```yaml
# Avoid - waits 60 seconds
for i in $(seq 1 60); do sleep 1; done

# Prefer - quick check
curl -f --max-time 10 http://service/health
```
