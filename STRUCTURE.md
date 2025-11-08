# 📁 plattform-gitops Repository Structure

This repository implements a **GitOps-based deployment strategy** using ArgoCD for managing AKS platform services across multiple environments (dev, test, prod).

## 🏗️ Architecture Pattern

This repository follows the **App of Apps** pattern with environment-specific overlays:

```
plattform-gitops/
├── bootstrap/                    # ArgoCD bootstrap configuration (App of Apps)
│   ├── root-app.yaml            # Root Application that deploys all apps
│   ├── appproject-platform.yaml # Platform AppProject (admin-level)
│   ├── appproject-applications.yaml # Applications AppProject (dev-level)
│   └── apps/                    # App of Apps definitions
│       ├── monitoring.yaml      # Monitoring stack Application
│       ├── security.yaml        # Security stack Application
│       └── ingress.yaml         # Ingress controller Application
│
├── apps/                        # Application manifests and Helm values
│   ├── monitoring/              # Monitoring stack
│   │   ├── kube-prometheus-stack.yaml  # Prometheus, Grafana, Alertmanager
│   │   └── loki.yaml                   # Loki logging stack
│   ├── security/                # Security and compliance tools
│   │   ├── kyverno.yaml                # Policy engine
│   │   ├── kyverno-policies.yaml       # Pod Security Standards
│   │   ├── kubescape.yaml              # Security scanner (CIS, NSA, MITRE)
│   │   └── tetragon.yaml               # eBPF-based runtime security
│   ├── ingress/                 # Ingress controllers
│   │   └── ingress-nginx.yaml          # NGINX Ingress Controller
│   └── applications/            # Customer applications (optional)
│
└── environments/                # Environment-specific configurations
    ├── dev/
    │   ├── README.md            # Dev environment documentation
    │   └── values.yaml          # Dev-specific values (domain, retention, etc.)
    ├── test/
    │   ├── README.md            # Test environment documentation
    │   └── values.yaml          # Test-specific values
    └── prod/
        ├── README.md            # Prod environment documentation
        └── values.yaml          # Prod-specific values
```

---

## 🔄 GitOps Workflow

### 1️⃣ Bootstrap Phase (One-time setup)
After Terraform creates the AKS cluster and installs ArgoCD via Helm:

```bash
# Apply the root application to ArgoCD
kubectl apply -f bootstrap/root-app.yaml

# This triggers the App of Apps pattern:
# root-app → deploys apps from bootstrap/apps/ → each app deploys from apps/<category>/
```

### 2️⃣ Application Deployment
ArgoCD automatically:
- Monitors this Git repository
- Detects changes in `apps/` directories
- Deploys/updates applications based on Helm charts or manifests
- Syncs according to `syncPolicy` (automated or manual)

### 3️⃣ Environment-Specific Overrides
Use Kustomize or Helm values from `environments/<env>/values.yaml` to customize:
- Domain names
- Retention policies
- Storage sizes
- Certificate issuers (staging vs. prod)
- Alert webhooks

---

## 📦 Application Categories

### **Monitoring** (`apps/monitoring/`)
- **kube-prometheus-stack**: Full Prometheus Operator stack
  - Prometheus for metrics
  - Grafana with Keycloak SSO
  - Alertmanager with Teams notifications
  - ServiceMonitors and PodMonitors
- **loki-stack**: Centralized logging
  - Loki for log aggregation
  - Promtail for log collection

### **Security** (`apps/security/`)
- **kyverno**: Policy engine for Kubernetes
  - Pod Security Standards enforcement
  - Mutating/validating admission policies
- **kyverno-policies**: Pre-built policy library
- **kubescape**: Security posture scanning
  - CIS Benchmarks
  - NSA/CISA hardening guides
  - MITRE ATT&CK framework
- **tetragon**: eBPF-based runtime security
  - Process execution monitoring
  - Network policy enforcement

### **Ingress** (`apps/ingress/`)
- **ingress-nginx**: NGINX Ingress Controller
  - LoadBalancer integration with Azure
  - Prometheus metrics exporter
  - cert-manager integration

---

## 🔐 RBAC and Access Control

### AppProjects
Two AppProjects enforce different permission levels:

#### **platform** (Admin-level)
- Full cluster access
- Manages infrastructure components
- Can deploy to any namespace
- Used by: monitoring, security, ingress apps

#### **applications** (Developer-level)
- Restricted to `app-*` namespaces
- Limited resource types
- No cluster-scoped resources
- Used by: customer applications

### Keycloak Integration
All tools with web UIs use Keycloak for SSO:
- **ArgoCD**: OIDC authentication with role mapping
- **Grafana**: OAuth2 proxy with Keycloak
- **Prometheus**: (if UI exposed) via OAuth2 Proxy

---

## 🚀 Deployment Order

Critical dependencies must be respected:

```
1. Ingress NGINX (creates LoadBalancer, assigns IP)
   ↓
2. DNS records (created by Terraform, points to Ingress IP)
   ↓
3. cert-manager (already installed via Terraform)
   ↓
4. Keycloak (requires TLS certificate)
   ↓
5. Monitoring stack (Grafana SSO requires Keycloak)
   ↓
6. Security tools (policies applied after cluster is stable)
   ↓
7. Applications (customer workloads)
```

**ArgoCD Sync Waves** can be used to enforce this order (add `argocd.argoproj.io/sync-wave` annotations).

---

## 🔧 Configuration Management

### Secrets
**Never commit secrets to Git!**

Use one of:
1. **Sealed Secrets**: Encrypt secrets via `kubeseal`
2. **External Secrets Operator**: Sync from Azure Key Vault
3. **ArgoCD Vault Plugin**: Inject secrets at sync time

### Environment Variables
Use `environments/<env>/values.yaml` for:
- Domain names (e.g., `dev.example.com`)
- Storage sizes
- Retention periods
- Feature flags
- Resource limits

### Helm Values Override
Each Application can reference environment-specific values:

```yaml
source:
  helm:
    valueFiles:
      - ../../environments/dev/values.yaml
```

---

## 📊 Observability Stack

### Metrics (Prometheus + Grafana)
- **Prometheus**: Scrapes metrics from all components
- **Grafana**: Visualizes metrics with pre-built dashboards
- **Alertmanager**: Routes alerts to Microsoft Teams

### Logs (Loki + Promtail)
- **Promtail**: Collects logs from all pods
- **Loki**: Stores and indexes logs
- **Grafana**: Query logs via LogQL

### Security Scanning (Kubescape)
- Continuous compliance scanning
- CIS Kubernetes Benchmark
- NSA/CISA hardening checks
- MITRE ATT&CK detection

### Runtime Security (Tetragon)
- eBPF-based process monitoring
- Network flow observability
- Syscall tracing

---

## 🛠️ Maintenance and Operations

### Adding a New Application
1. Create manifest in `apps/<category>/<app-name>.yaml`
2. Add Application CR in `bootstrap/apps/<app-name>.yaml`
3. Commit and push
4. ArgoCD syncs automatically

### Updating an Application
1. Edit the manifest in `apps/<category>/`
2. Update Helm chart version or values
3. Commit and push
4. ArgoCD detects drift and syncs

### Rollback
```bash
# Via ArgoCD CLI
argocd app rollback <app-name> <revision>

# Or via Git
git revert <commit-sha>
git push
```

### Debugging
```bash
# Check ArgoCD app status
argocd app get <app-name>

# View sync history
argocd app history <app-name>

# Logs from ArgoCD
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller
```

---

## 🔗 Integration with IaC Repository

This repository is deployed **after** the `iac-client-cluster` Terraform completes:

```
iac-client-cluster (Terraform)
  ↓ creates AKS + ArgoCD
plattform-gitops (GitOps)
  ↓ deploys applications
```

**Terraform** provisions:
- AKS cluster
- Azure Key Vault
- DNS zones
- ArgoCD (initial Helm install)
- cert-manager
- Keycloak

**ArgoCD** (this repo) manages:
- Monitoring stack
- Security tools
- Ingress controllers
- Application deployments

---

## 📚 References

### DevOps Best Practices
- [GitOps Principles](https://devops.com/gitops/)
- [App of Apps Pattern](https://devops.com/argocd-app-of-apps-pattern/)
- [Zero Trust Security](https://devops.com/zero-trust-kubernetes/)

### Official Documentation
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Kyverno Policies](https://kyverno.io/policies/)
- [Prometheus Operator](https://prometheus-operator.dev/)
- [cert-manager](https://cert-manager.io/)

---

## ✅ Validation Checklist

Before deploying to production:

- [ ] All secrets managed via Azure Key Vault or Sealed Secrets
- [ ] Keycloak SSO configured for all UIs
- [ ] DNS records point to correct Ingress IP
- [ ] TLS certificates issued by Let's Encrypt
- [ ] Prometheus scraping all targets
- [ ] Grafana dashboards accessible via SSO
- [ ] Alertmanager sending test alerts to Teams
- [ ] Kyverno policies in enforce mode
- [ ] Kubescape scans passing (or documented exceptions)
- [ ] Tetragon policies active
- [ ] ArgoCD Projects correctly restrict access
- [ ] Backup strategy for Grafana dashboards and Prometheus data

---

## 🆘 Support and Troubleshooting

### Common Issues

**ArgoCD Out of Sync**
- Check for manual changes: `kubectl get <resource> -o yaml`
- Force sync: `argocd app sync <app-name> --force`

**Certificate Not Issuing**
- Verify DNS propagation: `nslookup <domain>`
- Check cert-manager logs: `kubectl logs -n cert-manager -l app=cert-manager`

**Grafana SSO Not Working**
- Verify Keycloak client secret in Grafana config
- Check Keycloak realm and client configuration

**Prometheus Not Scraping**
- Verify ServiceMonitor labels match Prometheus selector
- Check NetworkPolicies allow traffic

---

**Last Updated**: November 2025  
**Maintained By**: Cloud Platform Team
