---
name: k8s-platform-operations
description: Use when performing cluster health checks, responding to incidents and alerts, planning and managing capacity, conducting maintenance operations, managing backups and recovery, or creating and following runbooks
---

# Kubernetes Platform Operations

Manage day-2 operations of Kubernetes platforms including monitoring, incident response, capacity planning, and operational excellence.

## Keywords

kubernetes, operations, monitoring, incident, capacity, maintenance, backup, recovery, health check, runbook, on-call, escalation, node, pod, troubleshooting, performing, responding, planning, managing, conducting, creating

## When to Use This Skill

- Performing cluster health checks
- Responding to incidents and alerts
- Planning and managing capacity
- Conducting maintenance operations
- Managing backups and recovery
- Creating and following runbooks

## Related Skills

- [k8s-namespace-troubleshooting](../k8s-namespace-troubleshooting) - Namespace-scoped diagnosis
- [k8s-platform-tenancy](../k8s-platform-tenancy) - Tenant management
- [k8s-security-hardening](../k8s-security-hardening) - Security operations
- [k8s-continual-improvement](../k8s-continual-improvement) - SLOs and metrics
- [k8s-security-redteam](../k8s-security-redteam) - Validate security posture
- [flux-troubleshooting](../flux-troubleshooting) - GitOps troubleshooting

## Quick Reference

| Task | Command |
|------|---------|
| Cluster health | `kubectl get nodes && kubectl get --raw='/healthz?verbose'` |
| All pods status | `kubectl get pods -A \| grep -v Running` |
| Resource usage | `kubectl top nodes && kubectl top pods -A` |
| Recent events | `kubectl get events -A --sort-by='.lastTimestamp'` |

## Health Monitoring

### Cluster Health Checks
```bash
kubectl get nodes -o wide
kubectl top nodes
kubectl get pods -n kube-system
kubectl get pods -n platform-system
kubectl get --raw='/healthz?verbose'
kubectl get --raw='/livez?verbose'
kubectl get --raw='/readyz?verbose'
kubectl get --raw='/healthz/etcd'
```

### Key Metrics to Monitor

| Metric | Warning | Critical | Action |
|--------|---------|----------|--------|
| Node CPU | >70% | >85% | Scale or optimize |
| Node Memory | >75% | >90% | Scale or evict |
| Pod restarts | >3/hr | >10/hr | Investigate |
| API latency p99 | >500ms | >1s | Check etcd/load |
| etcd disk | >70% | >85% | Expand/compact |
| PVC usage | >75% | >90% | Expand or clean |

## Incident Response

### Severity Levels

| Level | Impact | Response | Examples |
|-------|--------|----------|----------|
| P1 | Platform down | 15 min | API unreachable, etcd failure |
| P2 | Major degradation | 30 min | Node failures, ingress down |
| P3 | Partial impact | 2 hours | Single tenant affected |
| P4 | Minor issue | 24 hours | Non-critical alerts |

### Incident Workflow
```
1. DETECT    → Alert fires or user report
2. TRIAGE    → Assess severity and impact
3. COMMUNICATE → Update status page/channel
4. INVESTIGATE → Gather evidence
5. MITIGATE  → Restore service (temporary fix OK)
6. RESOLVE   → Fix root cause
7. REVIEW    → Post-incident analysis
```

### Escalation Path
```
L1 (On-call) → 15 min no progress
    ↓
L2 (Senior SRE) → 30 min no progress
    ↓
L3 (Platform Lead) → Critical impact
    ↓
Management (if customer-facing outage)
```

### On-Call Handoff Format

Include: **Active Issues** (severity + status), **Recent Changes** (what + when), **Upcoming Maintenance**, **Watchlist**, **Notes**

## Common Runbooks

### Node NotReady
```bash
# 1. Check node status
kubectl describe node ${NODE_NAME}

# 2. Check kubelet (SSH to node)
journalctl -u kubelet -n 100 --no-pager

# 3. Check resources
kubectl top node ${NODE_NAME}

# 4. If unrecoverable
kubectl cordon ${NODE_NAME}
kubectl drain ${NODE_NAME} --ignore-daemonsets --delete-emptydir-data
```

### Pod CrashLoopBackOff
```bash
# 1. Get details
kubectl describe pod ${POD} -n ${NS}

# 2. Check logs
kubectl logs ${POD} -n ${NS}
kubectl logs ${POD} -n ${NS} --previous

# 3. Check events
kubectl get events -n ${NS} --sort-by='.lastTimestamp' | grep ${POD}

# 4. Check resources
kubectl get pod ${POD} -n ${NS} -o jsonpath='{.spec.containers[*].resources}'
```

### High Memory Pressure
```bash
# 1. Find top consumers
kubectl top pods -A --sort-by=memory | head -20

# 2. Check evictions
kubectl get pods -A --field-selector=status.phase=Failed | grep Evicted

# 3. Check node conditions
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}: {.status.conditions[?(@.type=="MemoryPressure")].status}{"\n"}{end}'

# 4. Identify leaks (high restarts)
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name} restarts={.status.containerStatuses[0].restartCount}{"\n"}{end}' | sort -t= -k2 -rn | head -20
```

## Capacity Planning

### Resource Analysis
```bash
# Cluster summary
kubectl top nodes --no-headers | awk '{cpu+=$3; mem+=$5} END {print "Avg CPU:", cpu/NR"%", "Avg Mem:", mem/NR"%"}'

# Namespace consumption
kubectl top pods -A --no-headers | awk '{ns[$1]+=$3} END {for(n in ns) print n, ns[n]"m"}' | sort -k2 -rn

# Over-provisioned (requests >> actual)
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name} req={.spec.containers[0].resources.requests.cpu}{"\n"}{end}'
```

### Capacity Thresholds
- **Green**: <60% - Healthy headroom
- **Yellow**: 60-80% - Plan scaling
- **Red**: >80% - Scale immediately

## Maintenance Operations

### Node Maintenance
```bash
# 1. Cordon (prevent new pods)
kubectl cordon ${NODE}

# 2. Drain (evict pods)
kubectl drain ${NODE} \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=60 \
  --timeout=5m

# 3. Perform maintenance...

# 4. Uncordon
kubectl uncordon ${NODE}
```

### Rolling Restart
```bash
kubectl rollout restart deployment/${NAME} -n ${NS}
kubectl rollout status deployment/${NAME} -n ${NS}
```

### Certificate Check
```bash
kubeadm certs check-expiration
```

## Backup & Recovery

### etcd Backup
```bash
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%Y%m%d).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

### Velero Backup
```bash
velero backup create platform-$(date +%Y%m%d) \
  --include-namespaces platform-system \
  --ttl 720h

velero schedule create daily-platform \
  --schedule="0 2 * * *" \
  --include-namespaces platform-system
```

## Operational Checklists

### Daily
- [ ] Review alerting dashboard
- [ ] Check node health
- [ ] Verify backup completion
- [ ] Review capacity metrics

### Weekly
- [ ] Audit resource quotas
- [ ] Check certificate expiry
- [ ] Review pending updates
- [ ] Update runbooks

### Monthly
- [ ] Capacity planning review
- [ ] Security patch assessment
- [ ] Cost optimization review
- [ ] Tenant usage reports

## Post-Incident Review Format

Structure reviews with: **Summary** (duration, severity, impact), **Timeline** (time + event table), **Root Cause**, **What Went Well**, **What Could Be Improved**, **Action Items** (action, owner, due date)

## Common Mistakes

| Mistake | Why It Fails | Instead |
|---------|--------------|---------|
| Draining a node without cordoning first | New pods schedule onto the node during drain | Always `kubectl cordon` before `kubectl drain` |
| Skipping the COMMUNICATE step in incidents | Stakeholders make assumptions; duplicate investigations start | Update status channel before deep-diving |
| Running etcd backup without verifying restore procedure | Backup may be corrupt or incompatible; you won't know until you need it | Test restore to a non-production cluster periodically |
| Applying certificate rotation without checking dependent services | Services using the old cert break silently | Inventory cert consumers before rotation |
| Ignoring "Warning" events because pods are Running | Warnings often precede failures (e.g., mounting issues, throttling) | Review `kubectl get events` as part of daily checks |

## MCP Tools

- `mcp__flux-operator-mcp__get_kubernetes_resources`
- `mcp__flux-operator-mcp__get_kubernetes_logs`
- `mcp__flux-operator-mcp__get_kubernetes_metrics`
