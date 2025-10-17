---
id: TASK-004
title: Audit All LoadBalancer IP Configurations
status: completed
priority: medium
created: 2025-10-17
updated: 2025-10-17
completed: 2025-10-17
assignee: unassigned
labels: [infrastructure, networking, documentation]
retrospective: docs/retrospectives/retro-20251017-111558.md
---

## Description

Comprehensive audit of all LoadBalancer services and their IP assignments to ensure consistency across the cluster after the rebuild. Document all IPs in a central location for reference.

## Context

After the minikube cluster rebuild, LoadBalancer IPs were reassigned. This audit ensures all services have stable IP assignments and that these are documented for future reference and troubleshooting.

## Acceptance Criteria

- [x] Registry: 192.168.7.21:5000
- [x] ChartMuseum: 192.168.7.18:8080
- [x] PostgreSQL: 192.168.7.17:5432
- [x] Ingress-nginx: 192.168.7.20:80/443
- [x] Istio Gateway: 192.168.7.19:80/443
- [x] Document all IPs in beckerkube-tasks/docs/architecture.md
- [x] Verify all services are accessible at their assigned IPs

## Audit Results

### Infrastructure Services

| Service | Type | IP Address | Port(s) | Status |
|---------|------|------------|---------|--------|
| registry | LoadBalancer | 192.168.7.21 | 5000 | ✅ Operational |
| chartmuseum | LoadBalancer | 192.168.7.18 | 8080 | ✅ Operational |
| postgresql | LoadBalancer | 192.168.7.17 | 5432 | ✅ Operational |
| ingress-nginx | LoadBalancer | 192.168.7.20 | 80, 443 | ✅ Operational |
| istio-gateway | LoadBalancer | 192.168.7.19 | 80, 443 | ✅ Operational |

### Verification Commands Used

```bash
# List all LoadBalancer services
kubectl get svc -A -o wide | grep LoadBalancer

# Test registry
curl -k https://192.168.7.21:5000/v2/_catalog

# Test ChartMuseum
curl http://192.168.7.18:8080/api/charts

# Test PostgreSQL
nc -zv 192.168.7.17 5432

# Test ingress-nginx
curl -k https://192.168.7.20

# Test istio-gateway
curl -k https://192.168.7.19
```

## IP Address Allocation

IP range: 192.168.7.17 - 192.168.7.21 (5 addresses allocated)

```
192.168.7.17 - PostgreSQL (database)
192.168.7.18 - ChartMuseum (helm charts)
192.168.7.19 - Istio Gateway (service mesh ingress)
192.168.7.20 - Ingress-Nginx (http/https ingress)
192.168.7.21 - Registry (container images)
```

## Configuration Locations

### Registry IP References
- beckerkube/infra/registry/registry-service.yaml (if using LoadBalancer annotation)
- beckerkube/apps/*/helmrelease.yaml (image repository URLs)
- All project .env.build.local files
- Build scripts in each service repository

### Database IP References
- beckerkube/apps/*/helmrelease.yaml (database connection strings)
- Application ConfigMaps or Secrets

### Ingress IP References
- No direct references (services use ingress hostnames)
- External access uses LoadBalancer IP directly

## Progress Log

- 2025-10-17 09:00: Started LoadBalancer IP audit
- 2025-10-17 09:15: Listed all LoadBalancer services with `kubectl get svc -A`
- 2025-10-17 09:20: Verified registry at 192.168.7.21:5000
- 2025-10-17 09:25: Verified ChartMuseum at 192.168.7.18:8080
- 2025-10-17 09:30: Verified PostgreSQL at 192.168.7.17:5432
- 2025-10-17 09:35: Verified ingress-nginx at 192.168.7.20
- 2025-10-17 09:40: Verified Istio Gateway at 192.168.7.19
- 2025-10-17 10:00: Documented IPs in docs/architecture.md
- 2025-10-17 10:15: Task completed successfully

## Documentation Updated

- Created beckerkube-tasks/docs/architecture.md with full IP allocation table
- Included service descriptions and purposes
- Added verification commands for future reference

## Recommendations

### For Future Cluster Rebuilds

1. **Use Static IP Assignments**: Configure LoadBalancer services with specific IP addresses in service annotations
   ```yaml
   metadata:
     annotations:
       metallb.universe.tf/address-pool: cluster-services
       metallb.universe.tf/loadBalancerIPs: 192.168.7.21
   ```

2. **Automated IP Detection**: Create script to detect LoadBalancer IPs and update configuration files
   ```bash
   #!/bin/bash
   # detect-registry-ip.sh
   REGISTRY_IP=$(kubectl get svc -n registry registry -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
   echo "Registry IP: $REGISTRY_IP"
   # Update .env.build.local files...
   ```

3. **Centralized Configuration**: Store LoadBalancer IPs in a ConfigMap or central config file
   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: cluster-ips
     namespace: kube-system
   data:
     registry: "192.168.7.21:5000"
     chartmuseum: "192.168.7.18:8080"
     postgresql: "192.168.7.17:5432"
   ```

4. **Documentation**: Keep docs/architecture.md updated with any IP changes

5. **Service Discovery**: Prefer Kubernetes DNS names for in-cluster communication
   - Use `registry.registry.svc.cluster.local:5000` instead of IP
   - LoadBalancer IPs only needed for external access

## Notes

- LoadBalancer IPs in Minikube can change on cluster deletion/recreation
- Consider using MetalLB with static IP pool configuration
- Document IP allocation in beckerkube repository as well
- All services are healthy and accessible at documented IPs
- No conflicts or duplicate IP assignments found
