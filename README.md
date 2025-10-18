# BeckerKube Task Management

Task and feature management repository for the BeckerKube Kubernetes cluster infrastructure.

## Overview

This repository tracks all work items, bugs, features, and operational tasks for the BeckerKube cluster. It serves as the central source of truth for infrastructure work and complements the main [beckerkube](../beckerkube) GitOps repository.

## Repository Structure

```
beckerkube-tasks/
├── README.md                 # This file
├── .github/
│   └── ISSUE_TEMPLATE/      # GitHub issue templates
├── tasks/                   # General infrastructure tasks
│   ├── active/              # Currently in progress
│   ├── backlog/             # Not yet started
│   ├── completed/           # Finished tasks
│   └── blocked/             # Waiting on dependencies
├── bugs/                    # Bug tracking
│   ├── active/              # Being worked on
│   ├── investigating/       # Under investigation
│   └── resolved/            # Fixed bugs
├── features/                # Feature development
│   ├── planned/             # Future features
│   └── in-progress/         # Active development
└── docs/                    # Documentation
    ├── architecture.md      # Cluster architecture overview
    ├── runbooks/           # Operational procedures
    └── decisions/          # Architecture decision records
```

## Task Format

All task files use the following markdown frontmatter format:

```markdown
---
id: TASK-XXX
title: Brief title
status: [backlog|active|blocked|completed]
priority: [critical|high|medium|low]
created: YYYY-MM-DD
updated: YYYY-MM-DD
assignee: [name or "unassigned"]
labels: [infrastructure, security, builds, etc]
---

## Description
Detailed description of the task

## Context
Background information and why this task is needed

## Acceptance Criteria
- [ ] Criterion 1
- [ ] Criterion 2

## Dependencies
- Links to other tasks or external dependencies

## Progress Log
- YYYY-MM-DD: Progress update
```

## Workflow

### Creating New Tasks

1. Create a new markdown file in the appropriate directory:
   - `tasks/backlog/` for general infrastructure work
   - `bugs/investigating/` for new bugs
   - `features/planned/` for new feature requests

2. Use the next available ID (TASK-XXX, BUG-XXX, or FEAT-XXX)

3. Fill in all frontmatter fields

4. Add detailed description and acceptance criteria

### Moving Tasks

When task status changes, move the file to the appropriate directory:

```bash
# Starting work on a task
mv tasks/backlog/TASK-001.md tasks/active/

# Completing a task
mv tasks/active/TASK-001.md tasks/completed/

# Blocking a task
mv tasks/active/TASK-001.md tasks/blocked/
```

### Updating Tasks

1. Update the `updated` field in frontmatter
2. Add entry to Progress Log section
3. Check off acceptance criteria as completed
4. Move file if status changes

## Priority Levels

- **Critical**: Cluster is down or severely degraded, immediate action required
- **High**: Significant impact on operations, should be addressed soon
- **Medium**: Important but not urgent, normal priority
- **Low**: Nice to have, address when time permits

## Common Labels

- `infrastructure`: Core cluster infrastructure
- `security`: Security policies, RBAC, network policies
- `builds`: Container image builds and registry
- `deployment`: Application deployments and Helm releases
- `flux`: Flux CD and GitOps workflow
- `monitoring`: Prometheus, Grafana, logging
- `networking`: Ingress, load balancers, service mesh
- `storage`: Persistent volumes and storage classes
- `debugging`: Troubleshooting and investigation

## Integration with BeckerKube

This repository complements the main beckerkube repository:

- **beckerkube**: GitOps manifests, Helm charts, cluster configuration
- **beckerkube-tasks**: Work item tracking, documentation, runbooks

When completing tasks that involve infrastructure changes:

1. Update task with progress in beckerkube-tasks
2. Make changes in beckerkube repository
3. Commit and push to beckerkube
4. Wait for Flux to reconcile
5. Verify deployment
6. Mark task as completed in beckerkube-tasks

## Quick Commands

### View Active Tasks
```bash
ls -lh tasks/active/
```

### Search Tasks
```bash
grep -r "registry" tasks/
```

### Count Tasks by Status
```bash
echo "Active: $(ls tasks/active/ | wc -l)"
echo "Backlog: $(ls tasks/backlog/ | wc -l)"
echo "Completed: $(ls tasks/completed/ | wc -l)"
echo "Blocked: $(ls tasks/blocked/ | wc -l)"
```

### View Recent Updates
```bash
find tasks/ -type f -name "*.md" -mtime -7
```

## Related Repositories

- [beckerkube](../beckerkube) - Kubernetes GitOps infrastructure
- [mtg_dev_agents](../mtg_dev_agents) - Multi-agent development system
- [triager](../triager) - Issue triage automation
- [ffl](../ffl) - FFL backend application
- [midwestmtg](../midwestmtg) - MidwestMTG backend application

## Getting Started

Current critical tasks are in `tasks/active/`. Start there to understand immediate priorities.

For cluster architecture and component overview, see [docs/architecture.md](docs/architecture.md).

For operational procedures, check [docs/runbooks/](docs/runbooks/).
