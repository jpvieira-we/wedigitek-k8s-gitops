# 🚀 ArgoCD Installation Quick Start

## ✅ Validações Realizadas

- ✅ Sintaxe YAML válida
- ✅ Estrutura de manifests correta
- ✅ Helm values configurado
- ✅ RBAC policies definidas
- ✅ Resources limits configurados

---

## 📦 Instalação Bootstrap (Executar apenas UMA VEZ)

### Opção 1: Via Helm (Recomendado)

```bash
# 1. Adicionar repositório Helm
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# 2. Instalar ArgoCD usando o values.yaml deste repo
helm install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --version 7.7.12 \
  -f infrastructure/argocd/values.yaml

# 3. Aguardar pods ficarem ready
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/name=argocd-server \
  -n argocd --timeout=300s

# 4. Obter senha admin
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo

# 5. Port-forward para acessar (desenvolvimento)
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Acessar: https://localhost:8080
# User: admin / Pass: (senha do passo 4)
```

### Opção 2: Via kubectl (Alternativa)

```bash
# Download do manifesto oficial
curl -o /tmp/argocd-install.yaml \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Aplicar
kubectl create namespace argocd
kubectl apply -n argocd -f /tmp/argocd-install.yaml

# Aguardar e seguir passos 3-5 acima
```

---

## 🔄 Migrar para GitOps Self-Management

Após o bootstrap, ArgoCD deve gerenciar a si mesmo:

```bash
# Aplicar o manifest de self-management
kubectl apply -f infrastructure/argocd/argocd-self-management.yaml

# Verificar Application
kubectl get application -n argocd argocd

# A partir daqui, qualquer mudança no Git será aplicada automaticamente!
```

---

## 🔧 Instalação da CLI

```bash
# Linux
curl -sSL -o /tmp/argocd \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 /tmp/argocd /usr/local/bin/argocd
rm /tmp/argocd

# Login
argocd login localhost:8080
# ou
argocd login argocd.wedigitek.com
```

---

## 📊 Verificação

```bash
# Ver pods do ArgoCD
kubectl get pods -n argocd

# Ver Applications
kubectl get applications -n argocd

# Ver logs do server
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-server -f
```

---

## 🌐 Próximos Passos

1. ✅ ArgoCD instalado
2. ⏳ Instalar Ingress Controller (nginx)
3. ⏳ Instalar Cert-Manager (SSL/TLS)
4. ⏳ Configurar DNS (argocd.wedigitek.com)
5. ⏳ Deploy Root Application (App-of-Apps)
6. ⏳ Migrar repos GitLab → GitHub
7. ⏳ Deploy primeira aplicação

---

## ⚠️ IMPORTANTE

- **Ingress está DESABILITADO** no values.yaml inicial
- Após instalar Ingress Controller e Cert-Manager, habilitar via self-management
- GitLab permanece intacto (não será deletado)
- Todas as mudanças no Git após self-management serão aplicadas automaticamente

---

**Validação realizada em**: 2026-02-09
