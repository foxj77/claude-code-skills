# Incident Runbooks

## Node NotReady

### Symptoms
- Node shows NotReady status
- Pods on node in Unknown/Terminating state
- Alerts: NodeNotReady, NodeUnreachable

### Diagnosis
```bash
kubectl describe node ${NODE}
kubectl get events --field-selector involvedObject.name=${NODE}
ssh ${NODE} "journalctl -u kubelet -n 100"
ssh ${NODE} "systemctl status kubelet"
```

### Resolution
1. Check node network connectivity
2. Check kubelet service status
3. Check disk space and inodes
4. Check memory pressure
5. Restart kubelet if needed
6. If unrecoverable, cordon, drain, replace

---

## Pod CrashLoopBackOff

### Symptoms
- Pod repeatedly restarting
- Status shows CrashLoopBackOff
- High restart count

### Diagnosis
```bash
kubectl describe pod ${POD} -n ${NS}
kubectl logs ${POD} -n ${NS} --previous
kubectl get events -n ${NS} | grep ${POD}
```

### Common Causes
- Application error (check logs)
- Missing dependencies
- Resource limits too low
- Liveness probe failing
- Missing secrets/configmaps

---

## High API Server Latency

### Symptoms
- kubectl commands slow
- Alerts: KubeAPILatencyHigh
- Tenant complaints

### Diagnosis
```bash
kubectl get --raw='/metrics' | grep apiserver_request_duration
kubectl top pods -n kube-system | grep apiserver
etcdctl endpoint status --write-out=table
```

### Resolution
1. Check etcd health and latency
2. Review audit log for expensive queries
3. Check for webhook timeout issues
4. Scale API server if needed

---

## PersistentVolume Issues

### Symptoms
- PVC stuck in Pending
- Pod can't start due to volume

### Diagnosis
```bash
kubectl describe pvc ${PVC} -n ${NS}
kubectl get events -n ${NS} | grep ${PVC}
kubectl get pv | grep ${PVC}
```

### Common Causes
- No matching PV available
- StorageClass misconfigured
- Node affinity mismatch
- CSI driver issues

---

## Ingress Not Working

### Symptoms
- 502/503 errors
- Timeouts
- SSL certificate errors

### Diagnosis
```bash
kubectl get ingress -A
kubectl describe ingress ${INGRESS} -n ${NS}
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller
kubectl get events -n ${NS}
```

### Resolution
1. Verify service exists and has endpoints
2. Check ingress controller logs
3. Verify TLS secret exists
4. Check network policies
