---
name: k8s-namespace-troubleshooting
description: Use when a user reports a problem in a specific namespace, when investigating unhealthy workloads in a namespace, when diagnosing pod failures or restarts, when checking namespace events and resource status, when performing end-to-end namespace health assessment, or when triaging application issues inside Kubernetes
---

# Kubernetes Namespace Troubleshooting

Perform comprehensive, namespace-scoped investigation of Kubernetes workloads. Collect events, pod status, node health, resource consumption, configuration issues, and application logs, then summarise findings, diagnose root causes, assess impact, and prescribe remediation.

## Keywords

kubernetes, namespace, troubleshoot, troubleshooting, debug, diagnose, investigate, problem, issue, error, unhealthy, failing, broken, pod, deployment, statefulset, daemonset, job, cronjob, service, ingress, events, logs, restart, crashloop, pending, oomkill, eviction, namespace health, application, workload

## When to Use This Skill

- A user says "I have a problem in namespace X" or "something is wrong in namespace X"
- Pods are failing, restarting, pending, or evicted in a namespace
- Services or ingresses are not responding
- Applications are returning errors or degraded performance
- Namespace events show warnings or errors
- A general health check of everything running in a namespace is needed

## Related Skills

- [k8s-platform-operations](../k8s-platform-operations) - Cluster-wide health checks and incident response
- [k8s-security-hardening](../k8s-security-hardening) - Security context and policy issues
- [k8s-platform-tenancy](../k8s-platform-tenancy) - Resource quotas and limit ranges
- [k8s-continual-improvement](../k8s-continual-improvement) - SLOs and capacity planning
- [flux-troubleshooting](../flux-troubleshooting) - GitOps reconciliation failures
- [Shared: Pod Security Context](../_shared/references/pod-security-context.md)
- [Shared: Network Policies](../_shared/references/network-policies.md)

## Quick Reference

| Task | Command |
|------|---------|
| All resources in namespace | `kubectl get all -n ${NS}` |
| Warning events | `kubectl get events -n ${NS} --field-selector type=Warning --sort-by='.lastTimestamp'` |
| Unhealthy pods | `kubectl get pods -n ${NS} \| grep -Ev 'Running\|Completed'` |
| Pod resource usage | `kubectl top pods -n ${NS} --sort-by=memory` |
| Recent pod logs | `kubectl logs -n ${NS} deploy/${NAME} --tail=100` |
| Describe failing pod | `kubectl describe pod ${POD} -n ${NS}` |

---

## Investigation Workflow

This skill follows a five-phase workflow. Each phase builds on the previous one.

```
Phase 1: COLLECT     → Gather all namespace data (resources, events, status)
    ↓
Phase 2: SUMMARISE   → Produce a namespace health summary
    ↓
Phase 3: DIAGNOSE    → Identify root causes from collected evidence
    ↓
Phase 4: IMPACT      → Assess severity and blast radius
    ↓
Phase 5: REMEDIATE   → Prescribe fixes (automated or user-directed)
```

---

## Phase 1: Collect

Gather comprehensive data from the namespace. Run all commands targeting the user-provided namespace. Collect the output of every section below before moving to Phase 2.

### 1.1 Namespace Status and Configuration
```bash
# Confirm namespace exists and check labels/annotations
kubectl get namespace ${NS} -o yaml

# Check resource quotas (may explain scheduling failures)
kubectl get resourcequota -n ${NS}
kubectl describe resourcequota -n ${NS}

# Check limit ranges (may explain OOMKills or throttling)
kubectl get limitrange -n ${NS}
kubectl describe limitrange -n ${NS}
```

### 1.2 Events (Critical — Always Check First)
```bash
# All events sorted by time (most recent last)
kubectl get events -n ${NS} --sort-by='.lastTimestamp'

# Warning events only
kubectl get events -n ${NS} --field-selector type=Warning --sort-by='.lastTimestamp'

# Event counts to spot recurring issues
kubectl get events -n ${NS} -o json | \
  jq -r '[.items[] | {reason: .reason, name: .involvedObject.name, count: .count}] | group_by(.reason) | map({reason: .[0].reason, total: (map(.count) | add)}) | sort_by(-.total) | .[] | "\(.total)\t\(.reason)"'
```

### 1.3 Pods — Status, Restarts, and Conditions
```bash
# Full pod listing with status
kubectl get pods -n ${NS} -o wide

# Unhealthy pods (not Running or Completed)
kubectl get pods -n ${NS} | grep -Ev 'Running|Completed'

# High restart counts
kubectl get pods -n ${NS} -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .status.containerStatuses[*]}{.restartCount}{" "}{end}{"\n"}{end}' | awk '{sum=0; for(i=2;i<=NF;i++) sum+=$i; if(sum>0) print $1"\t"sum" restarts"}' | sort -t$'\t' -k2 -rn

# OOMKilled containers
kubectl get pods -n ${NS} -o json | \
  jq -r '.items[] | .metadata.name as $pod | .status.containerStatuses[]? | select(.lastState.terminated.reason == "OOMKilled") | "\($pod)\t\(.name)\tOOMKilled"'

# Pending pods — why are they not scheduled?
kubectl get pods -n ${NS} --field-selector=status.phase=Pending -o name | \
  xargs -I{} kubectl describe {} -n ${NS} | grep -A5 "Events:"

# Container image details (spot wrong tags, missing images)
kubectl get pods -n ${NS} -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .spec.containers[*]}{.image}{" "}{end}{"\n"}{end}'
```

### 1.4 Workload Controllers
```bash
# Deployments — check desired vs ready replicas
kubectl get deployments -n ${NS} -o wide

# Deployments not at full availability
kubectl get deployments -n ${NS} -o json | \
  jq -r '.items[] | select(.status.availableReplicas != .status.replicas) | "\(.metadata.name)\tavailable=\(.status.availableReplicas // 0)/\(.status.replicas)"'

# StatefulSets
kubectl get statefulsets -n ${NS} -o wide

# DaemonSets — check desired vs ready
kubectl get daemonsets -n ${NS} -o wide

# Jobs — check for failures
kubectl get jobs -n ${NS}
kubectl get jobs -n ${NS} -o json | \
  jq -r '.items[] | select(.status.failed > 0) | "\(.metadata.name)\tfailed=\(.status.failed)"'

# CronJobs — check schedule and last run
kubectl get cronjobs -n ${NS}
```

### 1.5 Services, Endpoints, and Networking
```bash
# Services
kubectl get services -n ${NS} -o wide

# Endpoints — empty endpoints mean no backing pods
kubectl get endpoints -n ${NS}
kubectl get endpoints -n ${NS} -o json | \
  jq -r '.items[] | select((.subsets // []) | length == 0) | "\(.metadata.name)\tNO ENDPOINTS"'

# Ingresses
kubectl get ingress -n ${NS}

# Network policies (may be blocking traffic)
kubectl get networkpolicies -n ${NS}
```

### 1.6 Configuration — ConfigMaps, Secrets, PVCs
```bash
# ConfigMaps
kubectl get configmaps -n ${NS}

# Secrets (names only — never print secret data)
kubectl get secrets -n ${NS}

# PersistentVolumeClaims — check bound status and capacity
kubectl get pvc -n ${NS}
kubectl get pvc -n ${NS} -o json | \
  jq -r '.items[] | select(.status.phase != "Bound") | "\(.metadata.name)\t\(.status.phase)"'
```

### 1.7 Resource Consumption
```bash
# Pod CPU and memory (requires metrics-server)
kubectl top pods -n ${NS} --sort-by=memory
kubectl top pods -n ${NS} --sort-by=cpu

# Requests vs limits vs actual (identify over/under provisioning)
kubectl get pods -n ${NS} -o json | \
  jq -r '.items[] | .metadata.name as $pod | .spec.containers[] | "\($pod)\t\(.name)\treq_cpu=\(.resources.requests.cpu // "none")\tlim_cpu=\(.resources.limits.cpu // "none")\treq_mem=\(.resources.requests.memory // "none")\tlim_mem=\(.resources.limits.memory // "none")"'
```

### 1.8 Node Health (for Nodes Running Namespace Pods)
```bash
# Find which nodes host pods in this namespace
kubectl get pods -n ${NS} -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort -u

# Check conditions on those nodes
kubectl get pods -n ${NS} -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort -u | \
  xargs -I{} sh -c 'echo "--- {} ---" && kubectl describe node {} | grep -A5 "Conditions:"'

# Node resource pressure
kubectl get pods -n ${NS} -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort -u | \
  xargs -I{} kubectl top node {}
```

### 1.9 Application Logs
```bash
# Logs from all pods in the namespace (last 100 lines each)
for pod in $(kubectl get pods -n ${NS} -o jsonpath='{.items[*].metadata.name}'); do
  echo "=== ${pod} ==="
  kubectl logs -n ${NS} ${pod} --all-containers --tail=100 2>&1
done

# Previous container logs (for crashed/restarted containers)
for pod in $(kubectl get pods -n ${NS} -o jsonpath='{.items[*].metadata.name}'); do
  echo "=== ${pod} (previous) ==="
  kubectl logs -n ${NS} ${pod} --all-containers --previous --tail=50 2>&1 || echo "No previous logs"
done

# Grep for common error patterns across all pod logs
for pod in $(kubectl get pods -n ${NS} -o jsonpath='{.items[*].metadata.name}'); do
  kubectl logs -n ${NS} ${pod} --all-containers --tail=200 2>/dev/null | \
    grep -iE 'error|exception|fatal|panic|timeout|refused|denied|failed|oom|kill' | \
    head -20 | sed "s/^/${pod}: /"
done
```

### 1.10 HPA and VPA (Autoscaling)
```bash
# Horizontal Pod Autoscalers
kubectl get hpa -n ${NS}
kubectl describe hpa -n ${NS}

# Vertical Pod Autoscalers (if installed)
kubectl get vpa -n ${NS} 2>/dev/null
```

### 1.11 Flux / GitOps Resources (If Present)
```bash
# Check if any Flux resources target this namespace
kubectl get kustomizations.kustomize.toolkit.fluxcd.io -A -o json | \
  jq -r '.items[] | select(.spec.targetNamespace == "'${NS}'" or .metadata.namespace == "'${NS}'") | "\(.metadata.namespace)/\(.metadata.name)\tReady=\(.status.conditions[-1].status)\tMessage=\(.status.conditions[-1].message)"' 2>/dev/null

kubectl get helmreleases.helm.toolkit.fluxcd.io -n ${NS} 2>/dev/null
```

---

## Phase 2: Summarise

After collecting data, produce a structured health summary. This summary must be presented to the user before deeper diagnosis.

### Namespace Health Summary Template

```markdown
## Namespace Health Summary: ${NS}

### Overview
- **Total pods**: X (Y running, Z unhealthy)
- **Deployments**: X/Y at desired replicas
- **StatefulSets**: X/Y at desired replicas
- **Jobs**: X succeeded, Y failed
- **Services**: X total, Y with empty endpoints
- **PVCs**: X bound, Y unbound
- **Warning events**: X in last hour

### Unhealthy Resources
| Resource | Type | Status | Issue |
|----------|------|--------|-------|
| pod-name | Pod | CrashLoopBackOff | Exit code 1, 47 restarts |
| deploy-name | Deployment | 1/3 ready | 2 pods Pending |
| pvc-name | PVC | Pending | No matching PV |

### Resource Consumption
| Pod | CPU (actual/limit) | Memory (actual/limit) | Pressure |
|-----|--------------------|-----------------------|----------|
| pod-a | 450m/500m | 490Mi/512Mi | HIGH |
| pod-b | 10m/500m | 50Mi/512Mi | LOW |

### Recent Warning Events (Top 5)
| Count | Reason | Object | Message |
|-------|--------|--------|---------|
| 23 | BackOff | pod/app-x | Back-off restarting failed container |
| 12 | FailedScheduling | pod/app-y | Insufficient memory |

### Node Health (Hosting Namespace Pods)
| Node | CPU% | Mem% | Conditions |
|------|------|------|------------|
| node-1 | 82% | 91% | MemoryPressure=True |
| node-2 | 45% | 60% | Ready=True |
```

---

## Phase 3: Diagnose

Analyse the collected evidence to identify root causes. Work through this decision tree for each unhealthy resource.

### Pod Diagnostic Decision Tree

```
Pod not Running?
├─ Pending
│   ├─ "Insufficient cpu/memory" → Node capacity or resource quota exhausted
│   ├─ "no nodes match selector" → Node affinity/selector mismatch
│   ├─ "persistentvolumeclaim not found" → PVC missing or unbound
│   ├─ "0/N nodes are available" → Check taints, tolerations, affinity
│   └─ No events at all → Scheduler may be down
├─ CrashLoopBackOff
│   ├─ Exit code 1 → Application error (check logs)
│   ├─ Exit code 126/127 → Command not found (wrong image or entrypoint)
│   ├─ Exit code 137 → OOMKilled (raise memory limits)
│   ├─ Exit code 139 → Segfault (application bug or incompatible image)
│   └─ Exit code 143 → SIGTERM (graceful shutdown issue)
├─ ImagePullBackOff
│   ├─ "not found" → Wrong image name or tag
│   ├─ "unauthorized" → Missing or wrong imagePullSecret
│   └─ "timeout" → Registry unreachable (network/DNS)
├─ Init:Error / Init:CrashLoopBackOff
│   └─ Init container failing → Check init container logs separately
├─ CreateContainerConfigError
│   ├─ "secret not found" → Referenced Secret missing
│   └─ "configmap not found" → Referenced ConfigMap missing
├─ Evicted
│   └─ Node under pressure → Check node conditions, DiskPressure/MemoryPressure
└─ Terminating (stuck)
    ├─ Finalizer blocking deletion → Check finalizers
    └─ Process not responding to SIGTERM → Check graceful shutdown
```

### Service Connectivity Diagnostic

```
Service not reachable?
├─ Endpoints empty?
│   ├─ Yes → Label selector mismatch between Service and Pods
│   └─ No → Endpoints exist, check:
│       ├─ Port mismatch (service port vs container port)
│       ├─ Network policy blocking traffic
│       ├─ Pod readiness probe failing
│       └─ DNS resolution issue
├─ Ingress not routing?
│   ├─ Ingress class missing or wrong
│   ├─ TLS certificate issue
│   ├─ Backend service name/port mismatch
│   └─ Ingress controller not running
```

### Common Root Cause Patterns

| Symptom | Likely Root Cause | Evidence to Confirm |
|---------|-------------------|---------------------|
| All pods Pending | ResourceQuota exhausted | `kubectl describe resourcequota -n ${NS}` shows at limit |
| Pods OOMKilled | Memory limits too low | Container `lastState.terminated.reason == OOMKilled` |
| Pods CrashLooping | Application error | Exit code 1 + error in logs |
| Pods ImagePullBackOff | Wrong image/tag or missing secret | Event message contains "not found" or "unauthorized" |
| Service 503s | No ready endpoints | `kubectl get endpoints` shows empty subsets |
| PVC Pending | No matching PV or StorageClass | Event "no persistent volumes available" |
| Intermittent failures | Node under pressure | Node conditions show pressure flags |
| Slow response | CPU throttling | CPU usage near limits, high throttle counts |
| Connection refused | Container port mismatch | Service targetPort != container port |
| Pods scheduled but not starting | ConfigMap/Secret missing | CreateContainerConfigError in events |

---

## Phase 4: Impact Assessment

For each identified problem, assess its severity and blast radius.

### Severity Classification

| Severity | Criteria | Response |
|----------|----------|----------|
| **Critical** | All replicas down, data loss risk, entire namespace non-functional | Immediate action required |
| **High** | Degraded availability, partial outage, >50% replicas unhealthy | Action within 1 hour |
| **Medium** | Single replica down (redundancy covering), elevated restarts | Action within 4 hours |
| **Low** | Warning events, non-critical resource pressure, cosmetic issues | Action within 24 hours |

### Impact Assessment Template

```markdown
### Impact Assessment

| Problem | Severity | Affected Resources | User Impact | Blast Radius |
|---------|----------|--------------------|-------------|--------------|
| OOMKill on api-server | Critical | deploy/api (0/3 ready) | API completely unavailable | All consumers of this API |
| PVC Pending for cache | Medium | sts/redis (1/3 ready) | Degraded cache performance | Requests hitting this shard |
| CronJob failing | Low | cronjob/cleanup | Stale data accumulating | No immediate user impact |
```

---

## Phase 5: Remediate

For each diagnosed problem, prescribe remediation steps. Clearly distinguish between actions the skill can take directly and those requiring user decision or cluster-admin intervention.

### Automated Remediation (Skill Can Execute)

These actions are safe to perform directly when the diagnosis is clear:

```bash
# Restart a stuck deployment (safe — rolling restart, no downtime)
kubectl rollout restart deployment/${NAME} -n ${NS}

# Delete a pod stuck in Terminating (force)
kubectl delete pod ${POD} -n ${NS} --grace-period=0 --force

# Delete Evicted pods (cleanup)
kubectl get pods -n ${NS} --field-selector=status.phase=Failed | \
  grep Evicted | awk '{print $1}' | xargs kubectl delete pod -n ${NS}

# Delete failed Jobs (cleanup)
kubectl delete jobs -n ${NS} --field-selector=status.successful=0

# Scale a deployment to trigger fresh pod scheduling
kubectl scale deployment/${NAME} -n ${NS} --replicas=0
kubectl scale deployment/${NAME} -n ${NS} --replicas=${DESIRED}

# Trigger rollout of latest ReplicaSet (if rollout stuck)
kubectl rollout undo deployment/${NAME} -n ${NS}
kubectl rollout status deployment/${NAME} -n ${NS}
```

### User-Directed Remediation

These actions require user decision, access, or changes to source manifests:

| Problem | Remediation | Action Required By |
|---------|-------------|--------------------|
| OOMKilled | Increase memory limits in deployment manifest | User (manifest change) |
| ResourceQuota exhausted | Request quota increase or reduce workload | Namespace owner / platform team |
| Image not found | Correct image name/tag in deployment manifest | User (manifest change) |
| ImagePullSecret missing | Create secret: `kubectl create secret docker-registry` | User (credentials needed) |
| PVC Pending (no PV) | Provision storage or check StorageClass availability | Platform team |
| Node pressure | Scale cluster or redistribute workloads | Platform team |
| Network policy blocking | Review and update NetworkPolicy rules | User / security team |
| CPU throttling | Increase CPU limits or optimise application | User (manifest or code change) |
| Readiness probe failing | Fix probe configuration or application health endpoint | User (manifest or code change) |
| Ingress misconfigured | Correct ingress rules, TLS config, or backend service ref | User (manifest change) |
| ConfigMap/Secret missing | Create the missing ConfigMap or Secret resource | User (may need values from team) |
| CronJob schedule wrong | Update cron expression in CronJob manifest | User (manifest change) |
| Stuck finalizer | Remove finalizer from resource (with caution) | User (understand why it exists first) |

### Remediation Report Template

```markdown
## Remediation Report: ${NS}

### Immediate Actions Taken
| Action | Resource | Result |
|--------|----------|--------|
| Restarted deployment | deploy/api-server | Pods coming up |
| Cleaned evicted pods | 12 pods removed | Namespace cleaned |

### Actions Requiring User Intervention
| Priority | Action | Details | Who |
|----------|--------|---------|-----|
| Critical | Increase memory limit | api-server OOMKilled at 512Mi, recommend 1Gi | App team |
| High | Fix image tag | deploy/worker using tag `v2.3.1` which doesn't exist | App team |
| Medium | Request quota increase | CPU quota 90% consumed, 2 pods Pending | Platform team |

### Preventive Recommendations
- Set up PodDisruptionBudgets for critical deployments
- Configure resource requests and limits for all containers
- Add readiness and liveness probes
- Implement HPA for variable-load workloads
- Review and right-size resource requests based on actual usage
```

---

## Common Error Patterns — Quick Diagnosis

A quick-lookup table for when the user describes a specific symptom:

| User Says | Check First | Command |
|-----------|-------------|---------|
| "Pods keep restarting" | Restart counts and OOMKill | `kubectl get pods -n ${NS}` + describe top restarter |
| "App is slow" | CPU throttling and resource usage | `kubectl top pods -n ${NS}` + check limits |
| "Can't connect to service" | Endpoints and network policies | `kubectl get endpoints -n ${NS}` |
| "Pods won't start" | Events for scheduling failures | `kubectl get events -n ${NS} --field-selector type=Warning` |
| "Deployment stuck" | Rollout status | `kubectl rollout status deploy/${NAME} -n ${NS}` |
| "Out of space" | PVC usage and node disk pressure | `kubectl get pvc -n ${NS}` + node conditions |
| "Permission denied" | RBAC and security context | `kubectl auth can-i --list -n ${NS}` |
| "Image pull error" | Image name and pull secrets | `kubectl describe pod ${POD} -n ${NS}` |
| "Namespace feels broken" | Full Phase 1 collection | Run all collection commands |

---

## MCP Tools Available

When the appropriate MCP servers are connected, prefer these over raw kubectl where available:

- `mcp__flux-operator-mcp__get_kubernetes_resources` - Query any Kubernetes resource
- `mcp__flux-operator-mcp__get_kubernetes_logs` - Retrieve pod logs
- `mcp__flux-operator-mcp__get_kubernetes_metrics` - Get resource consumption metrics

### MCP Collection Strategy

When MCP tools are available, use this sequence instead of kubectl:

```
1. get_kubernetes_resources(kind=Namespace, name=${NS})           → Namespace config
2. get_kubernetes_resources(kind=Event, namespace=${NS})          → All events
3. get_kubernetes_resources(kind=Pod, namespace=${NS})             → Pod status
4. get_kubernetes_resources(kind=Deployment, namespace=${NS})      → Controller status
5. get_kubernetes_resources(kind=StatefulSet, namespace=${NS})     → StatefulSet status
6. get_kubernetes_resources(kind=Service, namespace=${NS})         → Services
7. get_kubernetes_resources(kind=Endpoints, namespace=${NS})       → Endpoint health
8. get_kubernetes_resources(kind=Ingress, namespace=${NS})         → Ingress config
9. get_kubernetes_resources(kind=PersistentVolumeClaim, namespace=${NS}) → Storage
10. get_kubernetes_resources(kind=ResourceQuota, namespace=${NS})  → Quotas
11. get_kubernetes_metrics(namespace=${NS})                        → Resource usage
12. get_kubernetes_logs(namespace=${NS}, pod=${UNHEALTHY_POD})     → Targeted logs
```

---

## Behavioural Guidelines

When using this skill, follow these principles:

1. **Always collect before concluding** — Run Phase 1 fully. Do not guess root causes from partial data.
2. **Present the summary first** — Give the user the Phase 2 health summary before diving into diagnosis. Let them see the landscape.
3. **Be explicit about severity** — Every identified problem must have a severity rating from Phase 4.
4. **Distinguish auto-fix from user-fix** — Clearly separate what you can safely do from what requires the user's decision.
5. **Never print secret values** — List secret names only. Never decode or display secret data.
6. **Respect the namespace boundary** — This is a namespace-scoped investigation. Only look at nodes or cluster resources when they directly affect this namespace's workloads.
7. **Handle empty namespaces** — If the namespace has no resources, say so. It may be newly created, or everything may have been deleted.
8. **Handle large namespaces** — If there are >50 pods, focus on unhealthy resources first. Summarise healthy resources in aggregate.
9. **Correlate across resources** — A pod failure may be caused by a node issue, a missing secret, or a quota limit. Cross-reference findings.
10. **Recommend prevention** — After fixing immediate issues, suggest probes, PDBs, resource limits, and monitoring to prevent recurrence.
