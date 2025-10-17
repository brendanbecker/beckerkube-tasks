---
id: TASK-002
title: Rebuild and Push All Service Images with Version Increments
status: active
priority: critical
created: 2025-10-17
updated: 2025-10-17
assignee: unassigned
labels: [builds, deployment, registry]
---

## Description

All services need to be rebuilt and pushed with incremented version numbers, not just "latest" tags. After the cluster rebuild, images need to be pushed to the new registry IP with proper semantic versioning.

This task tracks the rebuild and push of all service container images to ensure they are available in the registry for Flux deployments.

## Context

After the cluster rebuild:
- **CCbot**: Successfully built and pushed (v2.0.2) ✅
- **FFL**: Build failed due to wrong registry IP, needs rebuild with version 0.1.18
- **Midwestmtg**: Built but not pushed (missing --push flag), needs version 0.1.12+
- **Triager**: Build failed, needs investigation and version 0.1.1+
- **MTG Dev Agents**: Still building evaluator, various versions

Registry URL: 192.168.7.21:5000

## Service Version Matrix

| Service | Current Chart Version | Target Image Version | Status |
|---------|----------------------|---------------------|---------|
| CCbot | 2.0.1 | 2.0.2 | ✅ Complete |
| FFL Backend | 0.1.17 | 0.1.18 | ❌ Failed (wrong IP) |
| MidwestMTG Backend | >=0.1.11 <1.0.0 | 0.1.12 | ⚠️ Built, not pushed |
| Triager Orchestrator | >=0.1.0 <1.0.0 | 0.1.1 | ❌ Build failure |
| MTG Dev Agents - Orchestrator | Check chart | TBD | 🔄 In progress |
| MTG Dev Agents - Worker | Check chart | TBD | 🔄 In progress |
| MTG Dev Agents - Evaluator | Check chart | TBD | 🔄 In progress |
| MTG Dev Agents - Stenographer | Check chart | TBD | 🔄 In progress |

## Acceptance Criteria

### FFL
- [ ] Build all FFL containers with version 0.1.18
- [ ] Push all FFL images to registry 192.168.7.21:5000
- [ ] Update ffl HelmRelease in beckerkube to version 0.1.18
- [ ] Verify images in registry: `curl -k https://192.168.7.21:5000/v2/ffl/backend/tags/list`

### MidwestMTG
- [ ] Build all midwestmtg containers with version 0.1.12
- [ ] Push all midwestmtg images to registry 192.168.7.21:5000
- [ ] Update midwestmtg HelmRelease in beckerkube to version 0.1.12
- [ ] Verify images in registry: `curl -k https://192.168.7.21:5000/v2/midwestmtg/backend/tags/list`

### Triager
- [ ] Debug and fix triager build failure (see TASK-005)
- [ ] Build all triager containers with version 0.1.1
- [ ] Push all triager images to registry 192.168.7.21:5000
- [ ] Update triager HelmRelease in beckerkube to version 0.1.1
- [ ] Verify images in registry: `curl -k https://192.168.7.21:5000/v2/triager/orchestrator/tags/list`

### MTG Dev Agents
- [ ] Complete current evaluator build
- [ ] Build all remaining mtg_dev_agents containers
- [ ] Push all mtg_dev_agents images to registry 192.168.7.21:5000
- [ ] Update all mtg_dev_agents HelmReleases in beckerkube
- [ ] Verify images in registry for all agents

### Post-Build Verification
- [ ] Verify all images are in registry catalog: `curl -k https://192.168.7.21:5000/v2/_catalog`
- [ ] Check image sizes are reasonable
- [ ] Verify image manifests are valid
- [ ] Document final versions in this task

## Build Commands

### FFL (version 0.1.18)
```bash
cd ~/projects/ffl
REGISTRY_URL=192.168.7.21:5000 VERSION=0.1.18 ./scripts/build.sh --push all
```

### MidwestMTG (version 0.1.12)
```bash
cd ~/projects/midwestmtg
REGISTRY_URL=192.168.7.21:5000 VERSION=0.1.12 ./scripts/build.sh --push
```

### Triager (version 0.1.1)
```bash
# First investigate failure (TASK-005), then:
cd ~/projects/triager
REGISTRY_URL=192.168.7.21:5000 VERSION=0.1.1 ./scripts/build.sh --push
```

### MTG Dev Agents
```bash
cd ~/projects/mtg_dev_agents
# Check current build status first
make build
# Or build specific agents:
make build-orchestrator
make build-worker
make build-evaluator
make build-stenographer
```

## Dependencies

- **TASK-001**: Must be completed first (registry IP configuration)
- **TASK-005**: Triager build failure must be resolved

## Blocks

- **TASK-003**: Flux reconciliation depends on all images being available

## Progress Log

- 2025-10-17 09:00: CCbot successfully built and pushed (v2.0.2)
- 2025-10-17 09:30: FFL build failed due to wrong registry IP in .env.build.local
- 2025-10-17 09:35: Midwestmtg built successfully but not pushed (missing --push flag)
- 2025-10-17 09:40: Triager build failed with exit code 1
- 2025-10-17 10:00: MTG Dev Agents evaluator build started
- 2025-10-17 10:15: Fixed registry IPs in all .env.build.local files (TASK-001)
- 2025-10-17 10:40: Ready to retry FFL and midwestmtg builds

## Next Steps

1. **Immediate**: Retry FFL build with corrected registry IP
   ```bash
   cd ~/projects/ffl
   REGISTRY_URL=192.168.7.21:5000 VERSION=0.1.18 ./scripts/build.sh --push all
   ```

2. **Immediate**: Push midwestmtg images that were already built
   ```bash
   cd ~/projects/midwestmtg
   REGISTRY_URL=192.168.7.21:5000 VERSION=0.1.12 ./scripts/build.sh --push
   ```

3. **High Priority**: Wait for MTG Dev Agents build to complete, then verify

4. **High Priority**: Investigate triager build failure (TASK-005)

5. **After All Builds**: Update beckerkube HelmRelease files with new versions

6. **Verification**: Check registry catalog and verify all images are present

## Notes

- Always use explicit version tags, not "latest"
- Version numbers should follow semantic versioning (MAJOR.MINOR.PATCH)
- Update Helm chart versions to match or increment appropriately
- Document any build issues or special considerations
- Consider automating version increment in CI/CD pipeline
- Keep track of which versions are deployed in each environment

## Troubleshooting

If builds fail:
1. Check Docker daemon is running
2. Verify registry is accessible: `curl -k https://192.168.7.21:5000/v2/_catalog`
3. Check registry certificate is trusted
4. Verify .env.build.local has correct registry URL
5. Check build logs for specific errors
6. Ensure sufficient disk space for image layers
