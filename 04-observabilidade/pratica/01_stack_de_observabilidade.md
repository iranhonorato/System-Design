# A Stack de Observabilidade na Prática

**Pré-requisito:** a parte de sinais, até [../sinais/04_escolhendo_o_sinal_certo.md](../sinais/04_escolhendo_o_sinal_certo.md), e [../../02-containers/docker/04_compose_e_producao.md](../../02-containers/docker/04_compose_e_producao.md)

> Segue a documentação do OpenTelemetry (Python e Collector), do Prometheus, do Grafana Loki (*OpenTelemetry ingestion*), do Grafana Tempo e do Grafana (*Provisioning*, *Exemplars*, *Trace to logs*). As versões das imagens estão fixadas em versões de referência; antes de usar em um projeto, confira as mais recentes.

---

## Por que este arquivo existe

A parte de sinais apresentou cada peça separadamente. Este capítulo as junta em um laboratório completo, que roda na sua máquina com um único `docker compose up`: os três microsserviços da loja instrumentados com OpenTelemetry, um Collector, um banco para cada sinal e o Grafana para consultar tudo.

A pergunta central é: *como montar, de ponta a ponta, uma pilha de observabilidade em que as duas perguntas clássicas sejam respondidas em painéis reais, e em que seja possível ir de um gráfico a um trace e de um trace aos logs com um clique?* O capítulo segue a ordem do caminho da telemetria do [capítulo de fundamentos](../fundamentos/01_monitoramento_e_observabilidade.md): instrumentar, coletar, armazenar, consultar.

---

## 1. O que vamos montar

```
 ┌──────────────── aplicação ────────────────┐
 │                                           │
 │  gerador de ──▶ loja ──▶ pedidos ──▶ pagamentos     (cada um com o SDK do OpenTelemetry)
 │  tráfego        :8000     :8000      :8000
 │                   │         │           │
 └───────────────────┼─────────┼───────────┼─┘
                     └─────────┼───────────┘  OTLP (gRPC :4317): traces, métricas e logs
                               ▼
                    ┌─────────────────────┐
                    │ OpenTelemetry       │
                    │ Collector           │
                    └──┬────────┬──────┬──┘
           métricas    │        │      │   logs
      (Prometheus raspa│:8889)  │      └──────────────┐
                       ▼        ▼ traces              ▼
                ┌──────────┐ ┌───────┐           ┌────────┐
                │Prometheus│ │ Tempo │           │  Loki  │
                │  :9090   │ │ :3200 │           │ :3100  │
                └─────┬────┘ └───┬───┘           └───┬────┘
                      └──────────┼───────────────────┘
                                 ▼
                          ┌────────────┐
                          │  Grafana   │ ◀── você, no navegador: http://localhost:3000
                          │   :3000    │
                          └────────────┘
```

| Componente | Papel | Capítulo onde foi apresentado |
|---|---|---|
| `loja`, `pedidos`, `pagamentos` | A aplicação, instrumentada com o OpenTelemetry | [Telemetria](../fundamentos/02_telemetria_e_opentelemetry.md) |
| `gerador-de-trafego` | Faz checkouts sem parar, para haver dados | |
| `otel-collector` | Recebe os três sinais e os envia a cada banco | [Telemetria](../fundamentos/02_telemetria_e_opentelemetry.md), seção 6 |
| `prometheus` | Guarda as métricas | [Métricas](../sinais/01_metricas.md) |
| `loki` | Guarda os logs | [Logs](../sinais/02_logs.md) |
| `tempo` | Guarda os traces | [Traces](../sinais/03_traces.md) |
| `grafana` | Consulta, painéis e a navegação entre sinais | [Escolhendo o sinal](../sinais/04_escolhendo_o_sinal_certo.md), seção 4 |

---

## 2. A estrutura do projeto

```
observabilidade-loja/
├── docker-compose.yml
├── servicos/
│   ├── Dockerfile
│   ├── requirements.txt
│   └── servicos/
│       ├── __init__.py
│       ├── loja.py
│       ├── pedidos.py
│       └── pagamentos.py
├── otel-collector.yaml
├── prometheus.yml
├── tempo.yaml
└── grafana/
    └── datasources.yaml
```

---

## 3. Os serviços

Os três serviços são versões enxutas da loja: a `loja` recebe o checkout e chama `pedidos`; `pedidos` valida o estoque e chama `pagamentos`; `pagamentos` "chama a operadora de cartão", que às vezes é lenta e às vezes recusa. Os defeitos são simulados de propósito, para que os painéis tenham o que mostrar.

### 3.1 `loja.py`

```python
import logging
import os

import httpx
from fastapi import FastAPI, HTTPException
from opentelemetry import metrics

PEDIDOS_URL = os.getenv("PEDIDOS_URL", "http://localhost:8001")

logging.basicConfig(level=logging.INFO)
log = logging.getLogger("loja")
meter = metrics.get_meter("loja")
checkouts = meter.create_counter(
    "loja.checkouts", unit="{checkout}", description="Checkouts finalizados, por resultado"
)

app = FastAPI()


@app.post("/checkout")
async def checkout(item: str = "caneca", quantidade: int = 1):
    async with httpx.AsyncClient(timeout=10) as cliente:
        resposta = await cliente.post(
            f"{PEDIDOS_URL}/pedidos", json={"item": item, "quantidade": quantidade}
        )
    if resposta.status_code >= 400:
        checkouts.add(1, {"resultado": "falha"})
        log.error("checkout falhou: pedidos respondeu %s", resposta.status_code)
        raise HTTPException(status_code=502, detail="falha ao criar o pedido")
    checkouts.add(1, {"resultado": "sucesso"})
    log.info("checkout concluído para %s", item)
    return resposta.json()
```

### 3.2 `pedidos.py`

```python
import asyncio
import logging
import os

import httpx
from fastapi import FastAPI, HTTPException
from opentelemetry import trace
from pydantic import BaseModel

PAGAMENTOS_URL = os.getenv("PAGAMENTOS_URL", "http://localhost:8002")

logging.basicConfig(level=logging.INFO)
log = logging.getLogger("pedidos")
tracer = trace.get_tracer("pedidos")

app = FastAPI()


class Pedido(BaseModel):
    item: str
    quantidade: int


@app.post("/pedidos", status_code=201)
async def criar_pedido(pedido: Pedido):
    with tracer.start_as_current_span("validar_estoque") as span:
        span.set_attribute("loja.item", pedido.item)
        await asyncio.sleep(0.02)  # simula uma consulta ao banco

    async with httpx.AsyncClient(timeout=10) as cliente:
        resposta = await cliente.post(
            f"{PAGAMENTOS_URL}/pagamentos", json={"valor": 49.9 * pedido.quantidade}
        )
    if resposta.status_code >= 400:
        log.warning("pagamento recusado para %s", pedido.item)
        raise HTTPException(status_code=402, detail="pagamento recusado")
    log.info("pedido criado para %s", pedido.item)
    return {"pedido": 123, "pagamento": resposta.json()}
```

### 3.3 `pagamentos.py`

```python
import asyncio
import logging
import random

from fastapi import FastAPI, HTTPException
from opentelemetry import trace
from pydantic import BaseModel

logging.basicConfig(level=logging.INFO)
log = logging.getLogger("pagamentos")
tracer = trace.get_tracer("pagamentos")

app = FastAPI()


class Pagamento(BaseModel):
    valor: float


@app.post("/pagamentos")
async def pagar(pagamento: Pagamento):
    bandeira = random.choice(["visa", "master", "elo"])
    with tracer.start_as_current_span("chamar_operadora") as span:
        span.set_attribute("pagamento.bandeira", bandeira)
        # A operadora é lenta em 5% das chamadas, mas em 30% das chamadas com "elo"
        lento = random.random() < (0.30 if bandeira == "elo" else 0.05)
        span.set_attribute("operadora.lenta", lento)
        await asyncio.sleep(1.5 if lento else 0.05)
        if random.random() < 0.05:  # 5% são recusadas
            span.set_status(trace.StatusCode.ERROR, "recusado pela operadora")
            log.error("operadora recusou o pagamento de %.2f (%s)", pagamento.valor, bandeira)
            raise HTTPException(status_code=502, detail="operadora indisponível")
    log.info("pagamento de %.2f aprovado (%s)", pagamento.valor, bandeira)
    return {"status": "aprovado"}
```

Repare no que **não** está no código: nenhum endereço de Collector, nenhuma configuração de exportação, nenhuma propagação manual de contexto. O código só usa a **API** do OpenTelemetry nos pontos de domínio (o contador de checkouts, os spans `validar_estoque` e `chamar_operadora`, o atributo `pagamento.bandeira`). Todo o resto vem da instrumentação automática, configurada por variáveis de ambiente, como no [capítulo de telemetria](../fundamentos/02_telemetria_e_opentelemetry.md).

Repare também em como **o mesmo problema** aparece em cada serviço: uma recusa da operadora é um `502` em `pagamentos`, vira um `402` em `pedidos` (que a trata como "pagamento recusado") e volta a ser um `502` na `loja`. Quem olha só as métricas de `pedidos` filtrando por `5xx` não vê o problema. É um exemplo pequeno de por que vale olhar o sistema pela borda (a `loja`, onde o cliente está) e de por que os traces, que mostram a cadeia inteira, ajudam tanto.

### 3.4 A imagem

Uma única imagem serve aos três serviços; a variável `SERVICO` escolhe qual módulo executar.

```text
# servicos/requirements.txt
fastapi==0.162.0
uvicorn==0.54.0
httpx==0.28.1
opentelemetry-distro==0.66b1
opentelemetry-exporter-otlp==1.45.1
opentelemetry-instrumentation-fastapi==0.66b1
opentelemetry-instrumentation-httpx==0.66b1
opentelemetry-instrumentation-logging==0.66b1
```

```dockerfile
# servicos/Dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY servicos/ ./servicos/
RUN useradd --create-home app
USER app
# "exec" faz o uvicorn substituir o shell e ser o PID 1 (recebe o SIGTERM do docker stop)
CMD ["sh", "-c", "exec opentelemetry-instrument uvicorn servicos.${SERVICO}:app --host 0.0.0.0 --port 8000"]
```

O `CMD` usa a forma *shell* (`sh -c`) para que a variável `${SERVICO}` seja expandida, e o `exec` resolve o problema dos sinais que o [capítulo de imagens](../../02-containers/docker/02_imagens_e_dockerfile.md) descreveu para essa forma: o shell é substituído pelo processo da aplicação, que passa a ser o PID 1 e recebe o `SIGTERM` diretamente.

---

## 4. O Collector

É o mesmo arquivo do [capítulo de telemetria](../fundamentos/02_telemetria_e_opentelemetry.md), com uma opção a mais no exporter do Prometheus, que liga os **exemplares**:

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
  memory_limiter:
    check_interval: 1s
    limit_percentage: 80
    spike_limit_percentage: 25
  batch: {}

exporters:
  otlp/tempo:
    endpoint: tempo:4317
    tls:
      insecure: true
  prometheus:
    endpoint: 0.0.0.0:8889
    enable_open_metrics: true          # expõe os exemplares (trace_id) junto com as métricas
    resource_to_telemetry_conversion:
      enabled: true                    # service.name vira o rótulo service_name
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

---

## 5. Os bancos de cada sinal

### 5.1 Prometheus

O Prometheus raspa as métricas que o Collector expõe na porta 8889 (e as dele mesmo):

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: otel-collector
    static_configs:
      - targets: ["otel-collector:8889"]

  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]
```

No Compose (seção 7), ele é iniciado com `--enable-feature=exemplar-storage`, para guardar os exemplares que o Collector envia.

### 5.2 Tempo

```yaml
# tempo.yaml
server:
  http_listen_port: 3200

distributor:
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317

storage:
  trace:
    backend: local                 # em produção: s3, gcs ou azure
    local:
      path: /var/tempo/traces
    wal:
      path: /var/tempo/wal
```

O `endpoint: 0.0.0.0:4317` precisa estar explícito: as versões recentes dos componentes que recebem OTLP passaram a ouvir só em `localhost` por padrão, e, dentro de um container, `localhost` é o próprio container, a confusão do [capítulo de volumes e redes](../../02-containers/docker/03_volumes_e_redes.md). Sem o `0.0.0.0`, o Collector, em outro container, não conseguiria entregar os traces.

### 5.3 Loki

O Loki é usado com a configuração padrão da imagem, que já aceita logs em OTLP no caminho `/otlp` e guarda os atributos dos logs (como o `trace_id`) como **metadados estruturados**. Ao receber logs por OTLP, ele transforma alguns atributos do recurso em rótulos indexados, como o `service.name`, que vira o rótulo `service_name`. É assim que as consultas da seção 9 filtram por serviço.

---

## 6. O Grafana, já conectado aos três sinais

O Grafana pode ser configurado por arquivos (*provisioning*), em vez de cliques. Este arquivo cria as três fontes de dados **e as ligações entre elas**:

```yaml
# grafana/datasources.yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    uid: prometheus
    url: http://prometheus:9090
    isDefault: true
    jsonData:
      exemplarTraceIdDestinations:          # MÉTRICA → TRACE: exemplares clicáveis
        - name: trace_id
          datasourceUid: tempo

  - name: Tempo
    type: tempo
    uid: tempo
    url: http://tempo:3200
    jsonData:
      tracesToLogsV2:                       # TRACE → LOGS: o botão "Logs for this span"
        datasourceUid: loki
        spanStartTimeShift: "-5m"
        spanEndTimeShift: "5m"
        customQuery: true
        query: '{service_name="$${__span.tags["service.name"]}"} | trace_id="$${__trace.traceId}"'
      serviceMap:
        datasourceUid: prometheus

  - name: Loki
    type: loki
    uid: loki
    url: http://loki:3100
    jsonData:
      derivedFields:                        # LOG → TRACE: o trace_id vira um link
        - name: trace_id
          matcherType: label
          matcherRegex: trace_id
          datasourceUid: tempo
          url: "$${__value.raw}"
          urlDisplayLabel: "Ver trace"
```

Cada bloco de `jsonData` é uma das ligações da tabela do [capítulo de escolha do sinal](../sinais/04_escolhendo_o_sinal_certo.md), seção 4.2:

- **`exemplarTraceIdDestinations`**: os pontos de exemplar nos gráficos do Prometheus abrem o trace correspondente no Tempo.
- **`tracesToLogsV2`**: a partir de um span, abre no Loki os logs do mesmo serviço com o mesmo `trace_id`.
- **`derivedFields`**: em um log, o campo `trace_id` vira um link para o trace.

O `$$` é necessário porque o Grafana substitui `$VARIAVEL` nos arquivos de provisionamento por variáveis de ambiente. Com `$$`, ele mantém um `$` literal, que é a sintaxe das variáveis internas do Grafana (`${__trace.traceId}`).

---

## 7. O `docker-compose.yml`

```yaml
# docker-compose.yml
x-otel-comum: &otel-comum                  # variáveis comuns aos três serviços
  OTEL_EXPORTER_OTLP_ENDPOINT: http://otel-collector:4317
  OTEL_EXPORTER_OTLP_PROTOCOL: grpc
  OTEL_SEMCONV_STABILITY_OPT_IN: http
  OTEL_METRIC_EXPORT_INTERVAL: "10000"
  OTEL_RESOURCE_ATTRIBUTES: service.version=1.0.0,deployment.environment.name=laboratorio

services:
  # ───────────────────────── aplicação ─────────────────────────
  loja:
    build: ./servicos
    ports:
      - "8000:8000"
    environment:
      <<: *otel-comum
      SERVICO: loja
      OTEL_SERVICE_NAME: loja
      PEDIDOS_URL: http://pedidos:8000

  pedidos:
    build: ./servicos
    environment:
      <<: *otel-comum
      SERVICO: pedidos
      OTEL_SERVICE_NAME: pedidos
      PAGAMENTOS_URL: http://pagamentos:8000

  pagamentos:
    build: ./servicos
    environment:
      <<: *otel-comum
      SERVICO: pagamentos
      OTEL_SERVICE_NAME: pagamentos

  gerador-de-trafego:
    image: curlimages/curl:8.13.0
    entrypoint: ["/bin/sh", "-c"]
    command: ["while true; do curl -s -o /dev/null -X POST http://loja:8000/checkout; sleep 0.3; done"]
    depends_on: [loja]

  # ───────────────────────── telemetria ─────────────────────────
  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.128.0
    command: ["--config=/etc/otelcol/config.yaml"]
    volumes:
      - ./otel-collector.yaml:/etc/otelcol/config.yaml:ro

  prometheus:
    image: prom/prometheus:v3.5.0
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --enable-feature=exemplar-storage
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
    ports:
      - "9090:9090"

  tempo:
    image: grafana/tempo:2.8.0
    command: ["-config.file=/etc/tempo.yaml"]
    volumes:
      - ./tempo.yaml:/etc/tempo.yaml:ro

  loki:
    image: grafana/loki:3.5.0

  grafana:
    image: grafana/grafana:12.0.0
    ports:
      - "3000:3000"
    environment:
      GF_AUTH_ANONYMOUS_ENABLED: "true"         # SÓ PARA O LABORATÓRIO (veja a seção 11)
      GF_AUTH_ANONYMOUS_ORG_ROLE: Admin
      GF_AUTH_DISABLE_LOGIN_FORM: "true"
    volumes:
      - ./grafana/datasources.yaml:/etc/grafana/provisioning/datasources/datasources.yaml:ro
    depends_on: [prometheus, tempo, loki]
```

Duas técnicas do Compose aparecem aqui pela primeira vez na trilha:

- **`x-otel-comum: &otel-comum`**: um bloco de extensão (as chaves que começam com `x-` são ignoradas pelo Compose) com uma **âncora** YAML (`&otel-comum`).
- **`<<: *otel-comum`**: insere o conteúdo da âncora em cada serviço. As variáveis comuns são escritas uma vez, e cada serviço só acrescenta as suas.

Repare também que **só três portas** são publicadas no host: a da `loja` (para testar à mão), a do Prometheus e a do Grafana. Todo o resto conversa pela rede interna do Compose, pelos nomes dos serviços, como no [capítulo de Compose](../../02-containers/docker/04_compose_e_producao.md).

---

## 8. Subindo o laboratório

```bash
docker compose up -d --build
docker compose ps
```

Em alguns segundos, o gerador começa a fazer checkouts, um atrás do outro, com 0,3 s de pausa entre eles (perto de 100 por minuto). Um teste manual:

```bash
curl -s -X POST "http://localhost:8000/checkout?item=caneca&quantidade=2"
# {"pedido":123,"pagamento":{"status":"aprovado"}}
```

| Endereço | O que é |
|---|---|
| `http://localhost:3000` | Grafana: o lugar de onde tudo é consultado |
| `http://localhost:9090` | Prometheus: útil para ver os alvos (*Status → Targets*) e testar consultas |

Para saber se a telemetria está chegando: no Prometheus, *Status → Targets* deve mostrar `otel-collector` como **UP**. Se algum sinal não aparecer, o primeiro lugar para olhar é o log do Collector: `docker compose logs otel-collector`.

---

## 9. Respondendo às duas perguntas

### 9.1 Pergunta 1: falhas por minuto, ao longo do dia

No Grafana, abra **Explore**, escolha a fonte **Prometheus** e use a consulta do [capítulo de métricas](../sinais/01_metricas.md):

```promql
sum(increase(http_server_request_duration_seconds_count{service_name="loja", http_response_status_code=~"5.."}[1m]))
```

O gráfico mostra as falhas de checkout por minuto. Mude o intervalo de tempo (no canto superior direito) para *Last 24 hours* depois que o laboratório estiver rodando há algumas horas, e a curva vira a **tendência do dia**. Para transformar a consulta em um painel permanente: *Add to dashboard*.

Com o laboratório como está (5% de recusas na operadora), a linha oscila em torno de 5 falhas por minuto (perto de 100 checkouts por minuto, vezes 5%). Duas variações para comparar:

```promql
# A métrica de negócio, contada pela própria aplicação
sum(increase(loja_checkouts_total{resultado="falha"}[1m]))

# A taxa de erro de cada serviço, lado a lado
  sum by (service_name) (rate(http_server_request_duration_seconds_count{http_response_status_code=~"5.."}[5m]))
/
  sum by (service_name) (rate(http_server_request_duration_seconds_count[5m]))
```

A última consulta mostra o efeito da seção 3.3: `loja` e `pagamentos` com cerca de 5% de erros `5xx`, e `pedidos` com zero, porque ele responde `402`.

### 9.2 Montando um painel RED

Crie um *dashboard* com três painéis, um para cada letra do método RED do [capítulo de métricas](../sinais/01_metricas.md), seção 6.2, separados por serviço:

```promql
# Rate: requisições por segundo
sum by (service_name) (rate(http_server_request_duration_seconds_count[5m]))

# Errors: proporção de 5xx (a consulta da seção 9.1)

# Duration: p95 da latência
histogram_quantile(0.95, sum by (le, service_name) (rate(http_server_request_duration_seconds_bucket[5m])))
```

No painel de latência, ligue a opção **Exemplars**. Pontos aparecem sobre as linhas: cada um é uma requisição real, com o seu `trace_id`. Os que ficam no alto, perto de 1,6 s, são as requisições lentas.

Repare no painel de latência: os **três** serviços têm um p95 alto, porque cada um espera pelos que chama. É a armadilha descrita no [capítulo de escolha do sinal](../sinais/04_escolhendo_o_sinal_certo.md): a métrica mostra que há lentidão, mas não diz onde ela nasce.

### 9.3 Pergunta 2: em qual chamada está a demora

Clique em um **exemplar** alto do painel de latência, e depois no link para o Tempo. (Ou, em **Explore**, escolha a fonte **Tempo** e busque os traces lentos com TraceQL.)

```traceql
{ resource.service.name = "loja" && name = "POST /checkout" && duration > 1s }
```

Abra um dos traces da lista. A cascata é a do [capítulo de traces](../sinais/03_traces.md):

```
 loja        POST /checkout                ██████████████████████████████████████  1,64 s
 loja        POST                           █████████████████████████████████████  1,61 s
 pedidos     POST /pedidos                   ████████████████████████████████████  1,60 s
 pedidos     validar_estoque                 █                                     0,02 s
 pedidos     POST                             ███████████████████████████████████  1,57 s
 pagamentos  POST /pagamentos                  ██████████████████████████████████  1,56 s
 pagamentos  chamar_operadora                  ██████████████████████████████████  1,55 s  ◀
```

O span `chamar_operadora`, no serviço `pagamentos`, concentra quase todo o tempo, e não tem filhos que o expliquem: a resposta à pergunta 2. Clique nele: entre os atributos estão `operadora.lenta = true` e a `pagamento.bandeira`.

Uma pergunta que ninguém previu ao montar os painéis: **a lentidão é maior com alguma bandeira?** Com o atributo no span, ela é respondida com uma consulta:

```traceql
{ name = "chamar_operadora" && span.pagamento.bandeira = "elo" && duration > 1s }
```

Compare a quantidade de resultados com a mesma consulta para `"visa"`: o código da seção 3.3 torna a operadora lenta em 30% das chamadas com Elo, e em 5% das outras. É o cenário 8 do [capítulo de escolha do sinal](../sinais/04_escolhendo_o_sinal_certo.md), e um exemplo concreto da diferença entre monitoramento e observabilidade do [capítulo de fundamentos](../fundamentos/01_monitoramento_e_observabilidade.md).

### 9.4 Do trace aos logs, e de volta

No trace aberto, clique no ícone de logs de um span do serviço `pagamentos` (*Logs for this span*). O Grafana abre o Loki com a consulta da seção 6, mostrando os logs daquela requisição naquele serviço. Para ver a requisição inteira nos três serviços, em **Explore** com a fonte **Loki**:

```logql
{service_name=~"loja|pedidos|pagamentos"} | trace_id="<cole aqui o trace_id>"
```

E o caminho inverso: busque os erros de pagamento e abra o trace de um deles pelo link **Ver trace** que aparece no campo `trace_id`:

```logql
{service_name="pagamentos"} | severity_text="ERROR"
```

Com isso, as três ligações do fluxo de investigação (métrica → trace → log, e log → trace) estão funcionando.

---

## 10. Experimentos

O laboratório fica mais instrutivo quando se provoca problemas e se observa como cada sinal reage.

**Derrubar um serviço.** Pare o serviço de pagamentos:

```bash
docker compose stop pagamentos
```

Em menos de um minuto, a taxa de erros da `loja` vai a 100% no painel RED. Nos traces, o span `pedidos → pagamentos` aparece com erro de conexão, e nenhum span do serviço `pagamentos` existe. Nos logs de `pedidos`, a exceção de conexão recusada. Volte com `docker compose start pagamentos` e veja os painéis se recuperarem.

**Quebrar a propagação de contexto.** No serviço `pedidos`, acrescente a variável `OTEL_PROPAGATORS: none` e recrie-o (`docker compose up -d pedidos`). Agora `pedidos` não injeta mais o `traceparent` ao chamar `pagamentos`. Busque um trace novo: ele termina em `pedidos`, e os spans de `pagamentos` aparecem em **outros** traces, soltos, sem pai. É o problema da seção 3.3 do [capítulo de traces](../sinais/03_traces.md), visto ao vivo. Remova a variável para restaurar.

**Mudar o perfil de falhas.** Altere as probabilidades no `pagamentos.py` (por exemplo, recusas de 5% para 20%), reconstrua com `docker compose up -d --build pagamentos` e observe a reação dos painéis. Mude também a versão em `OTEL_RESOURCE_ATTRIBUTES` (`service.version=1.1.0`) e compare as duas versões com `sum by (service_version) (...)` nas consultas: é a comparação de um deploy canary.

**Medir o custo.** No Prometheus, a consulta `count({__name__=~".+"})` mostra quantas séries existem. Acrescente ao contador de checkouts um atributo com valores ilimitados (como um número aleatório) e veja o número de séries crescer sem parar: a explosão de cardinalidade do [capítulo de métricas](../sinais/01_metricas.md), seção 7.

Para desligar tudo: `docker compose down` (os dados somem com os containers, porque o laboratório não usa volumes).

---

## 11. Do laboratório para a produção

O laboratório simplifica várias coisas de propósito. Em produção, cada uma muda:

| No laboratório | Em produção |
|---|---|
| Grafana sem login, com usuário anônimo administrador | Login integrado ao provedor de identidade da empresa, com permissões por equipe |
| Dados no disco dos containers, perdidos no `down` | Armazenamento persistente: Tempo e Loki em *object storage* (S3), Prometheus com volume e armazenamento de longo prazo (Thanos, Mimir) |
| Uma instância de cada componente | Réplicas e alta disponibilidade, ou serviços gerenciados |
| 100% dos traces guardados | Amostragem no final, no Collector, guardando todos os erros e lentos ([capítulo de traces](../sinais/03_traces.md), seção 5) |
| Collector sem limites de recursos | Collector com `memory_limiter` calibrado, recursos dimensionados e métricas próprias monitoradas |
| Tráfego sem TLS entre os componentes | TLS e autenticação entre aplicações, Collector e backends |
| Retenção padrão | Retenção definida por sinal (por exemplo, métricas por 13 meses, logs por 30 dias, traces por 7) |

> **Confusão comum:** "o Grafana só mostra gráficos, então não há problema em deixá-lo aberto na rede interna". ✅ **Mais preciso:** o Grafana é uma porta de entrada para **todos** os dados de telemetria, e os logs costumam conter informações sensíveis (identificadores de clientes, endereços internos, às vezes dados que nunca deveriam ter sido registrados, como mostrou o [capítulo de logs](../sinais/02_logs.md)). Um usuário anônimo com papel de administrador, como no laboratório, pode ainda criar fontes de dados novas, alterar alertas e, com algumas fontes, executar consultas arbitrárias em bancos de dados. Em qualquer ambiente com dados reais, o Grafana exige autenticação, permissões por equipe e acesso restrito pela rede, como qualquer sistema que guarda dados de produção.

---

## Próximo passo

O laboratório responde às perguntas quando alguém as faz: abrir um painel, buscar um trace, filtrar os logs. Mas ninguém fica olhando painéis o dia todo. Falta definir **o que é "funcionar bem"** para a loja, em números, e fazer o sistema **avisar** quando isso deixar de ser verdade, sem acordar ninguém por falsos alarmes.

O [02_slos_e_alertas.md](02_slos_e_alertas.md) trata de SLIs, SLOs e orçamentos de erro, e de como transformá-los em alertas que valem a pena.
