# 🚀 Platform GitOps Quick Start

## Prerequisites
- AKS cluster deployed via `iac-client-cluster` Terraform
- ArgoCD installed in cluster
- `kubectl` configured with cluster access
- ArgoCD CLI installed (optional)

## Initial Setup

### 1. Bootstrap ArgoCD
```bash
# Apply AppProjects
kubectl apply -f bootstrap/appproject-platform.yaml
kubectl apply -f bootstrap/appproject-applications.yaml

# Deploy root application (App of Apps)
kubectl apply -f bootstrap/root-app.yaml
```

### 2. Verify Deployment
```bash
# Check ArgoCD applications
kubectl get applications -n argocd

# Check application status
argocd app list

# Watch sync progress
argocd app sync root-app
```

### 3. Access Services
All services are accessible via Ingress with TLS:

- **ArgoCD**: https://argocd.<your-domain>
- **Grafana**: https://grafana.<your-domain>
- **Keycloak**: https://keycloak.<your-domain>

## Environment-Specific Deployment

### Development
```bash
# Update domain in environments/dev/values.yaml
# Commit and push - ArgoCD auto-syncs
```

### Production
```bash
# Update environments/prod/values.yaml
# Commit and push
# Manual sync required in prod (unless configured otherwise)
argocd app sync <app-name>
```

## Adding a New Application

1. Create application manifest:
```yaml
# apps/applications/my-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: applications
  source:
    repoURL: https://github.com/your-org/your-repo.git
    targetRevision: main
    path: manifests/
  destination:
    server: https://kubernetes.default.svc
    namespace: app-myapp
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

2. Register in App of Apps:
```yaml
# bootstrap/apps/my-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: applications
  source:
    repoURL: https://github.com/alexandercoen/plattform-gitops.git
    targetRevision: main
    path: apps/applications
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
```

3. Commit and push - ArgoCD deploys automatically.

## Common Operations

### Sync Application
```bash
argocd app sync <app-name>
```

### Rollback Application
```bash
argocd app rollback <app-name> <revision>
```

### View Application Details
```bash
argocd app get <app-name>
```

### Access Grafana Dashboards
1. Navigate to https://grafana.<your-domain>
2. Login via Keycloak SSO
3. View pre-configured dashboards for:
   - Kubernetes cluster overview
   - Node metrics
   - Pod metrics
   - Ingress traffic

### Check Security Compliance
```bash
# View Kubescape scan results
kubectl get configauditreports -A

# View Kyverno policy violations
kubectl get policyreports -A
```

## Troubleshooting

### ArgoCD Not Syncing
```bash
# Check application status
argocd app get <app-name>

# View sync errors
kubectl describe application <app-name> -n argocd
```

### Certificate Issues
```bash
# Check certificate status
kubectl get certificate -A

# Check cert-manager logs
kubectl logs -n cert-manager -l app=cert-manager
```

### Grafana SSO Not Working
```bash
# Check Keycloak client configuration
# Verify client secret in Grafana values
# Check Grafana logs
kubectl logs -n monitoring -l app.kubernetes.io/name=grafana
```

## Next Steps
- Review `STRUCTURE.md` for detailed architecture
- Configure environment-specific values in `environments/`
- Set up GitHub Actions for automated validation
- Configure backup strategies for persistent data

## Support
For issues or questions, refer to:
- `STRUCTURE.md` - Repository structure and patterns
- `README.MD` - Overall platform architecture
- ArgoCD documentation: https://argo-cd.readthedocs.io/
