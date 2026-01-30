# Tenant Onboarding Checklist

## Pre-Onboarding
- [ ] Tenant agreement signed
- [ ] Cost center identified
- [ ] Technical contact designated
- [ ] Service tier selected (Bronze/Silver/Gold/Platinum)
- [ ] Resource requirements estimated

## Namespace Setup
- [ ] Namespace created with standard labels
- [ ] Pod Security Standard applied (restricted)
- [ ] ResourceQuota configured per tier
- [ ] LimitRange with defaults applied

## Access Control
- [ ] RBAC roles created (admin, developer, viewer)
- [ ] RoleBindings for tenant users
- [ ] Service accounts created (if needed)
- [ ] SSO/OIDC groups mapped

## Network Isolation
- [ ] Default deny NetworkPolicy applied
- [ ] Intra-namespace policy applied
- [ ] Platform services access policy applied
- [ ] Egress policies configured

## Observability
- [ ] ServiceMonitor for tenant apps
- [ ] Alerting rules configured
- [ ] Grafana dashboards provisioned
- [ ] Log forwarding configured

## Documentation
- [ ] Tenant registered in platform catalog
- [ ] Access instructions sent
- [ ] Runbook templates provided
- [ ] Support escalation path documented

## Verification
- [ ] Tenant admin can deploy test workload
- [ ] Monitoring shows tenant metrics
- [ ] Logs visible in logging platform
- [ ] Network isolation verified
