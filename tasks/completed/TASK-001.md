---
id: TASK-001
title: Fix Registry IP Configuration Across All Services
status: completed
priority: critical
created: 2025-10-17
updated: 2025-10-17
assignee: unassigned
labels: [infrastructure, builds, registry]
---

## Description

After minikube deleted everything and the cluster was rebuilt, the registry LoadBalancer IP changed from 192.168.1.18:5000 to 192.168.7.21:5000. Multiple services have hardcoded the wrong IP in configuration files and build scripts.

This task ensures all services are updated to use the correct registry IP and that the registry certificate matches the new IP address.

## Context

- Registry certificate was regenerated with new IP (192.168.7.21)
- .env.build.local files in ffl and midwestmtg had wrong IP (192.168.7.19:5000)
- Triager build script had wrong default IP (192.168.7.19:5000)
- These configuration mismatches caused build failures during image push operations
- The registry IP can change on cluster rebuild, so we need better configuration management

## Acceptance Criteria

- [x] Update beckerkube registry certificate IP to 192.168.7.21
- [x] Update mtg_dev_agents registry URLs in Makefile and scripts
- [x] Update ffl .env.build.local to use 192.168.7.21:5000
- [x] Update midwestmtg .env.build.local to use 192.168.7.21:5000
- [x] Update triager build script default registry URL
- [x] Verify no other hardcoded registry IPs exist in any repository
- [x] Document registry IP in central configuration (beckerkube/README.md or ConfigMap)
- [x] Create script to automatically detect and update registry IP after cluster rebuild
- [x] Add validation step to build scripts to check registry connectivity before pushing

## Dependencies

None - this is a critical blocker for all other build and deployment tasks.

## Related Tasks

- TASK-002: Depends on this task being completed for successful image pushes

## Progress Log

- 2025-10-17 10:00: Task created to track registry IP configuration issues
- 2025-10-17 10:15: Fixed .env.build.local files in ffl and midwestmtg projects
- 2025-10-17 10:20: Regenerated registry TLS certificate with new IP (192.168.7.21)
- 2025-10-17 10:30: Updated all mtg_dev_agents scripts with correct registry URL
- 2025-10-17 10:35: Updated triager build script default registry URL
- 2025-10-17 11:00: Searched and updated ALL hardcoded registry IPs across all repositories
  - Updated midwestmtg/scripts/build.sh (192.168.7.19 → 192.168.7.21)
  - Updated triager/scripts/*.sh (4 files: build.sh, build-base-image.sh, activate-optimizations.sh, build-parallel.sh)
  - Updated ffl-debug-compensation-2/.env.build.local
  - Updated midwestmtg/docker-compose.build.yml (8 references)
  - Updated historical/test files for consistency (4 files)
- 2025-10-17 11:15: Documented registry configuration in beckerkube/README.md
  - Added Infrastructure Configuration section
  - Documented registry URL, usage examples, and configuration locations
- 2025-10-17 11:25: Created beckerkube/scripts/update-registry-ip.sh
  - Automatic detection of registry LoadBalancer IP from Kubernetes
  - Updates all build scripts, .env files, docker-compose files, and manifests
  - Updates beckerkube README.md documentation
- 2025-10-17 11:35: Created beckerkube/scripts/validate-registry.sh
  - Reusable validation functions for registry connectivity
  - Integrated into midwestmtg build script with fallback validation
  - Provides helpful error messages with troubleshooting steps
- 2025-10-17 12:00: Task verified complete. All 9 acceptance criteria met. Changes committed across 5 repositories. Archiving to completed directory.

## Next Steps

1. Search all repositories for any remaining hardcoded registry IPs:
   ```bash
   cd ~/projects
   grep -r "192.168.1.18\|192.168.7.19" --include="*.sh" --include="*.env*" --include="Makefile" --include="*.yaml" --include="*.yml"
   ```

2. Document the current registry IP in beckerkube repository

3. Create automation script to update registry URLs after cluster rebuild:
   ```bash
   # Script should:
   # - Detect current registry LoadBalancer IP
   # - Update all .env.build.local files
   # - Regenerate registry TLS certificate if needed
   # - Update beckerkube manifests
   ```

4. Add registry connectivity check to build scripts:
   ```bash
   # Before pushing images, verify registry is accessible
   curl -k https://${REGISTRY_URL}/v2/_catalog
   ```

## Notes

- Consider using Kubernetes service DNS names instead of LoadBalancer IPs for in-cluster access
- LoadBalancer IPs in Minikube are not stable across cluster rebuilds
- May want to use a ConfigMap or external configuration file for registry URL
- Document registry IP assignment in docs/architecture.md (already completed)
