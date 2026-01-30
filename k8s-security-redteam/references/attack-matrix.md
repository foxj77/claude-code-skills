# Kubernetes Attack Matrix (MITRE ATT&CK)

## Initial Access

| Technique | ID | Description | Test |
|-----------|-----|-------------|------|
| Valid Accounts | T1078 | Stolen credentials/tokens | Check for leaked tokens in repos |
| Exploit Public-Facing Application | T1190 | Vulnerable workload | Scan exposed services |
| Supply Chain Compromise | T1195 | Malicious image | Check image sources |

## Execution

| Technique | ID | Description | Test |
|-----------|-----|-------------|------|
| Container Administration Command | T1609 | kubectl exec | `kubectl auth can-i create pods/exec` |
| User Execution | T1204 | Malicious container | Deploy test pod |
| Scheduled Task/Job | T1053 | CronJob abuse | Check CronJob permissions |

## Persistence

| Technique | ID | Description | Test |
|-----------|-----|-------------|------|
| Account Manipulation | T1098 | Create service accounts | Check SA creation permissions |
| Implant Container | T1525 | Backdoor container | Look for unexpected containers |
| Valid Accounts | T1078.004 | Cloud account persistence | Check cloud IAM |

## Privilege Escalation

| Technique | ID | Description | Test |
|-----------|-----|-------------|------|
| Escape to Host | T1611 | Container breakout | Check for privileged pods |
| Exploitation for Privilege Escalation | T1068 | Kernel/runtime exploit | Check versions |
| Valid Accounts | T1078 | Escalate via RBAC | Check bind/escalate verbs |

## Defense Evasion

| Technique | ID | Description | Test |
|-----------|-----|-------------|------|
| Masquerading | T1036 | Look like legit workload | Review pod naming |
| Impair Defenses | T1562 | Disable logging | Check audit policy |
| Indicator Removal | T1070 | Delete pods/logs | Check delete permissions |

## Credential Access

| Technique | ID | Description | Test |
|-----------|-----|-------------|------|
| Unsecured Credentials | T1552 | Secrets in env/files | Enumerate secrets |
| Credentials from Password Stores | T1555 | Vault/secret stores | Check secret store access |
| Steal Application Access Token | T1528 | SA token theft | Check token mounts |

## Discovery

| Technique | ID | Description | Test |
|-----------|-----|-------------|------|
| Cloud Service Discovery | T1526 | Enumerate cluster | Check list permissions |
| Network Service Scanning | T1046 | Service discovery | Network policy gaps |
| Permission Groups Discovery | T1069 | RBAC enumeration | Check auth can-i |

## Lateral Movement

| Technique | ID | Description | Test |
|-----------|-----|-------------|------|
| Internal Spearphishing | T1534 | Access via workloads | Network policy gaps |
| Exploitation of Remote Services | T1210 | Service vulnerabilities | Scan internal services |
| Use Alternate Authentication | T1550 | Token reuse | Test token scope |

## Collection

| Technique | ID | Description | Test |
|-----------|-----|-------------|------|
| Data from Cloud Storage Object | T1530 | PV data access | Check PV permissions |
| Data from Information Repositories | T1213 | ConfigMaps/Secrets | Enumerate data stores |

## Impact

| Technique | ID | Description | Test |
|-----------|-----|-------------|------|
| Resource Hijacking | T1496 | Cryptomining | Monitor resource usage |
| Data Destruction | T1485 | Delete resources | Check delete permissions |
| Service Stop | T1489 | Delete deployments | Check delete permissions |

## Quick Reference Commands

```bash
# Check what you can do
kubectl auth can-i --list

# Find privileged paths
kubectl auth can-i create pods --subresource=exec
kubectl auth can-i create rolebindings
kubectl auth can-i get secrets

# Find privileged pods
kubectl get pods -A -o json | jq '.items[] | select(.spec.containers[].securityContext.privileged==true) | .metadata.name'
```
