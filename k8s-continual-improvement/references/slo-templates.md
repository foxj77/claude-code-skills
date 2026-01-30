# SLO Templates

## Platform Availability SLO

```yaml
name: platform-availability
description: "Kubernetes API server availability"

sli:
  type: availability
  metric: |
    sum(up{job="kubernetes-apiservers"})
    / count(up{job="kubernetes-apiservers"})

slo:
  target: 99.9%
  window: 30d

error_budget:
  monthly_minutes: 43.2
  alert_thresholds:
    - burn_rate: 14.4  # 5 day budget burn
      window: 1h
      severity: critical
    - burn_rate: 6     # 2 week budget burn
      window: 6h
      severity: warning

escalation:
  - level: 1
    condition: "budget < 50%"
    action: "Review and prioritize reliability work"
  - level: 2
    condition: "budget < 25%"
    action: "Freeze non-critical changes"
  - level: 3
    condition: "budget < 10%"
    action: "Incident response mode"
```

## API Latency SLO

```yaml
name: api-latency
description: "API server response time"

sli:
  type: latency
  metric: |
    histogram_quantile(0.99,
      sum(rate(apiserver_request_duration_seconds_bucket{verb!="WATCH"}[5m]))
      by (le)
    )

slo:
  target: 500ms
  percentile: p99
  window: 30d

error_budget:
  calculation: |
    Requests where latency > 500ms
    / Total requests
  target: < 1%
```

## Workload Deployment SLO

```yaml
name: deployment-success
description: "Successful deployments within SLA"

sli:
  type: success_rate
  metric: |
    sum(kube_deployment_status_condition{condition="Available",status="true"})
    / sum(kube_deployment_status_condition{condition="Available"})

slo:
  target: 99.5%
  window: 7d

measurement:
  - Deployment completes within 5 minutes
  - All replicas become ready
  - No rollback required
```

## Tenant Request SLO

```yaml
name: tenant-onboarding
description: "New tenant provisioning time"

sli:
  type: freshness
  measurement: "Time from request to ready namespace"

slo:
  target: "< 4 hours for standard requests"
  tiers:
    standard: 4h
    expedited: 1h
    emergency: 15m

tracking:
  ticket_field: "time_to_resolution"
  start: "ticket_created"
  end: "tenant_namespace_ready"
```

## Incident Response SLO

```yaml
name: incident-response
description: "Time to acknowledge and resolve incidents"

sli:
  type: freshness
  measurement: "Time from alert to acknowledgment/resolution"

slo:
  acknowledge:
    p1: 15m
    p2: 30m
    p3: 2h
    p4: 24h
  resolve:
    p1: 4h
    p2: 8h
    p3: 24h
    p4: 72h

tracking:
  source: "pagerduty/opsgenie"
  metrics:
    - mtta  # Mean Time To Acknowledge
    - mttr  # Mean Time To Resolve
```

## Prometheus Alert Rules

```yaml
groups:
- name: slo-alerts
  rules:
  # Availability SLO burn rate
  - alert: AvailabilitySLOBudgetBurn
    expr: |
      (
        1 - (sum(up{job="kubernetes-apiservers"}) / count(up{job="kubernetes-apiservers"}))
      ) > (1 - 0.999) * 14.4
    for: 5m
    labels:
      severity: critical
      slo: availability
    annotations:
      summary: "Availability SLO error budget burning fast"
      runbook: "https://runbooks/slo-availability"

  # Latency SLO breach
  - alert: LatencySLOBreach
    expr: |
      histogram_quantile(0.99,
        sum(rate(apiserver_request_duration_seconds_bucket{verb!="WATCH"}[5m]))
        by (le)
      ) > 0.5
    for: 10m
    labels:
      severity: warning
      slo: latency
    annotations:
      summary: "API latency exceeding SLO target"
```
