# Telemetria e OpenTelemetry

**Pré-requisito:** [01_monitoramento_e_observabilidade.md](01_monitoramento_e_observabilidade.md)

> Segue a documentação oficial do OpenTelemetry (*Concepts*, *Specification*, *Semantic Conventions*, *Collector* e *Python zero-code instrumentation*). O argumento de padronizar as preocupações operacionais entre equipes vem do Capítulo 17 de **Fundamentals of Software Architecture** e do Capítulo 8 de **Software Architecture: The Hard Parts**.

---

## Por que este arquivo existe

O capítulo anterior mostrou que observabilidade é uma propriedade construída **no código**: o sistema só pode ser entendido a partir dos sinais que ele emite. Este capítulo trata de **como** o código passa a emitir esses sinais, e de um problema que acompanhou a área por anos: cada ferramenta tinha sua própria biblioteca, seu próprio agente e seu próprio formato.

A pergunta central é: *como instrumentar uma aplicação uma única vez, de forma que métricas, logs e traces saiam em um formato padrão, cheguem a qualquer ferramenta e possam ser relacionados entre si?* A resposta é o **OpenTelemetry**, e este capítulo apresenta suas peças antes que os capítulos de sinais as usem.

---

## 1. O problema: instrumentação presa a um fornecedor

Imagine a loja, há alguns anos. A equipe escolheu um fornecedor de monitoramento, a "Ferramenta A", e instrumentou os três serviços com a biblioteca dela:

```python
import ferramenta_a                       # biblioteca proprietária

ferramenta_a.iniciar(chave="...")
with ferramenta_a.rastrear("chamar_operadora"):
    ...
ferramenta_a.contador("pagamentos.aprovados").incrementar()
```

Dois anos depois, o contrato fica caro e a empresa quer migrar para a "Ferramenta B". Isso significa **reescrever a instrumentação de todos os serviços**, cada `rastrear` e cada `contador`, com a biblioteca da Ferramenta B, testar tudo de novo e aceitar que os dados antigos e os novos não se comparam. Com dezenas de serviços, a migração vira um projeto de meses, e a ferramenta deixa de ser uma escolha para virar uma prisão.

Havia um segundo problema, mais sutil: **as bibliotecas de terceiros** (o framework web, o cliente HTTP, o driver do banco) não tinham como emitir telemetria sem escolher um fornecedor. Ou cada uma suportava todos os fornecedores, ou nenhum.

---

## 2. O que é o OpenTelemetry

O **OpenTelemetry** (abreviado **OTel**) é um projeto de código aberto, mantido pela CNCF (a mesma fundação do Kubernetes), que define **um padrão único para gerar, coletar e transmitir telemetria**: traces, métricas e logs. Ele nasceu em 2019 da fusão de dois projetos que resolviam partes do problema (o OpenTracing e o OpenCensus) e hoje é suportado por praticamente todas as ferramentas de observabilidade, abertas e comerciais.

A ideia central é separar duas coisas que antes vinham juntas:

```
 ANTES                                         COM OPENTELEMETRY

 código ──▶ biblioteca da Ferramenta A          código ──▶ API do OpenTelemetry (padrão)
              │                                               │
              ▼                                               ▼
         Ferramenta A                                   SDK + OTLP (protocolo padrão)
                                                              │
 trocar de ferramenta = reescrever o código                   ▼
                                                    Collector ──▶ Ferramenta A
                                                              ├──▶ Ferramenta B
                                                              └──▶ Prometheus, Tempo, Loki...

                                                trocar de ferramenta = mudar a configuração
```

**O que o OpenTelemetry é:** um padrão (a especificação), um conjunto de bibliotecas para cada linguagem, um coletor e um protocolo.

**O que ele não é:** um lugar para **guardar** ou **visualizar** dados.

> **Confusão comum:** "instalamos o OpenTelemetry; onde ficam os painéis?". ✅ **Mais preciso:** o OpenTelemetry cuida das duas primeiras etapas do caminho da telemetria (**instrumentar** e **coletar**, do [capítulo anterior](01_monitoramento_e_observabilidade.md)), e só. Ele não tem banco de dados nem interface: os dados precisam ser **enviados** para algum lugar que os armazene e mostre, como Prometheus, Tempo, Loki e Grafana, Jaeger, ou uma plataforma comercial. Essa é justamente a vantagem: a instrumentação fica neutra, e o destino é uma decisão de configuração, que pode mudar sem tocar no código, ou até apontar para dois destinos ao mesmo tempo durante uma migração.

---

## 3. As peças

```
┌──────────────────────── dentro da aplicação ────────────────────────┐
│                                                                     │
│  seu código ──────────┐       bibliotecas de instrumentação         │
│  (spans e métricas    │       (FastAPI, httpx, psycopg, logging...) │
│   do domínio)         │                    │                        │
│                       ▼                    ▼                        │
│               ┌────────────────────────────────────┐                │
│               │  API do OpenTelemetry              │ ← interfaces   │
│               └────────────────┬───────────────────┘                │
│               ┌────────────────▼───────────────────┐                │
│               │  SDK: amostragem, processamento,   │ ← implementação│
│               │  recurso (service.name...), lotes  │                │
│               └────────────────┬───────────────────┘                │
│                                │ exportador OTLP                    │
└────────────────────────────────┼────────────────────────────────────┘
                                 ▼  OTLP (gRPC :4317 ou HTTP :4318)
                    ┌─────────────────────────┐
                    │  OpenTelemetry Collector │ ── receivers → processors → exporters
                    └─────────────────────────┘
                                 ▼
                     Prometheus · Tempo · Loki · ...
```

| Peça | O que é |
|---|---|
| **Especificação** | O documento que define o que é um span, uma métrica, um log, como o contexto é propagado, como cada linguagem deve se comportar |
| **API** | As interfaces que o código chama: "crie um span", "incremente este contador". Leve e estável. Bibliotecas de terceiros dependem **só da API** |
| **SDK** | A implementação da API: decide quais traces guardar (amostragem), agrupa os dados em lotes e os envia. Configurado pela aplicação, nunca pelas bibliotecas |
| **Bibliotecas de instrumentação** | Pacotes que instrumentam frameworks populares sem você escrever nada: FastAPI, Flask, Django, httpx, requests, psycopg, Redis, o módulo `logging`... |
| **OTLP** | O protocolo padrão de envio (*OpenTelemetry Protocol*), por gRPC (porta 4317) ou HTTP (porta 4318) |
| **Collector** | Um serviço que recebe telemetria, processa e envia para um ou mais destinos (seção 6) |
| **Convenções semânticas** | Os nomes padronizados de atributos e métricas (seção 5) |

A separação entre **API** e **SDK** resolve o segundo problema da seção 1. O FastAPI pode chamar a API do OpenTelemetry para criar spans; se a aplicação não configurar nenhum SDK, essas chamadas não fazem nada e custam quase zero. Se configurar, os spans aparecem, para o destino que a aplicação escolher. A biblioteca nunca precisa saber qual é.

---

## 4. Instrumentando uma aplicação Python

Há duas formas de instrumentar, e na prática se usam as duas juntas.

### 4.1 Instrumentação automática (*zero-code*)

Para Python, o pacote `opentelemetry-distro` traz o comando `opentelemetry-instrument`, que **inicia a aplicação com a instrumentação já ligada**, sem alterar uma linha de código:

```bash
pip install opentelemetry-distro opentelemetry-exporter-otlp
opentelemetry-bootstrap -a install      # detecta as bibliotecas instaladas (fastapi, httpx...)
                                        # e instala os pacotes de instrumentação correspondentes
```

```bash
# Em vez de:   uvicorn servicos.pagamentos:app --port 8000
OTEL_SERVICE_NAME=pagamentos \
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317 \
OTEL_SEMCONV_STABILITY_OPT_IN=http \
opentelemetry-instrument uvicorn servicos.pagamentos:app --port 8000
```

Só com isso, cada requisição recebida pelo FastAPI vira um **span**, cada chamada feita com o `httpx` vira outro span (e o contexto é **propagado** para o serviço chamado, como mostra o [capítulo de traces](../sinais/03_traces.md)), são geradas **métricas** de duração das requisições, e cada mensagem do módulo `logging` vira um **log** com o `trace_id` da requisição em que foi escrita.

A configuração é toda feita por **variáveis de ambiente** padronizadas pela especificação, iguais em todas as linguagens:

| Variável | Para quê | Exemplo |
|---|---|---|
| `OTEL_SERVICE_NAME` | O nome do serviço, presente em todos os sinais | `pagamentos` |
| `OTEL_RESOURCE_ATTRIBUTES` | Outros atributos do serviço | `service.version=1.4.2,deployment.environment.name=producao` |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | Para onde enviar | `http://otel-collector:4317` |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | gRPC ou HTTP | `grpc` ou `http/protobuf` |
| `OTEL_TRACES_EXPORTER` / `OTEL_METRICS_EXPORTER` / `OTEL_LOGS_EXPORTER` | Qual exportador usar para cada sinal | `otlp`, `console` (imprime na tela), `none` |
| `OTEL_TRACES_SAMPLER` / `OTEL_TRACES_SAMPLER_ARG` | Amostragem de traces ([capítulo de traces](../sinais/03_traces.md)) | `parentbased_traceidratio` / `0.1` |
| `OTEL_METRIC_EXPORT_INTERVAL` | A cada quantos milissegundos enviar as métricas | `10000` |
| `OTEL_SEMCONV_STABILITY_OPT_IN` | Usar a versão estável das convenções de HTTP (seção 5) | `http` |

Para experimentar sem nenhum backend, `OTEL_TRACES_EXPORTER=console` (e o mesmo para métricas e logs) imprime tudo na saída do processo, em JSON. É a forma mais rápida de ver o que a instrumentação está produzindo.

Repare que tudo isso combina com o que o [capítulo de entrega contínua](../../03-entrega-continua/cd/01_entrega_e_implantacao_continua.md) defendeu: **a mesma imagem** roda em todos os ambientes, e só as variáveis mudam. O destino da telemetria é configuração, como a URL do banco.

Um detalhe de versão: tutoriais mais antigos pedem a variável `OTEL_PYTHON_LOGGING_AUTO_INSTRUMENTATION_ENABLED=true` para que os logs sejam enviados. Nas versões atuais, a instrumentação do módulo `logging` faz isso por conta própria, e essa variável passou a ativar um caminho **depreciado**, com um aviso na inicialização. Não a use em projetos novos.

### 4.2 Instrumentação manual

A instrumentação automática enxerga a **parte técnica**: requisições HTTP, consultas ao banco, chamadas a outros serviços. Ela não sabe nada sobre o **domínio**: que existe uma etapa chamada "validar estoque", que a chamada à operadora de cartão é a parte delicada, que "pagamento aprovado" é um número que o negócio acompanha. Isso só quem escreve o código sabe, e é acrescentado com a API:

```python
import time

from opentelemetry import metrics, trace

tracer = trace.get_tracer("pagamentos")
meter = metrics.get_meter("pagamentos")

pagamentos_processados = meter.create_counter(
    "pagamentos.processados",
    unit="{pagamento}",
    description="Pagamentos processados, por resultado",
)
duracao_operadora = meter.create_histogram(
    "pagamentos.operadora.duracao",
    unit="s",
    description="Tempo de resposta da operadora de cartão",
)


async def processar(pagamento):
    with tracer.start_as_current_span("chamar_operadora") as span:   # um span do domínio
        span.set_attribute("pagamento.bandeira", pagamento.bandeira)
        inicio = time.monotonic()
        resultado = await operadora.autorizar(pagamento)
        duracao_operadora.record(time.monotonic() - inicio, {"bandeira": pagamento.bandeira})

    pagamentos_processados.add(1, {"resultado": resultado.status})
    return resultado
```

O span manual `chamar_operadora` aparece **dentro** do span automático da requisição, porque o SDK mantém o "span atual" do contexto. As métricas manuais saem pelo mesmo exportador das automáticas. O código só conversa com a **API**: trocar o destino continua sendo configuração.

> **Confusão comum:** "com a instrumentação automática, o sistema já está observável". ✅ **Mais preciso:** a instrumentação automática entrega uma base valiosa (toda requisição, toda chamada HTTP e toda consulta viram spans e métricas, de graça), mas vê o sistema **pelo lado técnico**. Ela mostra que `POST /pagamentos` levou 1,5 segundo, e não mostra que o tempo foi gasto na operadora, qual era a bandeira do cartão ou que o pagamento foi recusado por limite. Também não sabe que "pagamentos aprovados por minuto" é a métrica mais importante do serviço. Um sistema bem observável combina as duas: a automática para a cobertura técnica, e alguns spans, atributos e métricas manuais nos pontos que importam para o negócio. Esses pontos costumam ser poucos, e valem muito.

---

## 5. Recursos e convenções semânticas

### 5.1 O recurso: quem está emitindo

Todo sinal carrega um **recurso** (*resource*): o conjunto de atributos que descreve **quem** o emitiu. Ele é definido uma vez, na inicialização, e anexado automaticamente a cada span, métrica e log:

| Atributo | Exemplo | Para quê |
|---|---|---|
| `service.name` | `pagamentos` | **O mais importante**: é por ele que tudo é filtrado e correlacionado |
| `service.version` | `1.4.2` (ou o hash do commit) | Comparar a versão nova com a antiga, como no deploy canary |
| `deployment.environment.name` | `producao`, `homologacao` | Separar ambientes |
| `service.instance.id` | um identificador único da réplica | Distinguir réplicas |
| `k8s.pod.name`, `k8s.namespace.name`, `k8s.node.name` | `pagamentos-7d9c-4xk2p` | Acrescentados pelo Collector no Kubernetes ([capítulo de Kubernetes](../pratica/03_observabilidade_no_kubernetes.md)) |
| `cloud.region` | `sa-east-1` | Separar regiões |

O `service.version` merece destaque. Com ele, a pergunta "o deploy das 18h piorou a latência?" vira um filtro: compare `service.version="a1b2c3d"` com a versão anterior. É exatamente o que a análise de um deploy canary faz.

### 5.2 Convenções semânticas: os mesmos nomes em todo lugar

Se o serviço Python chama o atributo do código de resposta HTTP de `status`, o serviço Java chama de `http_status` e o Node chama de `responseCode`, nenhuma consulta funciona para os três, e nenhum painel pronto entende os dados. As **convenções semânticas** do OpenTelemetry definem os nomes padronizados:

| Domínio | Exemplos de atributos |
|---|---|
| **HTTP** | `http.request.method`, `http.route`, `http.response.status_code`, `url.path`, `server.address` |
| **Banco de dados** | atributos da família `db.*`: o sistema de banco, a operação, o texto da consulta |
| **Mensageria** | atributos da família `messaging.*`: o sistema (Kafka, RabbitMQ), o destino (tópico, fila), a operação |
| **Exceções** | `exception.type`, `exception.message`, `exception.stacktrace` |

E as métricas também têm nomes padronizados. A duração das requisições recebidas por um servidor HTTP é `http.server.request.duration`, em **segundos**, com os atributos `http.request.method`, `http.route` e `http.response.status_code`.

As convenções de HTTP passaram por uma revisão que as declarou estáveis, com nomes diferentes dos antigos. Por compatibilidade, algumas bibliotecas ainda emitem os nomes antigos por padrão (por exemplo, `http.server.duration`, em **milissegundos**), e é para isso que serve a variável `OTEL_SEMCONV_STABILITY_OPT_IN=http` da seção 4: ela liga os nomes estáveis. Usar os nomes estáveis desde o início evita painéis e alertas que quebram quando o padrão da biblioteca mudar.

---

## 6. O OpenTelemetry Collector

### 6.1 Por que um coletor no meio

A aplicação poderia enviar os dados direto para o Prometheus, o Tempo e o Loki. Na prática, quase sempre há um **Collector** no meio, pelas mesmas razões que justificam um proxy reverso na frente de uma API:

| Sem Collector (aplicação → backends) | Com Collector (aplicação → Collector → backends) |
|---|---|
| Cada aplicação conhece o endereço e as credenciais de cada backend | Só o Collector conhece; as aplicações enviam para um único endereço |
| Trocar de backend exige mudar a configuração de todas as aplicações | Muda-se só o Collector |
| Se o backend ficar fora do ar, a aplicação precisa guardar e reenviar | O Collector faz fila e novas tentativas |
| Enriquecer, filtrar ou amostrar acontece (ou não) em cada aplicação | É feito em um lugar só, igual para todos os serviços |
| Amostragem baseada no trace inteiro é impossível (cada serviço só vê sua parte) | É possível (*tail sampling*, no [capítulo de traces](../sinais/03_traces.md)) |

É a resposta concreta ao problema do acoplamento operacional do capítulo anterior: a parte comum da telemetria (para onde vai, como é filtrada, quem tem as credenciais) sai das aplicações e fica em um componente da plataforma.

### 6.2 Receivers, processors e exporters

O Collector é configurado como um conjunto de **pipelines**, um por sinal. Cada pipeline tem três etapas:

```
                 ┌──────────────────── pipeline de traces ────────────────────┐
  OTLP ────────▶ │ receiver ──▶ processor ──▶ processor ──▶ exporter ──────────┼──▶ Tempo
  (das apps)     │  otlp        memory_limiter  batch       otlp/tempo        │
                 └────────────────────────────────────────────────────────────┘
                 ┌──────────────────── pipeline de métricas ──────────────────┐
  OTLP ────────▶ │ otlp ──▶ memory_limiter ──▶ batch ──▶ prometheus (:8889) ──┼──▶ Prometheus
                 └────────────────────────────────────────────────────────────┘     (raspa)
                 ┌──────────────────── pipeline de logs ──────────────────────┐
  OTLP ────────▶ │ otlp ──▶ memory_limiter ──▶ batch ──▶ otlphttp/loki ───────┼──▶ Loki
                 └────────────────────────────────────────────────────────────┘
```

| Etapa | O que faz | Exemplos |
|---|---|---|
| **Receivers** | Recebem dados | `otlp` (das aplicações), `prometheus` (raspa endpoints), `filelog` (lê arquivos de log), `hostmetrics` |
| **Processors** | Transformam no caminho | `memory_limiter` (protege o Collector de estourar a memória), `batch` (agrupa em lotes), `resource` e `attributes` (acrescentam ou removem atributos), `k8sattributes` (acrescenta Pod, namespace e node), `filter`, `tail_sampling` |
| **Exporters** | Enviam para os destinos | `otlp` (Tempo e outros que falam OTLP), `otlphttp`, `prometheus` (expõe as métricas para o Prometheus raspar), `debug` (imprime no log do Collector) |
| **Connectors** | Ligam um pipeline a outro | `spanmetrics`: gera métricas de taxa, erros e duração **a partir dos spans** |

O arquivo de configuração descreve exatamente o desenho acima:

```yaml
# otel-collector.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  memory_limiter:            # sempre o primeiro: recusa dados antes de faltar memória
    check_interval: 1s
    limit_percentage: 80
    spike_limit_percentage: 25
  batch: {}                  # envia em lotes, em vez de um dado por vez

exporters:
  otlp/tempo:
    endpoint: tempo:4317
    tls:
      insecure: true         # ambiente de laboratório, sem TLS entre os containers
  prometheus:
    endpoint: 0.0.0.0:8889   # o Prometheus raspa as métricas aqui
    resource_to_telemetry_conversion:
      enabled: true          # transforma service.name etc. em rótulos das métricas
  otlphttp/loki:
    endpoint: http://loki:3100/otlp

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlp/tempo]
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [prometheus]
    logs:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlphttp/loki]
```

O nome depois da barra (`otlp/tempo`, `otlphttp/loki`) é só um apelido, que permite ter vários componentes do mesmo tipo. Esse arquivo é o mesmo usado na [parte prática](../pratica/01_stack_de_observabilidade.md).

O Collector é distribuído em versões com conjuntos diferentes de componentes. A imagem `otel/opentelemetry-collector-contrib` traz praticamente todos (inclusive o exporter `prometheus` e o processor `tail_sampling`), e é a usada nesta pasta. Em produção, é comum montar uma distribuição própria só com os componentes necessários, pelo mesmo motivo de evitar plugins desnecessários no [capítulo de fundamentos do Jenkins](../../03-entrega-continua/jenkins/01_fundamentos_do_jenkins.md).

### 6.3 Onde o Collector roda

Há dois papéis, que costumam ser combinados:

| Papel | Onde | Para quê |
|---|---|---|
| **Agente** | Perto da aplicação: um sidecar no mesmo Pod, ou um DaemonSet (um por node) | Receber dados com baixa latência; ler logs dos arquivos do node; acrescentar os dados do Pod e do node |
| **Gateway** | Um serviço central, com algumas réplicas | Ponto único de saída: credenciais dos backends, amostragem de traces completos, filtros globais |

```
  ┌── node 1 ─────────────┐   ┌── node 2 ─────────────┐
  │ [loja] [pedidos]      │   │ [pagamentos] [loja]   │
  │     │     │           │   │     │          │      │
  │  Collector (agente)   │   │  Collector (agente)   │
  └──────────┬────────────┘   └──────────┬────────────┘
             └─────────────┬─────────────┘
                   Collector (gateway) ×2  ──▶ backends
```

O agente como DaemonSet é a mesma ideia de agente por node do [capítulo de configuração, estado e segurança do Kubernetes](../../02-containers/kubernetes/03_configuracao_estado_e_seguranca.md) (seção 5.1), e o sidecar é o padrão discutido pelos livros de arquitetura no [capítulo anterior](01_monitoramento_e_observabilidade.md). O [capítulo de Kubernetes](../pratica/03_observabilidade_no_kubernetes.md) monta essa topologia.

---

## 7. O OpenTelemetry e as ferramentas existentes

O OpenTelemetry não substitui o Prometheus, o Loki ou o Jaeger: ele os **alimenta**. E convive com instrumentações mais antigas:

- **Prometheus:** aplicações que já expõem `/metrics` no formato do Prometheus continuam funcionando; o Collector pode raspá-las (receiver `prometheus`) e o próprio Prometheus, desde a versão 3, aceita receber métricas em OTLP.
- **Logs existentes:** aplicações que só escrevem logs na saída padrão continuam funcionando; um agente (o receiver `filelog` do Collector, o Fluent Bit ou o Grafana Alloy) lê esses logs dos arquivos do node, como mostra o [capítulo de logs](../sinais/02_logs.md).
- **Jaeger e Zipkin:** o Collector recebe esses formatos e os converte.

Isso permite adotar o OpenTelemetry **aos poucos**: começar pelos traces de um serviço novo, depois as métricas, e migrar o resto conforme os serviços forem sendo alterados.

---

## Próximo passo

Este capítulo mostrou como a telemetria é produzida e transportada: a API e o SDK na aplicação, a instrumentação automática e a manual, os recursos e as convenções semânticas, e o Collector no meio do caminho. Falta entender cada sinal por dentro: como é armazenado, como é consultado e que perguntas responde bem ou mal.

A parte de sinais começa pelo mais usado e mais barato: o [../sinais/01_metricas.md](../sinais/01_metricas.md) mostra os tipos de métrica, o modelo do Prometheus, a linguagem de consulta PromQL, os percentis e os métodos para escolher o que medir. E responde à primeira pergunta clássica: como contar as requisições que falharam em cada minuto.
