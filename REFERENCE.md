# 📦 Platform Component Reference

## Installed Components

### Monitoring Stack
| Component | Version | Purpose | Namespace |
|-----------|---------|---------|-----------|
| kube-prometheus-stack | 56.0.0 | Metrics collection & alerting | monitoring |
| Grafana | (included) | Metrics visualization | monitoring |
| Prometheus | (included) | Time-series database | monitoring |
| Alertmanager | (included) | Alert routing to Teams | monitoring |
| Loki | 2.10.0 | Log aggregation | monitoring |
| Promtail | (included) | Log collection | monitoring |

### Security Stack
| Component | Version | Purpose | Namespace |
|-----------|---------|---------|-----------|
| Kyverno | 3.1.0 | Policy engine | kyverno |
| Kyverno Policies | 3.1.0 | Pod Security Standards | kyverno |
| Kubescape | 1.18.0 | Security scanning (CIS, NSA) | kubescape |
| Tetragon | 1.0.0 | eBPF runtime security | tetragon |

### Ingress
| Component | Version | Purpose | Namespace |
|-----------|---------|---------|-----------|
| NGINX Ingress | 4.9.0 | Ingress controller | ingress-nginx |

### Platform (installed via Terraform)
| Component | Version | Purpose | Namespace |
|-----------|---------|---------|-----------|
| ArgoCD | Latest | GitOps controller | argocd |
| cert-manager | Latest | TLS certificate automation | cert-manager |
| Keycloak | Latest | SSO/OIDC provider | keycloak |

---

## Service URLs (Template)

Replace `<domain>` with your environment domain:
- Dev: `dev.example.com`
- Test: `test.example.com`  
- Prod: `example.com`

### Platform Services
- **ArgoCD**: https://argocd.<domain>
- **Keycloak**: https://keycloak.<domain>

### Monitoring & Observability
- **Grafana**: https://grafana.<domain>
- **Prometheus**: https://prometheus.<domain> (optional)
- **Alertmanager**: https://alertmanager.<domain> (optional)

---

## Keycloak Configuration

### Realms
- **platform**: Main realm for all platform services

### Clients
| Client ID | Service | Redirect URI |
|-----------|---------|--------------|
| argocd | ArgoCD | https://argocd.<domain>/auth/callback |
| grafana | Grafana | https://grafana.<domain>/login/generic_oauth |
| prometheus | Prometheus | https://prometheus.<domain>/oauth2/callback |

### Roles
| Role | Description | Access Level |
|------|-------------|--------------|
| platform-admin | Full platform access | All services |
| platform-developer | Developer access | Limited to applications |
| platform-viewer | Read-only access | Monitoring only |

---

## ArgoCD Projects

### platform (Admin-level)
**Purpose**: Infrastructure and platform services

**Permissions**:
- Full cluster access
- All namespaces
- All resource types

**Namespaces**: monitoring, security, ingress-nginx, kyverno, kubescape, tetragon

**Applications**:
- kube-prometheus-stack
- loki
- kyverno
- kyverno-policies
- kubescape
- tetragon
- ingress-nginx

### applications (Developer-level)
**Purpose**: Customer workloads

**Permissions**:
- Limited to `app-*` namespaces
- Restricted resource types
- No cluster-scoped resources

**Namespaces**: app-*

**Applications**: User-deployed applications

---

## Prometheus Metrics

### Key Metrics to Monitor
```promql
# Node CPU usage
100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Node memory usage
100 * (1 - ((node_memory_MemAvailable_bytes) / (node_memory_MemTotal_bytes)))

# Pod CPU usage
sum(rate(container_cpu_usage_seconds_total[5m])) by (pod, namespace)

# Pod memory usage
sum(container_memory_working_set_bytes) by (pod, namespace)

# Ingress request rate
sum(rate(nginx_ingress_controller_requests[5m])) by (ingress, status)
```

### ServiceMonitors
All platform components expose metrics via ServiceMonitors:
- kube-state-metrics
- node-exporter
- ingress-nginx
- argocd-metrics
- kyverno-admission-controller

---

## Grafana Dashboards

### Pre-installed Dashboards
- **Kubernetes / Compute Resources / Cluster**: Overall cluster health
- **Kubernetes / Compute Resources / Namespace (Pods)**: Per-namespace metrics
- **Kubernetes / Compute Resources / Node (Pods)**: Per-node metrics
- **Kubernetes / Networking / Cluster**: Network traffic
- **NGINX Ingress controller**: Ingress metrics
- **ArgoCD**: Application sync status
- **Kyverno Policy Reporter**: Policy violations

### Custom Dashboards
Create additional dashboards for:
- Application-specific metrics
- Business KPIs
- Security alerts
- Cost tracking

---

## Alertmanager Rules

### Critical Alerts (PagerDuty/Teams)
- Node down
- Pod crash loop
- Persistent volume full
- Certificate expiring
- High error rate

### Warning Alerts (Teams only)
- High CPU usage
- High memory usage
- Pod pending too long
- Failed security scan

### Alert Routing
```yaml
route:
  receiver: 'teams-notifications'
  routes:
    - match:
        severity: critical
      receiver: 'teams-critical'
    - match:
        severity: warning
      receiver: 'teams-warnings'
```

---

## Kyverno Policies

### Enforced Policies
- **require-non-root**: Pods must run as non-root
- **restrict-privilege-escalation**: No privilege escalation
- **require-ro-rootfs**: Root filesystem must be read-only
- **disallow-capabilities**: Restrict Linux capabilities
- **require-network-policy**: All namespaces must have NetworkPolicies
- **require-resource-limits**: Pods must have CPU/memory limits

### Audit Policies
- **check-deprecated-apis**: Warn on deprecated Kubernetes APIs
- **validate-ingress-annotations**: Ensure proper annotations

---

## Kubescape Scans

### Frameworks
- **CIS Kubernetes Benchmark**: Industry standard
- **NSA/CISA Hardening Guide**: Government security standards
- **MITRE ATT&CK**: Attack pattern detection

### Scan Frequency
- Continuous: On pod creation
- Scheduled: Daily full cluster scan
- On-demand: Via CLI or API

### Compliance Targets
- Dev: 70% compliance
- Test: 85% compliance
- Prod: 95% compliance

---

## Tetragon Policies

### Monitored Events
- Process execution
- Network connections
- File access
- Syscalls

### Example Policies
```yaml
# Detect shell spawning in containers
apiVersion: cilium.io/v1alpha1
kind: TracingPolicy
metadata:
  name: detect-shell-spawn
spec:
  kprobes:
  - call: "sys_execve"
    syscall: true
    args:
    - index: 0
      type: "string"
    selectors:
    - matchBinaries:
      - operator: "In"
        values:
        - "/bin/bash"
        - "/bin/sh"
```

---

## Backup and Disaster Recovery

### What to Backup
- **Grafana dashboards**: Export JSON via API
- **Prometheus data**: Use Thanos or remote storage
- **Loki logs**: Persist to Azure Blob Storage
- **ArgoCD config**: Git is source of truth
- **Keycloak realms**: Export realm JSON

### Backup Schedule
- Grafana: Daily
- Prometheus: Continuous (remote write)
- Loki: Continuous (object storage)
- Keycloak: Daily

### Recovery Time Objectives
- Monitoring: 15 minutes (redeploy via ArgoCD)
- Security: 10 minutes (redeploy via ArgoCD)
- Applications: Varies by SLA

---

## Troubleshooting Commands

### ArgoCD
```bash
# Check application status
argocd app get <app-name>

# View sync errors
argocd app sync <app-name> --dry-run

# Refresh and hard sync
argocd app sync <app-name> --force --prune
```

### Prometheus
```bash
# Check Prometheus targets
kubectl port-forward -n monitoring svc/prometheus-operated 9090:9090
# Visit http://localhost:9090/targets

# Check alerts
# Visit http://localhost:9090/alerts
```

### Grafana
```bash
# Reset admin password
kubectl exec -n monitoring deployment/grafana -- grafana-cli admin reset-admin-password <new-password>
```

### Kyverno
```bash
# View policy reports
kubectl get policyreports -A

# View cluster policy reports
kubectl get clusterpolicyreports
```

### Kubescape
```bash
# Manual scan
kubectl exec -n kubescape deployment/kubescape -- kubescape scan framework cis
```

---

## Performance Tuning

### Prometheus Retention
- Dev: 7 days
- Test: 14 days
- Prod: 30 days

### Loki Retention
- Dev: 7 days
- Test: 14 days
- Prod: 30 days

### Storage Sizing
| Component | Dev | Test | Prod |
|-----------|-----|------|------|
| Prometheus | 20Gi | 30Gi | 50Gi |
| Loki | 10Gi | 20Gi | 30Gi |
| Grafana | 5Gi | 5Gi | 10Gi |

### Resource Requests/Limits
See `apps/monitoring/kube-prometheus-stack.yaml` for default values.
Adjust based on cluster size and workload.

---

## Security Hardening Checklist

- [ ] Keycloak SSO configured for all UIs
- [ ] TLS certificates from Let's Encrypt (not self-signed)
- [ ] Kyverno policies in enforce mode
- [ ] Kubescape scans passing >90%
- [ ] Tetragon policies active
- [ ] NetworkPolicies in all namespaces
- [ ] Pod Security Standards: Restricted
- [ ] No privileged containers
- [ ] No root containers
- [ ] Secrets in Azure Key Vault (not in Git)
- [ ] RBAC configured per least privilege
- [ ] Regular security updates via Renovate/Dependabot

---

**Last Updated**: November 2025  
**Version**: 1.0.0  
**Maintained By**: Platform Engineering Team
