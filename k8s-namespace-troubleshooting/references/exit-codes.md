# Container Exit Codes Reference

Quick-lookup reference for container exit codes encountered during namespace troubleshooting.

## Standard Exit Codes

| Exit Code | Signal | Meaning | Common Cause | Remediation |
|-----------|--------|---------|--------------|-------------|
| 0 | — | Success | Normal termination | No action needed |
| 1 | — | Application error | Unhandled exception, config error | Check application logs |
| 2 | — | Shell misuse | Bad shell script syntax | Fix entrypoint/command |
| 126 | — | Cannot execute | Permission denied on binary | Check image, file permissions |
| 127 | — | Command not found | Binary missing in image | Wrong image or entrypoint |
| 128 | — | Invalid exit argument | Exit called with non-integer | Fix application exit handling |
| 130 | SIGINT (2) | Interrupt | Ctrl+C or manual interrupt | Usually intentional |
| 134 | SIGABRT (6) | Abort | Application called abort() | Application bug, check core dump |
| 137 | SIGKILL (9) | Killed | OOMKilled or `kubectl delete --force` | Raise memory limits or fix leak |
| 139 | SIGSEGV (11) | Segfault | Memory access violation | Application bug, incompatible binary |
| 143 | SIGTERM (15) | Terminated | Graceful shutdown requested | Normal if during rollout; check if preStop hook needs more time |
| 255 | — | Exit status out of range | Runtime error or unknown exit | Check container runtime logs |

## Kubernetes-Specific Termination Reasons

| Reason | Description | Common Cause | Check |
|--------|-------------|--------------|-------|
| OOMKilled | Container exceeded memory limit | Memory leak or limit too low | `kubectl describe pod` → Last State |
| Evicted | Node under resource pressure | DiskPressure or MemoryPressure | Node conditions |
| DeadlineExceeded | Job exceeded activeDeadlineSeconds | Slow job or wrong deadline | Job spec |
| BackoffLimitExceeded | Job failed too many times | Persistent application error | Job events and logs |
| Completed | Container ran to completion | Normal for Jobs | No action |
| Error | Container exited with non-zero code | Application error | Container logs |

## Pod Phase Reference

| Phase | Meaning | Action |
|-------|---------|--------|
| Pending | Not yet scheduled or pulling images | Check events for scheduling failures |
| Running | At least one container running | Check individual container status |
| Succeeded | All containers exited with 0 | Normal for Jobs |
| Failed | At least one container exited non-zero | Check logs and exit codes |
| Unknown | State cannot be determined | Possible node communication issue |

## Pod Condition Reference

| Condition | Meaning When False |
|-----------|-------------------|
| PodScheduled | Scheduler cannot place the pod (resource/affinity issues) |
| Initialized | Init containers have not completed |
| ContainersReady | Not all containers are passing readiness probes |
| Ready | Pod is not ready to serve traffic |

## Container State Reference

| State | Description | Key Fields |
|-------|-------------|------------|
| Waiting | Container not yet running | `.reason` (e.g., CrashLoopBackOff, ImagePullBackOff, CreateContainerConfigError) |
| Running | Container is executing | `.startedAt` |
| Terminated | Container has exited | `.exitCode`, `.reason`, `.finishedAt` |
