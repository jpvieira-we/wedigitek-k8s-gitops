# 🏗️ Arquitetura GitOps - WE:DIGITEK

## 📚 Estratégia de Repositórios

### Separação de Responsabilidades

```
┌─────────────────────────────────────────────────────────────┐
│                    WE:DIGITEK Infrastructure                 │
└─────────────────────────────────────────────────────────────┘

┌──────────────────────────────┐  ┌──────────────────────────────┐
│   terraformServers (Repo 1)  │  │ wedigitek-k8s-gitops (Repo 2)│
│                              │  │                              │
│  • Azure Infrastructure      │  │  • Kubernetes Manifests      │
│  • AKS Cluster               │  │  • Application Configs       │
│  • ACR Registry              │  │  • ArgoCD Apps               │
│  • Networking                │  │  • Helm Charts               │
│  • Storage/Key Vault         │  │  • GitOps Workflows          │
│  • Terraform State           │  │                              │
│                              │  │  No Terraform State          │
│  GitHub Actions (IaC)        │  │  ArgoCD (GitOps)             │
└──────────────────────────────┘  └──────────────────────────────┘
           │                                    │
           │                                    │
           ▼                                    ▼
    ┌──────────────┐                  ┌─────────────────┐
    │ Azure Cloud  │◄─────────────────│  AKS Cluster    │
    │              │                  │                 │
    │ • VMs        │                  │ • Pods          │
    │ • Databases  │                  │ • Services      │
    │ • Networks   │                  │ • Ingresses     │
    └──────────────┘                  └─────────────────┘
```

## 🎯 Repositório 1: terraformServers

### Propósito
Gerenciar **infraestrutura Azure** usando Infrastructure as Code (Terraform).

### Estrutura
```
terraformServers/
├── modules/
│   └── aks/              # Módulo AKS reutilizável
├── environments/
│   ├── prd/              # Produção
│   ├── hom/              # Homologação (futuro)
│   └── dev/              # Desenvolvimento (futuro)
├── .github/workflows/
│   ├── terraform-plan.yml
│   └── terraform-apply.yml
└── README.md
```

### Recursos Gerenciados
- ✅ **AKS Cluster** (Kubernetes)
- ✅ **Azure Container Registry** (ACR)
- ✅ **Resource Groups**
- ✅ **Virtual Networks** (VNet)
- ✅ **Subnets**
- ✅ **Log Analytics Workspace**
- ✅ **Managed Identities**
- 🔜 **Key Vault** (secrets)
- 🔜 **Azure Database** (se necessário)

### Pipeline CI/CD
```
PR criado → terraform plan → review → merge → terraform apply
```

### State Management
- **Backend**: Azure Storage Account
- **Container**: `tfstate`
- **Lock**: Azure Blob Storage lease
- **Isolation**: Por ambiente (prd.tfstate, hom.tfstate, dev.tfstate)

### Permissões
- **Owner**: DevOps/SRE Team
- **Contributors**: Infrastructure Team
- **Read-only**: Developers

---

## 🎯 Repositório 2: wedigitek-k8s-gitops

### Propósito
Gerenciar **aplicações Kubernetes** usando GitOps (ArgoCD).

### Estrutura
```
wedigitek-k8s-gitops/
├── argocd/
│   ├── install/          # ArgoCD installation
│   └── apps/             # App-of-Apps
│       ├── dev/
│       ├── hom/
│       └── prd/
├── infrastructure/       # Infra K8s (não Azure)
│   ├── namespaces/
│   ├── ingress-nginx/
│   ├── cert-manager/
│   ├── sealed-secrets/
│   └── monitoring/
├── applications/         # Business apps
│   ├── dev/
│   ├── hom/
│   └── prd/
└── .github/workflows/
    ├── validate-manifests.yml
    └── sync-argocd.yml
```

### Recursos Gerenciados
- ✅ **Kubernetes Deployments**
- ✅ **Services**
- ✅ **Ingresses**
- ✅ **ConfigMaps**
- ✅ **Sealed Secrets**
- ✅ **HPA (Horizontal Pod Autoscaler)**
- ✅ **Ingress Controller** (NGINX)
- ✅ **Cert Manager** (Let's Encrypt)
- ✅ **Monitoring Stack** (Prometheus/Grafana/Loki)

### Pipeline GitOps
```
PR criado → validate manifests → merge → ArgoCD auto-sync → deploy
```

### State Management
- **Não usa Terraform state**
- **Cluster Kubernetes = Source of Truth**
- **ArgoCD monitora Git → Cluster**

### Permissões
- **Owner**: DevOps/SRE Team
- **Contributors**: All Developers
- **Read-only**: QA Team

---

## 🔄 Fluxo de Deploy Completo

### 1️⃣ Provisionar Infraestrutura (uma vez ou raramente)
```bash
# Repositório: terraformServers
cd ~/terraformServers
git checkout -b feat/add-new-node-pool
# Editar terraform
git commit -am "feat: add new node pool"
git push
# Criar PR → Review → Merge
# GitHub Actions executa terraform apply
```

### 2️⃣ Deploy de Aplicação (frequente)
```bash
# Repositório: wedigitek-k8s-gitops
cd ~/wedigitek-k8s-gitops
git checkout -b feature/add-api-users
# Criar manifests Kubernetes
mkdir -p applications/prd/api-users
# Adicionar deployment.yaml, service.yaml, etc
git commit -am "feat: add api-users application"
git push
# Criar PR → Review → Merge
# ArgoCD detecta mudança e faz sync automático
```

### 3️⃣ Atualizar Versão de Aplicação (CI/CD completo)
```bash
# Repositório da aplicação (ex: api-users-app)
git tag v1.2.3
git push --tags
# GitHub Actions:
#   1. Build Docker image
#   2. Push para ACR
#   3. Atualiza image tag em wedigitek-k8s-gitops
#   4. ArgoCD detecta e faz deploy automático
```

---

## 🔐 Segurança

### Repositório terraformServers
- ✅ Branch protection em `main`
- ✅ Require PR reviews (2 approvers)
- ✅ Terraform state criptografado
- ✅ Secrets via Azure Key Vault
- ✅ OIDC authentication (GitHub → Azure)

### Repositório wedigitek-k8s-gitops
- ✅ Branch protection em `main` e `develop`
- ✅ Require PR reviews (1 approver)
- ✅ Sealed Secrets (secrets criptografados no Git)
- ✅ ArgoCD RBAC
- ✅ Kubernetes RBAC por namespace

---

## 📊 Ambientes

### DEV (Development)
- **Branch**: `feature/*`
- **Namespace**: `*-dev`
- **Sync**: Manual ou automático
- **Infra**: Shared AKS cluster

### HOM (Homologação/Staging)
- **Branch**: `develop`
- **Namespace**: `*-hom`
- **Sync**: Automático
- **Infra**: Shared AKS cluster

### PRD (Produção)
- **Branch**: `main`
- **Namespace**: `*-prd`
- **Sync**: Automático com approval
- **Infra**: AKS cluster dedicado (ou separação por namespace)

---

## 🎓 Boas Práticas

### ✅ DO
- Separar infra (Terraform) de apps (GitOps)
- Usar ArgoCD para deploy de aplicações
- Versionar tudo no Git
- Automatizar com CI/CD
- Usar Sealed Secrets para dados sensíveis
- Implementar RBAC granular

### ❌ DON'T
- Misturar Terraform de infra com manifests K8s
- Fazer deploy manual via `kubectl apply`
- Commitar secrets em plain text
- Dar acesso admin a todos
- Usar branch `main` para desenvolvimento

---

## 🚀 Próximos Passos

1. ✅ Finalizar estrutura `terraformServers`
2. ✅ Provisionar AKS via PR merge
3. ⏳ Criar estrutura completa `wedigitek-k8s-gitops`
4. ⏳ Instalar ArgoCD no cluster
5. ⏳ Configurar App-of-Apps pattern
6. ⏳ Migrar primeira aplicação (MongoDB → Atlas)
7. ⏳ Migrar aplicações do GitLab para GitHub
8. ⏳ Deploy aplicações no AKS
9. ⏳ Validação e cutover

---

**Cliente**: WE:DIGITEK  
**Consultoria**: 4EVER CONSULTING  
**Data**: 2026-02-07

**Co-Authored-By: Warp <agent@warp.dev>**
