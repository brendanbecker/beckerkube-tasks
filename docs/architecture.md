# BeckerKube Cluster Architecture

## Overview

BeckerKube is a Kubernetes cluster infrastructure managed via GitOps using Flux CD. The cluster runs on Minikube for local development and testing.

## Core Components

### GitOps and Deployment

- **Flux CD**: Continuous delivery tool that automatically syncs cluster state from Git repository
  - Source: beckerkube repository
  - Reconciliation interval: 1 minute
  - Auto-sync enabled for most resources

### Infrastructure Services

#### Container Registry
- **Service**: Docker Registry v2
- **LoadBalancer IP**: 192.168.7.21:5000
- **Purpose**: Private container image registry for all cluster applications
- **TLS**: Self-signed certificate
- **Storage**: Persistent volume backed

#### Helm Chart Repository
- **Service**: ChartMuseum
- **LoadBalancer IP**: 192.168.7.18:8080
- **Purpose**: Private Helm chart repository
- **Storage**: Persistent volume backed

#### Database
- **Service**: PostgreSQL
- **LoadBalancer IP**: 192.168.7.17:5432
- **Purpose**: Shared database for cluster applications
- **Databases**:
  - `stenographer` - Development logging for mtg_dev_agents
  - `triager` - Issue triage automation
  - Additional application databases

### Networking

#### Ingress Controllers
- **ingress-nginx**
  - LoadBalancer IP: 192.168.7.20:80/443
  - Purpose: HTTP/HTTPS ingress for most services
  - TLS termination

- **Istio Gateway**
  - LoadBalancer IP: 192.168.7.19:80/443
  - Purpose: Service mesh ingress
  - Advanced traffic management

### Monitoring and Observability

- **Prometheus**: Metrics collection and alerting
- **Grafana**: Metrics visualization and dashboards
- **Fluent Bit**: Log forwarding and aggregation

### Security

- **Pod Security Admission**: Enforces pod security standards
- **RBAC**: Role-based access control for all services
- **Network Policies**: Segmentation and traffic control
- **Sealed Secrets**: Encrypted secrets management

## Application Deployments

### MTG Dev Agents
Multi-agent development system with A2A protocol:
- **Orchestrator**: Strategic planning and coordination
- **Generic Worker**: Core implementation tasks
- **Evaluator**: Code quality and rules compliance
- **Stenographer**: Development logging and documentation

Registry: 192.168.7.21:5000/mtg-dev-agents/*

### Fantasy Football League (FFL)
Backend services for fantasy football application:
- **Backend API**: REST API service
- **Frontend**: Web application

Registry: 192.168.7.21:5000/ffl/*

### MidwestMTG
Backend services for Magic: The Gathering tournament management:
- **Backend API**: REST API service
- **Frontend**: Web application

Registry: 192.168.7.21:5000/midwestmtg/*

### Triager
Issue triage automation system:
- **Orchestrator**: Task coordination
- **Workers**: Issue processing

Registry: 192.168.7.21:5000/triager/*

### CCBot
Discord bot for community management:
- **Bot Service**: Discord integration

Registry: 192.168.7.21:5000/ccbot/*

## LoadBalancer IP Assignments

| Service | IP Address | Ports | Purpose |
|---------|------------|-------|---------|
| Registry | 192.168.7.21 | 5000 | Container image registry |
| ChartMuseum | 192.168.7.18 | 8080 | Helm chart repository |
| PostgreSQL | 192.168.7.17 | 5432 | Shared database |
| ingress-nginx | 192.168.7.20 | 80, 443 | HTTP/HTTPS ingress |
| Istio Gateway | 192.168.7.19 | 80, 443 | Service mesh ingress |

## Persistent Storage

- **Storage Class**: standard (Minikube default)
- **Volumes**:
  - Registry: Container images
  - ChartMuseum: Helm charts
  - PostgreSQL: Database data
  - Application-specific volumes as needed

## Build and Deployment Workflow

1. **Code Changes**: Developer commits to application repository
2. **Build**: Container images built with version tags
3. **Push**: Images pushed to private registry (192.168.7.21:5000)
4. **Update Manifests**: HelmRelease files updated in beckerkube repo
5. **Git Push**: Changes committed to beckerkube repository
6. **Flux Sync**: Flux detects changes and reconciles cluster state
7. **Deployment**: Kubernetes pulls new images and updates deployments
8. **Verification**: Health checks and monitoring confirm successful deployment

## Security Architecture

### Network Segmentation
- Default deny network policies
- Explicit allow rules for required communication
- Namespace isolation

### Authentication and Authorization
- Service accounts with minimal required permissions
- RBAC policies for all components
- Pod security standards enforcement

### Secret Management
- Sealed Secrets for encrypted storage in Git
- Secrets injected as environment variables or mounted volumes
- Rotation procedures documented in runbooks

## Disaster Recovery

### Backup Strategy
- Persistent volume snapshots
- Database backups (PostgreSQL dump)
- Git as source of truth for configuration

### Recovery Procedures
See [docs/runbooks/](runbooks/) for detailed procedures:
- Cluster rebuild from scratch
- Service restoration
- Data recovery

## Monitoring and Alerting

### Metrics
- Node metrics (CPU, memory, disk)
- Pod metrics (resource usage, restart counts)
- Application metrics (exposed via /metrics endpoints)

### Logs
- Centralized logging via Fluent Bit
- Log retention policies
- Search and analysis capabilities

### Alerts
- Critical: Cluster or service down
- High: Resource exhaustion, repeated failures
- Medium: Performance degradation
- Low: Information and warnings

## Known Issues and Limitations

### Minikube Specific
- LoadBalancer IPs can change on cluster rebuild
- Limited to single-node cluster
- Performance constraints based on host resources

### Workarounds
- Document LoadBalancer IPs in this file
- Automated IP update scripts for registry configuration
- Regular cluster state backups

## Future Enhancements

See [features/planned/](../features/planned/) for planned improvements:
- High availability setup
- Automated backup and restore
- Enhanced monitoring and alerting
- Service mesh adoption across all services
- Progressive delivery with Flagger

## Related Documentation

- [README.md](../README.md) - Task management overview
- [runbooks/](runbooks/) - Operational procedures
- [decisions/](decisions/) - Architecture decision records
- [beckerkube repository](../../beckerkube) - GitOps manifests and charts
