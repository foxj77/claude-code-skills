# CIS Kubernetes Benchmark Reference

## Control Plane Components

### 1.1 API Server

| Control | Description | Check |
|---------|-------------|-------|
| 1.1.1 | Anonymous auth disabled | `--anonymous-auth=false` |
| 1.1.2 | Basic auth disabled | No `--basic-auth-file` |
| 1.1.3 | Token auth file not used | No `--token-auth-file` |
| 1.1.4 | Kubelet HTTPS | `--kubelet-https=true` |
| 1.1.5 | Kubelet client cert auth | `--kubelet-client-certificate` |
| 1.1.6 | Authorization mode | `--authorization-mode=RBAC,Node` |
| 1.1.7 | Admission controllers | Enable: NodeRestriction, PodSecurity |

### 1.2 Controller Manager

| Control | Description | Check |
|---------|-------------|-------|
| 1.2.1 | Profiling disabled | `--profiling=false` |
| 1.2.2 | Service account credentials | `--use-service-account-credentials=true` |
| 1.2.3 | SA token rotation | `--rotate-certificates=true` |

### 1.3 Scheduler

| Control | Description | Check |
|---------|-------------|-------|
| 1.3.1 | Profiling disabled | `--profiling=false` |
| 1.3.2 | Bind address | `--bind-address=127.0.0.1` |

### 1.4 etcd

| Control | Description | Check |
|---------|-------------|-------|
| 1.4.1 | Client cert auth | `--client-cert-auth=true` |
| 1.4.2 | Auto TLS disabled | `--auto-tls=false` |
| 1.4.3 | Peer cert auth | `--peer-client-cert-auth=true` |

## Worker Node Security

### 2.1 Kubelet

| Control | Description | Check |
|---------|-------------|-------|
| 2.1.1 | Anonymous auth disabled | `--anonymous-auth=false` |
| 2.1.2 | Authorization mode | `--authorization-mode=Webhook` |
| 2.1.3 | Client CA | `--client-ca-file` set |
| 2.1.4 | Read-only port | `--read-only-port=0` |
| 2.1.5 | Streaming connection timeout | `--streaming-connection-idle-timeout` |
| 2.1.6 | Protect kernel defaults | `--protect-kernel-defaults=true` |
| 2.1.7 | Event qps | `--event-qps=0` (or appropriate rate) |

## Policies

### 3.1 RBAC

| Control | Description |
|---------|-------------|
| 3.1.1 | Minimize cluster-admin usage |
| 3.1.2 | Minimize wildcard permissions |
| 3.1.3 | Minimize service account permissions |
| 3.1.4 | Audit roles and rolebindings |

### 3.2 Pod Security

| Control | Description |
|---------|-------------|
| 3.2.1 | Minimize privileged containers |
| 3.2.2 | Minimize containers with allowPrivilegeEscalation |
| 3.2.3 | Minimize root containers |
| 3.2.4 | Minimize NET_RAW capability |
| 3.2.5 | Minimize added capabilities |

## Automated Checking

```bash
# Run kube-bench
docker run --rm -v /etc:/etc:ro -v /var:/var:ro \
  aquasec/kube-bench:latest run --targets=node

# Run kubescape
kubescape scan framework cis-v1.23-t1.0.1
```
