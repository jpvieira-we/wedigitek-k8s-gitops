# ArgoCD Installation

## 📥 Download ArgoCD Manifests

```bash
# Download latest stable version
curl -o argocd/install/install.yaml \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

## 🚀 Install ArgoCD

```bash
# Create namespace
kubectl create namespace argocd

# Apply installation
kubectl apply -n argocd -f argocd/install/install.yaml

# Wait for pods to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=300s
```

## 🔐 Get Initial Admin Password

```bash
# Get password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo

# Username: admin
# Password: (output do comando acima)
```

## 🌐 Access ArgoCD UI

### Option 1: Port Forward
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Access: https://localhost:8080
```

### Option 2: Ingress (Production)
```bash
kubectl apply -f argocd/install/ingress.yaml
# Access: https://argocd.wedigitek.com
```

## 🔧 Install ArgoCD CLI

```bash
# Linux
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64

# Login
argocd login localhost:8080
```

## 📋 Configure GitHub Repository

```bash
# Add repository
argocd repo add https://github.com/wedigitek/wedigitek-k8s-gitops.git \
  --username <github-username> \
  --password <github-token>
```

## 🎯 Deploy Root Application

```bash
kubectl apply -f argocd/apps/root-app.yaml
```

## ✅ Verify Installation

```bash
# Check all pods
kubectl get pods -n argocd

# Check ArgoCD version
argocd version

# List applications
argocd app list
```
