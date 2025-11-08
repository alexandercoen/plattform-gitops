# GitHub Actions Configuration

This repository uses GitHub Actions for automated validation and deployment.

## Required Secrets

Configure these secrets in GitHub repository settings (`Settings > Secrets and variables > Actions`):

### ArgoCD Integration
- `ARGOCD_SERVER`: ArgoCD server URL (e.g., `argocd.dev.client-box.com`) done
- `ARGOCD_AUTH_TOKEN`: ArgoCD authentication token

#### How to Get ArgoCD Token
```bash
# Login to ArgoCD
argocd login <argocd-server> --username admin

# Generate token (never expires)
argocd account generate-token --account admin
```

### Azure Integration (Optional)
If using Azure Key Vault for secrets management:
- `AZURE_CLIENT_ID`: Service Principal Client ID
- `AZURE_CLIENT_SECRET`: Service Principal Secret
- `AZURE_TENANT_ID`: Azure Tenant ID
- `AZURE_SUBSCRIPTION_ID`: Azure Subscription ID

#### How to Create Service Principal
```bash
# Create SP with minimal permissions
az ad sp create-for-rbac --name "github-actions-gitops" \
  --role Reader \
  --scopes /subscriptions/<subscription-id>/resourceGroups/<rg-name>
```

### Microsoft Teams (Optional)
For deployment notifications:
- `TEAMS_WEBHOOK_URL`: Microsoft Teams incoming webhook URL   done

#### How to Get Teams Webhook (Updated for 2025)

**Method 1: Workflows (Recommended)**
1. Open Teams channel
2. Click `...` (More options) > `Workflows`
3. Search for "Post to a channel when a webhook request is received"
4. Click **Add workflow**
5. Select the team and channel
6. Copy the webhook URL provided
7. Click **Add workflow**

**Method 2: Power Automate (Enterprise)**
1. Go to https://make.powerautomate.com
2. Create new flow: **Automated cloud flow**
3. Select trigger: "When a HTTP request is received"
4. Add action: "Post message in a chat or channel" (Teams)
5. Configure channel and message format
6. Save and copy the HTTP POST URL

**Note**: The legacy "Incoming Webhook" connector is deprecated. Use Workflows for new integrations.

---

## Workflows

### `validate.yaml`
**Trigger**: Pull requests and pushes to `main`

**Purpose**: 
- Validate YAML syntax
- Check for secrets in repository
- Validate ArgoCD Application manifests
- Run security scans with Trivy

**Status**: ✅ Required to pass before merge

### `sync.yaml`
**Trigger**: Pushes to `main` or manual workflow dispatch

**Purpose**:
- Login to ArgoCD
- Trigger sync of root-app
- Wait for health checks
- Notify success/failure

**Status**: 🔄 Automatic on merge to `main`

---

## Workflow Configuration

### Enabling Auto-Sync
To enable automatic ArgoCD sync on merge:

1. Add secrets to repository (see above)
2. Ensure ArgoCD server is accessible from GitHub Actions runners
3. Merge to `main` branch triggers sync

### Disabling Auto-Sync
To require manual deployment:

1. Remove `sync.yaml` workflow or disable it
2. Use `argocd app sync` command manually after merge

---

## Environments and Branches

### Branch Strategy
- `main`: Production-ready code
- `develop`: Integration branch (optional)
- `feature/*`: Feature branches

### Environment Mapping
- `main` branch → Prod environment (manual approval)
- `develop` branch → Test environment (auto-sync)
- Feature branches → Dev environment (auto-sync)

---

## Security Best Practices

### Secrets Management
- ✅ Store secrets in GitHub encrypted secrets
- ✅ Use short-lived tokens where possible
- ✅ Rotate tokens regularly (quarterly)
- ❌ Never commit secrets to repository
- ❌ Never echo secrets in workflow logs

### Access Control
- Limit who can approve workflow runs
- Use environment protection rules
- Require PR reviews before merge
- Enable branch protection on `main`

### Audit Trail
- All deployments logged in GitHub Actions
- ArgoCD sync history available
- Git history provides full audit trail

---

## Troubleshooting Workflows

### Workflow Fails: "Invalid token"
```bash
# Regenerate ArgoCD token
argocd account generate-token --account admin

# Update GitHub secret
gh secret set ARGOCD_AUTH_TOKEN --body "<new-token>"
```

### Workflow Fails: "Cannot connect to ArgoCD server"
- Check ArgoCD server is accessible from internet
- Verify firewall/NSG rules allow HTTPS
- Check DNS resolution

### Workflow Fails: "Application not found"
- Verify ArgoCD application exists: `argocd app get root-app`
- Check application name in workflow matches actual name

---

## Manual Workflow Execution

### Via GitHub UI
1. Go to `Actions` tab
2. Select workflow (e.g., `Sync to ArgoCD`)
3. Click `Run workflow`
4. Select branch
5. Click `Run workflow`

### Via GitHub CLI
```bash
# Trigger sync workflow
gh workflow run sync.yaml --ref main

# Check workflow status
gh run list --workflow=sync.yaml
```

---

## Monitoring Workflows

### GitHub Actions Dashboard
- View all workflow runs in `Actions` tab
- Filter by status, branch, or workflow
- Download logs for debugging

### Notifications
Configure GitHub to send notifications on workflow failures:
1. Go to `Settings > Notifications`
2. Enable `Actions` notifications
3. Choose delivery method (email, mobile)

### Integrate with Microsoft Teams
Add Teams notification to workflows:
```yaml
- name: Notify Teams on Failure
  if: failure()
  run: |
    curl -H 'Content-Type: application/json' \
      -d '{"text": "GitOps sync failed!"}' \
      ${{ secrets.TEAMS_WEBHOOK_URL }}
```

---

## Cost Optimization

### GitHub Actions Minutes
- Public repos: Unlimited (free)
- Private repos: 2000 min/month (free tier)

### Reduce Usage
- Cache dependencies
- Avoid unnecessary workflow runs
- Use `paths` filter to trigger only on relevant changes:
```yaml
on:
  push:
    branches: [main]
    paths:
      - 'apps/**'
      - 'bootstrap/**'
      - 'environments/**'
```

---

**Need Help?**
- GitHub Actions docs: https://docs.github.com/actions
- ArgoCD docs: https://argo-cd.readthedocs.io/
- Open an issue in this repository
