# Namespace Troubleshooting Checklist

A printable checklist for systematic namespace investigation. Use alongside the main skill.

## Pre-Flight

- [ ] Confirm namespace name with user
- [ ] Confirm cluster context is correct (`kubectl config current-context`)
- [ ] Confirm you have read access to the namespace (`kubectl auth can-i get pods -n ${NS}`)

## Phase 1: Collect

### Core Resources
- [ ] Namespace exists and check labels/annotations
- [ ] Events (all, then warnings only)
- [ ] Pods (status, restarts, conditions)
- [ ] Deployments (desired vs available)
- [ ] StatefulSets (desired vs ready)
- [ ] DaemonSets (desired vs ready)
- [ ] Jobs and CronJobs (success/failure)

### Networking
- [ ] Services
- [ ] Endpoints (check for empty subsets)
- [ ] Ingresses
- [ ] Network policies

### Configuration
- [ ] ConfigMaps (list)
- [ ] Secrets (names only — never print values)
- [ ] PersistentVolumeClaims (bound status)
- [ ] ResourceQuotas
- [ ] LimitRanges

### Compute
- [ ] Pod resource usage (CPU/memory)
- [ ] Request vs limit vs actual comparison
- [ ] HPA / VPA status

### Infrastructure
- [ ] Nodes hosting namespace pods (conditions, capacity)
- [ ] Node resource pressure

### Application
- [ ] Current logs (all unhealthy pods)
- [ ] Previous logs (crashed/restarted containers)
- [ ] Error pattern grep across logs

### GitOps (if applicable)
- [ ] Flux Kustomizations targeting namespace
- [ ] HelmReleases in namespace

## Phase 2: Summarise
- [ ] Total resource counts
- [ ] List of unhealthy resources with status
- [ ] Resource consumption table
- [ ] Top warning events
- [ ] Node health for relevant nodes

## Phase 3: Diagnose
- [ ] Each unhealthy resource has an identified root cause
- [ ] Decision tree followed for each pod issue
- [ ] Cross-resource correlations checked (e.g., node pressure → pod eviction)

## Phase 4: Impact
- [ ] Each problem assigned a severity (Critical/High/Medium/Low)
- [ ] Blast radius documented
- [ ] User impact described

## Phase 5: Remediate
- [ ] Safe automated actions identified and executed (if approved)
- [ ] User-directed actions listed with clear instructions
- [ ] Preventive recommendations provided
- [ ] Remediation report generated
