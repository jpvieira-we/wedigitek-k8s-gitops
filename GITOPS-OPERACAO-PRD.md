# Operação GitOps em Produção - WE:DIGITEK

## Objetivo

Este documento explica o fluxo operacional de produção da WE:DIGITEK com:

- repositórios de aplicação;
- repositório GitOps;
- ArgoCD;
- cluster AKS;
- Azure Key Vault via CSI Driver.

Também descreve como uma mudança chega em produção, como validar, como atualizar serviços já existentes e como resolver os principais problemas encontrados no dia a dia.

---

## Visão geral da arquitetura

O fluxo de produção segue esta ordem:

1. um time altera código em um repositório de aplicação;
2. após merge para a branch principal da aplicação, é gerada/publicada uma nova imagem;
3. o repositório GitOps é atualizado para apontar para a nova versão ou novo manifesto;
4. o ArgoCD monitora o repositório GitOps;
5. o ArgoCD reconcilia o estado desejado no cluster AKS;
6. o AKS aplica a mudança em `Deployment`, `Service`, `Ingress`, `SecretProviderClass` e demais recursos.

Em produção, o repositório que representa a verdade operacional do cluster é o GitOps.

---

## Repositórios envolvidos

### 1. Repositórios de aplicação

São os repositórios onde vive o código-fonte de cada serviço.

Exemplos:

- `we-services-devices`
- `we-api`
- outros serviços da plataforma

Normalmente, o fluxo é:

1. abrir branch;
2. desenvolver;
3. abrir PR;
4. fazer merge para `main`;
5. pipeline publica imagem Docker.

### 2. Repositório GitOps

Repositório:

- `https://github.com/wedigitek/wedigitek-k8s-gitops.git`

Este repositório guarda os manifests Kubernetes e as aplicações ArgoCD.

Ele é o ponto central da operação em produção.

Tudo que deve existir no cluster de forma persistente precisa estar aqui.

### 3. ArgoCD

O ArgoCD observa o repositório GitOps e compara:

- o estado desejado no Git;
- o estado real no cluster AKS.

Se houver diferença, ele sincroniza automaticamente quando a `Application` estiver com `automated sync` habilitado.

### 4. AKS

Cluster de produção validado nesta operação:

- contexto Kubernetes: `aks-wedigitek-prd`

---

## Estrutura do repositório GitOps

Principais diretórios:

```text
argocd/
	apps/
		prd/
applications/
	prd/
infrastructure/
docs/
```

### `applications/prd/<app>`

Contém os manifests de uma aplicação em produção.

Exemplo de estrutura:

```text
applications/prd/broker-mqtt/
	deployment.yaml
	ingress.yaml
	kustomization.yaml
	namespace.yaml
	secretproviderclass-files.yaml
	service.yaml
```

### `argocd/apps/prd/<app>.yaml`

Contém a `Application` do ArgoCD apontando para o path correto em `applications/prd/<app>`.

Exemplo lógico:

```yaml
spec:
	source:
		repoURL: https://github.com/wedigitek/wedigitek-k8s-gitops.git
		targetRevision: main
		path: applications/prd/broker-mqtt
```

---

## Fluxo operacional completo para mandar mudança para produção

## Cenário A - mudança de código de uma aplicação

Use este fluxo quando houve alteração no código do serviço.

### Passo 1 - merge no repositório da aplicação

Depois que o PR é aprovado e entra em `main` no repositório da aplicação:

1. a pipeline gera uma nova imagem;
2. a imagem é publicada no registry configurado.

Exemplo comum:

- `ghcr.io/wedigitek/we-services-devices:main`

### Passo 2 - atualizar o repositório GitOps

Se o manifesto usa tag fixa diferente da desejada, atualizar o `deployment.yaml` da aplicação correspondente.

Exemplo:

- alterar imagem em `applications/prd/service-devices/deployment.yaml`

Em muitos casos, o padrão atual de produção está usando `:main`.

### Passo 3 - commit e push no GitOps

Após ajustar o manifesto:

1. commitar;
2. fazer push para `main` no repositório GitOps oficial.

### Passo 4 - ArgoCD detecta a mudança

Com `automated sync` habilitado, o ArgoCD reconcilia automaticamente.

### Passo 5 - validar no cluster

Validar:

1. `Application` no ArgoCD;
2. rollout do `Deployment`;
3. pods saudáveis;
4. logs se necessário.

---

## Cenário B - mudança de infraestrutura Kubernetes da aplicação

Use este fluxo quando a mudança é no manifesto e não no código.

Exemplos:

- novo `Service`;
- novo `Ingress`;
- novo `SecretProviderClass`;
- mount de arquivo vindo do Key Vault;
- alteração de `env`;
- ajuste de `resources`;
- ajuste de namespace;
- habilitar volume CSI.

Fluxo:

1. editar manifests dentro de `applications/prd/<app>`;
2. se a app ainda não existir no ArgoCD, criar `argocd/apps/prd/<app>.yaml`;
3. validar render com `kubectl kustomize`;
4. commitar;
5. push para `main` do GitOps oficial;
6. validar sincronização no ArgoCD;
7. validar rollout no AKS.

---

## Fluxo prático após merge para `main`

## O que acontece depois do merge

Depois que uma alteração entra em `main` do repositório GitOps oficial:

1. o GitHub passa a ter a nova verdade declarativa;
2. o `repo-server` do ArgoCD consulta esse repositório;
3. o `application-controller` recalcula o estado desejado;
4. o ArgoCD compara Git vs cluster;
5. se houver diferença, sincroniza;
6. o cluster passa a refletir a alteração.

Em resumo:

$$GitOps\ main \rightarrow ArgoCD \rightarrow AKS$$

---

## Como atualizar e refletir no Argo e no cluster

## 1. Atualizar manifests

Alterar o arquivo correto em `applications/prd/...`.

Exemplos:

- imagem: `deployment.yaml`
- hostname: `ingress.yaml`
- Key Vault CSI: `secretproviderclass-files.yaml`
- portas: `service.yaml`

## 2. Validar localmente

Renderizar antes de subir:

```bash
kubectl kustomize applications/prd/broker-mqtt
kubectl kustomize applications/prd/service-devices
```

## 3. Commitar e fazer push

O push precisa ir para o repositório usado pela `Application` do ArgoCD.

Importante:

- se a `Application` aponta para `wedigitek/wedigitek-k8s-gitops.git`, o push precisa existir no repositório oficial da organização;
- push apenas em fork não basta.

## 4. Validar no ArgoCD

Exemplo:

```bash
kubectl -n argocd get application broker-mqtt-prd service-devices-prd -o wide
```

Esperado:

- `SYNC STATUS: Synced`
- `HEALTH STATUS: Healthy`

## 5. Validar no AKS

Exemplo:

```bash
kubectl -n broker-mqtt get deploy broker-mqtt
kubectl -n service-devices get deploy service-devices service-devices-mongo
```

Esperado:

- réplicas prontas;
- rollout concluído;
- pods em `Running`.

---

## Padrão atual validado para broker-mqtt e service-devices

Nesta operação foi consolidado o fluxo GitOps destes dois serviços.

### `broker-mqtt`

Recursos versionados:

- namespace;
- service account;
- deployment;
- service interno;
- service público `LoadBalancer`;
- ingress websocket;
- `SecretProviderClass` com integração Key Vault.

### `service-devices`

Recursos versionados:

- namespace;
- deployment principal;
- deployment `service-devices-mongo`;
- services;
- `SecretProviderClass` com integração Key Vault.

### Secrets montados via Key Vault CSI

Secrets utilizados:

- `broker-mqtt-ca-crt`
- `broker-mqtt-ca-key`
- `broker-mqtt-server-crt`
- `broker-mqtt-server-key`
- `broker-mqtt-service-devices-password`
- `broker-mqtt-vmq-pwd`
- `broker-mqtt-acl`

### Vault validado

- `kv-wedigitek-prd`

### Resultado operacional validado

- `broker-mqtt-prd`: `Synced` e `Healthy`
- `service-devices-prd`: `Synced` e `Healthy`

---

## Como adicionar uma aplicação nova em produção

### 1. Criar diretório da aplicação

Exemplo:

```text
applications/prd/minha-app/
```

### 2. Criar os manifests mínimos

Normalmente:

- `namespace.yaml`
- `deployment.yaml`
- `service.yaml`
- `kustomization.yaml`

Se necessário também:

- `ingress.yaml`
- `secretproviderclass-env.yaml`
- `secretproviderclass-files.yaml`
- `hpa.yaml`
- `configmap.yaml`

### 3. Criar a `Application` do ArgoCD

Em:

```text
argocd/apps/prd/minha-app.yaml
```

### 4. Validar render

```bash
kubectl kustomize applications/prd/minha-app
```

### 5. Commit + push

### 6. Validar no ArgoCD e cluster

---

## Como atualizar imagem de uma aplicação

### Método recomendado

Editar o `deployment.yaml` no GitOps.

Exemplo:

```yaml
image: ghcr.io/wedigitek/we-services-devices:main
```

Se a política da equipe mudar para tags versionadas, atualizar para algo como:

```yaml
image: ghcr.io/wedigitek/we-services-devices:v1.2.3
```

Depois:

1. commit;
2. push;
3. validar ArgoCD;
4. validar rollout.

### O que não fazer

Evitar fazer apenas:

```bash
kubectl set image ...
```

Motivo:

- o ArgoCD tende a reverter para o estado do Git;
- isso gera drift e confusão operacional.

---

## Como funciona o Azure Key Vault neste projeto

O padrão utilizado é `Secrets Store CSI Driver` com provider Azure.

Fluxo:

1. o secret existe no Key Vault;
2. um `SecretProviderClass` referencia esse secret;
3. o pod monta os arquivos via volume CSI;
4. a aplicação lê o conteúdo do path configurado.

### Quando usar

Use Key Vault CSI quando o serviço precisar de:

- certificados;
- chaves privadas;
- arquivos ACL;
- senhas em arquivo;
- material sensível que não deve ficar versionado no Git.

### Exemplo real

No `broker-mqtt`:

- `vmq.acl`
- `vmq.passwd`
- `ca.crt`
- `ca.key`
- `broker.crt`
- `broker.key`

No `service-devices`:

- `mqtt_password`
- `ca.crt`
- `ca.key`
- também foram deixados `vmq.acl` e `vmq.passwd` montados no volume CSI para alinhamento com o conjunto compartilhado.

---

## Checklist operacional antes de mandar para produção

- manifest renderiza com `kubectl kustomize`;
- path do ArgoCD aponta para o diretório correto;
- `repoURL` da `Application` aponta para o repositório oficial esperado;
- imagem existe no registry;
- secrets existem no Key Vault;
- `SecretProviderClass` aponta para os nomes corretos dos secrets;
- namespace correto;
- portas corretas;
- service selector correto;
- ingress hostname correto;
- rollout anterior está saudável.

---

## Checklist operacional depois de mandar para produção

- `kubectl -n argocd get application <app> -o wide`;
- `Synced` e `Healthy`;
- `kubectl -n <ns> get deploy`;
- `kubectl -n <ns> get pods`;
- logs se necessário;
- teste funcional do endpoint;
- validação de secrets montados se a mudança envolver Key Vault.

---

## Principais problemas e como resolver

## 1. `Application` do ArgoCD fica com `Unknown` e `app path does not exist`

### Causa comum

- o path em `spec.source.path` está errado;
- o push foi feito apenas no fork local;
- a `Application` aponta para outro repositório;
- o ArgoCD ainda não viu o commit correto.

### Como resolver

1. confirmar o `repoURL` da `Application`;
2. confirmar o path exato em `applications/prd/...`;
3. confirmar que o commit está no repositório oficial usado pelo ArgoCD;
4. se necessário, recriar a `Application`;
5. aguardar reconciliação do repo-server.

### Caso real desta operação

Os apps `broker-mqtt-prd` e `service-devices-prd` inicialmente ficaram com esse erro até o commit estar confirmado no repositório oficial esperado e o ArgoCD reconciliar corretamente.

---

## 2. mudança foi aplicada com `kubectl`, mas depois sumiu

### Causa

Drift: o cluster foi alterado manualmente, mas o Git continua com outro estado.

### Como resolver

1. versionar a mudança no GitOps;
2. fazer push;
3. deixar o ArgoCD reconciliar.

---

## 3. secret existe no Key Vault, mas não aparece no pod

### Causa comum

- `SecretProviderClass` errado;
- `objectName` incorreto;
- `objectAlias` diferente do path esperado pela aplicação;
- pod antigo sem restart;
- identity/permite acesso incorretos.

### Como resolver

1. validar o secret no Key Vault;
2. conferir `SecretProviderClass`;
3. reiniciar o deployment;
4. validar arquivos dentro do pod.

---

## 4. serviço sobe, mas aplica ACL/arquivo errado

### Causa comum

- arquivo default da imagem está sendo usado;
- arquivo do Key Vault não foi montado no path certo;
- volume existe, mas faltou `subPath`.

### Como resolver

1. verificar `volumeMounts`;
2. conferir o path final dentro do container;
3. inspecionar o conteúdo do arquivo no pod.

Exemplo real:

- o `broker-mqtt` estava com ACL default (`topic #`) antes da montagem correta de `broker-mqtt-acl`.

---

## 5. push foi feito, mas o ArgoCD ainda não sincronizou

### Causa comum

- reconciliação ainda não aconteceu;
- diferença entre fork e upstream;
- `Application` apontando para repo diferente do que recebeu o push.

### Como resolver

1. conferir `repoURL`;
2. conferir o commit no remoto correto;
3. aguardar alguns minutos e revalidar;
4. se necessário, recriar a `Application` ou forçar refresh operacional.

---

## 6. deployment atualizado, mas pods não sobem

### Causa comum

- imagem inexistente;
- env inválida;
- secret ausente;
- service account/identity sem permissão;
- porta ou readiness quebrada.

### Como resolver

1. `kubectl describe pod`;
2. `kubectl logs`;
3. validar image tag;
4. validar mounts e envs.

---

## Comandos úteis de operação

### Ver Applications no ArgoCD

```bash
kubectl -n argocd get application
kubectl -n argocd get application broker-mqtt-prd service-devices-prd -o wide
```

### Ver detalhes de uma Application

```bash
kubectl -n argocd describe application broker-mqtt-prd
```

### Ver deployments

```bash
kubectl -n broker-mqtt get deploy
kubectl -n service-devices get deploy
```

### Ver pods

```bash
kubectl -n broker-mqtt get pods -o wide
kubectl -n service-devices get pods -o wide
```

### Ver arquivos montados dentro do pod

```bash
kubectl -n broker-mqtt exec <pod> -- ls -la /etc/vernemq/certs
kubectl -n service-devices exec <pod> -- ls -la /run/secrets
```

### Ver SecretProviderClass

```bash
kubectl -n broker-mqtt get secretproviderclass broker-mqtt-keyvault-files -o yaml
kubectl -n service-devices get secretproviderclass service-devices-keyvault-files -o yaml
```

### Renderizar manifests localmente

```bash
kubectl kustomize applications/prd/broker-mqtt
kubectl kustomize applications/prd/service-devices
```

---

## Boas práticas recomendadas

- sempre tratar GitOps como fonte da verdade;
- evitar hotfix manual sem versionar depois;
- validar `repoURL` e `path` da `Application`;
- usar Key Vault para material sensível em arquivo;
- manter nomenclatura consistente entre Key Vault, alias e path montado;
- validar rollout depois de qualquer merge em produção;
- registrar serviços novos em `applications/prd/...` e `argocd/apps/prd/...` no mesmo PR.

---

## Estado validado nesta entrega

Foi validado em produção:

- `broker-mqtt-prd` sincronizado e saudável;
- `service-devices-prd` sincronizado e saudável;
- `broker-mqtt` com `vmq.acl` e `vmq.passwd` vindos do Key Vault;
- `service-devices` com secrets MQTT/TLS montados corretamente;
- GitOps persistido no repositório oficial;
- ArgoCD refletindo a revisão correta no cluster.

---

## Referência rápida de operação

Se precisar resumir em uma linha:

$$merge\ no\ GitOps\ main \Rightarrow ArgoCD\ reconcilia \Rightarrow AKS\ aplica$$

Se a mudança não refletir:

1. conferir `Application`;
2. conferir `repoURL`;
3. conferir `path`;
4. conferir commit no remoto correto;
5. conferir rollout no cluster.

