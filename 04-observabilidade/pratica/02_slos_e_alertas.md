# SLOs e Alertas

**Pré-requisito:** [01_stack_de_observabilidade.md](01_stack_de_observabilidade.md) e [../sinais/01_metricas.md](../sinais/01_metricas.md)

> Segue os Capítulos 4 (*Service Level Objectives*) e 6 (*Monitoring Distributed Systems*) de *Site Reliability Engineering* (Google, 2016) e o Capítulo 5 (*Alerting on SLOs*) de *The Site Reliability Workbook* (Google, 2018), de onde vem a tabela de *burn rate* em múltiplas janelas. A configuração segue a documentação do Prometheus (*Recording rules*, *Alerting rules*) e do Alertmanager.

---

## Por que este arquivo existe

O laboratório do capítulo anterior responde a qualquer pergunta, desde que alguém abra o Grafana e a faça. Na prática, isso leva a dois problemas opostos, comuns em quase toda equipe:

- **Painéis que ninguém olha.** Dezenas de gráficos bonitos, e o problema é descoberto pelos clientes, no atendimento.
- **Alertas demais.** Avisos de CPU a 80%, de disco a 70%, de um Pod que reiniciou uma vez, que chegam a toda hora e não exigem nenhuma ação. Em poucas semanas a equipe deixa de lê-los, e o alerta importante se perde no meio. Esse fenômeno se chama **fadiga de alertas**.

Os dois têm a mesma raiz: falta uma definição clara, em números, do que significa **o sistema estar funcionando bem para os usuários**. A pergunta central deste capítulo é: *como definir esse "funcionar bem" de forma mensurável, e como transformar essa definição em alertas que só acordam alguém quando os usuários estão realmente sendo afetados?* A resposta usa três ideias da engenharia de confiabilidade (SRE): **SLI**, **SLO** e **orçamento de erro**.

---

## 1. SLI, SLO e SLA

### 1.1 As três siglas

| Sigla | Nome | O que é | Exemplo na loja |
|---|---|---|---|
| **SLI** | *Service Level Indicator* | Uma **medida** da qualidade do serviço, do ponto de vista do usuário | A proporção de checkouts que não falharam |
| **SLO** | *Service Level Objective* | O **objetivo** para um SLI, em um período | 99,9% dos checkouts sem falha, em 30 dias |
| **SLA** | *Service Level Agreement* | Um **contrato** com consequências (multa, crédito) se o objetivo não for cumprido | "Se a disponibilidade mensal ficar abaixo de 99,5%, o cliente recebe 10% de crédito" |

A ordem importa: o SLI é a régua, o SLO é a meta interna da equipe, e o SLA, quando existe, é a promessa externa, sempre **mais folgada** que o SLO. Se o SLA fosse igual ao SLO, toda vez que a equipe errasse a própria meta a empresa pagaria a multa; com o SLO mais exigente, a equipe percebe e reage antes de chegar ao contrato.

### 1.2 Um bom SLI é a proporção de eventos bons

A forma recomendada de um SLI é sempre a mesma:

```
          eventos bons
 SLI = ───────────────────  × 100%
        eventos válidos
```

Para a loja, dois SLIs cobrem o que um cliente percebe no checkout:

| SLI | Evento bom | Evento válido |
|---|---|---|
| **Disponibilidade** | Um `POST /checkout` que **não** terminou com `5xx` | Todo `POST /checkout` |
| **Latência** | Um `POST /checkout` que levou **até 250 ms** | Todo `POST /checkout` |

Em PromQL, com as métricas do [capítulo anterior](01_stack_de_observabilidade.md):

```promql
# SLI de disponibilidade nos últimos 30 dias
  sum(rate(http_server_request_duration_seconds_count{service_name="loja", http_route="/checkout", http_response_status_code!~"5.."}[30d]))
/
  sum(rate(http_server_request_duration_seconds_count{service_name="loja", http_route="/checkout"}[30d]))

# SLI de latência: proporção de checkouts com até 250 ms
  sum(rate(http_server_request_duration_seconds_bucket{service_name="loja", http_route="/checkout", le="0.25"}[30d]))
/
  sum(rate(http_server_request_duration_seconds_count{service_name="loja", http_route="/checkout"}[30d]))
```

O limite de latência foi escolhido como **250 ms**, e não 300, por um motivo prático: o SLI de latência é a proporção de um **bucket** do histograma, e os limites padrão do OpenTelemetry para durações HTTP incluem `0.25`, mas não `0.3` ([capítulo de métricas](../sinais/01_metricas.md), seção 5). Ou se escolhe o limite do SLO entre os buckets existentes, ou se configura um bucket no valor desejado.

Três critérios para escolher SLIs:

- **Do ponto de vista do usuário.** "CPU abaixo de 70%" não é um SLI: o usuário não percebe a CPU, percebe a demora e o erro.
- **Na borda.** Medir na `loja` (onde a requisição do cliente chega), e não em cada serviço interno, captura o efeito de todos eles. Ainda melhor, quando possível: medir no load balancer ou no Ingress, que enxerga até as requisições que nem chegaram à aplicação.
- **Poucos.** Dois ou três SLIs por jornada importante do usuário (checkout, busca, login) bastam.

---

## 2. Escolhendo o objetivo

### 2.1 O que cada "nove" significa

Um SLO de disponibilidade define quanto de falha é aceitável. Em 30 dias, isso equivale a:

| SLO | Falha permitida | Em tempo de indisponibilidade total, em 30 dias |
|---|---|---|
| 99% | 1% | 7 h 12 min |
| 99,5% | 0,5% | 3 h 36 min |
| **99,9%** | **0,1%** | **43 min 12 s** |
| 99,95% | 0,05% | 21 min 36 s |
| 99,99% | 0,01% | 4 min 19 s |

Cada nove a mais divide a falha permitida por dez, e costuma multiplicar o custo: mais réplicas, mais zonas de disponibilidade, mais automação, mais gente de plantão, mais cuidado em cada deploy.

> **Confusão comum:** "o objetivo deve ser 100%: nenhuma falha é aceitável". ✅ **Mais preciso:** 100% é impossível de garantir e caro de perseguir, e nem os usuários conseguem percebê-lo. A conexão de celular, o Wi-Fi e o próprio aparelho do cliente falham com mais frequência do que um serviço com 99,99%; a partir de certo ponto, a confiabilidade extra do servidor se perde no caminho. Além disso, um SLO de 100% proíbe qualquer risco: nenhum deploy pode falhar, nenhum experimento pode dar errado, e a equipe acaba entregando cada vez menos. A pergunta certa é **"qual nível de confiabilidade os usuários precisam, e a partir de onde deixam de notar a diferença?"**. Para uma loja, 99,9% no checkout costuma ser um bom ponto de partida; para um relatório interno gerado de madrugada, 99% pode ser mais que suficiente.

### 2.2 Dependências se multiplicam

Se o checkout depende de três serviços **em sequência**, e cada um tem 99,9% de disponibilidade, a disponibilidade do caminho completo é, no máximo, o **produto**:

```
 0,999 × 0,999 × 0,999 ≈ 0,997     →  99,7%, e não 99,9%
```

Um SLO para a jornada do usuário precisa considerar todas as dependências do caminho, inclusive as externas, como a operadora de cartão. Se a operadora oferece 99,5%, nenhum esforço interno garante 99,9% no checkout, a não ser que o sistema seja capaz de funcionar quando ela falha (com uma segunda operadora, por exemplo).

---

## 3. O orçamento de erro

### 3.1 A ideia

Se o SLO é 99,9%, então **0,1% das requisições pode falhar**. Esse 0,1% é o **orçamento de erro** (*error budget*): uma quantidade de falha que a equipe **pode gastar**, e que é renovada a cada período.

Com 3 milhões de checkouts por mês e um SLO de 99,9%, o orçamento é de **3 mil checkouts com falha** no mês. Ele é consumido por qualquer coisa que faça checkouts falharem: um deploy com defeito, a operadora fora do ar, um node que caiu.

```
 orçamento de erro restante no mês (SLO 99,9%)

 100% │████████████████
      │                ████          ← um deploy com defeito consumiu 20%
  75% │                    ██████████████
      │                                  ████████        ← instabilidade da operadora
  50% │                                          ████████████████████
      │
   0% └──────────────────────────────────────────────────────────────▶
      dia 1                       dia 15                         dia 30
```

### 3.2 Para que serve

O orçamento transforma uma discussão difícil ("estamos indo rápido demais?") em uma regra objetiva, combinada antes, chamada **política de orçamento de erro**:

| Situação do orçamento | O que a equipe faz |
|---|---|
| **Sobrando** | Pode arriscar mais: deploys mais frequentes, experimentos, migrações |
| **Acabando** | Cautela: deploys menores, mais revisão, prioridade para as causas das falhas |
| **Esgotado** | Congelam-se as mudanças que não sejam correções de confiabilidade até o orçamento se recuperar |

Isso conecta a observabilidade à entrega contínua. As métricas DORA do [primeiro capítulo de CD](../../03-entrega-continua/cd/01_entrega_e_implantacao_continua.md) mostraram que velocidade e estabilidade andam juntas; o orçamento de erro é o mecanismo que mantém esse equilíbrio no dia a dia. Uma equipe que nunca gasta o orçamento provavelmente está sendo cautelosa demais, e poderia entregar mais rápido.

---

## 4. Princípios de um bom alerta

Antes de escrever regras, vale fixar o que um alerta deve ser. O livro de SRE do Google resume em uma frase: **um alerta que acorda alguém deve exigir uma ação humana, urgente e inteligente**. Daí saem quatro princípios.

**1. Alerte sobre sintomas, não sobre causas.** Os usuários sentem **sintomas** (erros, demora), não causas (CPU alta, um Pod reiniciado). Um alerta de sintoma ("o checkout está falhando para 2% dos clientes") cobre **todas** as causas, inclusive as que ninguém previu. Um alerta de causa ("CPU acima de 80%") dispara quando não há problema algum e fica em silêncio quando o problema tem outra causa.

**2. Todo alerta precisa de uma ação.** Se a resposta a um alerta é "olhar e esperar passar", ele não deveria existir, ou deveria ser apenas um registro em um painel.

**3. Separe a urgência.** Nem tudo precisa acordar alguém às 3h:

| Tipo | Quando | Como chega |
|---|---|---|
| **Página** (*page*) | Usuários estão sendo afetados agora, e a situação vai piorar se ninguém agir | Notificação de plantão, que toca até alguém responder |
| **Chamado** (*ticket*) | Algo precisa de atenção em horas ou dias, mas não agora | Uma tarefa na fila da equipe, ou uma mensagem no canal |
| **Painel** | Informação útil, sem ação esperada | Só no Grafana |

**4. Cada alerta tem um roteiro.** O alerta deve dizer o que está acontecendo, qual o impacto, e onde está o **runbook**: um documento curto com os primeiros passos de investigação e as ações comuns (o painel a abrir, como fazer rollback, quem chamar).

> **Confusão comum:** "precisamos de alertas para CPU, memória, disco e reinícios de cada Pod, para não deixar nada passar". ✅ **Mais preciso:** esses alertas parecem prudentes, mas produzem exatamente a fadiga de alertas que faz as equipes ignorarem os avisos. A CPU a 85% em um serviço que responde rápido e sem erros não exige nada: o HPA do [capítulo de workloads](../../02-containers/kubernetes/02_workloads_rede_e_escala.md) existe justamente para isso. Um Pod que reinicia uma vez e volta é o Kubernetes funcionando. O que precisa acordar alguém é o **efeito no usuário**, medido pelos SLIs. Os sinais de causa continuam importantes, mas para **investigar** (nos painéis, depois que o alerta de sintoma chegou) e para alguns poucos alertas de **antecipação** que dão tempo de agir antes do sintoma (como o disco que vai encher em 4 horas, na seção 9).

---

## 5. Alertando sobre o orçamento: a taxa de consumo

### 5.1 Por que um limite simples não funciona

A primeira ideia seria: "alerte quando a taxa de erro passar de 0,1%", o próprio SLO. Ela falha nos dois sentidos:

- **Barulhenta demais:** um pico de 0,5% de erros durante 2 minutos dispara o alerta, mas consome uma fração ínfima do orçamento do mês. Ninguém precisava ser acordado.
- **Lenta demais:** se a janela for longa para evitar o barulho (por exemplo, 6 horas), uma queda total do checkout leva horas para ser detectada.

### 5.2 A taxa de consumo (*burn rate*)

A **taxa de consumo** compara a velocidade com que o orçamento está sendo gasto com a velocidade que o esgotaria **exatamente** no fim do período:

```
                     taxa de erro observada
 taxa de consumo = ──────────────────────────
                     taxa de erro do SLO (1 − SLO)
```

| Taxa de consumo | Com SLO de 99,9%, significa... | O orçamento de 30 dias acaba em... |
|---|---|---|
| 1 | 0,1% de erros | 30 dias (exatamente no prazo) |
| 6 | 0,6% de erros | 5 dias |
| 14,4 | 1,44% de erros | ~2 dias |
| 1.000 | 100% de erros (queda total) | 43 minutos |

Alertar sobre a **taxa de consumo** em vez da taxa de erro resolve os dois problemas: o alerta passa a significar "neste ritmo, vamos estourar o objetivo", e a urgência é proporcional à velocidade do consumo.

### 5.3 Várias janelas, várias urgências

O *Site Reliability Workbook* recomenda combinar **duas janelas por alerta** (uma longa, para ter certeza de que o problema é relevante, e uma curta, para que o alerta pare logo depois que o problema for resolvido) e **dois níveis de urgência**:

| Urgência | Janela longa | Janela curta | Taxa de consumo | Significa | Orçamento consumido ao disparar |
|---|---|---|---|---|---|
| **Página** | 1 hora | 5 minutos | 14,4 | O orçamento do mês acaba em ~2 dias | 2% |
| **Página** | 6 horas | 30 minutos | 6 | Acaba em ~5 dias | 5% |
| **Chamado** | 3 dias | 6 horas | 1 | Acaba no prazo, sem folga | 10% |

O alerta dispara só quando **as duas janelas** estão acima da taxa. A janela longa garante que não é um pico passageiro; a curta garante que o problema ainda está acontecendo agora.

---

## 6. As regras no Prometheus

### 6.1 Regras de gravação: pré-calculando as taxas

Calcular a taxa de erro em várias janelas, em cada avaliação de cada alerta, é caro. As **regras de gravação** (*recording rules*) calculam uma expressão periodicamente e guardam o resultado como uma série nova, que os alertas e painéis consultam:

```yaml
# regras/slo-checkout.yml
groups:
  - name: slo-checkout-sli
    interval: 30s
    rules:
      - record: loja:checkout_erros:taxa5m
        expr: |
          sum(rate(http_server_request_duration_seconds_count{service_name="loja", http_route="/checkout", http_response_status_code=~"5.."}[5m]))
          /
          sum(rate(http_server_request_duration_seconds_count{service_name="loja", http_route="/checkout"}[5m]))

      - record: loja:checkout_erros:taxa30m
        expr: |
          sum(rate(http_server_request_duration_seconds_count{service_name="loja", http_route="/checkout", http_response_status_code=~"5.."}[30m]))
          /
          sum(rate(http_server_request_duration_seconds_count{service_name="loja", http_route="/checkout"}[30m]))

      - record: loja:checkout_erros:taxa1h
        expr: |
          sum(rate(http_server_request_duration_seconds_count{service_name="loja", http_route="/checkout", http_response_status_code=~"5.."}[1h]))
          /
          sum(rate(http_server_request_duration_seconds_count{service_name="loja", http_route="/checkout"}[1h]))

      - record: loja:checkout_erros:taxa6h
        expr: |
          sum(rate(http_server_request_duration_seconds_count{service_name="loja", http_route="/checkout", http_response_status_code=~"5.."}[6h]))
          /
          sum(rate(http_server_request_duration_seconds_count{service_name="loja", http_route="/checkout"}[6h]))
```

O nome segue a convenção do Prometheus para regras de gravação, `nível:métrica:operação`: fica claro, só pelo nome, de onde a série vem.

### 6.2 Regras de alerta

```yaml
  - name: slo-checkout-alertas
    rules:
      - alert: CheckoutConsumindoOrcamentoRapido
        expr: |
          loja:checkout_erros:taxa1h > (14.4 * 0.001)
          and
          loja:checkout_erros:taxa5m > (14.4 * 0.001)
        labels:
          severity: page
          service_name: loja
        annotations:
          summary: "Checkout com {{ $value | humanizePercentage }} de erros: o orçamento do mês acaba em ~2 dias"
          runbook_url: "https://wiki.exemplo.com/runbooks/checkout-erros"

      - alert: CheckoutConsumindoOrcamento
        expr: |
          loja:checkout_erros:taxa6h > (6 * 0.001)
          and
          loja:checkout_erros:taxa30m > (6 * 0.001)
        labels:
          severity: page
          service_name: loja
        annotations:
          summary: "Checkout com {{ $value | humanizePercentage }} de erros nas últimas horas"
          runbook_url: "https://wiki.exemplo.com/runbooks/checkout-erros"
```

| Campo | Para quê |
|---|---|
| `expr` | A condição. Enquanto ela devolver algum resultado, o alerta está ativo |
| `for` (opcional) | Exige que a condição se mantenha por um tempo antes de disparar. Com duas janelas, normalmente não é necessário |
| `labels` | Classificam o alerta. O `severity` decide o encaminhamento no Alertmanager (seção 7) |
| `annotations` | O texto para as pessoas: resumo com o valor atual (`$value`) e o link do runbook |

E, na configuração do Prometheus, os arquivos de regras e o endereço do Alertmanager:

```yaml
# prometheus.yml (acréscimos)
rule_files:
  - /etc/prometheus/regras/*.yml

alerting:
  alertmanagers:
    - static_configs:
        - targets: ["alertmanager:9093"]
```

Se você acrescentar essas regras ao laboratório do [capítulo anterior](01_stack_de_observabilidade.md), o primeiro alerta vai **disparar e não parar**: o serviço de pagamentos recusa 5% dos pagamentos por construção, e 5% de erros é uma taxa de consumo de 50 para um SLO de 99,9%. É uma boa lição: ou o sistema está muito abaixo do objetivo, ou o objetivo foi escolhido sem medir antes o que o sistema entrega. Na prática, o primeiro SLO de um serviço costuma ser definido **depois** de algumas semanas medindo o SLI, um pouco acima do que o sistema já entrega, e é ajustado com o tempo.

---

## 7. O Alertmanager: do alerta à pessoa certa

O Prometheus decide **quando** um alerta está ativo. O **Alertmanager** decide **o que fazer com ele**: para quem enviar, por qual canal, como agrupar alertas parecidos e quando ficar em silêncio.

```
 Prometheus ──alertas ativos──▶ Alertmanager
                                    │
                    ┌───────────────┼──────────────────┐
                    │ agrupa        │ encaminha        │ silencia / inibe
                    │ (30 Pods com  │ (severity=page → │ (manutenção programada;
                    │  o mesmo erro │  plantão; o resto│  um alerta "serviço fora
                    │  = 1 aviso)   │  → canal da      │  do ar" esconde os alertas
                    │               │  equipe)         │  derivados dele)
                    └───────────────┴──────────────────┘
                                    │
                    ┌───────────────┴──────────────┐
                    ▼                              ▼
          plantão (PagerDuty, Opsgenie)    canal da equipe (Slack, e-mail)
```

```yaml
# alertmanager.yml
route:
  receiver: canal-da-equipe              # o destino padrão
  group_by: [alertname, service_name]    # alertas iguais do mesmo serviço viram um só aviso
  group_wait: 30s                        # espera 30 s para juntar alertas que chegam juntos
  group_interval: 5m                     # novos alertas do mesmo grupo: no máximo a cada 5 min
  repeat_interval: 4h                    # lembra a cada 4 h enquanto o alerta continuar ativo
  routes:
    - matchers: [severity="page"]
      receiver: plantao

receivers:
  - name: canal-da-equipe
    slack_configs:
      - api_url_file: /etc/alertmanager/segredos/slack_url
        channel: "#loja-alertas"
        send_resolved: true              # avisa também quando o problema acabar

  - name: plantao
    pagerduty_configs:
      - routing_key_file: /etc/alertmanager/segredos/pagerduty_key

inhibit_rules:
  - source_matchers: [alertname="LojaForaDoAr"]   # se a loja inteira está fora do ar...
    target_matchers: [severity="ticket"]          # ...não envie os chamados derivados
    equal: [service_name]
```

As URLs e chaves dos destinos são lidas de **arquivos** (`api_url_file`, `routing_key_file`), e não escritas na configuração, pelo mesmo motivo de toda a trilha: o arquivo de configuração vai para o Git, os segredos não.

Dois recursos completam o Alertmanager:

- **Silêncios** (*silences*): pausar alertas por um período, por exemplo durante uma manutenção programada, pela interface ou pela linha de comando (`amtool`).
- **Inibições**: a regra `inhibit_rules` acima evita que uma única causa gere dezenas de avisos. Se a loja inteira está fora do ar, os alertas de latência, de erros em cada rota e de cada dependência são consequências, e só atrapalham.

O Grafana também tem um sistema de alertas próprio (*Grafana Alerting*), com os mesmos conceitos (regras, encaminhamento, silêncios) e a vantagem de criar alertas sobre consultas de qualquer fonte, inclusive Loki e Tempo. Os princípios deste capítulo valem para os dois.

---

## 8. Depois do alerta

### 8.1 Plantão e runbooks

O alerta é só o começo. Uma equipe que recebe páginas precisa de:

- uma **escala de plantão** clara, com quem responde primeiro e quem é a segunda linha;
- **runbooks** curtos para cada alerta, com o painel a abrir, as perguntas a fazer (o fluxo métrica → trace → log do [capítulo de escolha do sinal](../sinais/04_escolhendo_o_sinal_certo.md)) e as ações conhecidas, como o rollback do [pipeline de CD](../../03-entrega-continua/cd/03_pipeline_de_cd_na_pratica.md);
- um critério simples para **escalar**: se em 15 minutos não há uma hipótese, chama-se mais alguém.

### 8.2 O tempo de recuperação começa na detecção

A métrica DORA de **tempo de recuperação** ([primeiro capítulo de CD](../../03-entrega-continua/cd/01_entrega_e_implantacao_continua.md)) se divide em etapas, e a observabilidade atua em todas:

```
 a falha começa ──▶ alguém é avisado ──▶ a causa é encontrada ──▶ o serviço volta
                │                   │                        │
           tempo para          tempo para                 tempo para
           DETECTAR            DIAGNOSTICAR               CORRIGIR
           (alertas de SLO)    (traces, logs,             (rollback automático,
                               correlação)                 feature flag)
```

Uma equipe que leva 40 minutos para perceber um problema não consegue um tempo de recuperação de 15 minutos, por mais rápido que seja o seu rollback.

### 8.3 Revisões pós-incidente

Depois de um incidente relevante, a equipe escreve uma **revisão pós-incidente** (*postmortem*): o que aconteceu, a linha do tempo, o impacto (quanto do orçamento de erro foi consumido), as causas e as ações para que não se repita. A regra mais importante é que ela seja **sem culpados** (*blameless*): o objetivo é entender por que o sistema e o processo permitiram o erro, e não quem errou. Uma pessoa que teme ser punida esconde informações, e sem informação não há aprendizado. Uma ação comum nessas revisões é justamente **acrescentar telemetria**: "não conseguimos saber qual bandeira estava falhando; vamos registrar a bandeira nos spans".

---

## 9. Alertas que não vêm de SLOs

Alguns alertas úteis não medem a experiência atual do usuário, mas antecipam problemas ou garantem que a própria observabilidade está funcionando:

```yaml
  - name: antecipacao
    rules:
      # O disco vai encher nas próximas 4 horas, pela tendência das últimas 6
      - alert: DiscoVaiEncherEm4Horas
        expr: predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[6h], 4 * 3600) < 0
        for: 30m
        labels: { severity: ticket }

      # O Collector parou de ser raspado: sem ele, nenhuma telemetria chega
      - alert: CollectorSemDados
        expr: absent(up{job="otel-collector"} == 1)
        for: 5m
        labels: { severity: page }

      # Um alerta que está SEMPRE ativo: se ele parar de chegar, o próprio caminho dos alertas quebrou
      - alert: Watchdog
        expr: vector(1)
        labels: { severity: none }
```

- **`predict_linear`** estende a tendência de um medidor para o futuro: é o tipo de alerta de **causa** que vale a pena, porque dá tempo de agir antes do sintoma.
- **`absent`** detecta a **falta** de dados. Um sistema de alertas que só olha para valores altos não percebe quando os valores **somem**, e um Collector parado significa zero erros nos painéis, o que parece ótimo.
- **`Watchdog`** é um alerta permanente, enviado a um serviço externo que avisa se ele **deixar** de chegar (o padrão *dead man's switch*). Ele protege contra a falha mais silenciosa de todas: o Prometheus ou o Alertmanager fora do ar, sem ninguém para alertar sobre isso.

A esses somam-se alertas de prazos conhecidos, como certificados TLS perto do vencimento e backups que não rodaram.

---

## 10. SLOs e a entrega contínua

Os SLIs deste capítulo são as mesmas consultas que tornam a entrega contínua segura:

- A **análise automática de um deploy canary** ([capítulo de estratégias de deploy](../../03-entrega-continua/cd/02_estrategias_de_deploy.md)) compara o SLI de erros e de latência da versão nova com o da estável, filtrando pelo `service.version`.
- O **smoke test** do [pipeline de CD](../../03-entrega-continua/cd/03_pipeline_de_cd_na_pratica.md) verifica alguns pontos logo após o deploy; os alertas de taxa de consumo continuam verificando nas horas seguintes.
- A **política de orçamento de erro** (seção 3) pode ser automatizada no pipeline: se o orçamento do mês estiver esgotado, o job de produção exige uma aprovação extra.

O [próximo capítulo](03_observabilidade_no_kubernetes.md) mostra a versão Kubernetes dessa integração, com um `AnalysisTemplate` do Argo Rollouts consultando o Prometheus.

---

## Próximo passo

Este capítulo definiu o que é "funcionar bem" (SLIs e SLOs), quanto de falha é aceitável (o orçamento de erro) e como alertar só quando importa (taxa de consumo, Alertmanager). Tudo foi feito sobre o laboratório do Compose, com três serviços em uma máquina.

O [03_observabilidade_no_kubernetes.md](03_observabilidade_no_kubernetes.md) leva a observabilidade para o cluster da parte de [Kubernetes](../../02-containers/SUMARIO.md): as métricas do próprio cluster, a coleta de logs dos nodes, os traces com o OpenTelemetry Operator, os eventos do Kubernetes, e a análise de métricas no deploy canary.
