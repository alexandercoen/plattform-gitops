# Contributing to plattform-gitops

## Development Workflow

### Making Changes
1. Create a feature branch from `main`
2. Make your changes
3. Test locally using ArgoCD's dry-run feature
4. Create a Pull Request

### Branch Naming
- `feature/` - New features
- `fix/` - Bug fixes
- `docs/` - Documentation updates
- `security/` - Security improvements

### Testing Changes

#### Validate YAML Syntax
```bash
# Validate all YAML files
find . -name "*.yaml" -o -name "*.yml" | xargs yamllint
```

#### Dry-run ArgoCD Application
```bash
# Test application without applying
argocd app create test-app --dry-run -f apps/monitoring/kube-prometheus-stack.yaml
```

#### Local Validation with kubectl
```bash
# Validate Kubernetes manifests
kubectl apply --dry-run=client -f apps/monitoring/
```

### Code Review Checklist
- [ ] No secrets committed to repository
- [ ] YAML files are properly formatted
- [ ] ArgoCD Applications use correct AppProject
- [ ] Environment-specific values are in `environments/` directory
- [ ] Documentation updated if needed
- [ ] GitHub Actions validation passes

### Security Guidelines
- **Never** commit secrets, passwords, or tokens
- Use Azure Key Vault or Sealed Secrets for sensitive data
- Reference secrets via External Secrets Operator
- Use Kyverno policies to prevent insecure configurations

### Deployment Process

#### Development
- Auto-sync enabled
- Changes deploy automatically on merge to `main`

#### Test
- Auto-sync enabled
- Manual approval recommended for breaking changes

#### Production
- **Manual sync only** (no auto-sync)
- Requires explicit approval
- Use GitHub tags for releases

### Rollback Procedure
```bash
# View application history
argocd app history <app-name>

# Rollback to previous version
argocd app rollback <app-name> <revision-number>
```

### Getting Help
- Review `STRUCTURE.md` for architecture details
- Check ArgoCD documentation: https://argo-cd.readthedocs.io/
- Open an issue for questions or problems

## Style Guide

### YAML Formatting
- Use 2 spaces for indentation
- No tabs
- Line length max 120 characters
- Add comments for complex configurations

### Naming Conventions
- Applications: `kebab-case` (e.g., `kube-prometheus-stack`)
- Namespaces: `kebab-case` (e.g., `monitoring`, `security`)
- Files: `kebab-case.yaml` (e.g., `kube-prometheus-stack.yaml`)

### Application Metadata
Always include:
```yaml
metadata:
  name: app-name
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
```

### Sync Policy
Platform applications (admin-level):
```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
  syncOptions:
    - CreateNamespace=true
```

Customer applications (dev-level):
```yaml
syncPolicy:
  automated:
    prune: false  # Prevent accidental deletion
    selfHeal: true
```

## Monitoring Changes

### View Sync Status
```bash
# Check all applications
argocd app list

# Get detailed status
argocd app get <app-name>
```

### View Logs
```bash
# ArgoCD logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller

# Application logs
kubectl logs -n <namespace> -l app=<app-name>
```

### Prometheus Metrics
ArgoCD exposes metrics at:
- Application sync status
- Sync duration
- Health status

Access via Grafana dashboard: **ArgoCD Application Metrics**

## Best Practices

### GitOps Principles
1. **Declarative**: Define desired state in Git
2. **Versioned**: All changes tracked in Git history
3. **Automated**: ArgoCD automatically syncs state
4. **Auditable**: Git log provides complete audit trail

### Security
- Use RBAC via AppProjects
- Enforce Pod Security Standards via Kyverno
- Scan for vulnerabilities with Kubescape
- Monitor runtime behavior with Tetragon

### Observability
- Add ServiceMonitors for Prometheus scraping
- Include proper labels for log aggregation
- Configure health checks (liveness/readiness)
- Set resource requests and limits

---

**Thank you for contributing to the platform!**
