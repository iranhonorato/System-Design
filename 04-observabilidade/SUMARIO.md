# Sumário: Observabilidade

Esta pasta trata de como saber se o sistema está funcionando bem em produção, e de como descobrir o porquê quando não está. As partes anteriores do repositório ensinaram a construir, empacotar, hospedar e entregar a aplicação; aqui, ela passa a **contar o que acontece dentro dela**, por meio de métricas, logs e traces, e a equipe aprende a escolher o sinal certo para cada pergunta. A pasta tem três partes:

- **`fundamentos/`**: o problema, a diferença entre monitorar e observar, os sinais, e como o código passa a emiti-los com o **OpenTelemetry**.
- **`sinais/`**: cada sinal por dentro (métricas, logs, traces) e um capítulo dedicado a **escolher o sinal certo** para cada pergunta, com cenários para praticar.
- **`pratica/`**: a pilha completa com Docker Compose, SLOs e alertas, e a observabilidade de um cluster Kubernetes.

---

## Trilha de Leitura

```
pré-requisitos: ../02-containers/ (e, de preferência, ../03-entrega-continua/)
         │
         ▼
FUNDAMENTOS
  ① fundamentos/01_monitoramento_e_observabilidade.md
  ② fundamentos/02_telemetria_e_opentelemetry.md
         │
         ▼
OS SINAIS
  ③ sinais/01_metricas.md
  ④ sinais/02_logs.md
  ⑤ sinais/03_traces.md
  ⑥ sinais/04_escolhendo_o_sinal_certo.md
         │
         ▼
NA PRÁTICA
  ⑦ pratica/01_stack_de_observabilidade.md
  ⑧ pratica/02_slos_e_alertas.md
  ⑨ pratica/03_observabilidade_no_kubernetes.md
```

Todos os capítulos usam a **mesma loja**, agora dividida em três microsserviços (`loja` → `pedidos` → `pagamentos`), em que uma chamada à operadora de cartão às vezes é lenta e às vezes falha. É esse sistema que gera os gráficos, os traces e os logs dos exemplos, e que roda de verdade no laboratório do capítulo ⑦.

### As duas perguntas que guiam a pasta

| Pergunta | Sinal | Onde é respondida |
|---|---|---|
| Uma equipe quer saber **quantas requisições falharam em cada minuto** e observar a **tendência ao longo do dia**. Qual sinal é mais adequado para essa visão agregada? | **Métricas** (um contador consultado com `increase()` por minuto) | ③, seção 4.3; discutida em ⑥, seção 2.1; no Grafana em ⑦, seção 9.1 |
| Uma requisição passa por **três microsserviços** e demora muito. A equipe quer localizar **em qual chamada ocorreu a maior demora**. Que dado ajuda mais diretamente? | **Traces** (os spans da requisição e o tempo próprio de cada um) | ⑤, seção 2; discutida em ⑥, seção 2.2; no Grafana em ⑦, seção 9.3 |

---

## ① [fundamentos/01_monitoramento_e_observabilidade.md](fundamentos/01_monitoramento_e_observabilidade.md)

**Pré-requisito:** a parte de containers.

Por que um sistema com vários serviços não pode mais ser investigado entrando nas máquinas. A diferença entre **monitoramento** (vigiar falhas previstas) e **observabilidade** (entender as imprevistas com os dados existentes). Os sinais (**métricas**, **logs**, **traces**, além de eventos e profiles), com uma analogia e uma tabela comparativa. Como os sinais se completam em uma investigação, as três categorias de métricas de Alex Xu, o caminho da telemetria (instrumentar, coletar, armazenar, consultar) e de quem é a responsabilidade.

> **Problema que deixa em aberto:** como o código passa a emitir esses sinais sem ficar preso a um fornecedor?

---

## ② [fundamentos/02_telemetria_e_opentelemetry.md](fundamentos/02_telemetria_e_opentelemetry.md)

**Pré-requisito:** ①

O problema da instrumentação presa a um fornecedor e o **OpenTelemetry** como padrão: API, SDK, bibliotecas de instrumentação, o protocolo **OTLP** e as convenções semânticas. Instrumentação **automática** (`opentelemetry-instrument` e as variáveis `OTEL_*`) e **manual** (spans e métricas do domínio). O **recurso** (`service.name`, `service.version`). O **Collector**: por que ele existe, receivers, processors e exporters, e os papéis de agente e gateway.

> **Problema que deixa em aberto:** como cada sinal funciona por dentro?

---

## ③ [sinais/01_metricas.md](sinais/01_metricas.md)

**Pré-requisito:** ②

Séries temporais, os tipos (**contador**, **medidor**, **histograma**) e os instrumentos do OpenTelemetry. O modelo do **Prometheus** (coleta por *pull*, convenções de nomes). **PromQL**: seleção, `rate` e `increase`, agregações, e a consulta de **falhas por minuto ao longo do dia**. Por que a **média engana** e como calcular **percentis** com histogramas. O que medir: **sinais de ouro**, **RED** e **USE**. **Cardinalidade** e por que identificadores não podem ser rótulos. Exemplares, a ponte para os traces.

> **Problema que deixa em aberto:** as métricas dizem que há falhas, mas não o que aconteceu em cada uma.

---

## ④ [sinais/02_logs.md](sinais/02_logs.md)

**Pré-requisito:** ③

Logs **estruturados** e por que a mensagem deve ser fixa e os detalhes, campos. **Níveis** e quando usar cada um. **Correlação** com o `trace_id` entre serviços. Para onde vão os logs de um container (a saída padrão, os arquivos do node, os agentes) e por que o `kubectl logs` não basta. **Loki** vs Elasticsearch, a **LogQL**, e por que contar logs não substitui uma métrica. O que **nunca** registrar (segredos, cartões, dados pessoais) e como controlar o custo.

> **Problema que deixa em aberto:** os logs não mostram onde, entre vários serviços, o tempo foi gasto.

---

## ⑤ [sinais/03_traces.md](sinais/03_traces.md)

**Pré-requisito:** ④

**Traces** e **spans** (atributos, eventos, status, tipos). Como ler uma **cascata** e encontrar a chamada mais lenta pelo **tempo próprio**, e os padrões que ela revela (N+1, chamadas em série). A **propagação de contexto** com o cabeçalho `traceparent`, onde ela quebra (filas, threads, proxies) e como propagar à mão. **Spans do domínio**. **Amostragem** na origem e no final (*tail sampling*). Jaeger, Tempo e **TraceQL**, mapa de serviços e métricas de spans.

> **Problema que deixa em aberto:** diante de uma pergunta concreta, qual sinal consultar primeiro?

---

## ⑥ [sinais/04_escolhendo_o_sinal_certo.md](sinais/04_escolhendo_o_sinal_certo.md)

**Pré-requisito:** ③, ④, ⑤

Os **critérios** para escolher o sinal, em tabela e em árvore de decisão. As **duas perguntas clássicas** resolvidas, com o motivo pelo qual os outros sinais respondem pior. **Quatorze cenários** para praticar, com as respostas escondidas (pedido que falhou, SLO semanal, vazamento de memória, alertas, CPU alta, deploy suspeito, mapa de dependências, bandeira de cartão, clientes afetados, pool esgotado, Pod reiniciado, mensagem não consumida, comparação entre semanas, stack trace). O **fluxo de investigação** métrica → trace → log em um incidente, e os erros mais comuns.

> **Problema que deixa em aberto:** ver tudo isso funcionando.

---

## ⑦ [pratica/01_stack_de_observabilidade.md](pratica/01_stack_de_observabilidade.md)

**Pré-requisito:** ⑥ e [../02-containers/docker/04_compose_e_producao.md](../02-containers/docker/04_compose_e_producao.md)

Um laboratório completo com **Docker Compose**: os três microsserviços instrumentados com OpenTelemetry, o **Collector**, **Prometheus**, **Loki**, **Tempo** e **Grafana**, com as fontes de dados e as **ligações entre os sinais** já configuradas. As duas perguntas respondidas em painéis reais, um painel **RED**, a navegação métrica → trace → log, **experimentos** (derrubar um serviço, quebrar a propagação, medir a cardinalidade) e o que muda do laboratório para a produção.

> **Problema que deixa em aberto:** definir o que é "funcionar bem" e ser avisado quando deixar de ser.

---

## ⑧ [pratica/02_slos_e_alertas.md](pratica/02_slos_e_alertas.md)

**Pré-requisito:** ⑦

**SLI**, **SLO** e **SLA**, e como escrever SLIs em PromQL. O que cada "nove" significa e por que 100% é o objetivo errado. O **orçamento de erro** e sua política. Os princípios de um bom alerta (sintomas, não causas). A **taxa de consumo** e os alertas em múltiplas janelas. **Regras de gravação e de alerta** no Prometheus, o **Alertmanager** (agrupamento, encaminhamento, inibição, silêncios), plantão, runbooks, revisões sem culpados, alertas de antecipação (`predict_linear`, `absent`, *Watchdog*) e a ligação com a entrega contínua.

> **Problema que deixa em aberto:** levar tudo isso para um cluster Kubernetes.

---

## ⑨ [pratica/03_observabilidade_no_kubernetes.md](pratica/03_observabilidade_no_kubernetes.md)

**Pré-requisito:** ⑧ e a parte de Kubernetes.

As **camadas** de um cluster e de onde vem cada sinal (node-exporter, kube-state-metrics, cAdvisor, eventos, OpenTelemetry), e por que o `metrics-server` não é monitoramento. As consultas mais usadas (memória em relação ao limit, *throttling*, reinícios, `OOMKilled`, Pods `Pending`). O **kube-prometheus-stack**, `ServiceMonitor` e `PrometheusRule`. **Logs** com um agente por node, **traces** com o **OpenTelemetry Operator** e os metadados do Kubernetes, os **eventos** do cluster, o roteiro de diagnóstico com os sinais, o **deploy canary guiado por métricas** com Argo Rollouts, as opções gerenciadas da AWS e os cuidados de custo.

> **Fim da trilha.** Volte ao [README](../README.md) para a visão completa do repositório.

---

## Referência Rápida por Assunto

| Quero entender... | Leia |
|---|---|
| A diferença entre monitoramento e observabilidade | ① |
| O que são métricas, logs e traces, e para que serve cada um | ①, ⑥ |
| O que é OpenTelemetry e o que ele não é | ② |
| Instrumentação automática e manual em Python | ② |
| O OpenTelemetry Collector (receivers, processors, exporters) | ② |
| Contador, medidor e histograma | ③ |
| Contar falhas por minuto e ver a tendência do dia (PromQL) | ③, ⑦ |
| Por que a média engana; percentis p95 e p99 | ③ |
| Sinais de ouro, RED e USE | ③ |
| Cardinalidade e rótulos que derrubam o Prometheus | ③ |
| Logs estruturados e níveis de log | ④ |
| Correlacionar logs de vários serviços (`trace_id`) | ④ |
| Logs em containers e no Kubernetes | ④, ⑨ |
| LogQL e Loki | ④, ⑦ |
| O que não registrar nos logs (segredos, LGPD) | ④ |
| Traces, spans e como ler uma cascata | ⑤ |
| Encontrar a chamada mais lenta entre vários serviços | ⑤, ⑦ |
| Propagação de contexto (`traceparent`) e filas | ⑤ |
| Amostragem de traces (*head* e *tail sampling*) | ⑤ |
| TraceQL e Tempo | ⑤, ⑦ |
| Qual sinal usar para cada pergunta | ⑥ |
| Ir de uma métrica a um trace e aos logs | ⑥, ⑦ |
| Montar Prometheus, Loki, Tempo e Grafana com Compose | ⑦ |
| SLI, SLO, SLA e orçamento de erro | ⑧ |
| Alertas que não geram fadiga (taxa de consumo) | ⑧ |
| Alertmanager | ⑧ |
| Métricas do cluster (OOMKilled, throttling, Pending) | ⑨ |
| kube-prometheus-stack, ServiceMonitor e PrometheusRule | ⑨ |
| OpenTelemetry Operator e instrumentação sem mudar a imagem | ⑨ |
| Eventos do Kubernetes | ⑨ |
| Deploy canary decidido por métricas | ⑨ |
| Observabilidade gerenciada na AWS | ⑨ |
