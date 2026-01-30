---
name: helm-chart-testing
description: Use when creating Helm test manifests, analyzing what to test in a Helm chart, writing smoke tests for Helm releases, implementing connectivity tests, validating configuration in deployed resources, or troubleshooting failing Helm tests
---

# Helm Chart Testing

Develop and write effective built-in Helm tests using the native Helm test framework, from analyzing what to test through implementing test manifests.

## Keywords

helm, test, testing, helm test, smoke test, validation, connectivity, test-success, test-failure, hook, helm.sh/hook, pod test, test annotation, release verification, post-install test, post-upgrade test

## When to Use This Skill

- Analyzing a Helm chart to determine what should be tested
- Writing Helm test manifests with pod annotations
- Creating smoke tests for new releases
- Implementing service connectivity tests
- Validating configuration in deployed resources
- Troubleshooting failing Helm tests
- Designing test strategies for complex charts

## Related Skills

- [helm-chart-development](../helm-chart-development) - Chart structure and templating
- [helm-chart-maintenance](../helm-chart-maintenance) - Testing in CI/CD pipelines
- [helm-chart-review](../helm-chart-review) - Reviewing test coverage and quality
- [k8s-platform-operations](../k8s-platform-operations) - Debugging test failures

## Quick Reference

| Task | Command/Approach |
|------|-----------------|
| Run all tests | `helm test RELEASE_NAME` |
| Run tests with logs | `helm test RELEASE_NAME --logs` |
| Run tests with timeout | `helm test RELEASE_NAME --timeout 5m` |
| Identify testable surface area | Analyze Service, Ingress, Deployment resources |
| Test service connectivity | `curl`/`wget` from test pod to Service endpoints |
| Test deployment readiness | Check rollout status and pod readiness probes |
| Test configuration | Verify ConfigMaps/Secrets mounted correctly |
| Keep test pods for debugging | `helm.sh/hook-delete-policy: hook-succeeded` |

## Overview

The Helm test framework allows you to define tests as Pod templates that run after a release. Tests are marked with special annotations and executed using the `helm test` command. This skill focuses on **designing and writing effective tests** rather than just executing them.

### How Helm Tests Work

1. **Define test pods** in your chart's `templates/` directory with the `helm.sh/hook: test` annotation
2. **Install the chart** normally with `helm install` or `helm upgrade`
3. **Run tests** with `helm test RELEASE_NAME`
4. Helm creates the test pods, executes them, and reports results

### Test Annotations

| Annotation | Purpose |
|------------|---------|
| `helm.sh/hook: test` | Marks pod as a test (runs during `helm test`) |
| `helm.sh/hook: test-success` | Runs after successful release |
| `helm.sh/hook: test-failure` | Runs after failed release |
| `helm.sh/hook-weight:` | Controls test execution order |
| `helm.sh/hook-delete-policy:` | Controls cleanup behavior |

## Test Analysis & Planning

Before writing tests, analyze your chart to determine what needs testing.

### What to Test by Resource Type

| Resource Type | Test Considerations |
|---------------|---------------------|
| **Services** | Endpoint reachability, correct ports, DNS resolution |
| **Deployments/StatefulSets** | Pod readiness, replica count, rollout status |
| **Ingress** | Route reachability, TLS certificates, host headers |
| **ConfigMaps/Secrets** | Values present, mounted correctly, not malformed |
| **PersistentVolumeClaims** | Volume mounted, read/write access |
| **CRDs** | Custom resource creation and reconciliation |
| **Jobs/CronJobs** | Job completion status, retry behavior |
| **ServiceAccounts/RBAC** | Permissions work as expected |

### Analysis Questions

When planning tests for a chart, ask:

- What is the **minimum viable verification** that the release worked?
- What are the **critical failure modes**?
- What **external dependencies** does the chart have?
- What configuration is **required vs. optional**?
- Can tests run **safely in production** environments?
- What would a user want to **verify immediately** after installation?

### Test Categories

| Category | Purpose | Examples |
|----------|---------|----------|
| **Smoke Tests** | Quick "is it alive" checks | Service health, pod readiness |
| **Functional Tests** | Verify specific behavior | API responses, database connectivity |
| **Integration Tests** | Verify external interactions | Upstream services, third-party APIs |
| **Data Validation** | Verify deployed state | ConfigMap content, environment variables |
| **Regression Tests** | Catch breaking changes | API contract validation |

## Writing Test Manifests

### Basic Test Pod Structure

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "myapp.fullname" . }}-test-connectivity"
  annotations:
    helm.sh/hook: test-success
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
spec:
  containers:
  - name: test
    image: curlimages/curl:latest
    command:
    - sh
    - -c
    - |
      # Test commands here
      curl -f http://myapp-service:8080/health
  restartPolicy: Never
```

### Test Pod Best Practices

1. **Use `restartPolicy: Never`** - Tests should run once and exit
2. **Set cleanup policy** - Usually `hook-succeeded` to clean up passing tests
3. **Use appropriate images** - Lightweight images (alpine, busybox, curlimages)
4. **Exit code 0 = pass** - Any non-zero exit code marks the test as failed
5. **Keep tests focused** - One test pod should test one thing well

### Common Test Container Images

| Image | Use Case |
|-------|----------|
| `curlimages/curl` | HTTP endpoint checks |
| `busybox` | Basic shell utilities |
| `alpine` | Shell with more tools |
| `bitnami/kubectl` | Kubernetes API queries |
| `nicolaka/netshoot` | Network debugging |
| `postgres:alpine` | Database connectivity |
| Language-specific | Complex validation logic |

## Test Implementation Examples

### Service Connectivity Test

Verifies that a service is reachable and responds correctly:

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
    image: curlimages/curl:latest
    command:
    - sh
    - -c
    - |
      set -e

      echo "Testing service endpoint..."

      # Test main endpoint is reachable
      curl -f http://{{ .Release.Name }}-service:8080/health || exit 1

      # Test with expected response content
      response=$(curl -s http://{{ .Release.Name }}-service:8080/health)
      echo "$response" | grep -q "status.*ok" || exit 1

      # Test secured endpoint (if auth enabled)
      {{- if .Values.auth.enabled }}
      echo "Testing authenticated endpoint..."
      curl -f http://{{ .Release.Name }}-service:8080/secure \
        -H "Authorization: Bearer {{ .Values.auth.testToken }}" || exit 1
      {{- end }}

      echo "All connectivity tests passed!"
  restartPolicy: Never
```

### Configuration Validation Test

Verifies that configuration is correctly applied:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ .Release.Name }}-config-test"
  annotations:
    helm.sh/hook: test-success
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
spec:
  containers:
  - name: config-test
    image: busybox:1.36
    command:
    - sh
    - -c
    - |
      set -e

      echo "Validating configuration..."

      # Check config file exists
      test -f /app/config.yaml || exit 1

      # Check expected values are present
      grep -q "logLevel: {{ .Values.logLevel }}" /app/config.yaml || exit 1
      grep -q "port: {{ .Values.service.port }}" /app/config.yaml || exit 1

      # Check required sections exist
      grep -q "database:" /app/config.yaml || exit 1
      grep -q "host:" /app/config.yaml || exit 1

      echo "Configuration validated successfully!"
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

Verifies deployment rollout is complete:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ .Release.Name }}-readiness-test"
  annotations:
    helm.sh/hook: test-success
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
spec:
  serviceAccountName: {{ include "myapp.fullname" . }}-test-sa
  containers:
  - name: kubectl-test
    image: bitnami/kubectl:latest
    command:
    - sh
    - -c
    - |
      set -e

      echo "Checking deployment readiness..."

      # Verify deployment is ready
      kubectl get deployment {{ .Release.Name }} -n {{ .Release.Namespace }} -o json | \
        jq -e '.status.readyReplicas == {{ .Values.replicaCount }}' || exit 1

      # Verify all pods are running
      kubectl get pods -n {{ .Release.Namespace }} -l app={{ include "myapp.name" . }} -o json | \
        jq -e '.items | length == {{ .Values.replicaCount }}' || exit 1

      # Verify no pods are failing
      kubectl get pods -n {{ .Release.Namespace }} -l app={{ include "myapp.name" . }} -o json | \
        jq -e '[.items[].status.phase | select(. != "Running")] | length == 0' || exit 1

      echo "Deployment is ready!"
  restartPolicy: Never
```

### Database Connection Test

Verifies database connectivity:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ .Release.Name }}-db-test"
  annotations:
    helm.sh/hook: test-success
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
spec:
  containers:
  - name: db-test
    image: postgres:16-alpine
    command:
    - sh
    - -c
    - |
      set -e

      echo "Testing database connection..."

      # Test TCP connectivity
      nc -zv {{ .Values.database.host }} {{ .Values.database.port }} || exit 1

      # Test database connection and query
      PGPASSWORD={{ .Values.database.password }} \
      psql -h {{ .Values.database.host }} \
            -p {{ .Values.database.port }} \
            -U {{ .Values.database.user }} \
            -d {{ .Values.database.name }} \
            -c "SELECT 1;" || exit 1

      # Verify expected schema exists
      PGPASSWORD={{ .Values.database.password }} \
      psql -h {{ .Values.database.host }} \
            -p {{ .Values.database.port }} \
            -U {{ .Values.database.user }} \
            -d {{ .Values.database.name }} \
            -c "SELECT EXISTS (SELECT FROM information_schema.tables WHERE table_name = '{{ .Values.database.mainTable }}');" | \
            grep -q "t" || exit 1

      echo "Database connection successful!"
  restartPolicy: Never
```

### TLS/HTTPS Test

Verifies TLS certificate and secure endpoints:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ .Release.Name }}-tls-test"
  annotations:
    helm.sh/hook: test-success
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
spec:
  containers:
  - name: tls-test
    image: curlimages/curl:latest
    command:
    - sh
    - -c
    - |
      set -e

      echo "Testing TLS endpoint..."

      # Test HTTPS is reachable
      curl -f https://{{ .Values.ingress.host }}/health || exit 1

      # Verify certificate is valid (not expired)
      cert_info=$(openssl s_client -connect {{ .Values.ingress.host }}:443 -servername {{ .Values.ingress.host }} </dev/null 2>/dev/null | openssl x509 -noout -dates)
      echo "$cert_info" | grep -q "notAfter" || exit 1

      # Test with SNI (if multiple certificates)
      curl -f https://{{ .Values.ingress.host }}/health \
        --resolve {{ .Values.ingress.host }}:443:$(getent hosts {{ .Values.ingress.host }} | awk '{print $1}') || exit 1

      echo "TLS test passed!"
  restartPolicy: Never
```

### Storage Read/Write Test

Verifies persistent volume mounting and write access:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ .Release.Name }}-storage-test"
  annotations:
    helm.sh/hook: test-success
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
spec:
  containers:
  - name: storage-test
    image: busybox:1.36
    command:
    - sh
    - -c
    - |
      set -e

      echo "Testing persistent storage..."

      # Check mount point exists
      test -d /data || exit 1

      # Test write access
      echo "$(date)" > /data/test-write.txt || exit 1

      # Test read back
      test -f /data/test-write.txt || exit 1
      grep -q "$(date +%Y)" /data/test-write.txt || exit 1

      # Clean up
      rm /data/test-write.txt

      echo "Storage test passed!"
  volumeMounts:
  - name: data
    mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: {{ include "myapp.fullname" . }}-data
  restartPolicy: Never
```

## Test Design Considerations

### Test Independence

Each test should verify one thing well and not depend on other tests:

```yaml
# Good: Single focused test
apiVersion: v1
kind: Pod
metadata:
  name: "{{ .Release.Name }}-test-health"
  annotations:
    helm.sh/hook: test-success
spec:
  # Tests only health endpoint
```

```yaml
# Avoid: Tests multiple unrelated things
apiVersion: v1
kind: Pod
metadata:
  name: "{{ .Release.Name }}-test-everything"
  annotations:
    helm.sh/hook: test-success
spec:
  # Tests health, config, database, storage all together
```

### Resource Management

Set appropriate limits to prevent resource exhaustion:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ .Release.Name }}-test"
  annotations:
    helm.sh/hook: test-success
spec:
  containers:
  - name: test
    image: curlimages/curl:latest
    resources:
      requests:
        cpu: 100m
        memory: 64Mi
      limits:
        cpu: 200m
        memory: 128Mi
    command: ...
  restartPolicy: Never
```

### Cleanup Policies

| Policy | Behavior | Use Case |
|--------|----------|----------|
| `hook-succeeded` | Delete after passing test | Normal operation |
| `hook-failed` | Delete after failing test | Cleanup failed tests |
| `before-hook-creation` | Delete before new test | Prevent conflicts |
| `never` | Never delete automatically | Debugging |

For debugging, use `never` or omit cleanup policy:

```yaml
metadata:
  annotations:
    helm.sh/hook: test-success
    helm.sh/hook-delete-policy: never
```

### Error Handling

Provide clear failure messages:

```yaml
command:
- sh
- -c
- |
  set -e  # Exit on any error

  echo "Testing service connectivity..."

  if ! curl -f http://service:8080/health; then
    echo "ERROR: Service health check failed"
    echo "Troubleshooting steps:"
    echo "1. Check if service is running: kubectl get svc"
    echo "2. Check pod logs: kubectl logs -l app=myapp"
    exit 1
  fi

  echo "Service is healthy!"
```

### Security Considerations

**Use least privilege for test service accounts:**

```yaml
# Define minimal RBAC for tests
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ include "myapp.fullname" . }}-test-sa
  annotations:
    helm.sh/hook: pre-install,pre-upgrade
    helm.sh/hook-weight: "-5"
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

**Don't embed secrets in test manifests:**

```yaml
# Avoid: Secrets in test
command:
- sh
- -c
- curl -H "Authorization: Bearer super-secret-key" http://service/

# Better: Use pre-existing test tokens or create test-specific credentials
command:
- sh
- -c
- curl -H "Authorization: Bearer $TEST_TOKEN" http://service/
env:
- name: TEST_TOKEN
  valueFrom:
    secretKeyRef:
      name: test-credentials
      key: token
```

## Organizing Tests in a Chart

### Flat Structure (Simple Charts)

```
mychart/
├── templates/
│   ├── tests/
│   │   ├── test-service-connectivity.yaml
│   │   ├── test-config-validation.yaml
│   │   ├── test-database-connection.yaml
│   │   └── test-readiness.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ...
```

### Grouped by Test Type (Complex Charts)

```
mychart/
├── templates/
│   ├── tests/
│   │   ├── smoke-tests/
│   │   │   └── test-basic-connectivity.yaml
│   │   ├── functional-tests/
│   │   │   ├── test-api-endpoints.yaml
│   │   │   └── test-data-persistence.yaml
│   │   └── integration-tests/
│   │       └── test-external-services.yaml
```

## Conditional Testing

### Enable/Disable Tests Globally

```yaml
{{- if .Values.tests.enabled }}
apiVersion: v1
kind: Pod
metadata:
  name: "{{ .Release.Name }}-test"
  annotations:
    helm.sh/hook: test-success
spec:
  # ... test definition
{{- end }}
```

In `values.yaml`:
```yaml
tests:
  enabled: true
```

### Test Specific Configurations

```yaml
{{- if .Values.metrics.enabled }}
apiVersion: v1
kind: Pod
metadata:
  name: "{{ .Release.Name }}-metrics-test"
  annotations:
    helm.sh/hook: test-success
spec:
  containers:
  - name: metrics-test
    image: curlimages/curl:latest
    command:
    - sh
    - -c
    - |
      curl -f http://{{ .Release.Name }}-service:{{ .Values.metrics.port }}/metrics
  restartPolicy: Never
{{- end }}
```

### Test Execution Order

Use `helm.sh/hook-weight` to control execution order:

```yaml
# Test 1: Run first (basic connectivity)
apiVersion: v1
kind: Pod
metadata:
  name: "{{ .Release.Name }}-test-connectivity"
  annotations:
    helm.sh/hook: test-success
    helm.sh/hook-weight: "-5"
spec:
  # ...

---
# Test 2: Run second (depends on connectivity)
apiVersion: v1
kind: Pod
metadata:
  name: "{{ .Release.Name }}-test-api"
  annotations:
    helm.sh/hook: test-success
    helm.sh/hook-weight: "0"
spec:
  # ...

---
# Test 3: Run last (complex integration)
apiVersion: v1
kind: Pod
metadata:
  name: "{{ .Release.Name }}-test-integration"
  annotations:
    helm.sh/hook: test-success
    helm.sh/hook-weight: "5"
spec:
  # ...
```

## Running and Debugging Tests

### Running Tests

```bash
# Run all tests for a release
helm test my-release

# Run tests and show logs
helm test my-release --logs

# Run tests with custom timeout
helm test my-release --timeout 10m

# Run tests in specific namespace
helm test my-release -n my-namespace

# Run tests from specific revision
helm test my-release --revision 2
```

### Debugging Failed Tests

```bash
# Get test pod status
kubectl get pods -n namespace -l helm.sh/hook=test

# View test pod logs
kubectl logs my-release-test-connectivity -n namespace

# Describe test pod for events
kubectl describe pod my-release-test-connectivity -n namespace

# Keep test pods for inspection
# In test manifest:
# helm.sh/hook-delete-policy: never
```

### Common Test Failures

| Symptom | Common Cause | Solution |
|---------|--------------|----------|
| Image pull errors | Wrong image name/registry | Verify image name and pull secrets |
| Connection refused | Service not ready yet | Add readiness test, increase timeout |
| Permission denied | Insufficient RBAC | Add service account and role |
| Command not found | Wrong base image | Use image with required tools |
| Timeout | Service startup too slow | Increase test timeout or add retry logic |

## Test Anti-Patterns

### Avoid Testing During Chart Installation

Tests should run with `helm test`, not during install/upgrade:

```yaml
# Don't do this - tests run during install
metadata:
  annotations:
    helm.sh/hook: post-install

# Do this - tests run with helm test
metadata:
  annotations:
    helm.sh/hook: test-success
```

### Avoid Over-Testing

Don't test what the platform already guarantees:

```yaml
# Don't test this - Kubernetes guarantees it
command:
- sh
- -c
- kubectl get pod | grep myapp

# Instead test application-level functionality
command:
- sh
- -c
- curl http://myapp-service/health
```

### Avoid Fragile Tests

```yaml
# Fragile: Depends on exact output format
command:
- sh
- -c
- response=$(curl http://service/health)
  [ "$response" == '{"status":"ok"}' ]

# Robust: Checks for expected content
command:
- sh
- -c
- curl http://service/health | grep -q "status.*ok"
```

### Avoid Long-Running Tests

```yaml
# Avoid: Tests that take minutes
command:
- sh
- -c
- for i in $(seq 1 60); do sleep 1; done

# Prefer: Quick validation checks
command:
- sh
- -c
- curl -f --max-time 10 http://service/health
```
