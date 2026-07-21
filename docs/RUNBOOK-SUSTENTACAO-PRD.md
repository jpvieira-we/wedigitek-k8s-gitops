# Runbook de Sustentacao - Producao

## 1. Objetivo e escopo

Este documento orienta a operacao e a sustentacao do ambiente Kubernetes de producao da WE:DIGITEK. Ele cobre arquitetura, deploy GitOps, verificacao de saude, testes, logs, observabilidade, variaveis, secrets, troubleshooting e escalonamento.

Nunca registre valores de secrets em chamados, e-mails, commits, screenshots ou logs compartilhados. Este documento informa onde localizar e como validar secrets sem revelar seu conteudo.

## 2. Ambiente validado

| Item | Valor |
|---|---|
| Cloud | Microsoft Azure |
| Tenant | Wedigitek |
| Assinatura | WDT-Internal-Services |
| Resource group | `rg-wedigitek-prd` |
| Cluster | `aks-wedigitek-prd` |
| Repositorio GitOps efetivo | `https://github.com/jpvieira-we/wedigitek-k8s-gitops.git` |
| Branch de producao | `main` |
| Argo CD | `https://argocd.trywefood.dev` |
| Grafana | `https://grafana.trywefood.dev` |
| Key Vault | `kv-wedigitek-prd` |
| IP publico NGINX | `4.236.208.42` |
| IP publico MQTT TLS | `4.157.234.228` |

Os IPs acima representam o estado validado em 21/07/2026. O DNS e o status do Service LoadBalancer sao as fontes autoritativas; valide-os antes de alterar firewall, allowlist ou registro DNS.

## 3. Arquitetura e responsabilidades

O ambiente separa infraestrutura Azure, configuracao Kubernetes e codigo das aplicacoes:

1. A infraestrutura Azure e provisionada fora deste repositorio.
2. Os repositorios das aplicacoes geram imagens no GHCR.
3. Este repositorio declara Deployments, Services, Ingresses, ConfigMaps, SecretProviderClasses e demais recursos Kubernetes.
4. O Argo CD acompanha a branch `main` e aplica automaticamente as diferencas no AKS.
5. O ingress NGINX publica os endpoints e o cert-manager administra os certificados TLS.
6. O Azure Key Vault fornece material sensivel por meio do Secrets Store CSI Driver.
7. Prometheus, Grafana, Loki, Promtail e OpenTelemetry compoem a observabilidade.

Fluxo de entrega:

```text
Codigo -> pipeline da aplicacao -> imagem GHCR -> alteracao GitOps
      -> Argo CD -> AKS -> Service/Ingress -> usuario
```

Fluxo de uma requisicao HTTPS:

```text
Cliente -> DNS -> 4.236.208.42:443 -> Azure LoadBalancer
    -> ingress-nginx -> Ingress pelo host -> Service ClusterIP
    -> EndpointSlice -> Pod na porta declarada
```

O MQTT possui dois caminhos: WebSocket seguro pelo NGINX no host `mqtt.trywefood.dev` e MQTT TLS direto pelo LoadBalancer `4.157.234.228:8883`.

O Git e a fonte de verdade. Alteracoes diretas com `kubectl edit`, `kubectl set image` ou `kubectl apply` podem ser revertidas pelo Argo CD e devem ser usadas apenas em diagnostico emergencial autorizado.

## 4. Acessos e ferramentas

Ferramentas recomendadas:

- Azure CLI (`az`)
- Kubernetes CLI (`kubectl`)
- Kustomize, incluido no `kubectl`
- Git e GitHub CLI (`gh`)
- navegador para Argo CD e Grafana

Login Azure:

```powershell
az login --use-device-code
az account show --output table
```

Obter credenciais e contexto do AKS quando `kubectl` estiver instalado:

```powershell
az aks get-credentials --resource-group rg-wedigitek-prd --name aks-wedigitek-prd --overwrite-existing
kubectl config current-context
kubectl get nodes
```

Sem `kubectl` local, execute uma consulta remota:

```powershell
az aks command invoke --resource-group rg-wedigitek-prd --name aks-wedigitek-prd --command "kubectl get nodes"
```

O acesso deve seguir menor privilegio. Nao compartilhe a senha administrativa do Argo CD; prefira contas nominativas e RBAC.

### Credenciais e senhas

Este repositorio e este runbook nao armazenam valores de senha, token, connection string ou certificado privado. A tabela indica onde cada credencial e administrada:

| Acesso | Identidade recomendada | Local do valor/controle |
|---|---|---|
| Azure Portal e CLI | conta Entra ID nominativa com MFA | Microsoft Entra ID e Azure RBAC |
| AKS | mesma conta Azure, convertida por `kubelogin` | Azure RBAC/Kubernetes RBAC e kubeconfig local |
| GitHub e GHCR | conta GitHub nominativa ou GitHub App | GitHub Organization, repository secrets e `ghcr-pull-secret` nos namespaces |
| Argo CD | conta nominativa/SSO e RBAC | `argocd-rbac-cm`; credencial administrativa no Secret do namespace `argocd` |
| Grafana | conta nominativa quando configurada | Secret `monitoring/prometheus-stack-grafana`, chaves `admin-user` e `admin-password` |
| Aplicacoes e bancos | identidade do workload | Key Vault `kv-wedigitek-prd`, SecretProviderClass e Secret Kubernetes sincronizado |

Para recuperar uma credencial administrativa em uma atividade autorizada, consulte diretamente o cofre ou Secret no terminal local e nao copie o valor para chamado, e-mail ou documento. Exemplos de identificacao, sem exibir valor:

```powershell
kubectl get secret -n argocd
kubectl get secret prometheus-stack-grafana -n monitoring \
  -o jsonpath="{.metadata.name}"
kubectl get secretproviderclass -A
az keyvault secret list --vault-name kv-wedigitek-prd --query "[].name" -o table
```

O responsavel pelo acesso deve registrar solicitante, aprovador, escopo, data de concessao e data de revogacao. Contas compartilhadas devem ser substituidas por contas nominativas sempre que a ferramenta permitir.

## 5. Estrutura do repositorio

| Caminho | Finalidade |
|---|---|
| `applications/prd/<app>/` | Recursos Kubernetes da aplicacao |
| `argocd/apps/prd/` | Applications monitoradas pelo Argo CD |
| `infrastructure/argocd/` | Configuracao do Argo CD |
| `infrastructure/monitoring/` | Prometheus, Grafana, Loki e OpenTelemetry |
| `docs/` | Arquitetura e runbooks |
| `.github/workflows/` | Validacoes automatizadas |

Cada aplicacao normalmente possui `namespace.yaml`, `deployment.yaml`, `service.yaml`, `kustomization.yaml` e, quando aplicavel, `ingress.yaml`, `hpa.yaml`, `configmap.yaml` e `secretproviderclass-*.yaml`.

## 6. Catalogo de endpoints publicos

| Aplicacao | Endpoint |
|---|---|
| we-admin | `https://admin.trywefood.dev` |
| we-api | `https://api.trywefood.dev`, `https://api.direct.trywefood.dev` |
| data-api | `https://data.trywefood.dev` |
| broker-mqtt WebSocket | `https://mqtt.trywefood.dev` |
| reports | `https://reports.services.trywefood.dev` |
| surveys | `https://surveys.trywefood.dev` |
| surveys-reports | `https://surveys.services.trywefood.dev` |
| quality-control-webserver | `https://qualitycontrol.trywefood.dev` |
| we-marketplace | `https://marketplace.trywefood.dev` |
| we-wallet | `https://wallet.trywefood.dev`, `https://wecampus.trywefood.dev` |
| web-app | `https://web.trywefood.dev`, `https://grsa.trywefood.dev` |
| wefood-drive | `https://drive.trywefood.dev` |
| wefood-onboarding | `https://onboarding.trywefood.dev` |
| wefood-realtime-server | `https://orders.trywefood.dev` |
| Argo CD | `https://argocd.trywefood.dev` |
| Grafana | `https://grafana.trywefood.dev` |

Nem todo servico possui uma rota HTTP valida em `/`. Um `404` na raiz pode significar que ingress e TLS estao acessiveis, mas a aplicacao exige uma rota especifica. Um `503` do NGINX normalmente indica Service sem endpoints prontos.

### Rede e portas principais

| Componente | Exposicao | Porta(s) | Endereco/uso |
|---|---|---|---|
| ingress-nginx | LoadBalancer publico | `80/TCP`, `443/TCP` | `4.236.208.42`; HTTP e HTTPS dos hosts publicados |
| broker-mqtt | LoadBalancer publico | `8883/TCP` | `4.157.234.228`; MQTT com TLS |
| broker-mqtt WebSocket | Ingress HTTPS | `443/TCP` | `mqtt.trywefood.dev` |
| MongoDB central | ClusterIP interno | `27017/TCP` | `mongodb.mongodb.svc.cluster.local` |
| Redis master | ClusterIP interno | `6379/TCP` | `redis-master.redis.svc.cluster.local` |
| Redis replicas | ClusterIP interno | `6379/TCP` | `redis-replicas.redis.svc.cluster.local` |
| Prometheus | ClusterIP interno | `9090/TCP` | coleta e consulta de metricas |
| Alertmanager | ClusterIP interno | `9093/TCP`, `9094/TCP/UDP` | roteamento e agrupamento de alertas |
| Grafana | Ingress HTTPS/ClusterIP | `443/TCP` externo, `80/TCP` interno | `grafana.trywefood.dev` |
| Loki | ClusterIP interno | `3100/TCP`, `9095/TCP` | API de logs e comunicacao interna |
| OpenTelemetry Collector | ClusterIP interno | `4317/TCP`, `4318/TCP`, `8889/TCP` | OTLP gRPC, OTLP HTTP e metricas exportadas |
| Blackbox Exporter | ClusterIP interno | `9115/TCP` | probes sinteticos HTTP/TCP |
| MongoDB Exporter | ClusterIP interno | `9216/TCP` | metricas do MongoDB |
| Redis Exporter | ClusterIP interno | `9121/TCP` | metricas do Redis |
| Node Exporter | interno por node | `9100/TCP` | metricas de sistema operacional |

ClusterIP e IP de Pod sao dinamicos e nao devem ser usados em DNS externo, firewall ou configuracao permanente. Consulte o valor atual:

```powershell
kubectl get service -A -o wide
kubectl get ingress -A -o wide
kubectl get endpointslice -A
kubectl get service -n ingress-nginx ingress-nginx-controller -o wide
kubectl get service -n broker-mqtt broker-mqtt-public -o wide
```

Para descobrir a porta de uma aplicacao especifica, compare `containerPort` no Deployment, `targetPort`/`port` no Service e o backend do Ingress. Esses tres pontos devem ser coerentes.

## 7. Rotina diaria de saude

### Argo CD

Na interface, verifique:

- `Sync Status = Synced`
- `Health Status = Healthy`
- ultima revisao aplicada
- recursos filhos e eventos da Application

Via CLI:

```powershell
kubectl get applications.argoproj.io -n argocd \
  -o custom-columns=NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status,REVISION:.status.sync.revision
```

Interpretacao:

- `Synced/Healthy`: estado esperado.
- `OutOfSync`: Git e cluster divergem.
- `Progressing`: rollout ou Job ainda em andamento; examine os recursos filhos.
- `Degraded`: recurso falhou.
- `Unknown`: Argo CD nao conseguiu comparar ou avaliar.

### Cluster e workloads

```powershell
kubectl get nodes -o wide
kubectl get pods -A
kubectl get deployments -A
kubectl get statefulsets -A
kubectl get cronjobs -A
kubectl get events -A --sort-by=.lastTimestamp
```

Para uma aplicacao:

```powershell
kubectl rollout status deployment/<deployment> -n <namespace> --timeout=300s
kubectl get pod,service,endpoints,ingress -n <namespace> -o wide
kubectl describe deployment/<deployment> -n <namespace>
```

## 8. Como testar uma aplicacao

1. Confirme `Synced/Healthy` no Argo CD.
2. Confirme replicas `READY = DESIRED` no Deployment.
3. Confirme pods `Running` e containers `Ready`.
4. Confirme que o Service possui EndpointSlices/endpoints.
5. Confirme ingress, DNS e certificado.
6. Execute o health check documentado pela aplicacao.
7. Execute um teste funcional autenticado com dados de teste, nunca com dados reais destrutivos.

Comandos:

```powershell
kubectl get deployment,pod,service,endpointslice,ingress -n <namespace> -o wide
Resolve-DnsName <host>
curl.exe -I https://<host>
```

Para distinguir ingress de backend:

- falha DNS: registro ausente ou incorreto;
- timeout/TLS: load balancer, ingress ou certificado;
- HTTP 503: backend sem endpoints prontos;
- HTTP 404: host acessivel, rota consultada inexistente;
- HTTP 401/403: servico acessivel, autenticacao/autorizacao exigida;
- HTTP 200/204/3xx esperado: conectividade funcional.

## 9. Logs e troubleshooting

Logs atuais:

```powershell
kubectl logs -n <namespace> deployment/<deployment> --tail=200
kubectl logs -n <namespace> deployment/<deployment> -f
```

Container especifico ou falha anterior:

```powershell
kubectl logs -n <namespace> <pod> -c <container> --tail=200
kubectl logs -n <namespace> <pod> -c <container> --previous --tail=200
```

Eventos e descricao:

```powershell
kubectl describe pod -n <namespace> <pod>
kubectl get events -n <namespace> --sort-by=.lastTimestamp
```

Erros frequentes:

| Sintoma | Causa provavel | Acao |
|---|---|---|
| `ImagePullBackOff` | tag/digest inexistente, blob removido ou credencial GHCR | testar o pull real, validar `ghcr-pull-secret` e publicar tag imutavel; listar tags nao comprova que o blob existe |
| `CrashLoopBackOff` | erro da aplicacao, runtime ou configuracao | usar `--previous`, conferir env/secrets e dependencias |
| `FailedMount` CSI | driver iniciando, identidade ou secret ausente | conferir pods CSI, SecretProviderClass e Key Vault |
| `FailedAttachVolume` apos retomada do AKS | `VolumeAttachment` CSI obsoleto ligado ao node anterior | confirmar que nenhum pod usa o volume, remover apenas o attachment obsoleto e recriar o pod |
| Service sem endpoints | pods nao prontos ou selector incorreto | comparar labels do pod e selector do Service |
| `OutOfSync` | drift ou commit novo | comparar diff no Argo; corrigir pelo Git |
| HTTP 503 | Service sem endpoints | validar rollout e readiness |

## 10. Variaveis, ConfigMaps e secrets

### Onde localizar a declaracao

- variaveis nao sensiveis: `applications/prd/<app>/configmap.yaml` e `env:` do `deployment.yaml`;
- importacao de ConfigMap/Secret: `envFrom:` do `deployment.yaml`;
- nomes e mapeamentos de secrets: `applications/prd/<app>/secretproviderclass-env.yaml`;
- arquivos sensiveis: `applications/prd/<app>/secretproviderclass-files.yaml`;
- volume e caminho de montagem: `volumes:` e `volumeMounts:` do `deployment.yaml`;
- valores reais: Azure Key Vault `kv-wedigitek-prd`.

O manifesto mostra `objectName` no Key Vault, chave criada no Secret Kubernetes e nome do Secret sincronizado. Ele nao deve conter o valor.

Listar apenas nomes no Key Vault:

```powershell
az keyvault secret list --vault-name kv-wedigitek-prd --query "[].name" --output table
```

Ver metadados de um secret, sem valor:

```powershell
az keyvault secret show --vault-name kv-wedigitek-prd --name <nome> \
  --query "{name:name,enabled:attributes.enabled,updated:attributes.updated,expires:attributes.expires}" --output table
```

Validar o SecretProviderClass e o mount:

```powershell
kubectl get secretproviderclass -n <namespace>
kubectl describe secretproviderclass <nome> -n <namespace>
kubectl get secret -n <namespace>
kubectl describe pod <pod> -n <namespace>
```

Nao use `kubectl get secret -o yaml`, `az keyvault secret show --query value` ou `env` dentro do pod em sessoes gravadas/compartilhadas. Para rotacao:

1. atualizar a versao no Key Vault;
2. aguardar a rotacao do CSI ou reiniciar o Deployment de forma controlada;
3. validar rollout e aplicacao;
4. revogar a versao anterior quando confirmado;
5. registrar somente nome, data e responsavel, nunca o valor.

## 11. Deploy e rollback via GitOps

### Imagens e GHCR

As imagens de aplicacao usam o GitHub Container Registry no formato `ghcr.io/wedigitek/<imagem>:<tag>`. O pipeline do repositorio da aplicacao compila e publica a imagem; o repositorio GitOps apenas seleciona a imagem que o Kubernetes deve executar.

Fluxo recomendado:

1. pipeline executa testes e build da aplicacao;
2. pipeline publica uma tag imutavel, por exemplo `v1.8.3`, e registra o digest;
3. PR altera somente o campo `image:` do Deployment GitOps;
4. validacao renderiza o Kustomize e revisa o diff;
5. merge em `main` e detectado pelo Argo CD;
6. Kubernetes baixa a imagem usando `imagePullSecrets: ghcr-pull-secret`;
7. probes e estrategia RollingUpdate controlam a entrada dos novos pods;
8. equipe valida rollout, logs, metricas e teste funcional.

Antes de promover, teste que o manifesto e a imagem existem. A listagem de tags do registry nao garante que todos os blobs referenciados continuam disponiveis. Em producao, prefira tag de release ou digest e preserve as imagens necessarias para rollback.

```powershell
kubectl kustomize applications/prd/<app>
kubectl get secret ghcr-pull-secret -n <namespace>
kubectl get deployment <deployment> -n <namespace> \
  -o jsonpath="{.spec.template.spec.containers[*].image}"
```

Nao exiba nem decodifique o conteudo de `ghcr-pull-secret` em sessoes compartilhadas. A rotacao deve atualizar a credencial autorizada em cada namespace consumidor e ser validada com um rollout controlado.

### Deploy

```powershell
git checkout main
git pull --ff-only
# editar applications/prd/<app>/...
kubectl kustomize applications/prd/<app>
git diff --check
git checkout -b fix/<descricao>
git add <arquivos>
git commit -m "fix(prd): <descricao>"
git push -u origin fix/<descricao>
```

Abra PR, obtenha aprovacao e faça merge. Quando o fluxo excepcional autorizado usar push direto, registre justificativa e revisao posterior.

Depois do merge:

```powershell
kubectl get application <app> -n argocd -w
kubectl rollout status deployment/<deployment> -n <namespace> --timeout=300s
```

Prefira tags imutaveis (`vX.Y.Z` ou digest). Tags flutuantes como `main`, `develop`, `rc` e `latest` podem ser removidas ou republicadas e dificultam rollback e auditoria.

### Como o Argo CD reconcilia

As Applications em `argocd/apps/prd/` apontam para os diretorios Kustomize em `applications/prd/`. O App-of-Apps registra as Applications e cada uma compara continuamente a branch `main` com os recursos do cluster. A politica automatizada usa sync, prune e self-heal quando declarados:

- **sync** cria ou atualiza recursos para igualar o Git;
- **prune** remove recursos que deixaram de existir no caminho GitOps;
- **self-heal** corrige alteracoes manuais no cluster;
- **health assessment** avalia rollout, disponibilidade e recursos filhos.

Um estado `Synced` significa que Git e cluster coincidem; nao substitui teste funcional. Um Deployment intencionalmente declarado com `replicas: 0` pode estar `Synced/Healthy`, pois zero e o estado desejado.

### Rollback

O rollback persistente deve reverter o commit GitOps:

```powershell
git revert <commit>
git push origin main
```

Depois, valide Argo CD, rollout, logs e endpoint. Nao use apenas `kubectl rollout undo`, pois o Argo CD pode reaplicar o estado do Git.

## 12. Observabilidade

### Prometheus

Coleta metricas do cluster e workloads por ServiceMonitor, PodMonitor e annotations. A stack inclui kube-state-metrics, metricas de nodes/pods e exporters dedicados para MongoDB e Redis. O Blackbox Exporter pode executar probes de endpoints.

Consultas PromQL uteis:

```promql
up
kube_deployment_status_replicas_unavailable > 0
sum by (namespace, pod) (rate(container_cpu_usage_seconds_total[5m]))
sum by (namespace, pod) (container_memory_working_set_bytes)
kube_pod_container_status_restarts_total
```

### Grafana

URL: `https://grafana.trywefood.dev`.

Possui datasource Prometheus e datasource Loki provisionado. Use dashboards para cluster, workloads, MongoDB, Redis e componentes da observabilidade. Para logs, abra **Explore**, selecione Loki e filtre por namespace/pod.

Responsabilidades da stack:

| Solucao | Responsabilidade |
|---|---|
| Prometheus | coleta e armazena series temporais de metricas |
| kube-state-metrics | expoe estado de objetos Kubernetes |
| Node Exporter | expoe CPU, memoria, disco e rede dos nodes |
| MongoDB/Redis Exporters | expoem metricas dos datastores |
| Blackbox Exporter | testa disponibilidade externa e protocolos |
| Alertmanager | agrupa, silencia e encaminha alertas |
| Grafana | dashboards, Explore e visualizacao unificada |
| Promtail | descobre pods e envia stdout/stderr para Loki |
| Loki | indexa labels e armazena logs por 7 dias |
| OpenTelemetry | recebe e processa telemetria OTLP das aplicacoes |

O caminho de logs e `container stdout/stderr -> Promtail -> Loki -> Grafana Explore`. O caminho de metricas e `exporter/ServiceMonitor -> Prometheus -> Grafana/Alertmanager`. O caminho de telemetria instrumentada e `aplicacao -> OTel Collector -> endpoint configurado`, com metricas expostas ao Prometheus.

Exemplos LogQL:

```logql
{namespace="we-api-prd"}
{namespace="we-api-prd", pod=~"we-api-.*"} |= "error"
{namespace="argocd"} |~ "(?i)error|failed"
```

### Loki e Promtail

Promtail coleta logs de containers e envia para `loki.monitoring:3100`. A retencao declarada do Loki e de `168h` (7 dias). Logs necessarios por mais tempo devem ser exportados para o mecanismo corporativo aprovado, respeitando dados pessoais e secrets.

### OpenTelemetry

O OpenTelemetry Collector recebe OTLP por gRPC `4317` e HTTP `4318`, processa a telemetria e expoe metricas para o Prometheus na porta `8889`. Aplicacoes Node compativeis podem receber auto-instrumentacao pela annotation:

```yaml
instrumentation.opentelemetry.io/inject-nodejs: "monitoring/nodejs-instrumentation"
```

Nao habilite auto-instrumentacao em runtimes Node antigos sem teste. Bibliotecas atuais podem usar sintaxe nao suportada e causar `CrashLoopBackOff`. Quando incompatível, remova a annotation para restaurar o servico e planeje a atualizacao do runtime da aplicacao antes de reativar a telemetria.

### Alertas

Um alerta deve conter no minimo ambiente, aplicacao, namespace, sintoma, inicio, impacto, link do dashboard e acao executada. Correlacione:

1. status Argo CD;
2. disponibilidade de replicas;
3. eventos Kubernetes;
4. metricas Prometheus;
5. logs Loki/aplicacao;
6. endpoint funcional.

## 13. Operacoes especiais

### MongoDB do we-api

O namespace `we-api-prd` possui MongoDB isolado e sincronizacao periodica declarada em `applications/prd/we-api/mongodb-sync-cronjob.yaml`. A origem fica no Secret `we-api-mongo-sync-source`, chave `SOURCE_MONGO_URI`; o valor nao deve ser exibido. Verifique Jobs e logs:

```powershell
kubectl get cronjob,job,pod -n we-api-prd
kubectl logs -n we-api-prd -l app=we-api-mongo-sync --tail=100
```

Deve existir somente o CronJob `we-api-mongo-sync`. Um CronJob duplicado fora do GitOps executa outro `mongodump | mongorestore --drop`, aumenta a carga de CPU e I/O e pode indisponibilizar o MongoDB central e seus consumidores. Compare periodicamente o cluster com o manifesto:

```powershell
kubectl get cronjob -n we-api-prd
kubectl get cronjob we-api-mongo-sync -n we-api-prd -o jsonpath="{.spec.schedule}"
```

Antes de remover um recurso duplicado, confirme no Argo CD e no repositorio que ele nao pertence ao estado declarado. A sincronizacao completa a cada cinco minutos deve ser revista caso a carga volte a afetar probes ou latencia; prefira sincronizacao incremental ou uma janela menos agressiva.

Quando o MongoDB central estiver saudavel, mas um consumidor antigo continuar com timeout de buffering do Mongoose, recrie o Deployment de forma controlada para renovar o estado da conexao e valide o rollout e o endpoint.

### Recuperacao apos parada do AKS

Apos iniciar um cluster que ficou parado, valide nodes, drivers CSI, `VolumeAttachment`, MongoDB e Redis antes de reiniciar todas as aplicacoes. Volumes podem permanecer associados ao identificador de um node anterior.

```powershell
kubectl get nodes
kubectl get pods -n kube-system -o wide
kubectl get volumeattachment
kubectl get statefulset,pod,pvc -n mongodb
kubectl get statefulset,pod,pvc -n redis
```

Nunca exclua PVC ou PV para corrigir attachment obsoleto. Preserve o disco, confirme o attachment incorreto e remova somente o objeto `VolumeAttachment` quando o volume nao estiver montado por outro pod.

### Reinicio controlado

Use somente quando a configuracao declarada esta correta e a falha e transitoria:

```powershell
kubectl rollout restart deployment/<deployment> -n <namespace>
kubectl rollout status deployment/<deployment> -n <namespace> --timeout=300s
```

Registre motivo, horario, responsavel e resultado. Se a falha voltar, corrija a causa na aplicacao ou no GitOps.

## 14. Backup, continuidade e seguranca

Antes de assumir RPO/RTO, confirme os procedimentos no plano corporativo de continuidade. Este repositorio nao comprova sozinho backup integral de PVCs, bancos e estado externo.

Recomendacoes operacionais:

- testar restauracao de bancos e volumes periodicamente;
- manter imagens versionadas disponiveis no registry;
- rotacionar credenciais e remover acessos antigos;
- usar contas nominativas e MFA;
- revisar RBAC de Azure, Kubernetes, Argo CD, GitHub e Grafana;
- registrar mudancas de producao;
- nunca inserir secrets no Git;
- manter contatos e escalonamento atualizados.

## 15. Checklist de incidente

1. Registrar horario, impacto e endpoint afetado.
2. Verificar Azure/AKS e nodes.
3. Verificar Argo CD (`Sync` e `Health`).
4. Verificar Deployment, pods, endpoints e ingress.
5. Coletar eventos e logs atuais/anteriores.
6. Correlacionar metricas e logs no Grafana.
7. Identificar mudanca recente no GitOps ou imagem.
8. Aplicar mitigacao autorizada.
9. Validar teste funcional e monitorar estabilidade.
10. Registrar causa raiz, evidencias sem secrets e acao preventiva.

## 16. Criterios de aceite da sustentacao

- acessos nominativos concedidos e testados;
- responsaveis por Azure, GitHub, Argo CD, Grafana e Key Vault definidos;
- alertas e canal de acionamento definidos;
- RTO/RPO e janela de atendimento acordados;
- fluxo de mudanca e aprovacao acordado;
- teste de deploy e rollback realizado;
- teste de consulta de logs e dashboards realizado;
- procedimento de rotacao de secret validado;
- inventario e documentacao revisados pelos responsaveis.

## 17. Referencias

- `README.md`
- `GITOPS-OPERACAO-PRD.md`
- `KEYVAULT_SETUP_GUIDE.md`
- `docs/architecture.md`
- `infrastructure/argocd/README.md`
- `infrastructure/monitoring/`
