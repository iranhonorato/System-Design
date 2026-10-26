# Escolhendo o Sinal Certo

**Pré-requisito:** os três capítulos anteriores: [01_metricas.md](01_metricas.md), [02_logs.md](02_logs.md) e [03_traces.md](03_traces.md)

> Organiza o conteúdo dos capítulos anteriores. O fluxo de investigação que combina os sinais segue a documentação de correlação do Grafana (*exemplars*, *trace to logs*) e do OpenTelemetry. O uso de eventos de deploy sobre métricas retoma as métricas DORA de [../../03-entrega-continua/cd/01_entrega_e_implantacao_continua.md](../../03-entrega-continua/cd/01_entrega_e_implantacao_continua.md).

---

## Por que este arquivo existe

Cada capítulo de sinais terminou com a seção "onde este sinal para". Este capítulo junta as três seções em uma habilidade prática: diante de uma pergunta concreta, **escolher o sinal que a responde mais diretamente**, saber **por que os outros responderiam pior**, e saber **como passar de um sinal para o outro** quando a investigação avança.

O capítulo tem quatro partes: os critérios de escolha, as duas perguntas clássicas resolvidas e comentadas, uma série de cenários para praticar (com as respostas escondidas, para você tentar antes de ler), e o fluxo de investigação que conecta os três sinais.

---

## 1. Os critérios

Quatro perguntas sobre a **pergunta** quase sempre bastam para escolher o sinal:

| Pergunte-se | Se a resposta for... | Sinal mais adequado |
|---|---|---|
| **É sobre quantidades ou sobre um item?** | Quantidades, frequências, proporções, tendências ("quantas", "com que frequência", "está piorando?") | **Métricas** |
| | Um item específico ("este pedido", "este cliente", "esta requisição") | **Logs** ou **traces** |
| **Envolve o caminho entre serviços?** | Sim: "onde, ao longo do caminho, demorou ou falhou?" | **Traces** |
| **Preciso do detalhe do evento?** | Sim: a mensagem de erro, o motivo, o valor, o stack trace | **Logs** (ou eventos e exceções dentro dos spans) |
| **Vou alertar ou acompanhar por meses?** | Sim | **Métricas** (baratas, rápidas, exatas, retidas por muito tempo) |

E dois sinais complementares, para perguntas que os três principais não alcançam:

| A pergunta é... | Sinal |
|---|---|
| "**O que mudou** neste momento?" (um deploy, uma configuração, um Pod reiniciado) | **Eventos** (marcados sobre os gráficos) |
| "**Qual função do código** está gastando a CPU ou a memória?" | **Profiles** |

O mesmo raciocínio, em forma de árvore:

```
                          ┌──────────────────────────────┐
                          │ A pergunta é sobre QUANTO,    │
                          │ COM QUE FREQUÊNCIA ou sobre   │── sim ──▶  MÉTRICAS
                          │ uma TENDÊNCIA? Ou é para      │            (contadores, histogramas,
                          │ um ALERTA?                    │             medidores + PromQL)
                          └──────────────┬───────────────┘
                                         │ não: é sobre algo específico
                          ┌──────────────▼───────────────┐
                          │ É sobre ONDE, no caminho de  │── sim ──▶  TRACES
                          │ uma requisição por vários    │            (cascata, tempo próprio,
                          │ serviços, o tempo foi gasto  │             atributos dos spans)
                          │ ou o erro aconteceu?         │
                          └──────────────┬───────────────┘
                                         │ não
                          ┌──────────────▼───────────────┐
                          │ É sobre O QUE aconteceu e    │── sim ──▶  LOGS
                          │ POR QUÊ, com os detalhes do  │            (filtrar por pedido_id,
                          │ evento?                      │             cliente_id, trace_id)
                          └──────────────┬───────────────┘
                                         │ não
                       ┌─────────────────┴─────────────────┐
             "o que mudou?" ▼                              ▼ "qual função do código?"
                       EVENTOS                          PROFILES
```

---

## 2. As duas perguntas clássicas

### 2.1 Falhas por minuto, e a tendência ao longo do dia

> **Pergunta:** uma equipe quer saber **quantas requisições falharam em cada minuto** e **observar a tendência ao longo do dia**. Qual sinal é mais adequado para essa visão agregada?

**Resposta: métricas.** Especificamente, um **contador** de requisições (o `_count` do histograma de duração HTTP, separado pelo código de resposta), consultado com `increase()` em janelas de 1 minuto e exibido em um gráfico de 24 horas:

```promql
sum(increase(http_server_request_duration_seconds_count{service_name="loja", http_response_status_code=~"5.."}[1m]))
```

**Por que métricas.** A pergunta tem três palavras que apontam para elas: **"quantas"** (uma contagem, não uma descrição), **"em cada minuto"** (agregação em intervalos de tempo) e **"tendência ao longo do dia"** (uma série temporal longa). As métricas foram feitas exatamente para isso: a aplicação incrementa um contador a cada requisição, o armazenamento guarda só os totais, e a consulta de um dia inteiro devolve 1.440 pontos em milissegundos. O custo não depende do tráfego, e a contagem é **exata**, porque não depende de amostragem.

**Por que não logs.** Cada falha está nos logs, então seria possível contá-las (`count_over_time` na LogQL). Mas, para desenhar o gráfico de um dia, o sistema de logs precisaria **ler todas as linhas de erro de 24 horas** a cada vez que alguém abrisse o painel. É lento e caro, e frágil: se alguém mudar o texto ou o nível da mensagem de erro, a contagem muda sem ninguém perceber; se os logs forem amostrados, ela fica errada. Logs servem para **explicar** cada falha, não para contá-las de forma contínua.

**Por que não traces.** Os traces costumam ser **amostrados** (só uma fração é guardada), e uma contagem feita sobre uma amostra não é a contagem real. Mesmo sem amostragem, contar traces seria pagar pelo detalhe de cada requisição para obter um único número.

### 2.2 A requisição lenta que passa por três microsserviços

> **Pergunta:** uma requisição passa por **três microsserviços** e demora muito. A equipe quer **localizar em qual chamada ocorreu a maior demora**. Que dado ajuda mais diretamente?

**Resposta: o trace distribuído da requisição**, mais exatamente os seus **spans**, com a duração de cada chamada e a relação de pai e filho entre elas. Na cascata, o span com o maior **tempo próprio** (a duração menos a dos filhos) é a chamada em que o tempo foi gasto. No exemplo do [capítulo de traces](03_traces.md), 1.552 dos 1.640 ms estavam no span `chamar_operadora`, dentro do serviço de pagamentos.

**Por que traces.** A pergunta é sobre **o caminho de uma requisição** por **vários serviços** e sobre **onde, nesse caminho**, o tempo foi gasto. É a definição de um trace: um registro de cada etapa daquela requisição, de todos os serviços, ligadas pelo mesmo `trace_id` e posicionadas no mesmo eixo de tempo.

**Por que não métricas.** As métricas de latência dos três serviços estariam **todas altas**, porque a latência de cada serviço inclui a espera pelos serviços que ele chama: a `loja` espera `pedidos`, que espera `pagamentos`. O painel mostraria três serviços "lentos" sem dizer qual é a causa e quais são só vítimas. Além disso, as métricas são agregadas: mostram a latência típica, e não a daquela requisição específica. (Elas são, no entanto, o melhor ponto de **partida**: um alerta de latência e, a partir de um exemplar no gráfico, o trace de uma requisição lenta.)

**Por que não logs.** Os logs de cada serviço têm horários, e seria possível tentar calcular a duração de cada etapa subtraindo horários. Mas os relógios de máquinas diferentes nunca estão perfeitamente sincronizados (alguns milissegundos de diferença já distorcem a conta), nem toda etapa registra seu início e seu fim, e requisições simultâneas se misturam. Com o `trace_id` nos logs, eles **complementam** o trace, explicando **por que** a chamada lenta foi lenta.

---

## 3. Cenários para praticar

Para cada cenário, decida o sinal (ou a combinação de sinais) antes de abrir a resposta.

### Cenário 1
Um cliente liga para o atendimento: **o pedido 4512, feito ontem às 15h, falhou**, e ele quer saber por quê.

<details>
<summary>Resposta</summary>

**Logs**, filtrando pelo identificador do pedido: `{service_name=~".+"} | json | pedido_id="4512"` (ou o equivalente com metadados estruturados). Os logs trazem a mensagem e o motivo ("cartão expirado"). O `trace_id` presente nesses logs abre o **trace** da requisição, caso seja preciso ver em qual etapa a falha aconteceu. **Métricas não servem**: elas não guardam pedidos individuais, e um rótulo `pedido_id` explodiria a cardinalidade ([capítulo de métricas](01_metricas.md), seção 7). É por isso que os logs devem registrar os identificadores do domínio, e não só o `trace_id`: o atendimento chega com o número do pedido.

</details>

### Cenário 2
O produto definiu que **99% dos checkouts devem levar menos de 300 ms**, e quer saber se isso foi cumprido **na última semana**.

<details>
<summary>Resposta</summary>

**Métricas**, com o **histograma** de duração: a proporção de requisições abaixo de 300 ms é o bucket `le="0.3"` (se existir um limite exatamente ali) dividido pelo total, ao longo de 7 dias. Por ser uma proporção sobre todas as requisições de um período longo, precisa ser exata e barata. Esse é um **SLO**, o assunto do [capítulo de SLOs e alertas](../pratica/02_slos_e_alertas.md). Traces amostrados dariam um número aproximado; logs, um número caro.

</details>

### Cenário 3
A memória do serviço de pedidos **cresce devagar há quatro dias**, e o Pod é reiniciado por `OOMKilled` a cada dois dias. A equipe suspeita de um vazamento de memória.

<details>
<summary>Resposta</summary>

Dois sinais, em sequência. **Métricas** (um medidor de memória em uso, ao longo de dias) confirmam o padrão: crescimento contínuo, queda brusca a cada reinício. Para descobrir **onde** no código a memória está sendo retida, o sinal certo são os **profiles** de memória (ou uma ferramenta de análise de heap). Logs e traces não mostram consumo de memória.

</details>

### Cenário 4
A equipe quer ser **avisada automaticamente** quando a taxa de erro do serviço de pagamentos passar de 2% por mais de 5 minutos.

<details>
<summary>Resposta</summary>

**Métricas.** Alertas avaliam uma consulta a cada poucos segundos, o tempo todo, e precisam de uma resposta rápida, barata e exata: a proporção de requisições `5xx` sobre o total, com `rate()` em uma janela de 5 minutos. É a base das regras de alerta do [capítulo de SLOs e alertas](../pratica/02_slos_e_alertas.md).

</details>

### Cenário 5
O serviço de pagamentos está com a **CPU sempre em 90%**, mesmo com pouco tráfego. A equipe quer saber **qual parte do código** está consumindo a CPU.

<details>
<summary>Resposta</summary>

**Profiles** de CPU, que mostram o tempo de CPU por função (normalmente como um *flame graph*). As métricas confirmam o sintoma (CPU alta) e os traces podem mostrar que **um tipo de requisição** é lento, mas nenhum dos dois aponta a função ou a linha responsável.

</details>

### Cenário 6
Depois do **deploy das 18h**, a latência do checkout subiu. A equipe quer saber **se foi o deploy**.

<details>
<summary>Resposta</summary>

**Métricas e eventos**, juntos. O **evento** de deploy marcado sobre o gráfico de latência mostra se a mudança coincide com ele. A confirmação vem das **métricas separadas pela versão** (o atributo `service.version` do recurso, do [capítulo de telemetria](../fundamentos/02_telemetria_e_opentelemetry.md)): se a latência da versão nova é maior que a da anterior no mesmo horário, a causa está no deploy. É exatamente a comparação que a análise de um deploy **canary** automatiza ([capítulo de estratégias de deploy](../../03-entrega-continua/cd/02_estrategias_de_deploy.md)). Para saber **o que** ficou mais lento na versão nova, compare **traces** de requisições lentas das duas versões.

</details>

### Cenário 7
A equipe nova precisa saber **quais serviços dependem do serviço de estoque**, para avaliar o impacto de uma mudança nele. A documentação de arquitetura está desatualizada.

<details>
<summary>Resposta</summary>

**Traces**, na forma do **mapa de serviços** (*service graph*) derivado deles ([capítulo de traces](03_traces.md), seção 6). Como ele é calculado a partir das chamadas reais, mostra quem de fato chama o serviço de estoque, com a taxa de requisições de cada um, inclusive dependências que ninguém lembrava que existiam.

</details>

### Cenário 8
Os erros de pagamento parecem acontecer **só com cartões de uma bandeira**. A equipe quer confirmar.

<details>
<summary>Resposta</summary>

Para **explorar** a hipótese agora: **traces** (uma consulta TraceQL filtrando por `span.pagamento.bandeira` e `status = error`) ou **logs** (agrupando os erros pelo campo `bandeira`), desde que esse atributo tenha sido registrado. Se a pergunta se tornar **recorrente**, vale transformá-la em uma **métrica** com o rótulo `bandeira`: ele tem poucos valores possíveis, então não causa problema de cardinalidade. Esse é um exemplo do valor dos atributos de domínio: sem `bandeira` nos spans ou nos logs, a hipótese não poderia ser verificada com os dados existentes.

</details>

### Cenário 9
Um erro em produção afetou clientes por 20 minutos. A equipe precisa saber **quais clientes** foram afetados, para avisá-los.

<details>
<summary>Resposta</summary>

**Logs**, filtrados pelo período e pelo erro, agrupando pelo campo `cliente_id`. A métrica de erros diz **quantas** requisições falharam (útil para dimensionar o impacto), mas não **quais clientes**: um rótulo `cliente_id` na métrica é justamente o erro de cardinalidade do [capítulo de métricas](01_metricas.md). Traces também guardam o `cliente_id` como atributo, mas, se forem amostrados, a lista sairá incompleta.

</details>

### Cenário 10
As requisições ao serviço de pedidos começaram a demorar, mas a CPU dele está baixa. A suspeita é que o **pool de conexões com o banco** esteja esgotado.

<details>
<summary>Resposta</summary>

**Métricas** de saturação, pelo método **USE** ([capítulo de métricas](01_metricas.md), seção 6.3): medidores de conexões em uso, conexões livres e requisições **esperando** por uma conexão. Um pool com todas as conexões em uso e uma fila crescendo confirma a hipótese. Nos **traces**, o mesmo problema aparece como um intervalo vazio antes do span da consulta: o tempo gasto esperando uma conexão livre.

</details>

### Cenário 11
Um Pod do serviço de pagamentos **reiniciou às 3h da manhã**. A equipe quer saber por quê.

<details>
<summary>Resposta</summary>

Uma combinação. Os **eventos** do Kubernetes e as **métricas** de estado do cluster dizem **como** o container terminou (por exemplo, `OOMKilled`, código 137, ou falha na liveness probe), como mostra o [capítulo de Kubernetes](../pratica/03_observabilidade_no_kubernetes.md). Os **logs** da execução anterior do container, guardados no armazenamento central (o `kubectl logs --previous` só funciona enquanto o Pod existir), mostram o que ele estava fazendo antes de morrer.

</details>

### Cenário 12
O serviço de pedidos publica uma mensagem "pedido criado" em uma fila, e o serviço de e-mail deveria enviar a confirmação. Um cliente diz que **nunca recebeu o e-mail**.

<details>
<summary>Resposta</summary>

**Traces**, desde que o contexto seja propagado nos **cabeçalhos da mensagem** ([capítulo de traces](03_traces.md), seção 3.3): o trace do checkout mostraria o span `PRODUCER` da publicação e, se a mensagem foi consumida, o span `CONSUMER` do serviço de e-mail, com seu status. Se não houver span de consumo, a mensagem não foi processada. Os **logs** do serviço de e-mail, filtrados pelo `pedido_id` ou pelo `trace_id`, mostram o que aconteceu no envio (um endereço inválido, um erro do provedor de e-mail).

</details>

### Cenário 13
O time de produto quer comparar **quantos checkouts aconteceram hoje até as 14h** com o mesmo horário **da semana passada**.

<details>
<summary>Resposta</summary>

**Métricas de negócio**, como o contador `loja_checkouts_total`. A PromQL compara períodos com o modificador `offset`: `sum(increase(loja_checkouts_total{resultado="sucesso"}[14h]))` contra a mesma expressão com `offset 1w`. Como as métricas são guardadas por muito tempo a um custo baixo, comparações entre semanas e meses são naturais. Logs de uma semana atrás podem nem existir mais, dependendo da retenção.

</details>

### Cenário 14
Uma requisição falhou com `500`. A equipe quer ver **a mensagem e o stack trace exatos** da exceção.

<details>
<summary>Resposta</summary>

**Logs** (o registro de nível `ERROR` com a exceção) ou o **evento de exceção dentro do span** que falhou, se a exceção foi registrada com `record_exception` ([capítulo de traces](03_traces.md), seção 4). As métricas só dizem que houve um `500`.

</details>

### Resumo dos cenários

| Pergunta | Sinal principal | Complemento |
|---|---|---|
| Quantas falhas por minuto, tendência do dia | **Métricas** | |
| Em qual chamada, entre três serviços, está a demora | **Traces** | Logs com o `trace_id` |
| Por que o pedido 4512 falhou | **Logs** | Trace pelo `trace_id` |
| 99% abaixo de 300 ms na semana? | **Métricas** (histograma) | |
| Vazamento de memória | **Métricas** | Profiles |
| Alertar sobre taxa de erro | **Métricas** | |
| Qual função consome a CPU | **Profiles** | |
| O deploy piorou a latência? | **Métricas** por versão + **eventos** | Traces das duas versões |
| Quem depende do serviço de estoque | **Traces** (mapa de serviços) | |
| Só uma bandeira tem erro? | **Traces** ou **logs** | Métrica, se recorrente |
| Quais clientes foram afetados | **Logs** | |
| Pool de conexões esgotado? | **Métricas** (saturação) | Traces |
| Por que o Pod reiniciou | **Eventos** e **métricas** do cluster | Logs da execução anterior |
| A mensagem foi consumida? | **Traces** (com propagação na fila) | Logs |
| Checkouts hoje vs semana passada | **Métricas** de negócio | |
| Stack trace de uma exceção | **Logs** ou evento no span | |

---

## 4. Os sinais juntos: o fluxo de investigação

Nos cenários, a resposta quase sempre teve um **sinal principal e um complemento**. Uma investigação real percorre os sinais em sequência, e o que a torna rápida é poder **pular de um para o outro** sem procurar de novo.

### 4.1 Um incidente, passo a passo

```
 18:06  ALERTA (métrica)
        "taxa de erro do checkout acima de 2% há 5 min"
              │
              ▼
 ① PAINEL RED (métricas)                    → QUAL serviço e DESDE QUANDO?
   Os três serviços mostram erros e latência subindo às 18h00.
   O marcador de EVENTO mostra: "deploy pagamentos a1b2c3d, 17:58".
   A latência por service.version: só a versão nova está lenta.
              │ clique em um EXEMPLAR na faixa lenta do gráfico
              ▼
 ② TRACE (Tempo)                            → ONDE, no caminho?
   Cascata: 1.552 de 1.640 ms em "chamar_operadora" (pagamentos).
   O span tem status Error e um evento "timeout da operadora".
              │ "ver logs deste trace" (filtra pelo trace_id)
              ▼
 ③ LOGS (Loki)                              → POR QUÊ?
   pagamentos  WARNING  "timeout da operadora; nova tentativa"  (×3)
   pagamentos  ERROR    "operadora não respondeu" timeout_configurado=0.5s
              │
              ▼
 CAUSA: o deploy das 17:58 reduziu o timeout da operadora de 2 s para 0,5 s.
        A operadora responde em ~1,5 s: toda chamada estoura, é repetida 3 vezes e falha.
 AÇÃO:  rollback para a versão anterior (o mesmo mecanismo do deploy, como no capítulo de CD).
```

Cada passo respondeu a uma pergunta diferente, com o sinal adequado a ela, e cada passo levou ao seguinte por um **identificador compartilhado**. Sem essas ligações, a mesma investigação exigiria procurar manualmente, em três ferramentas, por horários aproximados.

### 4.2 O que precisa existir para que os sinais se conectem

| Ligação | O que a torna possível | Onde foi visto |
|---|---|---|
| Métrica → trace | **Exemplares**: o `trace_id` anexado a observações do histograma | [Métricas](01_metricas.md), seção 8 |
| Trace → logs | O **`trace_id`** e o **`span_id`** em cada log | [Logs](02_logs.md), seção 3 |
| Logs → trace | O mesmo `trace_id`, clicável na ferramenta de logs | [Logs](02_logs.md), seção 3 |
| Qualquer sinal → qualquer sinal | O **mesmo `service.name`** (e os mesmos atributos de recurso) nos três | [Telemetria](../fundamentos/02_telemetria_e_opentelemetry.md), seção 5 |
| Métricas → "o que mudou" | **Eventos** de deploy e de configuração marcados nos gráficos, com a **versão** como atributo | Seção 3, cenário 6 |
| Um serviço → o próximo | **Propagação de contexto** em todas as chamadas, inclusive filas | [Traces](03_traces.md), seção 3 |

Instrumentar com o OpenTelemetry cobre a maior parte dessa tabela de uma vez: o mesmo SDK produz os três sinais com o mesmo recurso, propaga o contexto, coloca o `trace_id` nos logs e pode anexar exemplares às métricas. A [parte prática](../pratica/01_stack_de_observabilidade.md) monta essas ligações no Grafana.

---

## 5. Os erros mais comuns na escolha

| Erro | Por que é um problema | O que fazer |
|---|---|---|
| **Usar logs como métricas** (painéis e alertas contando linhas de log) | Caro, lento e frágil: quebra quando a mensagem muda | Um contador na aplicação; logs para explicar |
| **Usar métricas como logs** (rótulos com `pedido_id`, `cliente_id`, URL completa) | Explosão de cardinalidade: o servidor de métricas cai ou a conta explode | Identificadores nos logs e nos atributos dos spans |
| **Traces sem propagação** | Cada serviço gera um trace separado: a cascata entre serviços nunca aparece | Instrumentar todos os clientes HTTP e as filas |
| **Amostragem na origem que descarta os erros** | Os traces que interessam são justamente os que faltam | Amostragem no final (*tail sampling*): todos os erros e lentos |
| **Só métricas técnicas** | O serviço "saudável" que parou de aprovar pagamentos passa despercebido | Métricas de negócio ao lado das técnicas |
| **Nomes diferentes para o mesmo serviço** nos três sinais | Nenhuma ligação entre os sinais funciona | `service.name` definido uma vez, pelo SDK |
| **Coletar tudo, em detalhe máximo** | Custo alto e sinal útil escondido no ruído | Poucos dados, bem escolhidos e bem conectados |

> **Confusão comum:** "precisamos escolher um dos três sinais para começar, e o melhor é o de logs, porque ele tem tudo". ✅ **Mais preciso:** os logs têm o **detalhe** de tudo, mas não respondem bem a perguntas agregadas (são caros para contar) nem a perguntas sobre o caminho entre serviços (os horários de máquinas diferentes não se alinham). Os sinais não competem: cada um responde a um tipo de pergunta que os outros respondem mal, e o valor maior está nas **ligações** entre eles. Se for preciso priorizar, a ordem que costuma dar mais retorno em sistemas com vários serviços é: **métricas RED de cada serviço** (para saber que algo está errado e onde), **traces com propagação** (para saber em qual chamada) e **logs estruturados com `trace_id`** (para saber por quê). Com o OpenTelemetry, as três coisas vêm praticamente juntas.

---

## Próximo passo

A parte de sinais termina aqui. Você sabe o que cada sinal é, como ele é produzido e consultado, e como escolher e combinar os três. Falta ver tudo isso **funcionando**: os três microsserviços da loja instrumentados com OpenTelemetry, um Collector, Prometheus, Loki, Tempo e Grafana, e as duas perguntas clássicas respondidas em painéis reais.

O [../pratica/01_stack_de_observabilidade.md](../pratica/01_stack_de_observabilidade.md) monta essa pilha completa com Docker Compose.
