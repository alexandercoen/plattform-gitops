# Example Application Deployment

This is a sample application to demonstrate how to deploy workloads via ArgoCD.

## Structure
```
applications/
├── sample-app/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
```

## Deployment
1. Add your application manifests in this directory
2. Create an ArgoCD Application CR in `bootstrap/apps/`
3. Commit and push
4. ArgoCD automatically syncs the application

## RBAC
Applications in this directory are governed by the `applications` AppProject, which:
- Restricts deployment to `app-*` namespaces
- Limits resource types (no ClusterRoles, CRDs, etc.)
- Enforces developer-level access control

## Best Practices
- Use Helm charts for complex applications
- Store secrets in Azure Key Vault
- Use proper resource requests/limits
- Enable health checks (liveness/readiness probes)
- Add Prometheus ServiceMonitor for metrics
