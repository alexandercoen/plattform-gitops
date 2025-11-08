# ArgoCD Bootstrap

This directory contains the **App of Apps** pattern configuration for ArgoCD.

## Files

### `root-app.yaml`
The root ArgoCD Application that bootstraps all other applications. Apply this first:
```bash
kubectl apply -f bootstrap/root-app.yaml
```

### `appproject-*.yaml`
ArgoCD AppProjects that define RBAC boundaries:
- **platform**: Admin-level access for infrastructure components
- **applications**: Developer-level access for workloads

### `apps/`
Individual Application CRs that ArgoCD deploys from the `root-app`:
- `monitoring.yaml`: Deploys monitoring stack
- `security.yaml`: Deploys security tools
- `ingress.yaml`: Deploys ingress controller

## Workflow
1. Terraform creates AKS cluster and installs ArgoCD
2. Apply AppProjects: `kubectl apply -f bootstrap/appproject-*.yaml`
3. Apply root app: `kubectl apply -f bootstrap/root-app.yaml`
4. ArgoCD reads `bootstrap/apps/*.yaml` and deploys each application
5. Each application syncs from its respective directory in `apps/`

## Sync Waves
Use annotations to control deployment order:
```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "1"  # Deploy first
```

Lower numbers deploy first.
