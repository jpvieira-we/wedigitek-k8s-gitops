# ArgoCD Bootstrap - GitOps Installation

## 🚀 Instalação Bootstrap (Primeira vez)

O ArgoCD precisa ser instalado manualmente pela primeira vez para depois gerenciar a si mesmo via GitOps.

### Opção 1: Bootstrap via Helm (Recomendado)

```bash
# Adicionar repo Helm do ArgoCD
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Instalar ArgoCD
helm install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --version 7.7.12 \
  -f infrastructure/argocd/values.yaml

# Aguardar pods
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/name=argocd-server \
  -n argocd --timeout=300s
```

### Opção 2: Bootstrap via kubectl (Alternativo)

```bash
# Download manifesto oficial
curl -o /tmp/argocd-install.yaml \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Aplicar
kubectl create namespace argocd
kubectl apply -n argocd -f /tmp/argocd-install.yaml

# Aguardar
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/name=argocd-server \
  -n argocd --timeout=300s
```

---

## 🔐 Acesso Inicial

### 1. Obter senha admin

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

### 2. Port Forward (Desenvolvimento)

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Acessar: https://localhost:8080
# User: admin
# Pass: (senha obtida no passo 1)
```

### 3. Instalar CLI do ArgoCD

```bash
# Linux
curl -sSL -o /tmp/argocd \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 /tmp/argocd /usr/local/bin/argocd
rm /tmp/argocd

# Login CLI
argocd login localhost:8080
```

---

## 🔄 Migrar para GitOps Self-Management

Após o bootstrap manual, o ArgoCD deve gerenciar a si mesmo via GitOps:

```bash
# Aplicar a Application do ArgoCD que aponta para este repo
kubectl apply -f infrastructure/argocd/argocd-application.yaml

# A partir desse momento, mudanças no Git serão aplicadas automaticamente
```

**Importante**: Após isso, o ArgoCD gerencia suas próprias configurações via Git!

---

## 📦 Deploy de Infraestrutura Base

### Root Application (App-of-Apps)

```bash
# Deploy da aplicação raiz que gerencia todas as outras
kubectl apply -f argocd/apps/root-app-prd.yaml
```

Isso irá criar automaticamente todas as aplicações de infraestrutura:
- Ingress NGINX
- Cert-Manager
- Sealed Secrets
- Monitoring Stack (Prometheus/Grafana/Loki)

---

## 🌐 Configurar Ingress (Produção)

### 1. Criar DNS Record
Apontar `argocd.wedigitek.com` para o IP do Load Balancer:

```bash
# Obter IP do Ingress Controller
kubectl get svc -n ingress-nginx ingress-nginx-controller \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

### 2. Aguardar Certificado SSL
```bash
# Verificar certificado
kubectl get certificate -n argocd argocd-tls
kubectl describe certificate -n argocd argocd-tls
```

### 3. Acessar via HTTPS
```
https://argocd.wedigitek.com
```

---

## 🔧 Configurações Importantes

### Adicionar Repositório GitHub Privado

```bash
# Via CLI
argocd repo add https://github.com/wedigitek/<repo>.git \
  --username <github-username> \
  --password <github-token>

# Via UI
# Settings → Repositories → Connect Repo
```

### Configurar SSO (GitHub OAuth)

1. Criar OAuth App no GitHub: https://github.com/organizations/wedigitek/settings/applications
2. Atualizar `infrastructure/argocd/values.yaml` com client ID/secret
3. Commit e push (ArgoCD aplicará automaticamente)

### Configurar Notificações Slack

1. Criar Slack App e obter webhook URL
2. Criar Secret com token:
```bash
kubectl create secret generic argocd-notifications-secret \
  -n argocd \
  --from-literal=slack-token=<webhook-url>
```
3. Descomentar seção de notifications no values.yaml

---

## 📊 Monitoramento

### Metrics via Prometheus

ArgoCD expõe métricas no formato Prometheus:
- `argocd-metrics:8082/metrics` - Application Controller
- `argocd-server-metrics:8083/metrics` - API Server
- `argocd-repo-server:8084/metrics` - Repo Server

ServiceMonitors são criados automaticamente se o Prometheus Operator estiver instalado.

### Logs

```bash
# Server
kubectl logs -n argocd deployment/argocd-server -f

# Application Controller
kubectl logs -n argocd deployment/argocd-application-controller -f

# Repo Server
kubectl logs -n argocd deployment/argocd-repo-server -f
```

---

## 🔄 Sync Policies

### Automated Sync (Recomendado para infra)
```yaml
syncPolicy:
  automated:
    prune: true      # Remove recursos deletados do Git
    selfHeal: true   # Corrige drift automaticamente
```

### Manual Sync (Recomendado para apps críticas)
```yaml
syncPolicy:
  automated: null  # Desabilita sync automático
```

---

## 🆘 Troubleshooting

### ArgoCD não sincroniza
```bash
# Forçar sync
argocd app sync <app-name> --force

# Ver diferenças
argocd app diff <app-name>

# Ver logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller
```

### Reset senha admin
```bash
kubectl -n argocd patch secret argocd-secret \
  -p '{"stringData": {"admin.password": "'$(htpasswd -bnBC 10 "" newpassword | tr -d ':\n')'"}}'
```

### Remover finalizer travado
```bash
kubectl patch app <app-name> -n argocd \
  -p '{"metadata": {"finalizers": null}}' \
  --type merge
```

---

## 📚 Referências

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Argo Helm Charts](https://github.com/argoproj/argo-helm)
- [Best Practices](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/)
