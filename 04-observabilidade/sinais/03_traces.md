# Traces: Rastreamento Distribuído

**Pré-requisito:** [02_logs.md](02_logs.md)

> Segue a documentação do OpenTelemetry (*Traces*, *Context propagation*, *Sampling*), a recomendação **W3C Trace Context** e a documentação do Grafana Tempo (*TraceQL*). A ideia de rastreamento distribuído foi popularizada pelo artigo *Dapper, a Large-Scale Distributed Systems Tracing Infrastructure* (Google, 2010), que inspirou o Zipkin, o Jaeger e o próprio OpenTelemetry.

---

## Por que este arquivo existe

Uma requisição de checkout passa por três microsserviços (`loja` → `pedidos` → `pagamentos`) e está demorando muito: 1,6 segundo, quando o normal é 150 milissegundos. A equipe quer localizar **em qual chamada** ocorre a maior demora. Que dado ajuda mais diretamente?

As **métricas** mostram a latência de cada serviço, mas têm uma armadilha: os **três** serviços aparecem lentos. A `loja` espera por `pedidos`, que espera por `pagamentos`, então a latência medida em cada um **inclui** o tempo das chamadas que ele faz. O painel mostra três números altos e não diz qual deles é a causa e quais são só vítimas. Os **logs** mostram eventos de cada serviço, mas reconstruir a duração de cada etapa comparando horários de máquinas diferentes é impreciso. O dado que responde diretamente é o **trace** daquela requisição: ele mostra cada chamada como um **span**, com início, duração e quem chamou quem, e torna visível em qual delas o tempo foi gasto.

Este capítulo explica como os traces funcionam: o que são traces e spans, como ler uma cascata de tempos (e o erro mais comum ao lê-la), como o contexto atravessa os serviços e onde essa travessia costuma quebrar, como acrescentar spans do domínio, e como decidir quais traces guardar sem perder justamente os que importam.

---

## 1. Traces e spans

Um **trace** é o registro do caminho completo de **uma** requisição pelo sistema. Ele é formado por **spans**: cada span representa **uma operação** com início e fim (receber uma requisição HTTP, fazer uma chamada a outro serviço, executar uma consulta no banco, ou um trecho do código que alguém quis medir).

Os spans formam uma **árvore**: cada span, exceto o primeiro, tem um **pai**, a operação que o originou. O primeiro, sem pai, é o **span raiz**.

```
 trace_id: 4bf92f3577b34da6a3ce929d0e0e4736  (o mesmo em todos os spans abaixo)

 span raiz ─ loja: POST /checkout                        span_id=a1   pai=—
   └─ loja → pedidos: POST                               span_id=b2   pai=a1
        └─ pedidos: POST /pedidos                        span_id=c3   pai=b2   ← outro serviço!
             ├─ validar_estoque                          span_id=d4   pai=c3
             └─ pedidos → pagamentos: POST               span_id=e5   pai=c3
                  └─ pagamentos: POST /pagamentos        span_id=f6   pai=e5   ← outro serviço!
                       └─ chamar_operadora               span_id=g7   pai=f6
```

### 1.1 O que cada span registra

| Campo | O que é | Exemplo |
|---|---|---|
| `trace_id` | O identificador do trace, **igual em todos os spans** dele | `4bf92f3577b34da6a3ce929d0e0e4736` |
| `span_id` | O identificador deste span | `00f067aa0ba902b7` |
| `parent_span_id` | O span pai (vazio na raiz) | |
| `name` | O nome da operação | `POST /checkout`, `chamar_operadora` |
| `kind` | O papel do span (veja abaixo) | `SERVER` |
| início e fim | Quando começou e terminou; a diferença é a duração | |
| **atributos** | Pares chave-valor com o contexto | `http.route="/checkout"`, `pedido.id=4512` |
| **eventos** | Acontecimentos com horário **dentro** do span | "nova tentativa", uma exceção |
| **status** | `Unset` (padrão), `Ok` ou `Error` | `Error: recusado pela operadora` |

O **tipo** (*kind*) diz qual lado de uma comunicação o span representa:

| Kind | Significado | Exemplo |
|---|---|---|
| `SERVER` | Recebeu uma requisição | `pedidos: POST /pedidos` |
| `CLIENT` | Fez uma requisição a outro serviço | `loja → pedidos: POST` |
| `INTERNAL` | Uma operação interna, sem comunicação | `validar_estoque` |
| `PRODUCER` / `CONSUMER` | Publicou / consumiu uma mensagem em uma fila | publicar "pedido criado" no Kafka |

Repare que cada chamada entre serviços aparece **duas vezes**: um span `CLIENT` no serviço que chama e um span `SERVER` no serviço chamado. A diferença entre as durações dos dois é o tempo gasto **na rede** (e em filas, proxies e balanceadores no caminho).

Uma diferença importante em relação às métricas: atributos de spans **podem ter valores ilimitados**. Colocar `pedido.id` ou `cliente.id` em um span é perfeitamente normal, e muito útil, porque cada span já é um registro individual. É o oposto da regra de cardinalidade das métricas do [capítulo de métricas](01_metricas.md).

---

## 2. Lendo a cascata: onde está a demora

### 2.1 A cascata do checkout lento

As ferramentas de traces (Jaeger, Grafana Tempo) mostram um trace como uma **cascata** (*waterfall*): cada span é uma barra, posicionada no tempo, com os filhos abaixo dos pais.

```
 0 ms            400 ms           800 ms          1.200 ms         1.640 ms
 ├───────────────┼────────────────┼────────────────┼────────────────┤
 ██████████████████████████████████████████████████████████████████████  loja POST /checkout      1.640 ms
   █████████████████████████████████████████████████████████████████████ loja → pedidos (CLIENT)  1.612 ms
    ████████████████████████████████████████████████████████████████████ pedidos POST /pedidos    1.603 ms
    █                                                                     validar_estoque             21 ms
      ███████████████████████████████████████████████████████████████████ pedidos → pagamentos     1.571 ms
       ██████████████████████████████████████████████████████████████████ pagamentos POST          1.565 ms
       █████████████████████████████████████████████████████████████████  chamar_operadora        1.552 ms  ◀
```

### 2.2 Tempo total e tempo próprio

Cada barra mostra o **tempo total** do span, que **inclui** o tempo de todos os seus filhos. Para encontrar a causa, o que importa é o **tempo próprio** (*self time*): o tempo do span **menos** o tempo dos filhos, ou seja, o trabalho que aquela operação fez sozinha.

| Span | Total | Filhos | **Tempo próprio** | O que significa |
|---|---|---|---|---|
| loja POST /checkout | 1.640 | 1.612 | **28 ms** | Processamento na loja |
| loja → pedidos (CLIENT) | 1.612 | 1.603 | **9 ms** | Rede entre loja e pedidos |
| pedidos POST /pedidos | 1.603 | 21 + 1.571 | **11 ms** | Processamento em pedidos |
| validar_estoque | 21 | 0 | **21 ms** | Consulta de estoque |
| pedidos → pagamentos (CLIENT) | 1.571 | 1.565 | **6 ms** | Rede entre pedidos e pagamentos |
| pagamentos POST | 1.565 | 1.552 | **13 ms** | Processamento em pagamentos |
| **chamar_operadora** | 1.552 | 0 | **1.552 ms** | **Espera pela operadora de cartão** |

A resposta à pergunta do início está na última linha: **95% do tempo do checkout foi gasto esperando a operadora de cartão**, na chamada feita pelo serviço de pagamentos. Os serviços `loja` e `pedidos` não têm problema nenhum: estão apenas esperando. Com essa informação, a investigação vai direto ao ponto certo (a operadora está lenta? só para uma bandeira? o *timeout* está alto demais?), em vez de passar horas otimizando a `loja`, que aparecia "lenta" nas métricas.

Muitas ferramentas calculam o tempo próprio sozinhas e destacam o **caminho crítico**: a sequência de spans que determina a duração total da requisição.

> **Confusão comum:** "o span com a barra mais longa é o culpado". ✅ **Mais preciso:** o span mais longo de um trace é **sempre** o raiz, porque ele contém todos os outros. A barra comprida da `loja` não diz nada sobre a `loja`: ela só reflete a espera pelos serviços chamados. O que localiza a causa é o **tempo próprio** de cada span, e normalmente a resposta está **no fundo da árvore**: o span mais profundo que ainda é longo e cujos filhos não explicam sua duração. Se nenhum filho explica a duração de um span (como o `chamar_operadora`, que não tem filhos), o tempo foi gasto ali dentro, ou em algo que não foi instrumentado. Um "buraco" sem filhos dentro de um span longo é, em si, uma pista: falta instrumentar aquela parte do código (uma consulta ao banco sem a instrumentação do driver, uma chamada HTTP com um cliente não instrumentado).

### 2.3 Padrões que a cascata revela

| O que aparece na cascata | O que costuma significar |
|---|---|
| Um span folha muito longo | A operação em si é lenta (consulta sem índice, dependência externa lenta) |
| Dezenas de spans curtos, iguais, **em sequência** | O problema **N+1**: uma consulta ao banco por item, em vez de uma consulta para todos |
| Chamadas em sequência que poderiam ser **paralelas** | Oportunidade de otimização: as barras deveriam estar sobrepostas |
| Grande diferença entre o span `CLIENT` e o `SERVER` correspondente | Tempo perdido na rede, em uma fila ou no balanceador |
| Um intervalo vazio entre o fim de um filho e o início do próximo | Processamento não instrumentado no pai |
| Span com status `Error` e um evento de exceção | O ponto em que o erro aconteceu (e não só onde ele foi percebido) |

---

## 3. Propagação de contexto: como o trace atravessa os serviços

### 3.1 O problema

O span `pedidos: POST /pedidos` foi criado **em outro processo**, em outra máquina, e mesmo assim sabe que pertence ao trace `4bf92f35...` e que seu pai é o span `b2` da loja. Como?

### 3.2 O cabeçalho `traceparent`

Quando a `loja` faz a chamada HTTP para `pedidos`, a instrumentação do cliente HTTP **injeta** o contexto do trace atual em um cabeçalho padronizado pela W3C, o **`traceparent`**:

```
POST /pedidos HTTP/1.1
Host: pedidos:8000
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
              │  └────────────── trace_id ──────────┘ └── span pai ──┘ │
              versão                                                   flags (01 = amostrado)
```

Do outro lado, a instrumentação do servidor em `pedidos` **extrai** o cabeçalho e cria o span `SERVER` como filho do span indicado, no mesmo trace. O mesmo acontece de `pedidos` para `pagamentos`.

```
   loja                                  pedidos                              pagamentos
 ┌──────────────────────┐             ┌──────────────────────┐             ┌──────────────────────┐
 │ span SERVER a1       │             │ span SERVER c3       │             │ span SERVER f6       │
 │ └ span CLIENT b2 ────┼─ HTTP ─────▶│ (pai = b2)           │             │ (pai = e5)           │
 │   injeta traceparent │ traceparent │ └ span CLIENT e5 ────┼─ HTTP ─────▶│                      │
 │   (trace, span b2)   │             │   injeta traceparent │ traceparent │                      │
 └──────────────────────┘             └──────────────────────┘             └──────────────────────┘
```

Com a instrumentação automática do OpenTelemetry nos dois lados (aqui, `httpx` no cliente e FastAPI no servidor), nada disso precisa ser escrito: injetar e extrair acontece sozinho. Um cabeçalho opcional, o **`tracestate`**, carrega informações específicas de fornecedores, e o **`baggage`** permite propagar pares chave-valor de negócio (por exemplo, `cliente.plano=premium`) para todos os serviços seguintes, que podem usá-los como atributos.

### 3.3 Onde a propagação quebra

A propagação só funciona se **todos os elos** da cadeia a fizerem. Quando um elo falha, o trace se parte em dois: o serviço seguinte inicia um trace novo, sem pai, e a cascata da loja termina "no vazio".

| Onde quebra | Por quê | Solução |
|---|---|---|
| **Cliente HTTP não instrumentado** | Uma biblioteca sem instrumentação não injeta o cabeçalho | Instalar a instrumentação dela, ou injetar manualmente |
| **Filas e mensageria** (Kafka, RabbitMQ, SQS) | Não há uma requisição HTTP: a mensagem é consumida depois, por outro processo | Injetar o contexto nos **cabeçalhos da mensagem** ao publicar e extraí-lo ao consumir (abaixo) |
| **Tarefas em segundo plano e threads** | O contexto é guardado por *thread*/tarefa, e uma thread nova começa sem ele | Passar o contexto explicitamente, ou usar a instrumentação de `threading`/`asyncio` |
| **Proxies e gateways que removem cabeçalhos** | Um proxy configurado para repassar só cabeçalhos conhecidos descarta o `traceparent` | Liberar `traceparent`, `tracestate` e `baggage`; ou instrumentar o próprio gateway |
| **Formatos diferentes** | Um serviço antigo usa o formato B3 do Zipkin, o outro, W3C | Configurar os propagadores para aceitar os dois (`OTEL_PROPAGATORS=tracecontext,baggage,b3`) |

Em mensageria, a propagação manual fica assim:

```python
from opentelemetry import trace
from opentelemetry.propagate import extract, inject
from opentelemetry.trace import SpanKind

tracer = trace.get_tracer("pedidos")

# Quem PUBLICA: injeta o contexto atual nos cabeçalhos da mensagem
def publicar_pedido_criado(pedido):
    with tracer.start_as_current_span("publicar pedido_criado", kind=SpanKind.PRODUCER):
        cabecalhos = {}
        inject(cabecalhos)                    # acrescenta "traceparent" ao dicionário
        fila.publicar("pedidos", corpo=pedido.json(), cabecalhos=cabecalhos)

# Quem CONSOME: extrai o contexto e cria o span como continuação do mesmo trace
def ao_receber(mensagem):
    contexto = extract(mensagem.cabecalhos)
    with tracer.start_as_current_span("processar pedido_criado", context=contexto, kind=SpanKind.CONSUMER):
        enviar_email_de_confirmacao(mensagem.corpo)
```

Para mensagens processadas muito depois, ou em lotes, o OpenTelemetry também permite **links** entre spans, em vez de uma relação de pai e filho: o span de processamento do lote "aponta" para os traces das mensagens que processou, sem que um trace de horas fique aberto.

---

## 4. Spans do domínio

A instrumentação automática cria os spans das requisições e das chamadas entre serviços. Os spans que tornam um trace realmente útil, como o `validar_estoque` e o `chamar_operadora` da cascata, são acrescentados no código, com a API do OpenTelemetry:

```python
from opentelemetry import trace
from opentelemetry.trace import StatusCode

tracer = trace.get_tracer("pagamentos")


async def pagar(pagamento):
    with tracer.start_as_current_span("chamar_operadora") as span:
        span.set_attribute("pagamento.bandeira", pagamento.bandeira)
        span.set_attribute("pedido.id", pagamento.pedido_id)

        for tentativa in range(1, 4):
            try:
                resultado = await operadora.autorizar(pagamento)
                break
            except TimeoutOperadora as erro:
                span.add_event("timeout da operadora", {"tentativa": tentativa})   # um evento no meio do span
                if tentativa == 3:
                    span.record_exception(erro)                                   # tipo, mensagem e stack trace
                    span.set_status(StatusCode.ERROR, "operadora não respondeu")
                    raise

        span.set_attribute("pagamento.resultado", resultado.status)
        return resultado
```

| Elemento | Para quê |
|---|---|
| **Atributos** | O contexto que permite filtrar depois: bandeira, pedido, resultado. Ajudam a responder "só acontece com uma bandeira?" |
| **Eventos** | Coisas que acontecem **durante** a operação, com horário: novas tentativas, um cache que falhou |
| **`record_exception`** | Anexa a exceção ao span, com o stack trace |
| **`set_status(ERROR)`** | Marca o span como falho, para que buscas e métricas por erro o encontrem |

Vale instrumentar à mão os pontos que importam para entender o sistema: chamadas a dependências externas sem instrumentação automática, etapas de negócio com nome próprio, laços que processam muitos itens (um span para o lote, e não um por item). Spans demais deixam o trace ilegível e caro; spans de menos deixam "buracos" na cascata.

---

## 5. Amostragem: quais traces guardar

### 5.1 O problema do volume

Um trace por requisição, com uma dúzia de spans cada, em um sistema com milhares de requisições por segundo, gera um volume (e um custo) maior que o dos logs. E a grande maioria desses traces é igual: requisições normais, rápidas, sem nada de interessante. Por isso os traces quase sempre são **amostrados**: só uma parte é guardada.

### 5.2 Amostragem na origem (*head sampling*)

A decisão é tomada **no início** do trace, no primeiro serviço, antes de se saber como a requisição vai terminar. Ela viaja no próprio `traceparent` (as *flags* `01` = amostrado), e todos os serviços seguintes respeitam a decisão, de forma que um trace nunca fica pela metade.

```bash
# Guarda 10% dos traces novos; se o serviço anterior já decidiu, segue a decisão dele
OTEL_TRACES_SAMPLER=parentbased_traceidratio
OTEL_TRACES_SAMPLER_ARG=0.1
```

É simples e barata, mas tem um defeito: como a decisão é tomada **antes**, ela é cega. Com 10%, descartam-se 90% dos traces normais, o que é ótimo, mas também 90% dos traces **com erro** e 90% dos **lentos**, justamente os que alguém vai querer ver.

### 5.3 Amostragem no final (*tail sampling*)

A decisão é tomada **depois** que o trace terminou, olhando para ele inteiro. Isso acontece no **OpenTelemetry Collector** ([capítulo de telemetria](../fundamentos/02_telemetria_e_opentelemetry.md)), com o processor `tail_sampling`: as aplicações enviam 100% dos spans, o Collector os segura por alguns segundos até o trace se completar, e aplica regras:

```yaml
processors:
  tail_sampling:
    decision_wait: 10s                 # espera os spans de um trace chegarem
    policies:
      - name: todos-os-erros
        type: status_code
        status_code: { status_codes: [ERROR] }
      - name: todos-os-lentos
        type: latency
        latency: { threshold_ms: 1000 }
      - name: amostra-do-resto
        type: probabilistic
        probabilistic: { sampling_percentage: 5 }
```

Com essas regras, **todo** trace com erro e **todo** trace acima de 1 segundo é guardado, e só 5% dos normais. O custo cai muito, e nada de importante se perde.

O preço é operacional: o Collector precisa guardar os spans em memória durante a espera, e **todos os spans de um mesmo trace** precisam chegar à **mesma instância** do Collector (que só assim enxerga o trace inteiro). Com várias réplicas de gateway, isso exige uma camada na frente que distribua os spans pelo `trace_id` (o exporter `loadbalancing` do Collector faz isso).

> **Confusão comum:** "com amostragem de 10%, perdemos 90% dos erros, então é melhor guardar todos os traces". ✅ **Mais preciso:** isso vale para a amostragem **na origem**, que decide antes de saber o resultado. Com amostragem **no final**, a decisão é tomada olhando o trace completo, e as regras podem guardar 100% dos erros e das requisições lentas e descartar a maioria das normais, que são repetitivas. Guardar todos os traces raramente compensa: o custo cresce com o tráfego, e os traces normais adicionais quase nunca são consultados. Duas outras peças completam o quadro. As **métricas** continuam contando 100% das requisições, porque não dependem da amostragem dos traces. E o Collector pode gerar métricas **a partir dos spans antes de amostrá-los** (o *connector* `spanmetrics`), de forma que os números de taxa, erros e duração fiquem exatos mesmo com poucos traces guardados.

---

## 6. Armazenando e consultando traces

| Ferramenta | Característica |
|---|---|
| **Jaeger** | Projeto da CNCF, criado no Uber; interface clássica de busca e cascata |
| **Grafana Tempo** | Guarda os traces em *object storage* (S3), com custo baixo; consulta com **TraceQL**; integrado ao Grafana com métricas e logs |
| **Zipkin** | O pioneiro aberto, inspirado no Dapper |
| **AWS X-Ray** | O serviço de traces da AWS |
| **Plataformas comerciais** | Datadog, Honeycomb, New Relic, Dynatrace, Grafana Cloud |

A **TraceQL**, do Tempo, busca traces pelos atributos e pela estrutura dos spans:

```traceql
# Traces em que o serviço de pagamentos teve um span com mais de 1 segundo
{ resource.service.name = "pagamentos" && duration > 1s }

# Traces com erro em qualquer serviço
{ status = error }

# Chamadas à operadora lentas, só para uma bandeira
{ name = "chamar_operadora" && span.pagamento.bandeira = "elo" && duration > 500ms }

# Traces em que a loja chamou pedidos e a requisição terminou com 5xx
{ resource.service.name = "loja" && span.http.response.status_code >= 500 }
```

A terceira consulta mostra o que torna os traces tão úteis para as falhas imprevistas do [capítulo de fundamentos](../fundamentos/01_monitoramento_e_observabilidade.md): com o atributo `pagamento.bandeira` no span, a pergunta "a lentidão é só com uma bandeira?" é respondida em uma consulta, sem nenhuma métrica ter sido criada para isso.

### Visões derivadas dos traces

Como os traces contêm quem chama quem, as ferramentas derivam deles duas visões úteis:

- **Mapa de serviços** (*service graph*): o desenho das dependências entre os serviços, com a taxa de requisições e de erros em cada ligação. Ele é gerado a partir dos traces reais, e por isso mostra a arquitetura **como ela é**, e não como está no diagrama da documentação.
- **Métricas de spans** (*span metrics*): taxa, erros e duração por serviço e por operação, calculadas a partir dos spans. São métricas RED ([capítulo de métricas](01_metricas.md)) obtidas sem instrumentação adicional.

---

## 7. Onde os traces param

Os traces mostram **onde** o tempo foi gasto e **onde** o erro aconteceu, em uma requisição. Eles não substituem os outros sinais:

- Um trace é **uma** requisição (ou uma amostra delas). Para saber **quantas** requisições estão lentas, e desde quando, as métricas continuam sendo a resposta.
- Um span diz que `chamar_operadora` falhou, com a exceção. Os detalhes que o código registrou naquele momento (a resposta exata da operadora, o motivo da recusa) estão nos **logs** com o mesmo `trace_id`.
- Um trace mostra que uma função levou 800 ms de CPU, mas não **qual linha** do código. Para isso existem os **profiles**.

---

## Próximo passo

Os três capítulos de sinais mostraram, cada um, o que um sinal faz bem e onde ele para. Na prática, diante de uma pergunta concreta, é preciso escolher rápido qual consultar primeiro, e saber como passar de um para o outro.

O [04_escolhendo_o_sinal_certo.md](04_escolhendo_o_sinal_certo.md) organiza essa decisão: os critérios para escolher, a resposta comentada das duas perguntas clássicas, uma lista de cenários com o sinal mais adequado para cada um, e o fluxo de investigação que conecta métricas, traces e logs.
