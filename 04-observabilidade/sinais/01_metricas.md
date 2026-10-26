# Métricas

**Pré-requisito:** [../fundamentos/02_telemetria_e_opentelemetry.md](../fundamentos/02_telemetria_e_opentelemetry.md)

> Segue a documentação do Prometheus (*Data model*, *Metric types*, *Querying*, *Histograms and summaries*, *Naming*) e das convenções semânticas do OpenTelemetry. Os **quatro sinais de ouro** vêm do Capítulo 6 de *Site Reliability Engineering* (Google, 2016); o método **RED** é de Tom Wilkie e o **USE**, de Brendan Gregg. As três categorias de métricas (máquina, agregada, negócio) são as do Capítulo 1 de **System Design Interview** (Alex Xu).

---

## Por que este arquivo existe

Uma equipe quer saber **quantas requisições falharam em cada minuto** e **observar a tendência ao longo do dia**: se os erros estão estáveis, crescendo devagar, ou se houve um pico às 18h. Qual sinal responde a isso?

Os logs têm cada erro registrado, então daria para contá-los. Mas contar milhões de linhas de log a cada minuto, para cada minuto do dia, é lento e caro, e fica impreciso se os logs forem amostrados. Os traces também registram as falhas, mas costumam ser **amostrados** (só uma fração das requisições é guardada), e uma contagem feita sobre uma amostra não é a contagem real. A pergunta é sobre **quantidades agregadas ao longo do tempo**, e esse é exatamente o trabalho das **métricas**: um contador de requisições com erro, incrementado pela aplicação, guardado como uma série temporal e consultado com uma função de taxa. A resposta, em PromQL, aparece na seção 4.

Este capítulo explica como as métricas funcionam por dentro: o que é uma série temporal, os tipos de métrica, o modelo do Prometheus, a linguagem de consulta, por que a média engana e como os percentis resolvem, o que medir em cada componente, e o erro que mais derruba sistemas de métricas, a **cardinalidade**.

---

## 1. O que é uma métrica

Uma **métrica** é uma medida numérica de algum aspecto do sistema, registrada ao longo do tempo. Cada combinação de **nome** e **rótulos** (*labels*) forma uma **série temporal**: uma sequência de pares (horário, valor).

```
 nome da métrica                          rótulos (labels)
 ┌──────────────────────────────────┐ ┌───────────────────────────────────────────────┐
 http_server_request_duration_seconds_count{service_name="loja", http_route="/checkout",
                                           http_response_status_code="502"}

 horário    18:00:00  18:00:15  18:00:30  18:00:45  18:01:00  ...
 valor          1021      1024      1029      1031      1040  ...   ← uma série temporal
```

Duas propriedades definem as métricas e explicam quando usá-las:

1. **São agregadas.** Uma métrica não guarda cada requisição, guarda um **número que as resume**: quantas foram, quanto tempo levaram no total, em quais faixas de duração caíram. Por isso ela responde muito bem a "quantas?", "com que frequência?" e "está piorando?", e não responde a "qual requisição?".
2. **O custo não depende do tráfego.** Uma série temporal ocupa o mesmo espaço com 10 ou 10 milhões de requisições por minuto, porque só o total é atualizado. O que aumenta o custo é o **número de séries** (seção 7), e não o volume de eventos.

Essas duas propriedades tornam as métricas o sinal certo para **painéis**, **tendências de longo prazo** (meses de histórico custam pouco) e **alertas** (consultar um número é rápido e barato).

---

## 2. Os tipos de métrica

### 2.1 Contador (*counter*)

Um número que **só aumenta** (ou volta a zero quando o processo reinicia). Conta coisas que acontecem: requisições, erros, pedidos criados, bytes enviados.

```
 checkouts_total
  1200 │                                  ╱
  1000 │                            ╱────╯
   800 │                     ╱─────╯
   600 │              ╱─────╯
   400 │        ╱────╯                       ← o valor bruto só cresce...
   200 │  ╱────╯
     0 └──────────────────────────────────▶
```

O valor bruto de um contador quase nunca interessa ("1.200 checkouts desde que o processo subiu" não diz nada). O que interessa é a **taxa**: quanto ele cresceu por segundo, ou em cada minuto. Essa é a função `rate()` da seção 4.

### 2.2 Medidor (*gauge*)

Um número que **sobe e desce**. Mede um estado em um instante: memória em uso, conexões abertas no pool do banco, tamanho de uma fila, temperatura, número de réplicas prontas.

```
 conexoes_em_uso
   20 │      ╭╮         ╭─╮
   15 │  ╭──╯╰╮   ╭───╯  ╰╮        ← o valor em si é o que importa
   10 │ ╭╯     ╰──╯        ╰───╮
    5 │─╯                       ╰──
    0 └──────────────────────────────▶
```

### 2.3 Histograma (*histogram*)

Registra a **distribuição** de valores observados, contando quantos caíram em cada faixa (*bucket*). É o tipo usado para durações e tamanhos. Um histograma de duração de requisições gera, na prática, várias séries:

```
 http_server_request_duration_seconds_bucket{le="0.05"}   812    ← requisições com duração ≤ 50 ms
 http_server_request_duration_seconds_bucket{le="0.1"}    955    ← ≤ 100 ms (inclui as de cima)
 http_server_request_duration_seconds_bucket{le="0.25"}   990
 http_server_request_duration_seconds_bucket{le="1"}      996
 http_server_request_duration_seconds_bucket{le="2.5"}   1024
 http_server_request_duration_seconds_bucket{le="+Inf"}  1024    ← todas
 http_server_request_duration_seconds_sum                 187.3  ← soma de todas as durações (s)
 http_server_request_duration_seconds_count              1024    ← quantas requisições
```

O rótulo `le` significa *less or equal* (menor ou igual): cada bucket conta as observações **até** aquele limite, por isso os valores são acumulados. A partir dos buckets, calculam-se os **percentis** (seção 5). E repare em um detalhe que vai importar na seção 4: o `_count` de um histograma de duração **também é um contador de requisições**. Não é preciso uma métrica separada para contar requisições: o histograma já conta.

O Prometheus tem ainda o tipo **summary**, que calcula os percentis na própria aplicação. Ele tem uma limitação séria: percentis calculados em réplicas diferentes não podem ser combinados (a confusão da seção 5 explica por quê). Para serviços com várias réplicas, prefira histogramas.

### 2.4 Os instrumentos do OpenTelemetry

No código, os tipos aparecem como **instrumentos** da API do OpenTelemetry ([capítulo de telemetria](../fundamentos/02_telemetria_e_opentelemetry.md)):

| Instrumento OTel | Tipo no Prometheus | Exemplo na loja |
|---|---|---|
| `Counter` | counter | `pagamentos.processados`, `loja.checkouts` |
| `UpDownCounter` | gauge | Itens no carrinho, requisições em andamento |
| `Histogram` | histogram | `http.server.request.duration`, `pagamentos.operadora.duracao` |
| `Gauge` / `ObservableGauge` | gauge | Memória em uso, tamanho da fila (lidos periodicamente) |

```python
from opentelemetry import metrics

meter = metrics.get_meter("loja")

checkouts = meter.create_counter(
    "loja.checkouts", unit="{checkout}", description="Checkouts finalizados, por resultado"
)

# No ponto do código em que o checkout termina:
checkouts.add(1, {"resultado": "sucesso"})      # ou {"resultado": "falha"}
```

---

## 3. O modelo do Prometheus

O **Prometheus** é o sistema de métricas de código aberto mais usado, e o padrão de fato no Kubernetes. Ele foi criado na SoundCloud em 2012 e foi o segundo projeto a entrar na CNCF, depois do próprio Kubernetes.

### 3.1 Coleta por *pull*: o Prometheus vai buscar

No modelo clássico, a aplicação **expõe** suas métricas em um endpoint HTTP (normalmente `/metrics`), em texto, e o Prometheus **busca** (*scrape*, raspa) esse endpoint periodicamente, por exemplo a cada 15 segundos:

```
             a cada 15 s: GET /metrics
 Prometheus ────────────────────────────▶ loja:8000/metrics
     │       ◀──────────────────────────  # TYPE loja_checkouts_total counter
     │                                    loja_checkouts_total{resultado="sucesso"} 1187
     │                                    loja_checkouts_total{resultado="falha"} 37
     ▼
 banco de séries temporais (TSDB) local
```

Cada endpoint raspado é um **alvo** (*target*), e o Prometheus acrescenta a cada série os rótulos `job` (o grupo de alvos) e `instance` (o endereço do alvo). Os alvos podem ser descobertos automaticamente: no Kubernetes, o Prometheus consulta a API do cluster e passa a raspar cada Pod novo sem configuração manual ([capítulo de Kubernetes](../pratica/03_observabilidade_no_kubernetes.md)).

O *pull* tem uma vantagem pouco óbvia: se um alvo para de responder, o Prometheus **percebe** (a série `up{job="loja"}` vai a zero). Em um modelo em que as aplicações enviam os dados, uma aplicação morta simplesmente fica em silêncio.

Também existe o caminho **push**: a aplicação envia as métricas (por OTLP, por exemplo) para um Collector, que as expõe para o Prometheus raspar, ou as envia ao Prometheus, que desde a versão 3 aceita receber OTLP. É o caminho usado na [parte prática](../pratica/01_stack_de_observabilidade.md), e o resultado no Prometheus é o mesmo.

### 3.2 Convenções de nomes

| Convenção | Exemplo | Por quê |
|---|---|---|
| Unidades de base no nome | `_seconds`, `_bytes` (nunca `_ms` ou `_mb`) | Evita confundir milissegundos com segundos em uma conta |
| Contadores terminam em `_total` | `loja_checkouts_total` | Deixa claro que é preciso aplicar `rate()` |
| Nome descreve o que é medido; rótulos, as dimensões | `http_server_request_duration_seconds{http_route="/checkout"}` | E não uma métrica separada por rota (`checkout_duration_seconds`) |

Ao passar pelo Collector, os nomes do OpenTelemetry são convertidos para esse padrão: `http.server.request.duration` (unidade `s`) vira `http_server_request_duration_seconds`, e o contador `loja.checkouts` vira `loja_checkouts_total`.

### 3.3 Retenção e armazenamento

O Prometheus guarda as séries em um banco local, por padrão por 15 dias. Para meses ou anos de histórico, alta disponibilidade e consultas sobre vários clusters, ele é combinado com sistemas de armazenamento de longo prazo compatíveis (Thanos, Grafana Mimir, Cortex) ou com serviços gerenciados (como o *Amazon Managed Service for Prometheus*).

---

## 4. PromQL: consultando métricas

A **PromQL** é a linguagem de consulta do Prometheus, usada nos painéis do Grafana e nas regras de alerta.

### 4.1 Selecionando séries

```promql
# Todas as séries desta métrica
http_server_request_duration_seconds_count

# Filtrando por rótulos: = igual, != diferente, =~ expressão regular, !~ não casa
http_server_request_duration_seconds_count{service_name="loja", http_response_status_code=~"5.."}
```

Uma consulta como essa devolve o **valor atual** de cada série (um *instant vector*). Acrescentando uma janela de tempo entre colchetes, ela devolve **todos os valores** daquela janela (um *range vector*), que é o que as funções de taxa precisam:

```promql
http_server_request_duration_seconds_count{service_name="loja"}[5m]
```

### 4.2 Taxas: `rate` e `increase`

| Função | O que calcula | Exemplo de leitura |
|---|---|---|
| `rate(contador[5m])` | A taxa média **por segundo** nos últimos 5 minutos | "2,4 requisições por segundo" |
| `increase(contador[1m])` | Quanto o contador **aumentou** no último minuto | "144 requisições neste minuto" |

As duas tratam automaticamente as **reinicializações**: quando um Pod reinicia, o contador volta a zero, e a função entende a queda como um recomeço, e não como uma taxa negativa. É o principal motivo para nunca fazer contas manuais com o valor bruto de um contador.

### 4.3 A pergunta do início: falhas por minuto, ao longo do dia

```promql
sum(
  increase(
    http_server_request_duration_seconds_count{service_name="loja", http_response_status_code=~"5.."}[1m]
  )
)
```

Lendo de dentro para fora:

1. **Seleciona** as séries de contagem de requisições do serviço `loja` cujo código de resposta começa com 5 (erros do servidor).
2. **`increase(...[1m])`**: para cada série (cada réplica, cada rota, cada código), calcula quantas requisições com erro houve no último minuto.
3. **`sum(...)`**: soma tudo, resultando em **um número**: falhas por minuto no serviço inteiro.

Colocada em um painel do Grafana com o intervalo de **24 horas**, essa consulta é calculada para cada ponto do gráfico, e o resultado é a curva de tendência do dia:

```
 falhas/min (loja)
   40 │                                              ╭╮
   30 │                                             ╭╯╰╮
   20 │                                            ╭╯  ╰╮       ← pico às 18h
   10 │╭╮ ╭╮  ╭─╮ ╭╮ ╭─╮ ╭╮  ╭╮ ╭─╮ ╭╮ ╭─╮ ╭╮ ╭╮ ╭╯    ╰─╮
    0 └┴┴─┴┴──┴─┴─┴┴─┴─┴─┴┴──┴┴─┴─┴─┴┴─┴─┴─┴┴─┴┴─┴────────┴──▶
      00h        06h        12h         18h         24h
```

Variações úteis da mesma consulta:

```promql
# Separando por serviço: uma linha para cada um
sum by (service_name) (increase(http_server_request_duration_seconds_count{http_response_status_code=~"5.."}[1m]))

# A TAXA de erro (proporção): mais útil que o número absoluto, porque não sobe só porque o tráfego subiu
  sum(rate(http_server_request_duration_seconds_count{service_name="loja", http_response_status_code=~"5.."}[5m]))
/
  sum(rate(http_server_request_duration_seconds_count{service_name="loja"}[5m]))

# Uma métrica de negócio: checkouts que falharam por minuto
sum(increase(loja_checkouts_total{resultado="falha"}[1m]))
```

A **taxa de erro** merece atenção: 40 falhas por minuto com 400 requisições por minuto (10%) é um incidente; 40 falhas por minuto com 40 mil (0,1%) pode ser normal. Painéis e alertas costumam usar a proporção, e o [capítulo de SLOs](../pratica/02_slos_e_alertas.md) a transforma no indicador central de disponibilidade.

Um ponto a decidir: **o que conta como falha?** Respostas `5xx` são erros do servidor. Respostas `4xx` normalmente são erros do cliente (uma senha errada, um produto inexistente) e não indicam que o sistema está com problema, mas há exceções: um `429 Too Many Requests` em excesso é sinal de saturação, e um aumento brusco de `401` depois de um deploy pode ser um defeito na autenticação. A consulta deve refletir essa decisão, e a decisão deve estar escrita.

> **Confusão comum:** "para saber o total de requisições por segundo, somo os contadores e depois aplico o `rate`" (`rate(sum(...))`). ✅ **Mais preciso:** a ordem certa é **`rate` primeiro, `sum` depois**: `sum(rate(...[5m]))`. O `rate` precisa olhar cada contador **individualmente** para perceber quando ele reinicia (uma queda para zero significa que o processo reiniciou). Se os contadores de três réplicas forem somados antes, a reinicialização de uma delas aparece como uma pequena queda na soma, que o `rate` não reconhece como reinício, e o resultado sai errado justamente nos momentos de instabilidade, quando ele mais importa. Na PromQL, a regra é sempre: **taxa por série, depois agregação**. (E o próprio Prometheus nem aceita `rate(sum(x))` diretamente, porque `rate` exige um *range vector*; o erro costuma aparecer escondido em subconsultas.)

### 4.4 Agregações

| Operação | Exemplo |
|---|---|
| `sum by (rótulo)` | Total por serviço: `sum by (service_name) (rate(...))` |
| `avg`, `min`, `max` | Memória média, mínima, máxima entre réplicas |
| `topk(5, ...)` | As 5 rotas mais lentas, os 5 Pods com mais memória |
| `avg_over_time(gauge[1h])`, `max_over_time(gauge[1h])` | Média ou pico de um medidor em uma janela |

Um alerta comum: `rate()` e `increase()` são funções **para contadores**. Aplicá-las a um medidor (memória, tamanho de fila) produz números sem sentido, porque um medidor que desce seria interpretado como reinício. Para medidores, use os valores diretamente, ou as funções `*_over_time`.

---

## 5. Latência: por que a média engana

### 5.1 O problema

Considere 100 requisições ao checkout: 99 levaram 50 ms e uma levou 10 segundos.

```
 média   = (99 × 0,05 + 10) ÷ 100 = 0,15 s    ← "o checkout leva 150 ms"
 mediana = 0,05 s                             ← metade leva até 50 ms
 p99     = 10 s                               ← 1% dos clientes espera 10 segundos
```

A média (150 ms) descreve **nenhuma** requisição real: não é a experiência da maioria (50 ms) nem a do cliente que esperou 10 segundos. E ela esconde exatamente o que importa: uma parte dos usuários está tendo uma experiência péssima.

Os **percentis** respondem melhor: o **p95** é o valor abaixo do qual estão 95% das requisições; o **p99**, 99%. "p95 = 300 ms" significa "95% das requisições levaram até 300 ms, e 5% levaram mais". Em sistemas com muito tráfego, esses 5% são milhares de pessoas por hora, e frequentemente os clientes mais valiosos (os que têm carrinhos maiores, que fazem mais chamadas).

### 5.2 Calculando percentis a partir de um histograma

```promql
# p95 da latência do checkout nos últimos 5 minutos
histogram_quantile(
  0.95,
  sum by (le) (
    rate(http_server_request_duration_seconds_bucket{service_name="loja", http_route="/checkout"}[5m])
  )
)
```

1. `rate(..._bucket[5m])`: a taxa de cada bucket (quantas requisições por segundo caíram em cada faixa).
2. `sum by (le)`: soma as réplicas, **mantendo** o rótulo `le` (as faixas).
3. `histogram_quantile(0.95, ...)`: estima, a partir das faixas, o valor abaixo do qual estão 95% das requisições.

É uma **estimativa**: o Prometheus sabe quantas requisições caíram entre 250 ms e 500 ms, mas não onde exatamente dentro da faixa. A precisão depende dos limites dos buckets, que por isso devem ficar próximos dos valores importantes (por exemplo, um limite exatamente no objetivo de 300 ms). O OpenTelemetry usa, para `http.server.request.duration`, limites padrão de 5 ms a 10 s (`0.005, 0.01, 0.025, 0.05, 0.075, 0.1, 0.25, 0.5, 0.75, 1, 2.5, 5, 7.5, 10`), adequados para APIs comuns. Versões recentes do Prometheus e do OpenTelemetry também suportam **histogramas nativos (exponenciais)**, que ajustam as faixas automaticamente e dão mais precisão com menos séries.

> **Confusão comum:** "cada réplica calcula seu p95, e o p95 do serviço é a média dos p95 das réplicas". ✅ **Mais preciso:** percentis **não podem ser combinados** por média (nem por soma, nem por máximo). Imagine duas réplicas: uma atendeu 1.000 requisições com p95 de 100 ms, a outra atendeu 10 requisições com p95 de 5 s. A média dos p95 (2,55 s) não tem relação nenhuma com o p95 real das 1.010 requisições, que fica perto de 100 ms. O jeito correto é combinar os **dados brutos** (os buckets dos histogramas de todas as réplicas, somados) e só então calcular o percentil, como faz o `sum by (le)` acima. É por isso que histogramas são preferíveis aos *summaries* (que entregam o percentil pronto, por réplica) e a qualquer métrica que já venha como "p95" calculado.

---

## 6. O que medir

Saber consultar não diz **o que** medir. Três métodos complementares são usados para cobrir um sistema sem esquecer nada importante, somados às métricas de negócio de Alex Xu do [capítulo de fundamentos](../fundamentos/01_monitoramento_e_observabilidade.md).

### 6.1 Os quatro sinais de ouro

O livro de SRE do Google afirma que, se só for possível medir quatro coisas de um sistema voltado a usuários, devem ser estas:

| Sinal | Pergunta | Exemplo na loja (PromQL) |
|---|---|---|
| **Latência** | Quanto tempo leva? (separando sucesso de erro: um erro rápido não é uma requisição rápida) | `histogram_quantile(0.95, sum by (le) (rate(http_server_request_duration_seconds_bucket{service_name="loja"}[5m])))` |
| **Tráfego** | Quanta demanda está chegando? | `sum(rate(http_server_request_duration_seconds_count{service_name="loja"}[5m]))` |
| **Erros** | Que proporção está falhando? | A taxa de erro da seção 4.3 |
| **Saturação** | Quão "cheio" está o recurso mais limitado? | Uso de memória em relação ao limit; conexões em uso no pool; tamanho da fila |

### 6.2 RED: para cada serviço

O método **RED** é uma versão compacta dos sinais de ouro, pensada para **serviços** (tudo o que recebe requisições):

- **R**ate: requisições por segundo;
- **E**rrors: requisições com erro por segundo (ou a proporção);
- **D**uration: a distribuição da duração (os percentis).

Um painel RED para cada serviço (`loja`, `pedidos`, `pagamentos`) responde, em segundos, à primeira pergunta de qualquer incidente: **qual serviço** está com problema.

### 6.3 USE: para cada recurso

O método **USE** é pensado para **recursos** (CPU, memória, disco, rede, pools de conexão, filas):

- **U**tilization: quanto do recurso está em uso (CPU a 70%);
- **S**aturation: quanto trabalho está **esperando** por ele (a fila de processos esperando CPU, as requisições esperando uma conexão livre do pool);
- **E**rrors: falhas do recurso (erros de disco, conexões recusadas).

A saturação é o sinal que costuma faltar, e o que mais avisa com antecedência. Um pool de conexões com 20 conexões, 20 em uso e 35 requisições esperando por uma está saturado, mesmo com a CPU do banco a 30%. É o tipo de causa que o RED mostra como "latência alta no serviço de pedidos", e que só o USE explica.

### 6.4 Juntando: o que medir na loja

| Componente | Método | Métricas |
|---|---|---|
| `loja`, `pedidos`, `pagamentos` | RED | Taxa, erros e percentis de `http.server.request.duration`, por rota |
| Chamadas entre serviços e à operadora | RED (do lado do cliente) | `http.client.request.duration`; o histograma manual `pagamentos.operadora.duracao` |
| Pods e nodes | USE | CPU, memória em relação ao limit, *throttling* de CPU, reinícios ([capítulo de Kubernetes](../pratica/03_observabilidade_no_kubernetes.md)) |
| Postgres e Redis | USE | Conexões em uso e em espera, consultas lentas, uso de memória do Redis |
| Negócio | Métricas próprias | Checkouts por minuto por resultado, pagamentos aprovados, valor vendido |

---

## 7. Cardinalidade: o erro que derruba sistemas de métricas

### 7.1 Cada combinação de rótulos é uma série

O custo de um sistema de métricas é proporcional ao **número de séries temporais**, e cada **combinação distinta de valores de rótulos** é uma série nova. O número de séries de uma métrica é, no pior caso, o **produto** do número de valores de cada rótulo:

```
 http_server_request_duration_seconds_bucket
   × 20 rotas (http_route)
   × 8 códigos de resposta (http_response_status_code)
   × 3 métodos (http_request_method)
   × 15 buckets (le)
   × 30 réplicas
   = 216.000 séries          ← já é bastante, mas o Prometheus lida bem
```

Agora alguém acrescenta um rótulo `cliente_id` "para poder filtrar por cliente", com 200 mil clientes ativos:

```
   216.000 × 200.000 = 43.200.000.000 séries     ← o Prometheus fica sem memória e cai
```

Esse fenômeno se chama **explosão de cardinalidade**, e é a forma mais comum de derrubar um servidor de métricas, ou de multiplicar a conta de um serviço comercial que cobra por série.

### 7.2 Regras práticas

| Bom rótulo (poucos valores, limitados) | Mau rótulo (valores ilimitados) |
|---|---|
| `http_route="/pedidos/{id}"` (o **modelo** da rota) | `url_path="/pedidos/4512"` (o caminho com o ID: um valor por pedido) |
| `http_response_status_code="502"` | A mensagem de erro completa |
| `resultado="falha"` | `pedido_id`, `cliente_id`, `email`, `trace_id` |
| `bandeira="visa"` | O número do cartão (e, além disso, um dado sensível) |
| `service_version="a1b2c3d"` (poucas versões ao mesmo tempo) | Um timestamp |

As convenções semânticas do OpenTelemetry já ajudam: a métrica de duração HTTP usa `http.route` (o modelo, como `/pedidos/{id}`), e não o caminho real da URL.

> **Confusão comum:** "preciso saber quais clientes estão tendo erro, então vou colocar o `cliente_id` como rótulo da métrica de erros". ✅ **Mais preciso:** métricas são o sinal errado para perguntas sobre **itens individuais** (este cliente, este pedido, esta requisição). Elas foram feitas para agregar, e um rótulo com valores ilimitados destrói justamente a propriedade que as torna baratas. A pergunta "quais clientes tiveram erro?" pertence aos **logs** (cada erro registrado com o `cliente_id`) ou aos **traces** (o `cliente_id` como atributo do span), que guardam detalhes individuais por natureza. A métrica responde "quantos erros, e está piorando?"; os logs e traces respondem "quem, qual, por quê". O [capítulo de escolha do sinal](04_escolhendo_o_sinal_certo.md) organiza essa divisão.

---

## 8. Onde as métricas param

As métricas dizem **que** algo está errado, **quanto** e **desde quando**, e fazem isso de forma barata, para qualquer volume de tráfego. Mas, por serem agregadas, elas não dizem **qual** requisição, **qual** cliente, nem **por que**. Quando o painel mostra que o p95 do checkout passou de 1,5 segundo às 18h, a próxima pergunta, "onde esse tempo está sendo gasto?", já não é uma pergunta de métricas.

Há uma ponte entre os mundos: os **exemplares** (*exemplars*). Ao registrar uma observação no histograma, o SDK do OpenTelemetry pode anexar a ela o `trace_id` da requisição que estava em andamento. No Grafana, esses exemplares aparecem como pontos sobre o gráfico de latência, e clicar em um ponto da faixa lenta abre **o trace de uma requisição lenta daquele momento**. É o primeiro passo do fluxo métrica → trace → log do [capítulo de fundamentos](../fundamentos/01_monitoramento_e_observabilidade.md), montado na [parte prática](../pratica/01_stack_de_observabilidade.md).

---

## Próximo passo

As métricas apontaram o problema: erros subindo às 18h, latência alta no serviço de pagamentos. Para saber **o que** exatamente aconteceu em cada falha (a mensagem, o pedido, o motivo da recusa), é preciso um sinal que guarde cada evento com seus detalhes.

O [02_logs.md](02_logs.md) trata dos logs: por que estruturá-los, como usar os níveis, como conectá-los aos traces, para onde vão quando a aplicação roda em containers, como consultá-los, e o que nunca deve ser registrado.
