---
name: k8s-continual-improvement
description: Use when defining and managing SLOs, optimizing cluster costs (FinOps), measuring and reducing toil, conducting capacity planning, assessing platform maturity, implementing feedback loops, or creating improvement roadmaps
---

# Kubernetes Continual Service Improvement

Continuously improve Kubernetes platforms including SLO management, cost optimization, performance tuning, FinOps, and platform maturity.

## Keywords

kubernetes, slo, sli, sla, error budget, cost optimization, finops, capacity, performance, improvement, maturity, metrics, toil, feedback, defining, managing, optimizing, measuring, reducing, planning, assessing, implementing, creating

## When to Use This Skill

- Defining and managing SLOs
- Optimizing cluster costs (FinOps)
- Measuring and reducing toil
- Conducting capacity planning
- Assessing platform maturity
- Implementing feedback loops
- Creating improvement roadmaps

## Related Skills

- [k8s-platform-operations](../k8s-platform-operations) - SLI data sources
- [k8s-platform-tenancy](../k8s-platform-tenancy) - Cost allocation
- [flux-gitops-patterns](../flux-gitops-patterns) - Change management

## Quick Reference

| Metric | Target | Calculation |
|--------|--------|-------------|
| Availability | 99.9% | uptime / total_time |
| Error Budget | 43.2 min/mo | (1 - SLO) * time_period |
| CPU Efficiency | >60% | actual / requested |
| MTTR | <4h P1 | mean(resolve_time - alert_time) |

## SLO Framework

### Service Level Indicators

**Availability:**
```yaml
- record: platform:availability:ratio_5m
  expr: |
    sum(up{job=~"kubernetes-.*"})
    / count(up{job=~"kubernetes-.*"})
```

**Latency (p99):**
```yaml
- record: platform:latency:p99_5m
  expr: |
    histogram_quantile(0.99,
      sum(rate(apiserver_request_duration_seconds_bucket{verb!="WATCH"}[5m]))
      by (le))
```

**Error Rate:**
```yaml
- record: platform:error_rate:ratio_5m
  expr: |
    sum(rate(apiserver_request_total{code=~"5.."}[5m]))
    / sum(rate(apiserver_request_total[5m]))
```

### SLO Targets

| Service | SLI | SLO | Error Budget/mo |
|---------|-----|-----|-----------------|
| API Server | Availability | 99.9% | 43.2 min |
| API Server | p99 Latency | <500ms | - |
| Ingress | Availability | 99.95% | 21.6 min |
| Workloads | Pod Start | <60s p95 | - |

### Error Budget Alerts
```yaml
- alert: ErrorBudgetBurnRate
  expr: |
    (1 - platform:availability:ratio_5m) > (1 - 0.999) * 14.4
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Error budget burning fast"
```

## Cost Optimization (FinOps)

### Efficiency Metrics

| Metric | Formula | Target |
|--------|---------|--------|
| CPU Efficiency | actual_cpu / requested_cpu | >60% |
| Memory Efficiency | actual_mem / requested_mem | >70% |
| Cost per Tenant | cluster_cost * (tenant_usage / total) | Track |
| Idle Resources | unused_capacity / total | <20% |

### Resource Analysis
```bash
# CPU efficiency
kubectl top pods -A --no-headers | awk '{
  split($3, cpu, "m"); actual+=cpu[1]
} END {print "Total CPU:", actual, "m"}'

# Find idle deployments
kubectl get deployments -A -o json | \
  jq -r '.items[] | select(.spec.replicas==0) | "\(.metadata.namespace)/\(.metadata.name)"'

# Unmounted PVCs
kubectl get pvc -A -o json | jq -r '.items[] | select(.status.phase=="Bound") | .metadata.name' > bound.txt
kubectl get pods -A -o json | jq -r '.items[].spec.volumes[]?.persistentVolumeClaim.claimName' | sort -u > mounted.txt
comm -23 <(sort bound.txt) <(sort mounted.txt)
```

### Cost Reduction Strategies

| Strategy | Savings | Effort |
|----------|---------|--------|
| Right-size requests | 20-40% | Medium |
| Spot/preemptible nodes | 60-80% | High |
| Cluster autoscaling | 10-30% | Low |
| Namespace quotas | Prevents waste | Low |
| Resource cleanup | 5-15% | Low |

### Cost Allocation Labels
```yaml
metadata:
  labels:
    cost-center: engineering
    team: platform
    environment: production
    application: api-gateway
```

## Toil Measurement

### Toil Indicators
- Manual, repetitive tasks
- No lasting value
- Scales with service size
- Automatable

### Toil Tracking
```yaml
toil_tasks:
  - name: "Manual tenant onboarding"
    frequency: "5/week"
    duration: "30min"
    annual_hours: 130
    automation_effort: "M"

  - name: "Certificate rotation"
    frequency: "4/year"
    duration: "2h"
    annual_hours: 8
    automation_effort: "S"
```

### Toil Reduction Target
- Current: X hours/week
- Target: 50% reduction in 6 months
- Method: Automation, self-service

## Platform Maturity Model

| Level | Name | Characteristics |
|-------|------|-----------------|
| 1 | Initial | Ad-hoc, manual |
| 2 | Managed | Documented, repeatable |
| 3 | Defined | Standardized, measured |
| 4 | Quantified | Data-driven, optimized |
| 5 | Optimizing | Continuous improvement |

### Capability Assessment
```yaml
capabilities:
  provisioning:
    current: 2
    target: 4
    gap: "No self-service"
  monitoring:
    current: 3
    target: 4
    gap: "Missing SLOs"
  security:
    current: 3
    target: 4
    gap: "Manual audits"
```

## Feedback Loops

### Tenant Satisfaction (NPS)
```yaml
survey:
  - "How satisfied with platform stability? (1-5)"
  - "How easy to deploy applications? (1-5)"
  - "How responsive is support? (1-5)"
  - "What should we improve?"
```

### Platform Metrics Dashboard
```yaml
dashboards:
  executive:
    - Availability %
    - Cost per tenant
    - Incident count
  tenant:
    - Resource usage
    - Deploy success rate
    - Error rates
  platform_team:
    - All SLIs
    - Error budget remaining
    - Capacity utilization
```

## Improvement Cadence

| Cadence | Activities |
|---------|------------|
| Weekly | Incident review, quick wins |
| Monthly | SLO review, cost analysis, backlog |
| Quarterly | Maturity assessment, OKRs |
| Annually | Strategy, tech radar, budget |

## Reporting Template

```markdown
# Platform Report - ${MONTH} ${YEAR}

## Availability
- SLO: 99.9% | Actual: ${ACTUAL}%
- Error Budget: ${REMAINING}% remaining

## Incidents
- P1: ${COUNT} | P2: ${COUNT}
- MTTR: ${MTTR}

## Cost
- Total: ${TOTAL}
- Per Tenant: ${AVG}
- MoM: ${CHANGE}%

## Capacity
- CPU: ${CPU}% | Memory: ${MEM}%

## Improvements
1. ${DELIVERED_1}
2. ${DELIVERED_2}

## Next Month
1. ${PLANNED_1}
2. ${PLANNED_2}
```

## Improvement Backlog Template

```markdown
## ${TITLE}

**Category**: Performance | Reliability | Security | Cost | UX
**Priority**: P1 | P2 | P3
**Effort**: S | M | L | XL

**Current**: ${PROBLEM}
**Target**: ${GOAL}
**Metrics**: Before: X → Target: Y
**Dependencies**: ${DEPS}
```
