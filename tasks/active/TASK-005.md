---
id: TASK-005
title: Fix HelmRelease Deployment Timeout and Hook Failures
status: active
priority: high
created: 2025-10-17
updated: 2025-10-17
assignee: unassigned
labels: [helm, deployment, troubleshooting]
blocks: TASK-003
---

## Description

Multiple HelmReleases are failing to deploy with timeout errors and pre-upgrade hook failures. After resolving the registry certificate issue in TASK-003, pods can now pull images successfully, but Helm deployments are still failing due to hook timeouts and context deadline exceeded errors.

## Context

Following the registry certificate fix, the cluster can now pull images from the local registry without ImagePullBackOff errors. However, HelmReleases for application services are failing with:

1. **Pre-upgrade hook failures**: "pre-upgrade hooks failed: timed out waiting for the condition"
   - Affects: ffl-backend, midwestmtg-backend

2. **Context deadline exceeded**: Helm operations timing out
   - Affects: triager services (ccbot, all workers, orchestrator, redis)
   - Affects: istio-ingressgateway, midwestmtg-frontend

3. **Velero upgrade job**: Init:ImagePullBackOff (external image not in our registry)

## Root Causes to Investigate

1. **Helm Hook Timeouts**: Default timeout may be too short for migration jobs
2. **Missing Dependencies**: Backend services may be waiting for database migrations
3. **Resource Constraints**: Pods may be pending due to resource limits
4. **Configuration Issues**: Missing ConfigMaps, Secrets, or invalid configurations
5. **Network Policies**: Pods may be unable to communicate with dependencies

## Acceptance Criteria

- [ ] Identify root cause of pre-upgrade hook failures for ffl-backend and midwestmtg-backend
- [ ] Identify root cause of context deadline exceeded for triager services
- [ ] Fix or increase timeout settings for Helm hooks if needed
- [ ] Ensure all required ConfigMaps and Secrets exist for each service
- [ ] Verify database migrations can complete successfully
- [ ] All HelmReleases reconcile successfully: `flux get helmreleases -A`
- [ ] All application pods running: `kubectl get pods -A`
- [ ] No CrashLoopBackOff or CreateContainerConfigError pods
- [ ] Services respond to health checks

## Affected Services

### FFL Namespace
- **ffl-backend** (v0.1.18): Pre-upgrade hook timeout
- **ffl-frontend**: Blocked by ffl-backend dependency

### MidwestMTG Namespace
- **midwestmtg-backend** (v0.1.13): Pre-upgrade hook timeout
- **midwestmtg-frontend** (v0.2.0): CrashLoopBackOff + ImagePullBackOff (2 replicas)
- **midwestmtg-discord-bot**: Blocked by midwestmtg-backend dependency

### Triager Namespace
- **triager-redis** (v0.1.0): Context deadline exceeded
- **ccbot** (v0.2.3): Context deadline exceeded
- **triager-orchestrator** (v0.1.1): Context deadline exceeded
- **triager-classifier-worker** (v0.1.2): Context deadline exceeded
- **triager-doc-generator-worker** (v0.1.2): Context deadline exceeded
- **triager-duplicate-worker** (v0.1.6): Context deadline exceeded
- **triager-git-manager-worker** (v0.1.2): Context deadline exceeded

### Infrastructure
- **istio-ingressgateway** (v1.26.2): ✅ NOW RUNNING (fixed by registry cert)
- **velero-upgrade-crds**: Init:ImagePullBackOff (external image)

## Diagnostic Commands

### Check HelmRelease Status
```bash
flux get helmreleases -A
kubectl get helmreleases -A -o yaml
```

### Check for Failed Hooks
```bash
kubectl get jobs -A
kubectl describe job <job-name> -n <namespace>
kubectl logs job/<job-name> -n <namespace>
```

### Check Pod Events
```bash
kubectl get events -A --sort-by='.lastTimestamp' | tail -50
kubectl describe pod <pod-name> -n <namespace>
```

### Check Helm Release Details
```bash
helm list -A
helm history <release-name> -n <namespace>
kubectl get secret -n <namespace> | grep helm
```

### Check HelmRelease Timeout Settings
```bash
kubectl get helmrelease <release-name> -n <namespace> -o yaml | grep -A 5 timeout
```

## Investigation Steps

1. **Check Helm Timeout Settings**:
   - Review HelmRelease manifests for timeout configurations
   - Check if default timeouts are too short for migration jobs
   - Increase timeouts if needed (e.g., from 5m to 10m)

2. **Examine Failed Migration Jobs**:
   ```bash
   kubectl get jobs -n ffl
   kubectl get jobs -n midwestmtg
   kubectl logs job/ffl-backend-migration -n ffl
   kubectl logs job/midwestmtg-backend-migration -n midwestmtg
   ```

3. **Check for Missing Secrets/ConfigMaps**:
   ```bash
   kubectl get configmaps -n ffl
   kubectl get secrets -n ffl
   kubectl get configmaps -n triager
   kubectl get secrets -n triager
   ```

4. **Review Database Connectivity**:
   - Verify PostgreSQL is accessible from application namespaces
   - Check database credentials in secrets
   - Test connection from a debug pod

5. **Check Resource Availability**:
   ```bash
   kubectl describe nodes
   kubectl top nodes
   kubectl top pods -A
   ```

## Potential Solutions

### Option 1: Increase Helm Timeout
Edit HelmRelease manifests to increase timeout:
```yaml
spec:
  timeout: 10m  # Increase from default 5m
```

### Option 2: Fix Migration Job Issues
- Review migration scripts for errors
- Ensure database schema is compatible
- Check for stuck or incomplete migrations

### Option 3: Suspend and Resume
Temporarily suspend problematic HelmReleases:
```bash
flux suspend helmrelease <name> -n <namespace>
# Fix underlying issue
flux resume helmrelease <name> -n <namespace>
```

### Option 4: Delete and Reinstall
For stuck releases, completely remove and let Flux reinstall:
```bash
helm uninstall <release> -n <namespace>
kubectl delete helmrelease <name> -n <namespace>
# Wait for Flux to recreate
```

## Success Criteria

Task is complete when:
- All HelmReleases show READY=True in `flux get helmreleases -A`
- All application pods are Running (except velero-upgrade-crds if external)
- No pods in ImagePullBackOff, CrashLoopBackOff, or CreateContainerConfigError
- Services respond to health check endpoints
- TASK-003 can be unblocked and completed

## Dependencies

**Unblocks:**
- **TASK-003**: Verify and Reconcile Flux Deployments

**Related:**
- **TASK-004**: Remove MTG Dev Agents from Cluster (lower priority, independent)

## Progress Log

- 2025-10-17 20:00: Task created to track HelmRelease deployment failures after registry cert fix

## Notes

- Registry certificate issue has been resolved - pods can now pull images
- This task focuses on Helm-level deployment issues, not registry issues
- May require coordination with service maintainers if schema changes are needed
- Velero upgrade job may need to be disabled or fixed separately (external image)
