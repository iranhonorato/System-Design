# Observabilidade no Kubernetes

**Pré-requisito:** [02_slos_e_alertas.md](02_slos_e_alertas.md) e a parte de Kubernetes, até [../../02-containers/kubernetes/04_do_compose_ao_cluster.md](../../02-containers/kubernetes/04_do_compose_ao_cluster.md)

> Segue a documentação do Kubernetes (*Resource metrics pipeline*, *Logging architecture*, *Events*), do *kube-prometheus-stack* e do Prometheus Operator, do OpenTelemetry Operator e do Collector (*k8sattributes*, *filelog*), do Grafana Alloy e do Argo Rollouts (*Analysis*). Na AWS, segue a documentação do CloudWatch Container Insights, do Amazon Managed Service for Prometheus e do AWS Distro for OpenTelemetry.

---

## Por que este arquivo existe

O laboratório do [primeiro capítulo prático](01_stack_de_observabilidade.md) rodava em uma máquina, com nomes fixos e poucos componentes. No cluster da parte de [Kubernetes](../../02-containers/SUMARIO.md), três coisas mudam:

- **Há mais o que observar.** Além da aplicação, o próprio cluster tem estado: nodes que ficam sem memória, Pods que não conseguem ser agendados, Deployments com menos réplicas prontas do que deveriam, containers mortos por `OOMKilled`.
- **Tudo é efêmero.** Pods mudam de nome, de IP e de node a cada deploy. Logs guardados no node somem com o Pod. Os sinais precisam carregar **de qual Pod, namespace e node** vieram, para que seja possível filtrá-los depois que o Pod já não existir.
- **A coleta precisa ser automática.** Ninguém configura à mão um alvo do Prometheus para cada Pod novo, nem instala um agente de logs em cada container.

A pergunta central é: *como observar um cluster Kubernetes e as aplicações que rodam nele, de forma que cada Pod novo seja coberto automaticamente e que os sinais continuem úteis depois que os Pods deixarem de existir?* O capítulo percorre as camadas do cluster, mostra de onde vem cada sinal, como coletá-los com as ferramentas mais usadas, e termina ligando tudo à entrega contínua: o deploy canary que só avança se as métricas permitirem.

---

## 1. As camadas que se observa

```
┌───────────────────────────────────────────────────────────────────────────────┐
│ ④ APLICAÇÃO     RED de cada serviço, traces, logs, métricas de negócio         │  ← OpenTelemetry
├───────────────────────────────────────────────────────────────────────────────┤
│ ③ CONTAINERS    CPU, memória em relação ao limit, throttling, reinícios        │  ← cAdvisor (no kubelet)
├───────────────────────────────────────────────────────────────────────────────┤
│ ② OBJETOS       Deployments com réplicas faltando, Pods Pending, HPA no máximo │  ← kube-state-metrics
│   DO CLUSTER    e os EVENTOS: OOMKilling, FailedScheduling, BackOff            │  ← API de eventos
├───────────────────────────────────────────────────────────────────────────────┤
│ ① NODES E       CPU, memória e disco dos nodes; API server, etcd, scheduler    │  ← node-exporter,
│   CONTROL PLANE                                                                │    métricas do control plane
└───────────────────────────────────────────────────────────────────────────────┘
```

| Camada | Pergunta típica | De onde vem |
|---|---|---|
| ① Nodes | "Algum node está sem memória ou sem disco?" | **node-exporter** (um DaemonSet que expõe as métricas do sistema operacional de cada node) |
| ① Control plane | "O API server está lento? O etcd está saudável?" | Métricas expostas pelos próprios componentes (nos clusters gerenciados, o provedor cuida da maior parte) |
| ② Objetos | "Quantos Pods estão `Pending`? O Deployment tem as réplicas que deveria?" | **kube-state-metrics** (lê o estado dos objetos na API do cluster e o expõe como métricas) |
| ② Eventos | "Por que este Pod foi reiniciado? Por que não foi agendado?" | **Eventos** do Kubernetes (seção 6) |
| ③ Containers | "Este container está perto do limit de memória? Está sendo desacelerado?" | **cAdvisor**, embutido no kubelet de cada node |
| ④ Aplicação | "O checkout está falhando? Em qual serviço está a demora?" | **OpenTelemetry**, como nos capítulos anteriores |

A camada ④ é a que interessa aos usuários, e é dela que vêm os SLIs do [capítulo de SLOs](02_slos_e_alertas.md). As camadas ① a ③ são as **causas** mais comuns dos problemas que aparecem na ④, e são consultadas durante a investigação.

---

## 2. As métricas do cluster

### 2.1 Duas fontes que se confundem

> **Confusão comum:** "o cluster já tem o `metrics-server` (é ele que faz o `kubectl top` e o HPA funcionarem), então já temos monitoramento". ✅ **Mais preciso:** o `metrics-server` do [capítulo de workloads](../../02-containers/kubernetes/02_workloads_rede_e_escala.md) guarda **só o valor atual** de CPU e memória de cada Pod e de cada node, em memória, para que o HPA e o `kubectl top` possam consultá-lo. Ele não guarda histórico (não responde "como estava a memória ontem às 18h?"), não tem métricas de rede, de disco, de reinícios ou de estado dos objetos, e não serve para alertas. Monitorar o cluster exige um sistema de métricas de verdade, normalmente o Prometheus, alimentado pelo cAdvisor, pelo kube-state-metrics e pelo node-exporter. Os dois convivem: o `metrics-server` para o auto-scaling, o Prometheus para a observabilidade.

### 2.2 As consultas que mais aparecem

| Pergunta | Consulta (PromQL) | Fonte |
|---|---|---|
| Uso de memória de cada Pod **em relação ao limit** | `sum by (pod) (container_memory_working_set_bytes{namespace="loja", container!=""}) / sum by (pod) (kube_pod_container_resource_limits{namespace="loja", resource="memory"})` | cAdvisor + kube-state-metrics |
| Proporção do tempo em que a CPU foi **desacelerada** (*throttling*) | `sum by (pod) (rate(container_cpu_cfs_throttled_periods_total{namespace="loja"}[5m])) / sum by (pod) (rate(container_cpu_cfs_periods_total{namespace="loja"}[5m]))` | cAdvisor |
| Reinícios na última hora | `sum by (pod) (increase(kube_pod_container_status_restarts_total{namespace="loja"}[1h]))` | kube-state-metrics |
| Containers cujo último término foi por falta de memória | `kube_pod_container_status_last_terminated_reason{namespace="loja", reason="OOMKilled"} == 1` | kube-state-metrics |
| Pods que não conseguem ser agendados | `sum(kube_pod_status_phase{namespace="loja", phase="Pending"})` | kube-state-metrics |
| Deployments com **menos réplicas prontas** do que o desejado | `kube_deployment_spec_replicas{namespace="loja"} - kube_deployment_status_replicas_available{namespace="loja"} > 0` | kube-state-metrics |
| Memória disponível em cada node | `node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes` | node-exporter |

Essas consultas são a versão em métricas do roteiro de diagnóstico do [capítulo do Compose ao cluster](../../02-containers/kubernetes/04_do_compose_ao_cluster.md): o `Pending`, o `OOMKilled` (o código 137 do [capítulo de arquitetura do Docker](../../02-containers/docker/01_arquitetura_e_isolamento.md)) e a CPU desacelerada pelo limit (o *throttling* do [capítulo de workloads](../../02-containers/kubernetes/02_workloads_rede_e_escala.md)). A diferença é que, como métricas, elas têm **histórico** e podem gerar **painéis e alertas** para o cluster inteiro.

Repare em `container_memory_working_set_bytes`: é essa a medida de memória que o kubelet compara com o limit para decidir um `OOMKilled`, e por isso é a que importa (e não o total de memória usada, que inclui cache que o kernel pode liberar).

---

## 3. O kube-prometheus-stack

### 3.1 Tudo de uma vez, com Helm

Montar Prometheus, Alertmanager, Grafana, node-exporter e kube-state-metrics à mão, com as permissões certas e os painéis de cada um, é muito trabalho. O chart Helm **kube-prometheus-stack** (citado no [capítulo do Compose ao cluster](../../02-containers/kubernetes/04_do_compose_ao_cluster.md)) instala tudo junto, já configurado:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoramento prometheus-community/kube-prometheus-stack \
  --namespace monitoramento --create-namespace
```

| O que é instalado | Para quê |
|---|---|
| **Prometheus Operator** | Um controller que gerencia o Prometheus a partir de objetos do Kubernetes (seção 3.2) |
| **Prometheus** e **Alertmanager** | As peças do [capítulo de SLOs](02_slos_e_alertas.md), rodando no cluster |
| **Grafana** | Já conectado ao Prometheus, com dezenas de painéis prontos (cluster, nodes, namespaces, Pods) |
| **node-exporter** | Um DaemonSet: um Pod por node |
| **kube-state-metrics** | Um Deployment que lê o estado dos objetos |
| **Regras de alerta prontas** | Para os problemas comuns do cluster: node sem memória, Pod em *CrashLoopBackOff*, disco enchendo, componentes do control plane fora do ar |

Os alertas prontos são, em sua maioria, **alertas de causa** (seção 4 do [capítulo de SLOs](02_slos_e_alertas.md)). Eles são úteis para quem opera o cluster, mas vale revisá-los: alguns são encaminhados como chamado, e não como página, justamente para evitar a fadiga de alertas.

### 3.2 Configuração como objetos: `ServiceMonitor` e `PrometheusRule`

Com o Prometheus Operator, a configuração do Prometheus deixa de ser um arquivo editado à mão e passa a ser feita com **objetos do Kubernetes** (os *Custom Resources* que o operator instala), aplicados com `kubectl apply` como qualquer outro manifesto. É o loop de reconciliação do [capítulo de fundamentos do Kubernetes](../../02-containers/kubernetes/01_fundamentos_e_arquitetura.md), aplicado ao monitoramento.

Um **`ServiceMonitor`** diz ao Prometheus para raspar os Pods por trás de um Service, escolhido por labels:

```yaml
# servicemonitor-otel-collector.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: otel-collector
  namespace: observabilidade
  labels:
    release: monitoramento          # ← o Prometheus do chart só enxerga ServiceMonitors com esta label
spec:
  selector:
    matchLabels:
      app: otel-collector           # os Services com esta label...
  endpoints:
    - port: prometheus              # ...na porta com este nome (a 8889 do exporter prometheus)
      interval: 15s
```

Um **`PrometheusRule`** carrega regras de gravação e de alerta, como as do [capítulo de SLOs](02_slos_e_alertas.md):

```yaml
# prometheusrule-slo-checkout.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: slo-checkout
  namespace: loja
  labels:
    release: monitoramento
spec:
  groups:
    - name: slo-checkout-sli
      rules:
        - record: loja:checkout_erros:taxa5m
          expr: |
            sum(rate(http_server_request_duration_seconds_count{service_name="loja", http_route="/checkout", http_response_status_code=~"5.."}[5m]))
            /
            sum(rate(http_server_request_duration_seconds_count{service_name="loja", http_route="/checkout"}[5m]))
        # ... as demais janelas e os alertas do capítulo de SLOs
```

Com isso, as regras de alerta de um serviço vivem **no repositório do serviço**, ao lado dos manifestos dele, e são aplicadas pelo mesmo pipeline (ou pelo mesmo Argo CD do [pipeline de CD](../../03-entrega-continua/cd/03_pipeline_de_cd_na_pratica.md)). Quem muda o serviço, muda os alertas dele.

> **Confusão comum:** "criei o `ServiceMonitor`, o `kubectl apply` não deu erro, mas o alvo não aparece no Prometheus". ✅ **Mais preciso:** o `ServiceMonitor` é só uma declaração, e o Prometheus Operator escolhe **quais** declarações levar em conta por um seletor de labels. No kube-prometheus-stack, por padrão, o Prometheus só considera `ServiceMonitor`s e `PrometheusRule`s que tenham a label `release` com o nome da instalação do Helm (no exemplo, `release: monitoramento`). Sem ela, o objeto é criado sem erro e ignorado em silêncio, o mesmo comportamento do Ingress sem controller do [capítulo de workloads](../../02-containers/kubernetes/02_workloads_rede_e_escala.md). Os outros dois motivos mais comuns: o `selector` do `ServiceMonitor` não casa com as labels do **Service** (e não do Pod), ou o nome em `port` não existe no Service. Em *Status → Targets* (ou *Service Discovery*), a interface do Prometheus mostra o que ele encontrou.

---

## 4. Logs no cluster

### 4.1 O caminho

O [capítulo de logs](../sinais/02_logs.md) mostrou que, no Kubernetes, os containers escrevem na saída padrão, o container runtime grava essas linhas em arquivos no node (`/var/log/pods/...`), e esses arquivos têm a vida do Pod. A coleta é feita por um **agente em cada node**, como um DaemonSet:

```
 ┌───────────────────────── node ─────────────────────────┐
 │                                                        │
 │  [Pod loja] [Pod pedidos] [Pod pagamentos]             │
 │      │ stdout    │ stdout       │ stdout               │
 │      ▼           ▼              ▼                      │
 │  /var/log/pods/loja_loja-7d9c..._<uid>/loja/0.log      │
 │  /var/log/pods/loja_pedidos-5f8d..._<uid>/pedidos/0.log│
 │                  │                                     │
 │       agente de logs (DaemonSet)                       │
 │       lê os arquivos, acrescenta namespace, Pod,       │
 │       container e labels, envia                        │
 └──────────────────┼─────────────────────────────────────┘
                    ▼
                  Loki
```

As opções de agente mais comuns:

| Agente | Observação |
|---|---|
| **Grafana Alloy** | O agente atual da Grafana (sucessor do Grafana Agent e do **Promtail**, que foi descontinuado); coleta logs, métricas e traces |
| **Fluent Bit** | Leve e muito usado, com destinos para quase tudo (Loki, Elasticsearch, CloudWatch) |
| **OpenTelemetry Collector** | Com o receiver `filelog`; tem a vantagem de usar o mesmo Collector e o mesmo processamento dos outros sinais |

### 4.2 Com o OpenTelemetry Collector

Um trecho da configuração do Collector rodando como DaemonSet:

```yaml
receivers:
  filelog:
    include: [/var/log/pods/*/*/*.log]      # os arquivos de log de todos os containers do node
    exclude: [/var/log/pods/observabilidade_*/*/*.log]   # não coleta os próprios logs
    include_file_path: true
    operators:
      - type: container                    # entende o formato do container runtime e extrai
                                           # namespace, Pod e container do caminho do arquivo

processors:
  k8sattributes:                           # consulta a API do cluster e acrescenta metadados
    extract:
      metadata: [k8s.namespace.name, k8s.pod.name, k8s.deployment.name, k8s.node.name]
      labels:
        - tag_name: app
          key: app
          from: pod
  memory_limiter:
    check_interval: 1s
    limit_percentage: 80
    spike_limit_percentage: 25
  batch: {}

service:
  pipelines:
    logs:
      receivers: [filelog]
      processors: [memory_limiter, k8sattributes, batch]
      exporters: [otlphttp/loki]
```

O DaemonSet precisa montar o diretório `/var/log/pods` do node (um volume do tipo `hostPath`) e de uma **ServiceAccount** com permissão de leitura de Pods, namespaces e ReplicaSets na API do cluster, porque é assim que o `k8sattributes` descobre a qual Deployment cada Pod pertence: o RBAC do [capítulo de configuração, estado e segurança](../../02-containers/kubernetes/03_configuracao_estado_e_seguranca.md), aplicado à telemetria.

Com os metadados, os logs de um Pod que já não existe continuam encontráveis pelo Deployment e pelo namespace:

```logql
{k8s_namespace_name="loja", k8s_deployment_name="pagamentos"} |= "recusado"
```

---

## 5. Traces e instrumentação com o OpenTelemetry Operator

### 5.1 Instrumentação sem alterar a imagem

No laboratório do Compose, a imagem dos serviços foi construída com o `opentelemetry-distro` e iniciada com `opentelemetry-instrument`. No Kubernetes, o **OpenTelemetry Operator** permite fazer a instrumentação automática **sem alterar a imagem**: ele injeta o SDK no Pod no momento da criação.

São dois objetos. Uma **`Instrumentation`** descreve como instrumentar:

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: padrao
  namespace: loja
spec:
  exporter:
    endpoint: http://otel-collector.observabilidade.svc:4318    # o Collector, pelo DNS do Service
  propagators: [tracecontext, baggage]
  sampler:
    type: parentbased_traceidratio
    argument: "1"                                                 # 100% na origem; a amostragem fica no Collector
  env:
    - name: OTEL_SEMCONV_STABILITY_OPT_IN
      value: http
```

E uma **anotação** no template do Pod pede a injeção:

```yaml
# no Deployment de cada serviço
spec:
  template:
    metadata:
      annotations:
        instrumentation.opentelemetry.io/inject-python: "true"
```

Ao criar o Pod, o operator acrescenta um *init container* que copia o SDK do OpenTelemetry para um volume compartilhado, e define no container da aplicação as variáveis (`OTEL_SERVICE_NAME` a partir do nome do Deployment, o endpoint, os propagadores, a amostragem) e o caminho para que o Python carregue a instrumentação. A aplicação passa a emitir traces, métricas e logs sem nenhuma mudança no Dockerfile. Há anotações equivalentes para Java, Node.js, .NET e Go.

O mesmo operator gerencia os próprios Collectors com um objeto **`OpenTelemetryCollector`**, que pode rodar como **Deployment** (o papel de gateway), **DaemonSet** (o papel de agente, como o da seção 4.2) ou **sidecar** (injetado em cada Pod), a topologia do [capítulo de telemetria](../fundamentos/02_telemetria_e_opentelemetry.md), seção 6.3.

### 5.2 Os metadados do Kubernetes nos traces

O processor `k8sattributes` da seção 4.2 também é usado nos pipelines de traces e métricas. Com ele, cada span e cada métrica ganham `k8s.pod.name`, `k8s.namespace.name` e `k8s.node.name`. Isso permite uma investigação que só existe em um cluster: "as requisições lentas estão todas em Pods do mesmo node?". Se estiverem, a causa provável está no node (um vizinho barulhento consumindo o disco ou a rede), e não na aplicação.

> **Confusão comum:** "com os metadados do Pod nas métricas, posso filtrar tudo por `k8s_pod_name`, então vale colocá-lo como rótulo em todas as métricas da aplicação". ✅ **Mais preciso:** o nome do Pod é um rótulo de **cardinalidade móvel**: ele tem poucos valores em um dado momento, mas valores **novos a cada deploy**, a cada escala e a cada reinício. Cada nome novo cria séries novas, e as antigas param de receber dados, mas continuam no banco até o fim da retenção. Em um serviço com 20 réplicas e vários deploys por dia, isso multiplica o número de séries ao longo do mês (o efeito tem nome, *churn* de séries). Para as métricas de negócio e os SLIs, que são agregados por serviço, o nome do Pod costuma ser desnecessário; para investigar um Pod específico, os traces e os logs já o carregam. Uma prática comum é manter `k8s.pod.name` nos traces e logs e removê-lo (ou nem acrescentá-lo) das métricas da aplicação, ficando com o Deployment e o namespace.

---

## 6. Eventos do Kubernetes

### 6.1 O sinal que costuma faltar

O Kubernetes registra **eventos** para tudo o que acontece com os objetos: um Pod agendado em um node, uma imagem baixada, um container morto por falta de memória, uma probe falhando, o HPA mudando o número de réplicas. São os eventos que o `kubectl describe` mostra no fim da saída, e que o [capítulo do Compose ao cluster](../../02-containers/kubernetes/04_do_compose_ao_cluster.md) usou para diagnosticar `Pending` e `CrashLoopBackOff`.

```bash
kubectl get events -n loja --sort-by=.lastTimestamp
# LAST SEEN  TYPE     REASON             OBJECT                         MESSAGE
# 2m         Warning  FailedScheduling   pod/pagamentos-5f8d9c7b6-2mxkq 0/3 nodes are available: 3 Insufficient memory.
# 40s        Warning  BackOff            pod/pedidos-7d9c8b6f5d-4xk2p   Back-off restarting failed container
# 12s        Normal   ScalingReplicaSet  deployment/loja                Scaled up replica set loja-6c9f7 to 5
```

O problema: por padrão, o cluster guarda os eventos por **apenas uma hora**. O Pod que reiniciou às 3h da manhã (o cenário 11 do [capítulo de escolha do sinal](../sinais/04_escolhendo_o_sinal_certo.md)) já não tem eventos quando a equipe chega às 9h.

### 6.2 Coletando os eventos

A solução é coletar os eventos continuamente e guardá-los como logs. O Collector faz isso com um receiver que observa a API de eventos (`k8s_events`, ou o mais geral `k8sobjects`); o Grafana Alloy e outras ferramentas têm componentes equivalentes. Como há um único fluxo de eventos para o cluster inteiro, esse receiver roda em **uma** réplica do Collector (no gateway, e não no DaemonSet, ou cada node enviaria os mesmos eventos).

```yaml
receivers:
  k8s_events:
    namespaces: [loja]          # ou todos, omitindo a lista
```

No Loki, os eventos viram consultáveis pelo período que a retenção permitir, e podem ser **marcados sobre os gráficos** do Grafana como anotações (cada `ScalingReplicaSet` ou `OOMKilling` aparece como uma linha vertical sobre o painel de latência), respondendo de imediato à pergunta "o que mudou neste momento?".

---

## 7. O roteiro de diagnóstico, com os sinais

O roteiro do [capítulo do Compose ao cluster](../../02-containers/kubernetes/04_do_compose_ao_cluster.md) usava `kubectl` para cada caso. Com a observabilidade montada, cada caso também pode ser investigado **no passado** e **para muitos Pods ao mesmo tempo**:

| Sintoma | Com `kubectl` (só o presente) | Com os sinais (presente e passado) |
|---|---|---|
| `Pending` | `kubectl describe pod` (eventos) | Métrica `kube_pod_status_phase{phase="Pending"}`; eventos `FailedScheduling` no Loki |
| `CrashLoopBackOff` | `kubectl logs --previous` | Logs de todas as execuções no Loki; `increase(kube_pod_container_status_restarts_total[1h])` |
| `OOMKilled` | `kubectl describe pod` (*Last State*) | `kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}`; o gráfico de memória em relação ao limit, mostrando a subida até o corte |
| Running, mas não Ready | `kubectl describe pod` (probe falhando) | Eventos `Unhealthy`; os logs e traces do endpoint de readiness |
| Lento, sem erros | (nada direto) | Traces com os metadados do Kubernetes: a lentidão se concentra em um node? Em uma versão? |
| Deployment com réplicas faltando | `kubectl get deploy` | `kube_deployment_spec_replicas - kube_deployment_status_replicas_available` |

---

## 8. O deploy canary guiado por métricas

O [capítulo de estratégias de deploy](../../03-entrega-continua/cd/02_estrategias_de_deploy.md) apresentou o canary com o **Argo Rollouts**, que avança a porcentagem do tráfego da versão nova passo a passo, e deixou em aberto o `AnalysisTemplate`, o objeto que decide se cada passo foi aprovado. Com o Prometheus no cluster, ele fica assim:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: taxa-de-erro-baixa
  namespace: loja
spec:
  args:
    - name: versao                          # a versão em análise
  metrics:
    - name: taxa-de-erro
      interval: 1m                          # mede a cada minuto...
      count: 5                              # ...cinco vezes
      failureLimit: 1                       # mais de uma medição reprovada = análise reprovada
      successCondition: result[0] < 0.01    # aprovado se a taxa de erro da versão nova < 1%
      provider:
        prometheus:
          address: http://monitoramento-kube-prometheus-prometheus.monitoramento.svc:9090
          query: |
            sum(rate(http_server_request_duration_seconds_count{service_name="loja", service_version="{{args.versao}}", http_response_status_code=~"5.."}[2m]))
            /
            sum(rate(http_server_request_duration_seconds_count{service_name="loja", service_version="{{args.versao}}"}[2m]))
```

E, no `Rollout` do serviço, a análise acompanha os passos do canary, recebendo a versão (a mesma tag da imagem, atualizada pelo pipeline, e também enviada como `service.version` no `OTEL_RESOURCE_ATTRIBUTES`):

```yaml
  strategy:
    canary:
      analysis:
        templates:
          - templateName: taxa-de-erro-baixa
        args:
          - name: versao
            value: "a1b2c3d"
      steps:
        - setWeight: 5
        - pause: { duration: 10m }
        - setWeight: 25
        - pause: { duration: 10m }
        - setWeight: 50
        - pause: { duration: 10m }
```

Se a taxa de erro da versão nova passar de 1% em mais de uma medição, o Argo Rollouts interrompe o canary e devolve todo o tráfego para a versão estável, sem ninguém intervir. A consulta é o SLI de disponibilidade do [capítulo de SLOs](02_slos_e_alertas.md), filtrado pela versão. É o ponto em que as duas últimas partes da trilha se encontram: **a observabilidade decide se a entrega continua**.

O mesmo vale para a latência (uma segunda métrica no template, com o `histogram_quantile` do [capítulo de métricas](../sinais/01_metricas.md)) e para métricas de negócio: um canary que mantém a taxa de erro HTTP baixa, mas derruba a taxa de pagamentos aprovados, também deve ser revertido.

---

## 9. Na AWS: as opções gerenciadas

No EKS, cada peça desta pasta tem uma alternativa gerenciada:

| Peça | Gerenciado na AWS |
|---|---|
| Prometheus | **Amazon Managed Service for Prometheus** (compatível com PromQL; o Prometheus ou o Collector do cluster envia as métricas para ele) |
| Grafana | **Amazon Managed Grafana** |
| Métricas e logs do cluster e dos containers | **CloudWatch Container Insights** |
| Logs | **CloudWatch Logs** |
| Traces e mapa de serviços | **AWS X-Ray** e o **CloudWatch Application Signals** |
| Collector | **AWS Distro for OpenTelemetry** (ADOT): uma distribuição do Collector e dos SDKs mantida pela AWS, com os exporters para os serviços acima |

O OpenTelemetry mantém a instrumentação igual: as aplicações continuam enviando OTLP para um Collector, e só os **exporters** do Collector mudam. As credenciais seguem o [capítulo de IAM](../../01-infraestrutura/aws/03_iam_identidades_e_permissoes.md): o Collector usa uma **IAM role** atribuída à sua ServiceAccount pelo **Pod Identity**, com permissão só para escrever no workspace do Prometheus gerenciado e nos grupos de logs da aplicação. Nenhuma chave de acesso fica guardada no cluster.

A escolha entre gerenciado e próprio é a mesma do [capítulo de fundamentos de infraestrutura](../../01-infraestrutura/fundamentos/01_o_que_e_infraestrutura.md): os serviços gerenciados trocam controle e, às vezes, custo, por não precisar operar bancos de séries temporais, de logs e de traces, que estão entre os sistemas mais trabalhosos de manter em alta disponibilidade.

---

## 10. Cuidados de custo e escala

A observabilidade de um cluster pode custar caro rápido. Os pontos que mais pesam:

- **Séries demais.** O cAdvisor e o kube-state-metrics expõem centenas de métricas, e muitas nunca são consultadas. O Prometheus permite **descartar** métricas e rótulos na coleta (com `metricRelabelings` no `ServiceMonitor`), e vale fazer isso para as mais volumosas que ninguém usa.
- **Churn de séries** pelos nomes de Pods (a confusão da seção 5.2).
- **Logs de componentes ruidosos.** Alguns componentes do próprio cluster (controllers, sidecars de service mesh) registram muito em nível `INFO`. Filtrar ou amostrar esses fluxos no agente evita pagar para guardar o que ninguém lê.
- **Traces sem amostragem.** Em um cluster com muito tráfego, o *tail sampling* no gateway do Collector ([capítulo de traces](../sinais/03_traces.md), seção 5) é o que mantém o custo sob controle.
- **Retenção por sinal.** Métricas agregadas por meses, logs e traces por dias, com os logs de auditoria em um caminho próprio.

---

## Fim da trilha

Com este capítulo, o repositório percorre o caminho inteiro de uma aplicação, do hardware ao incidente das 3h da manhã:

1. **Infraestrutura**: o que sustenta o software, e a nuvem da AWS em que ele roda (EC2, VPC, IAM).
2. **Containers**: o empacotamento com Docker e a orquestração com Kubernetes.
3. **Entrega contínua**: cada mudança verificada pelo CI, implantada pelo CD com estratégias seguras e rollback automático, no GitHub Actions ou no Jenkins.
4. **Observabilidade**: o sistema emitindo métricas, logs e traces conectados entre si; SLOs que definem o que é funcionar bem; alertas que só tocam quando os usuários são afetados; e as próprias métricas decidindo se um deploy continua.

As quatro partes se fecham em um ciclo: o código é entregue continuamente, a observabilidade mostra como ele se comporta em produção, e o que ela mostra (um SLO em risco, um canary reprovado, um incidente) volta como informação para as próximas mudanças. É esse ciclo, mais do que qualquer ferramenta, que a palavra **DevOps** descreve.

Para revisar a trilha completa e os caminhos mais curtos por objetivo, volte ao [README](../../README.md).
