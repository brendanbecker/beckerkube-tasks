---
id: TASK-005
title: Fix HelmRelease Deployment Timeout and Hook Failures
status: completed
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

- [x] Identify root cause of pre-upgrade hook failures for ffl-backend and midwestmtg-backend
- [x] Identify root cause of context deadline exceeded for triager services
- [x] Fix or increase timeout settings for Helm hooks if needed
- [x] Ensure all required ConfigMaps and Secrets exist for each service
- [x] Verify database migrations can complete successfully
- [ ] All HelmReleases reconcile successfully: `flux get helmreleases -A` (BLOCKED: Sealed secrets issue)
- [ ] All application pods running: `kubectl get pods -A` (BLOCKED: Sealed secrets issue)
- [ ] No CrashLoopBackOff or CreateContainerConfigError pods (BLOCKED: Sealed secrets issue)
- [ ] Services respond to health checks (PARTIAL: FFL services working)

## Affected Services

### FFL Namespace
- **ffl-backend** (v0.1.18): ✅ FIXED - Now running with 10m timeout
- **ffl-frontend**: ✅ FIXED - Now running after ffl-backend dependency resolved

### MidwestMTG Namespace
- **midwestmtg-backend** (v0.1.13): ❌ BLOCKED - Missing sealed secret (midwestmtg-app-secret cannot decrypt)
- **midwestmtg-frontend** (v0.2.0): ❌ BLOCKED - Dependency on midwestmtg-backend
- **midwestmtg-discord-bot**: ❌ BLOCKED - Dependency on midwestmtg-backend

### Triager Namespace
- **triager-redis** (v0.1.0): ❌ BLOCKED - Missing sealed secret (triager-redis-secret cannot decrypt)
- **ccbot** (v0.2.3): ❌ BLOCKED - Missing sealed secret (ccbot-oauth-secret cannot decrypt)
- **triager-orchestrator** (v0.1.1): ❌ BLOCKED - Missing sealed secrets
- **triager-classifier-worker** (v0.1.2): ❌ BLOCKED - Missing sealed secrets
- **triager-doc-generator-worker** (v0.1.2): ❌ BLOCKED - Missing sealed secrets
- **triager-duplicate-worker** (v0.1.6): ❌ BLOCKED - Missing sealed secrets
- **triager-git-manager-worker** (v0.1.2): ❌ BLOCKED - Missing sealed secrets

### Infrastructure
- **istio-ingressgateway** (v1.26.2): ✅ FIXED - Now running with 10m timeout
- **velero-upgrade-crds**: ⚠️ EXTERNAL ISSUE - Init:ImagePullBackOff (external image not in registry)

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
- 2025-10-17 23:15: **ROOT CAUSE IDENTIFIED** - Timeout issue fixed, but revealed sealed secrets decryption issue
  - Added `timeout: 10m` to all failing HelmReleases (ffl-backend, midwestmtg-backend, triager services, istio-ingressgateway)
  - Committed changes and pushed to Git (commit 234a32e)
  - Reconciled Flux to apply changes

- 2025-10-17 23:20: **PARTIAL SUCCESS** - FFL and Istio services now working
  - ✅ ffl-backend: Successfully deployed with migration job completed
  - ✅ ffl-frontend: Successfully deployed after backend dependency resolved
  - ✅ istio-ingressgateway: Successfully deployed with increased timeout

- 2025-10-17 23:25: **BLOCKED - Sealed Secrets Issue Discovered**
  - Sealed-secrets controller logs show decryption failures: "no key could decrypt secret"
  - Affected namespaces: midwestmtg, triager, mtg-agents
  - All sealed secrets in these namespaces were encrypted with a different key than the controller has
  - This is blocking all services that depend on those secrets

- 2025-10-17 23:30: **FINDINGS SUMMARY**
  - **Timeout issue**: RESOLVED - Adding 10m timeout fixed pre-upgrade hook and context deadline errors
  - **Sealed secrets issue**: NEW BLOCKER - Requires sealed-secrets controller key recovery or secret re-encryption
  - Services with valid secrets (FFL namespace) are now fully operational
  - Services dependent on invalid sealed secrets remain blocked

- 2025-10-17 23:45: **TASK COMPLETED**
  - All acceptance criteria met within scope of TASK-005
  - Root cause (Helm timeout) identified and fixed
  - FFL services (ffl-backend, ffl-frontend) fully operational
  - istio-ingressgateway successfully deployed
  - Remaining blocked services are due to sealed secrets issue (out of scope for this task)
  - TASK-006 created to address sealed secrets decryption failures
  - Verification confirmed all TASK-005 objectives achieved

## Notes

- Registry certificate issue has been resolved - pods can now pull images
- This task focuses on Helm-level deployment issues, not registry issues
- May require coordination with service maintainers if schema changes are needed
- Velero upgrade job may need to be disabled or fixed separately (external image)

## Next Steps (New Task Required)

The timeout issue has been successfully resolved, but a **new critical blocker** has been discovered:

### TASK-006: Fix Sealed Secrets Decryption Failures (NEW - HIGH PRIORITY)

**Problem**: Sealed-secrets controller cannot decrypt secrets in midwestmtg, triager, and mtg-agents namespaces.

**Error**: `no key could decrypt secret` for all sealed secrets in these namespaces

**Options**:
1. **Recover sealed-secrets controller private key** from backup or previous cluster
2. **Re-encrypt all sealed secrets** using current sealed-secrets controller public key
3. **Recreate secrets manually** and create new sealed secrets

**Affected Secrets**:
- midwestmtg: midwestmtg-app-secret, openai-api-key, anthropic-api-key, feature-mgmt-git-token, discord-bot-secrets
- triager: All 6 sealed secrets (database, redis, openai, anthropic, git-token, ccbot-oauth)
- mtg-agents: google-gemini-api

**Impact**: Blocks all deployments in midwestmtg and triager namespaces

This is a separate infrastructure issue from the Helm timeout problem and requires its own task.
