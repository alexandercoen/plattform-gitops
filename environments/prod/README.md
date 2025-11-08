# Environment: Production

## Configuration
- **Domain**: example.com
- **Cluster**: aks-prod
- **Resource Group**: rg-platform-prod
- **Keycloak**: https://keycloak.example.com
- **ArgoCD**: https://argocd.example.com
- **Grafana**: https://grafana.example.com

## Features
- Auto-sync: manual approval required
- Self-healing enabled
- Certificate issuer: letsencrypt-prod
- Monitoring retention: 30 days
- Log level: warn
- High availability: 3+ replicas
