# Flux Error Code Reference

## Kustomization Errors

### ReconciliationFailed
**Cause**: General reconciliation failure
**Check**: `flux logs --kind=Kustomization --name=<name>`
**Fix**: Review error message, fix manifests in Git

### DependencyNotReady
**Cause**: Parent Kustomization not ready
**Check**: `flux get kustomizations -A`
**Fix**: Resolve parent Kustomization first

### ArtifactFailed
**Cause**: Source artifact unavailable
**Check**: `flux get sources git -A`
**Fix**: Ensure Git repository is accessible, check credentials

### BuildFailed
**Cause**: Kustomize build failed
**Check**: `kubectl kustomize <path>` locally
**Fix**: Fix kustomization.yaml syntax

### HealthCheckFailed
**Cause**: Deployed resources not healthy
**Check**: `kubectl get pods -n <namespace>`
**Fix**: Check pod logs, resource limits, image availability

## HelmRelease Errors

### InstallFailed
**Cause**: Helm install failed
**Check**: `helm get notes <release> -n <namespace>`
**Fix**: Review values, check chart compatibility

### UpgradeFailed
**Cause**: Helm upgrade failed
**Check**: `flux logs --kind=HelmRelease --name=<name>`
**Fix**: Check values changes, review upgrade notes

### TestFailed
**Cause**: Helm test failed
**Check**: `helm test <release> -n <namespace>`
**Fix**: Review test pod logs

### GetLastReleaseFailed
**Cause**: Cannot retrieve Helm release state
**Check**: `helm list -n <namespace>`
**Fix**: May need to uninstall and reinstall

## Source Errors

### AuthenticationFailed
**Cause**: Git/Helm authentication failed
**Check**: Verify secret exists and is correct
**Fix**: Update deploy key or credentials

### CheckoutFailed
**Cause**: Git checkout failed
**Check**: `flux get sources git -A`
**Fix**: Verify branch/tag exists, check network

### StorageOperationFailed
**Cause**: Cannot store artifact
**Check**: `kubectl logs -n flux-system deploy/source-controller`
**Fix**: Check disk space, storage class
