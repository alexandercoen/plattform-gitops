

```markdown
# 🚀 Platform Delivery – Multi-Environment AKS, Terraform & GitOps Architektur

## 📌 Gesamtüberblick

Diese Plattform stellt eine vollständig **automatisierte**, **sichere** und **erweiterbare** Azure Kubernetes Service (AKS)-Umgebung bereit. Sie kombiniert:

- **Terraform** für Infrastructure-as-Code  
- **ArgoCD** für GitOps  
- **Keycloak** für SSO  
- **Let’s Encrypt** für TLS  
- **Prometheus, Grafana, Kyverno, Kubescape & Tetragon** für Monitoring & Security  

🎯 **Ziel:** Bereitstellung einer **standardisierten, mandantenfähigen** Cloud-Plattform für Endkunden – von Infrastruktur bis Applikationsbetrieb.

**Unterstützte Environments:**  
- `dev` – Development  
- `test` – Testing  
- `prod` – Production  

---

## 📁 Repository-Struktur

| Ebene                  | Repository           | Aufgabe                                                            | Tooling                    |
|------------------------|----------------------|---------------------------------------------------------------------|----------------------------|
| Infrastructure-as-Code | `iac-client-cluster` | Azure Ressourcen, AKS, Vault, DNS, ArgoCD, cert-manager, Keycloak  | Terraform + GitHub Actions |
| GitOps Plattform       | `plattform-gitops`   | Helm-basierte App-Deployments (Monitoring, Security, Environments) | ArgoCD + Helm + Keycloak   |

### plattform-gitops Directory Structure

```
plattform-gitops/
├── bootstrap/                      # ArgoCD App of Apps pattern
│   ├── root-app.yaml              # Root application
│   ├── appproject-platform.yaml   # Platform AppProject
│   ├── appproject-applications.yaml
│   └── apps/                      # Application manifests
│       ├── monitoring.yaml
│       ├── security.yaml
│       └── ingress.yaml
├── apps/                          # Application definitions
│   ├── monitoring/                # Prometheus, Grafana, Loki
│   ├── security/                  # Kyverno, Kubescape, Tetragon
│   ├── ingress/                   # NGINX Ingress Controller
│   └── applications/              # Customer applications
├── environments/                  # Environment-specific configs
│   ├── dev/
│   ├── test/
│   └── prod/
├── .github/workflows/             # CI/CD automation
├── STRUCTURE.md                   # Detailed structure docs
└── GETTING-STARTED.md            # Quick start guide
```

📖 **See [STRUCTURE.md](./STRUCTURE.md) for detailed architecture and patterns**

---

## 🏗️ Architekturübersicht

```

┌───────────────────────────────────────────────┐
│            iac-client-cluster (Terraform)     │
│───────────────────────────────────────────────│
│ • Azure RG, ACR, Vault, DNS                   │
│ • AKS mit Cilium CNI                          │
│ • ArgoCD, Keycloak, cert-manager, Ingress     │
└──────────────────────────────┬────────────────┘
▼
┌───────────────────────────────────────────────┐
│            plattform-gitops (ArgoCD)          │
│───────────────────────────────────────────────│
│ • Monitoring: Prometheus, Grafana, Loki       │
│ • Security: Kyverno, Kubescape, Tetragon      │
│ • cert-manager ClusterIssuer (Staging/Prod)   │
└──────────────────────────────┬────────────────┘
▼
┌───────────────────────────────────────────────┐
│                  AKS Cluster                  │
│───────────────────────────────────────────────│
│ • Cilium Networking, TLS, RBAC, Namespaces    │
└───────────────────────────────────────────────┘

```

---

## 🧱 Infrastruktur-Layer: `iac-client-cluster`

**Bereitgestellte Ressourcen:**

- Azure RG, DNS Zone, ACR, Vault, NSG  
- AKS (mit Cilium CNI)  
- NGINX Ingress Controller  
- cert-manager (Let's Encrypt)  
- Keycloak & ArgoCD  

---

## 🔁 Bereitstellungsreihenfolge (kritisch)

| Schritt | Beschreibung                                     | Voraussetzung      |
|--------:|--------------------------------------------------|--------------------|
|       1 | Core-Ressourcen (RG, ACR, Vault, DNS, NSG)       | –                  |
|       2 | AKS-Deployment mit Cilium CNI                    | Core vorhanden     |
|       3 | Ingress-Controller                               | AKS bereit         |
|       4 | Ingress-LoadBalancer IP abwarten                 | Ingress läuft      |
|       5 | DNS-Einträge für dev/test/prod + Tools erstellen | IP bekannt         |
|       6 | Installation cert-manager                        | DNS vorhanden      |
|       7 | TLS-Zertifikate (Let’s Encrypt)                  | cert-manager aktiv |
|       8 | ArgoCD & Keycloak (TLS aktiviert)                | Zertifikate gültig |

⚠️ **Achtung:** DNS & TLS erst nach Ingress-IP!

---

## 🔐 TLS & Domains (cert-manager + Let’s Encrypt)

**Ziel:** Automatische TLS-Zertifikate für alle Umgebungen & Subdomains.

| Environment | ArgoCD                       | Keycloak                       | Grafana                       | Prometheus                       | Alertmanager                       |
|-------------|------------------------------|--------------------------------|-------------------------------|----------------------------------|------------------------------------|
| dev         | `argocd.dev.client-box.com`  | `keycloak.dev.client-box.com`  | `grafana.dev.client-box.com`  | `prometheus.dev.client-box.com`  | `alertmanager.dev.client-box.com`  |
| test        | `argocd.test.client-box.com` | `keycloak.test.client-box.com` | `grafana.test.client-box.com` | `prometheus.test.client-box.com` | `alertmanager.test.client-box.com` |
| prod        | `argocd.prod.client-box.com` | `keycloak.prod.client-box.com` | `grafana.prod.client-box.com` | `prometheus.prod.client-box.com` | `alertmanager.prod.client-box.com` |

---

## ⚙️ GitOps Layer: `plattform-gitops`

**Aufgaben:**

- ArgoCD-Bootstrap & Helm-basierte App-Verwaltung  
- Automatische Environment-Trennung & Synchronisierung  

**Stacks:**

| Kategorie    | Komponenten                                                             |
|--------------|-------------------------------------------------------------------------|
| Monitoring   | Prometheus, Grafana, Loki, Alertmanager (mit Teams Webhook)             |
| Security     | Kyverno (Policies), Kubescape (Compliance), Tetragon (Runtime Security) |
| Automation   | cert-manager ClusterIssuer (Staging/Prod)                               |
| Environments | ArgoCD-Projekte & Namespaces für dev/test/prod                          |

---

## 📊 Monitoring Stack

- Vollautomatische TLS  
- Alerts über Microsoft Teams (via Alertmanager)  
- Keycloak-SSO für Grafana-Zugriff  

**Zugriffs-URLs:**

```

Grafana:      grafana.<env>.client-box.com
Prometheus:   prometheus.<env>.client-box.com
Alertmanager: alertmanager.<env>.client-box.com

```

---

## 🛡️ Security Stack

- **Kyverno**: Policy-Enforcement  
- **Kubescape**: CIS/NSA/MITRE-konforme Scans  
- **Tetragon**: Runtime-Sicherheit  

Alle über GitOps verwaltet. Ergebnisse in **Grafana** visualisiert.

---

## 🔍 Kubescape Integration – Security Observability

- Scanergebnisse als **Prometheus-Metriken**  
- Visualisierung in **dedizierten Dashboards**  
- Alertmanager meldet Findings an **Microsoft Teams**  

---

## 🧑‍💼 Identity & RBAC mit Keycloak

| Rolle | Beschreibung             | Zugriff                    |
|-------|--------------------------|----------------------------|
| admin | Plattform-Admins / SRE   | Alle Environments & Stacks |
| dev   | Kundenteams / Entwickler | Nur eigenes ArgoCD-Projekt |

- Zentrale SSO-Integration (Keycloak OIDC)  
- ArgoCD & Grafana nutzen OIDC-Tokens zur Rollenvergabe  

---

## 🔐 Sicherheit & Governance

- Secrets im Azure Key Vault  
- GitHub OIDC für Terraform (keine Secrets im Repo)  
- RBAC & Isolation über Namespaces und ArgoCD Projects  
- Policy & Compliance: Kyverno, Kubescape  

---

## 🌐 Zugriffspunkte

| Komponente   | URL Pattern                                |
|--------------|---------------------------------------------|
| ArgoCD       | `https://argocd.<env>.client-box.com`       |
| Keycloak     | `https://keycloak.<env>.client-box.com`     |
| Grafana      | `https://grafana.<env>.client-box.com`      |
| Prometheus   | `https://prometheus.<env>.client-box.com`   |
| Alertmanager | `https://alertmanager.<env>.client-box.com` |

Alle Endpunkte via **Let’s Encrypt** abgesichert.

---

## 🔄 Pipeline Flow

| Phase               | Tool      | Beschreibung                                |
|---------------------|-----------|---------------------------------------------|
| Infrastrukturaufbau | Terraform | Provisioniert AKS, Vault, DNS, cert-manager |
| Ingress-IP-Abfrage  | Terraform | Wartet auf LoadBalancer-IP                  |
| DNS & Zertifikate   | Terraform | Erstellt DNS-Records & TLS-Zertifikate      |
| ArgoCD Bootstrap    | Terraform | Verbindet ArgoCD mit GitOps-Repo            |
| GitOps Sync         | ArgoCD    | Deployt Monitoring, Security, Environments  |
| RBAC                | Keycloak  | Steuert den Zugriff über Rollen             |

---

## 🏁 Endzustand

✅ Mandantenfähige, mehrstufige AKS-Plattform  
✅ Automatisierte Bereitstellung (Infra + App)  
✅ Zertifikats-Management via Let’s Encrypt  
✅ Monitoring & Security integriert  
✅ GitOps-zentrierte Verwaltung (ArgoCD)  
✅ Keycloak-basiertes SSO & RBAC  

---

## 🧠 Vorteile

- 💡 **Vollständig automatisiert** – von Terraform bis ArgoCD  
- 🔒 **Security & Observability by Design**  
- 🌐 **Multi-Environment DNS & TLS out-of-the-box**  
- 👥 **Zentrale SSO- & RBAC-Lösung**  
- 📈 **Auditierbar, skalierbar, produktionsreif**  
