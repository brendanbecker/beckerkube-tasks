---
id: TASK-004
title: Remove MTG Dev Agents from Cluster
status: completed
priority: medium
created: 2025-10-17
updated: 2025-10-17
completed: 2025-10-17
assignee: claude-autonomous
labels: [cleanup, deployment, kubernetes]
---

## Description

Remove MTG Dev Agents (orchestrator, generic_worker, stenographer, evaluator) from the Kubernetes cluster. These agents are not currently deployed via HelmReleases and are not actively used in the current cluster configuration.

## Context

During TASK-002, we discovered that MTG Dev Agents:
- Have container images built and pushed to registry (tag 417e41b)
- Have Helm charts defined in mtg_dev_agents repository
- Have a namespace reference (mtg-agents) in beckerkube
- Only have an ingress.yaml in beckerkube/apps/mtg-agents (no HelmReleases)
- Are not actively running or configured for deployment

Since these agents are not currently needed in the cluster, we should clean up the references to avoid confusion and reduce cluster configuration complexity.

## Acceptance Criteria

- [x] Check current deployment status of mtg-agents namespace - Namespace existed but was empty
- [x] Verify no running pods or resources in mtg-agents namespace - Confirmed no resources
- [x] Remove mtg-agents from beckerkube/clusters/minikube/apps/kustomization.yaml - Removed from line 6
- [x] Remove or archive beckerkube/apps/mtg-agents directory - Archived to archive/mtg-agents-20251017
- [x] Delete mtg-agents namespace from cluster (if exists) - Deleted (force deleted after fixing sealed secret ref)
- [x] Archive mtg-agents sealed secret - Moved secret-google-gemini-api.sealed.yaml to archive
- [x] Remove sealed secret reference from secrets kustomization - Fixed reference in clusters/minikube/secrets/kustomization.yaml
- [ ] Update TASK-002 to reflect MTG Dev Agents removal decision - Not needed (TASK-002 already completed)
- [x] Commit and push changes to beckerkube repository - 3 commits: a79cb28, 543b65a, 1ff5806
- [x] Verify Flux reconciliation completes without errors - All kustomizations READY

## Commands to Run

### Check Current Status
```bash
# Check if mtg-agents namespace exists
kubectl get namespace mtg-agents

# Check for any resources in namespace
kubectl get all -n mtg-agents

# Check for any HelmReleases
kubectl get helmreleases -n mtg-agents
```

### Remove from Kustomization
```bash
cd /home/becker/projects/beckerkube

# Edit clusters/minikube/apps/kustomization.yaml
# Remove line: - ../../../apps/mtg-agents
```

### Clean Up Directory
```bash
# Option 1: Delete directory
rm -rf apps/mtg-agents

# Option 2: Archive for reference
mkdir -p archive/
git mv apps/mtg-agents archive/mtg-agents-$(date +%Y%m%d)
```

### Delete Namespace
```bash
# Only if namespace exists and has no critical resources
kubectl delete namespace mtg-agents
```

### Verify Flux
```bash
# Reconcile cluster configuration
flux reconcile kustomization clusters-minikube --with-source

# Check for errors
flux get kustomizations
```

## Dependencies

**After:**
- **TASK-002**: Should be marked complete or have note about MTG agents removal

## Progress Log

- 2025-10-17 15:45: Task created based on findings from TASK-002
- 2025-10-17 23:30: Task completed - All MTG Dev Agents references removed from cluster

### Completion Summary

Successfully removed all MTG Dev Agents infrastructure from the Kubernetes cluster:

**Actions Completed:**
1. ✅ Verified mtg-agents namespace existed but contained no resources (no pods, HelmReleases, or services)
2. ✅ Removed mtg-agents reference from clusters/minikube/apps/kustomization.yaml (line 6)
3. ✅ Archived apps/mtg-agents directory to archive/mtg-agents-20251017 (preserved ingress.yaml and kustomization.yaml for reference)
4. ✅ Archived Google Gemini API sealed secret to archive/mtg-agents-20251017/secret-google-gemini-api.sealed.yaml
5. ✅ Removed sealed secret reference from clusters/minikube/secrets/kustomization.yaml
6. ✅ Force deleted empty mtg-agents namespace from cluster
7. ✅ Committed and pushed all changes to beckerkube repository (commits: a79cb28, 543b65a, 1ff5806)
8. ✅ Verified Flux reconciliation - all kustomizations in READY state with no errors

**Files Modified (beckerkube repository):**
- clusters/minikube/apps/kustomization.yaml (removed mtg-agents reference)
- clusters/minikube/secrets/kustomization.yaml (removed Gemini secret reference)
- Moved apps/mtg-agents/ → archive/mtg-agents-20251017/
- Moved clusters/minikube/secrets/secret-google-gemini-api.sealed.yaml → archive/mtg-agents-20251017/

**Outcome:**
- Cluster configuration simplified and cleaned up
- No more confusing references to non-deployed MTG agents
- All archived files preserved in archive/mtg-agents-20251017/ for future reference if needed
- Container images remain in registry at tag 417e41b for potential future use
- Helm charts in mtg_dev_agents repository remain unchanged
- Flux reconciliation working correctly with no errors

**Note:** MTG Dev Agents can be re-deployed in the future by:
1. Restoring archived files from archive/mtg-agents-20251017/
2. Creating proper HelmRelease manifests
3. Re-adding reference to kustomization.yaml

## Next Steps

1. Verify current state of mtg-agents namespace and resources
2. Remove from kustomization.yaml
3. Archive or delete apps/mtg-agents directory
4. Delete namespace if it exists
5. Test Flux reconciliation
6. Update TASK-002 documentation

## Notes

- Keep container images in registry in case agents are needed in the future
- Helm charts in mtg_dev_agents repo remain unchanged
- If MTG agents are needed later, this can be reversed and proper HelmReleases created
- Document decision in cluster architecture notes

## Rationale

Removing unused components from cluster configuration:
- Reduces confusion about what's deployed vs what's just referenced
- Simplifies Flux reconciliation
- Keeps cluster configuration clean and maintainable
- Images remain available in registry for future use if needed
