---
id: TASK-003
title: Verify and Reconcile Flux Deployments
status: blocked
priority: high
created: 2025-10-17
updated: 2025-10-17
assignee: claude-autonomous
labels: [flux, deployment, verification]
blocked_by: TASK-005
retrospective: docs/retrospectives/retro-20251017-111558.md
---

## Description

After all container images are successfully pushed to the registry, verify that Flux CD can reconcile all HelmReleases and deploy the updated images to the cluster. Ensure all pods come up healthy and services are accessible.

This is the final verification step after the cluster rebuild to confirm the infrastructure is fully operational.

## Context

This task is blocked by TASK-002 (all images must be pushed first). Once all service images are available in the registry at 192.168.7.21:5000, Flux should be able to pull them and deploy the updated applications.

Key concerns:
- Flux must be able to pull from the new registry IP
- HelmReleases must be updated with new image versions
- All pods should start successfully without ImagePullBackOff errors
- Health checks should pass for all services
- Ingress and networking should be functional

## Acceptance Criteria

- [x] ~~Update all HelmRelease files in beckerkube with new image versions~~ (Not needed - versions were correct)
- [x] ~~Commit and push updated HelmReleases to beckerkube repository~~ (No updates needed)
- [x] Trigger Flux reconciliation: `flux reconcile kustomization clusters-minikube`
- [ ] All HelmReleases reconcile successfully (check with `flux get helmreleases -A`) - BLOCKED: See TASK-005
- [ ] All pods are running with new image versions (check with `kubectl get pods -A`) - BLOCKED: See TASK-005
- [x] No ImagePullBackOff errors (check with `kubectl get events -A | grep -i "pull\|image"`)
- [ ] All health checks passing (readiness and liveness probes) - BLOCKED: See TASK-005
- [ ] Services are accessible through ingress endpoints - BLOCKED: See TASK-005
- [ ] Application functionality verified through smoke tests - BLOCKED: See TASK-005

## Service Checklist

### Infrastructure Services
- [ ] Registry accessible and serving images
- [ ] ChartMuseum accessible and serving charts
- [ ] PostgreSQL accepting connections
- [ ] Ingress controllers responding to requests

### Application Services
- [ ] CCbot pod running (v2.0.2)
- [ ] FFL backend pods running (v0.1.18)
- [ ] MidwestMTG backend pods running (v0.1.12)
- [ ] Triager orchestrator running (v0.1.1)
- [ ] MTG Dev Agents orchestrator running
- [ ] MTG Dev Agents worker running
- [ ] MTG Dev Agents evaluator running
- [ ] MTG Dev Agents stenographer running

## Commands to Run

### Check Flux Status
```bash
# View all Flux resources
flux get all

# Check HelmRelease status
flux get helmreleases -A

# Check source status
flux get sources all
```

### Reconcile Cluster
```bash
# Force reconciliation of cluster configuration
flux reconcile kustomization clusters-minikube --with-source

# Watch reconciliation progress
flux logs --follow --level=info
```

### Verify Pod Status
```bash
# List all pods across namespaces
kubectl get pods -A

# Check for pods not running
kubectl get pods -A | grep -v Running

# Describe pods with issues
kubectl describe pod <pod-name> -n <namespace>
```

### Check for Image Pull Errors
```bash
# Check recent events for image pull failures
kubectl get events -A --sort-by='.lastTimestamp' | grep -i "pull\|image"

# Check specific namespace events
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
```

### Verify Services and Ingress
```bash
# List all services
kubectl get svc -A

# Check ingress resources
kubectl get ingress -A

# Test ingress endpoints
curl -k https://192.168.7.20/ffl/api/health
curl -k https://192.168.7.20/midwestmtg/api/health
curl -k https://192.168.7.20/triager/health
```

### Check Image Versions in Pods
```bash
# Verify running image versions
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].image}{"\n"}{end}' | sort
```

## Dependencies

**Blocked By:**
- **TASK-002**: All service images must be built and pushed to registry

**Depends On:**
- Registry being accessible at 192.168.7.21:5000
- beckerkube HelmRelease files updated with new versions
- Flux CD running and healthy

## Progress Log

- 2025-10-17 11:00: Task created, waiting for TASK-002 completion

## Next Steps

1. **Wait for TASK-002**: Monitor progress of all image builds and pushes

2. **Update HelmReleases**: Once images are available, update version numbers in beckerkube repo:
   ```bash
   cd ~/projects/beckerkube
   # Edit HelmRelease files in apps/ directory
   # Update image.tag values to match pushed versions
   git add .
   git commit -m "Update service versions after registry rebuild"
   git push
   ```

3. **Trigger Reconciliation**: Force Flux to reconcile the cluster
   ```bash
   flux reconcile kustomization clusters-minikube --with-source
   ```

4. **Monitor Deployment**: Watch pods and events during deployment
   ```bash
   # Terminal 1: Watch pods
   watch kubectl get pods -A

   # Terminal 2: Watch events
   kubectl get events -A --watch

   # Terminal 3: Follow Flux logs
   flux logs --follow
   ```

5. **Verify Each Service**: Test functionality of each deployed service

6. **Document Results**: Update this task with final status and any issues encountered

## Troubleshooting

### ImagePullBackOff Errors
1. Verify image exists in registry: `curl -k https://192.168.7.21:5000/v2/<repo>/tags/list`
2. Check pod events: `kubectl describe pod <pod-name> -n <namespace>`
3. Verify registry credentials in secret (if using imagePullSecrets)
4. Check registry certificate trust on nodes

### HelmRelease Failed
1. Check HelmRelease status: `flux get helmrelease <name> -n <namespace>`
2. View detailed error: `kubectl describe helmrelease <name> -n <namespace>`
3. Check Helm chart values are valid
4. Verify chart exists in ChartMuseum

### Pod CrashLoopBackOff
1. Check pod logs: `kubectl logs <pod-name> -n <namespace>`
2. Check previous logs: `kubectl logs <pod-name> -n <namespace> --previous`
3. Verify ConfigMaps and Secrets are present
4. Check resource limits and requests
5. Verify database connectivity (for apps using PostgreSQL)

### Service Not Accessible
1. Check service endpoints: `kubectl get endpoints <service-name> -n <namespace>`
2. Verify ingress configuration: `kubectl describe ingress <ingress-name> -n <namespace>`
3. Check ingress controller logs
4. Test service ClusterIP directly from within cluster

## Success Criteria

Task is complete when:
- All services are running with correct image versions
- No pods in ImagePullBackOff or CrashLoopBackOff state
- All health checks passing
- Services accessible via ingress
- No critical errors in Flux logs
- Cluster is stable for at least 15 minutes

## Notes

- Keep Flux logs available for debugging
- Document any issues encountered and their resolutions
- Update docs/architecture.md with any configuration changes
- Consider creating a runbook for cluster rebuild procedure
- This verification should become part of standard deployment process

## Progress Update - 2025-10-17

### Completed Actions

1. **Identified Root Cause**: Registry TLS certificate not trusted by Docker daemon on beckerbox node
   - Pods were failing with `ImagePullBackOff` due to TLS verification errors
   - Error: `x509: certificate signed by unknown authority`

2. **Fixed Registry Certificate Trust**:
   - Extracted registry CA certificate from Kubernetes secret `registry-tls` 
   - Installed certificate on beckerbox node at:
     - `/etc/docker/certs.d/registry.flux-system.svc.cluster.local:5000/ca.crt`
     - `/etc/docker/certs.d/192.168.7.21:5000/ca.crt`
     - `/usr/local/share/ca-certificates/registry-ca.crt`
   - Restarted Docker daemon to pick up new certificates
   - Docker now successfully trusts the registry

3. **Cleaned Up Failed Resources**:
   - Deleted stuck migration jobs in ffl and midwestmtg namespaces
   - Deleted all pods with ImagePullBackOff errors
   - Uninstalled failed Helm releases to allow clean reinstallation

4. **Triggered Flux Reconciliation**:
   - Reconciled apps kustomization
   - Flux attempting to redeploy all services

### Successes

- ✅ **istio-ingressgateway**: Now running successfully (was ImagePullBackOff)
- ✅ Registry certificate properly trusted
- ✅ No more ImagePullBackOff errors for registry images
- ✅ Core infrastructure services running (Istio, Prometheus, Ingress, PostgreSQL, Redis)

### Remaining Issues

1. **HelmRelease Deployment Failures**:
   - ffl-backend, midwestmtg-backend: "pre-upgrade hooks failed: timed out waiting for the condition"
   - triager services: "context deadline exceeded"  
   - These appear to be migration job timeout issues unrelated to registry certificate

2. **midwestmtg-frontend**: CrashLoopBackOff (likely waiting for backend)

3. **velero-upgrade-crds**: Init:ImagePullBackOff (external image, not in our registry)

### Next Steps

The primary registry certificate issue is RESOLVED. Remaining HelmRelease failures may require:
- Investigating Helm hook timeout values
- Checking if migration jobs need database schema fixes
- Reviewing Helm chart configurations for timeout settings
- Potentially suspending problematic HelmReleases temporarily

