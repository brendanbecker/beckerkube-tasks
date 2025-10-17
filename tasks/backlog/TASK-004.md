---
id: TASK-004
title: Remove MTG Dev Agents from Cluster
status: backlog
priority: medium
created: 2025-10-17
updated: 2025-10-17
assignee: unassigned
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

- [ ] Check current deployment status of mtg-agents namespace
- [ ] Verify no running pods or resources in mtg-agents namespace
- [ ] Remove mtg-agents from beckerkube/clusters/minikube/apps/kustomization.yaml
- [ ] Remove or archive beckerkube/apps/mtg-agents directory
- [ ] Delete mtg-agents namespace from cluster (if exists)
- [ ] Update TASK-002 to reflect MTG Dev Agents removal decision
- [ ] Commit and push changes to beckerkube repository
- [ ] Verify Flux reconciliation completes without errors

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
