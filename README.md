# wedigitek-k8s-gitops

## Documentação principal

- Sustentação de produção: [docs/RUNBOOK-SUSTENTACAO-PRD.md](docs/RUNBOOK-SUSTENTACAO-PRD.md)
- Operação GitOps em produção: [GITOPS-OPERACAO-PRD.md](GITOPS-OPERACAO-PRD.md)
- Arquitetura geral: [docs/architecture.md](docs/architecture.md)
- Azure Key Vault: [KEYVAULT_SETUP_GUIDE.md](KEYVAULT_SETUP_GUIDE.md)
- ArgoCD: [infrastructure/argocd/README.md](infrastructure/argocd/README.md)

## Operações recentes (AKS)

### 1) Namespace MQTT
- Namespace criado: `mqtt-prd`

### 2) Sincronização Mongo produção -> AKS
- Destino: `we-api-prd/we-api-aks-mongo`
- Fonte: URI definida em secret `we-api-mongo-sync-source`
- Estratégia: `mongodump | mongorestore --drop` com `readPreference=secondaryPreferred`
- Frequência contínua: a cada 5 minutos em `applications/prd/we-api/mongodb-sync-cronjob.yaml`
- Ajuste de conectividade: jobs de sync usam label `app=we-api-mongo-sync` para compatibilidade com `NetworkPolicy` do Mongo

### 3) Tags de imagem (prd)
- Alteradas de `:rc` para `:main` nos manifests de deployment em `applications/prd/**/deployment.yaml`
- Mudança persistente via GitOps (ArgoCD reconcilia a partir deste repositório)

## Validação operacional
- HPA e métricas do cluster validadas antes das mudanças
- Mongo destino saudável e com sincronização contínua ativa
- Mudanças aplicadas sem alteração de código nos repositórios de aplicação