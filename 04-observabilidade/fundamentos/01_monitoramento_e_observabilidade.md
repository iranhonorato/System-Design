# Monitoramento e Observabilidade

**Pré-requisito:** a parte de containers ([../../02-containers/SUMARIO.md](../../02-containers/SUMARIO.md)) e, de preferência, a de entrega contínua ([../../03-entrega-continua/SUMARIO.md](../../03-entrega-continua/SUMARIO.md)).

> O Capítulo 1 de **System Design Interview** (Alex Xu) inclui *logging, métricas e automação* entre os elementos indispensáveis de um sistema que cresceu, e propõe três categorias de métricas que este capítulo usa. O Capítulo 9 de **Fundamentals of Software Architecture** (Mark Richards & Neal Ford) aponta o *logging distribuído* como uma das dificuldades centrais das arquiteturas distribuídas, e o Capítulo 17 discute como padronizar o monitoramento entre equipes com o padrão sidecar. **Software Architecture: The Hard Parts** classifica logging e monitoramento como *acoplamento operacional*. A distinção entre monitoramento e observabilidade segue Charity Majors, Liz Fong-Jones e George Miranda (*Observability Engineering*, 2022).

---

## Por que este arquivo existe

A trilha de entrega contínua terminou com uma pergunta que vários capítulos deixaram em aberto: **como saber se o sistema está funcionando bem em produção?**

- O deploy *canary* do [capítulo de estratégias de deploy](../../03-entrega-continua/cd/02_estrategias_de_deploy.md) só avança se "a taxa de erros da versão nova" for aceitável. De onde vem esse número?
- O roteiro de diagnóstico do [capítulo do Compose ao cluster](../../02-containers/kubernetes/04_do_compose_ao_cluster.md) dependia de `kubectl logs`. E quando o Pod já foi recriado e os logs dele sumiram?
- As métricas DORA do [primeiro capítulo de CD](../../03-entrega-continua/cd/01_entrega_e_implantacao_continua.md) incluem o **tempo de recuperação**. Ninguém se recupera de uma falha que ainda não percebeu.

Esta pasta responde a essas perguntas. Este primeiro capítulo monta o mapa: qual é o problema, qual a diferença entre monitorar e observar, quais são os **sinais** (métricas, logs e traces) e que tipo de pergunta cada um responde. Os capítulos seguintes aprofundam cada sinal, ensinam a escolher o certo para cada situação e montam tudo na prática.

---

## 1. O problema: "o site está lento"

São 3 da manhã e chega uma mensagem: *"os clientes estão reclamando que o checkout está lento"*.

**Com um servidor só**, o caminho é conhecido. Você entra por SSH na EC2 do [capítulo de EC2](../../01-infraestrutura/aws/01_ec2_servidores_na_nuvem.md), roda `top` para ver a CPU, `docker compose logs -f api` para ler os logs e, com sorte, encontra a causa em alguns minutos. Tudo o que importa está em uma máquina.

**Com a arquitetura atual da loja**, o mesmo checkout passa por três serviços (`loja`, `pedidos` e `pagamentos`), cada um com várias réplicas, em Pods espalhados por vários nodes de um cluster:

```
 usuário ──▶ Ingress ──▶ loja ──▶ pedidos ──▶ pagamentos ──▶ operadora de cartão (externa)
                         ×4         ×3          ×3
                     (réplicas em Pods que mudam de nome, de node e de IP a cada deploy)
```

Agora as perguntas simples ficam difíceis:

- **Em qual dos três serviços está a lentidão?** Ou será no banco? Ou na operadora de cartão?
- **Em quais réplicas?** Todas, ou só as de um node com problema?
- **Desde quando?** Começou com o deploy das 18h, ou vem piorando aos poucos?
- **Para quem?** Todos os clientes, ou só os que pagam com um certo cartão?
- **Onde estão os logs?** Em 10 Pods diferentes, alguns já destruídos.

Richards e Ford descrevem exatamente esse cenário: arquiteturas distribuídas têm **dezenas a centenas de logs**, cada um em um lugar e com um formato diferente, o que torna difícil rastrear um problema, e as ferramentas de consolidar logs, sozinhas, "só arranham a superfície" dessa complexidade. Entrar em cada máquina para investigar deixa de ser viável. O sistema precisa **emitir, por conta própria, os dados que permitem entendê-lo de fora**. Esses dados se chamam **telemetria**, e a capacidade de usá-los para entender o sistema se chama **observabilidade**.

---

## 2. Monitoramento e observabilidade

Os dois termos são usados como sinônimos, mas descrevem coisas diferentes.

### 2.1 Monitoramento: vigiar o que já se sabe que pode falhar

**Monitorar** é acompanhar um conjunto de indicadores **escolhidos de antemão** e ser avisado quando eles saem do normal: CPU acima de 90%, disco quase cheio, taxa de erros acima de 1%, health check falhando. É o painel de um carro: velocidade, combustível, temperatura do motor, e uma luz que acende quando algo conhecido dá errado.

O monitoramento responde bem às **falhas previstas**: as que alguém já imaginou, para as quais alguém criou um gráfico e um alerta. São os *known unknowns*: você sabe que o disco pode encher, só não sabe quando.

### 2.2 Observabilidade: entender o que ninguém previu

O termo vem da **teoria de controle** (Rudolf Kálmán, 1960): um sistema é *observável* quando é possível deduzir seu estado interno a partir das suas saídas externas. Aplicado a software, a definição que se consolidou é:

> **Observabilidade** é a capacidade de entender o que está acontecendo dentro de um sistema, inclusive em situações que ninguém previu, **fazendo perguntas novas aos dados que ele já emite**, sem precisar alterar o código nem fazer um novo deploy para coletar mais informação.

A diferença está nas **falhas imprevistas**, os *unknown unknowns*. O checkout está lento, mas só para clientes de um estado, só com um tipo de cartão, só nas réplicas que foram para um node novo ontem. Nenhum painel foi criado para essa combinação, porque ninguém a imaginou. Em um sistema observável, é possível chegar a ela **filtrando e agrupando os dados existentes**: "mostre as requisições lentas de checkout das últimas duas horas, agrupadas por bandeira do cartão e por node".

| | Monitoramento | Observabilidade |
|---|---|---|
| **Pergunta típica** | "O indicador X saiu do normal?" | "Por que *isto* está acontecendo?" |
| **Tipo de falha** | Prevista (*known unknowns*) | Imprevista (*unknown unknowns*) |
| **Ferramenta típica** | Painéis e alertas pré-definidos | Consulta livre sobre dados ricos, correlação entre sinais |
| **Quando a pergunta é nova** | Exige criar um gráfico novo, às vezes um deploy | Responde com os dados que já existem |
| **Relação** | Uma **atividade** | Uma **propriedade** do sistema, que permite (entre outras coisas) monitorá-lo |

> **Confusão comum:** "observabilidade é só um nome novo para monitoramento" (ou "é uma ferramenta que se compra"). ✅ **Mais preciso:** monitoramento é uma **atividade** (vigiar indicadores conhecidos); observabilidade é uma **propriedade do sistema** (o quanto ele se deixa entender a partir do que emite). Um sistema observável é fácil de monitorar, mas um sistema monitorado não é necessariamente observável: é possível ter cinquenta painéis bonitos e, diante de uma falha nova, não ter dado nenhum que explique o que aconteceu. Por isso a observabilidade não se compra pronta: uma ferramenta só consegue mostrar os dados que a aplicação emite. Se os logs não dizem qual pedido falhou, se as métricas não separam o erro por rota, se os serviços não propagam o contexto de uma requisição para o seguinte, nenhum produto recupera essa informação depois. Ela é construída **no código**, por quem escreve o código, e é por isso que é um assunto de quem desenvolve, e não só de quem opera.

---

## 3. Os sinais

A telemetria chega em alguns formatos, chamados **sinais**. Os três principais costumam ser chamados de "pilares da observabilidade". Cada um é um tipo de dado diferente, com um custo diferente, e é bom para um tipo diferente de pergunta.

Uma analogia ajuda a separá-los. Pense em uma frota de caminhões de entrega:

- As **métricas** são o **painel de controle da frota**: quantas entregas por hora, a velocidade média, quantos caminhões estão parados. Números agregados, que mostram tendências e chamam atenção quando algo foge do normal, mas não contam a história de nenhuma entrega específica.
- Os **logs** são o **diário de bordo** de cada motorista: "08:12, saí do depósito", "09:40, cliente ausente, deixei com o vizinho". Registros detalhados de eventos, cada um com contexto, mas espalhados entre centenas de diários.
- Um **trace** é o **rastreamento GPS de uma entrega específica**, do depósito até a porta do cliente, mostrando quanto tempo foi gasto em cada trecho: 10 minutos no depósito, 25 na estrada, **40 parado no trânsito da ponte**, 5 na entrega. É o que mostra *onde* o tempo foi gasto em um percurso que atravessa vários lugares.

### 3.1 Métricas

Uma **métrica** é uma **medida numérica, agregada ao longo do tempo**: "requisições por segundo", "porcentagem de respostas com erro", "uso de memória", "tempo de resposta do percentil 95". Cada métrica gera uma **série temporal**, uma sequência de valores ao longo do tempo, que vira um gráfico.

```
 erros/min
   40 │                                         ╭─╮
   30 │                                        ╭╯ ╰╮
   20 │                                       ╭╯   ╰─╮
   10 │ ─╮╭──╮╭─╮╭───╮╭──╮╭─╮╭──╮╭───╮╭──╮╭──╯       ╰──
    0 └──────────────────────────────────────────────────▶
      00h      04h      08h      12h      16h      20h   24h
                                              ▲ algo mudou às 18h
```

A propriedade mais importante das métricas é que **o custo não cresce com o tráfego**: contar 10 ou 10 milhões de requisições por minuto ocupa o mesmo espaço, porque só o total é guardado. Isso as torna o sinal ideal para visões **agregadas**, **tendências** de longo prazo e **alertas**. O preço é que, ao agregar, se perde o detalhe: a métrica diz que 37 requisições falharam às 18h04, mas não diz quais.

### 3.2 Logs

Um **log** é um **registro de um evento discreto**, com horário e contexto: "18:04:12, pedido 4512, pagamento recusado pela operadora: cartão expirado". Ele carrega o **detalhe** que a métrica descarta: a mensagem de erro, o *stack trace*, o identificador do pedido, o cliente.

```json
{"horario": "2026-10-26T18:04:12Z", "nivel": "ERROR", "servico": "pagamentos",
 "mensagem": "operadora recusou o pagamento", "pedido_id": 4512,
 "motivo": "cartao_expirado", "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736"}
```

O preço do detalhe é o **volume**: cada evento gera um registro, então o custo cresce **junto com o tráfego**. Um sistema com muito movimento pode produzir terabytes de log por dia.

### 3.3 Traces

Um **trace** (rastro) registra **o caminho completo de uma única requisição** através de todos os serviços que ela atravessa. Ele é composto de **spans**: cada span é uma operação (uma chamada HTTP, uma consulta ao banco, um trecho de código) com início, duração e relação de "quem chamou quem". O resultado é uma visão em cascata:

```
 trace 4bf92f35...  POST /checkout                                    total: 1.640 ms
 ├─ loja        POST /checkout          ████████████████████████████████████████ 1.640 ms
 │  └─ loja → pedidos   POST            ███████████████████████████████████████  1.612 ms
 │     └─ pedidos  POST /pedidos         ██████████████████████████████████████   1.603 ms
 │        ├─ validar_estoque             █                                          21 ms
 │        └─ pedidos → pagamentos POST    █████████████████████████████████████   1.571 ms
 │           └─ pagamentos POST /pagamentos ████████████████████████████████████  1.565 ms
 │              └─ chamar_operadora          ███████████████████████████████████ 1.552 ms  ◀ aqui
```

É o único sinal que mostra **onde, ao longo de um caminho que atravessa vários serviços**, o tempo foi gasto ou o erro aconteceu. Na cascata acima, fica evidente que quase todo o 1,6 segundo do checkout foi gasto esperando a operadora de cartão, dentro do serviço de pagamentos.

### 3.4 Os sinais lado a lado

| | Métricas | Logs | Traces |
|---|---|---|---|
| **O que é** | Números agregados ao longo do tempo | Registros de eventos individuais | O caminho de uma requisição entre serviços |
| **Granularidade** | Agregada (por minuto, por serviço, por rota) | Um evento | Uma requisição, com todas as suas etapas |
| **Pergunta típica** | "Quanto? Com que frequência? Está piorando?" | "O que exatamente aconteceu neste evento?" | "Por onde a requisição passou e onde demorou?" |
| **Custo de armazenamento** | Baixo e estável (não cresce com o tráfego) | Alto (cresce com o tráfego) | Alto (normalmente se guarda só uma amostra) |
| **Bom para alertas?** | Sim: é o sinal natural para alertar | Pouco (caro e lento para agregar) | Pouco (amostrado) |
| **Retenção típica** | Meses a anos | Dias a semanas | Dias |
| **Ferramentas abertas comuns** | Prometheus | Loki, Elasticsearch/OpenSearch | Jaeger, Tempo |

As duas perguntas mais clássicas da área ficam respondidas pela própria tabela, e o [capítulo de escolha do sinal](../sinais/04_escolhendo_o_sinal_certo.md) as discute em detalhe:

- *Quantas requisições falharam em cada minuto, e qual a tendência ao longo do dia?* Uma pergunta **agregada** e **ao longo do tempo**: **métricas**.
- *Uma requisição passa por três microsserviços e demora muito: em qual chamada está a maior demora?* Uma pergunta sobre o **caminho de uma requisição**, através de vários serviços: **traces**.

### 3.5 Outros sinais

Além dos três principais, aparecem com frequência:

- **Eventos:** registros de mudanças importantes no sistema, como um deploy, uma alteração de configuração ou um Pod reiniciado. Marcados sobre os gráficos de métricas, eles respondem a "o que mudou às 18h?".
- **Profiles** (perfis contínuos): amostras de **onde a CPU e a memória são gastas dentro do código**, função por função. Respondem a "qual função está consumindo a CPU deste serviço?", algo que nenhum dos outros três sinais mostra. O OpenTelemetry está padronizando os profiles como o quarto sinal.

---

## 4. Os sinais se completam

Nenhum sinal sozinho resolve uma investigação. O fluxo típico usa os três, em sequência, cada um levando ao próximo:

```
 ① MÉTRICA                     ② TRACE                          ③ LOG
 "o que está errado?"          "onde está o problema?"          "por que aconteceu?"

 o alerta dispara:             abre-se um trace de uma           filtram-se os logs pelo
 a latência p95 do checkout    requisição lenta daquele          trace_id: o serviço de
 passou de 1,5 s às 18h        período: o tempo está todo no     pagamentos registrou
                               span "chamar_operadora", dentro   "operadora respondendo em
          │                    do serviço de pagamentos          1,5 s, timeout aumentado
          │                             │                        para 2 s no deploy das 18h"
          └──── exemplar ───────────────┘──────── trace_id ──────────────┘
```

Para que essa navegação funcione, os sinais precisam **compartilhar identificadores**:

- o **nome do serviço** igual nos três (`pagamentos`, e não `pagamentos` nas métricas e `payment-svc` nos logs);
- o **`trace_id`** dentro de cada log, para ir de um trace aos logs daquela requisição;
- **exemplares** (*exemplars*) nas métricas: um `trace_id` de exemplo anexado a um ponto do gráfico, para ir de "a latência subiu aqui" a "eis uma requisição lenta deste momento".

Montar essa correlação é o objetivo da [parte prática](../pratica/01_stack_de_observabilidade.md).

---

## 5. O que observar

Saber **quais** sinais existem não diz **o que** medir. Alex Xu propõe três categorias de métricas, que valem como ponto de partida para qualquer sistema:

| Categoria | Exemplos na loja | Quem se importa |
|---|---|---|
| **Nível de máquina** (*host*) | CPU, memória, disco, rede de cada node e de cada container | Quem opera a infraestrutura |
| **Nível agregado** (de um componente inteiro) | Latência e taxa de erros do serviço de pagamentos; desempenho da camada de banco ou de cache como um todo | Quem desenvolve e opera os serviços |
| **Negócio** | Checkouts por minuto, taxa de pagamentos aprovados, receita por hora, usuários ativos | Produto e negócio, e todos os outros |

As métricas de negócio costumam ser esquecidas, e são justamente as que detectam os problemas mais graves. Um serviço pode estar com CPU normal, latência normal e zero erros HTTP e, mesmo assim, ter parado de aprovar pagamentos porque uma regra nova recusa todos os cartões de uma bandeira. Nenhuma métrica técnica percebe isso; a métrica "pagamentos aprovados por minuto" cai a zero na hora.

O [capítulo de métricas](../sinais/01_metricas.md) apresenta os métodos mais usados para escolher as métricas técnicas de cada componente: os **quatro sinais de ouro** do livro de SRE do Google, o método **RED** (para serviços) e o método **USE** (para recursos).

---

## 6. O caminho da telemetria

Todo sistema de observabilidade, de qualquer fornecedor, tem as mesmas quatro etapas:

```
 ① INSTRUMENTAR        ② COLETAR             ③ ARMAZENAR             ④ CONSULTAR
 o código emite os     um agente recebe,     cada sinal vai para     painéis, buscas,
 sinais                processa e envia      um banco próprio        alertas

 ┌─────────────┐      ┌──────────────┐      ┌────────────────────┐   ┌──────────────┐
 │ loja        │─┐    │              │ ───▶ │ métricas: Prometheus│──▶│              │
 │ pedidos     │─┼──▶ │  Collector   │ ───▶ │ logs:     Loki      │──▶│   Grafana    │──▶ alertas
 │ pagamentos  │─┘    │              │ ───▶ │ traces:   Tempo     │──▶│              │
 └─────────────┘      └──────────────┘      └────────────────────┘   └──────────────┘
  SDK do OpenTelemetry   OpenTelemetry Collector
```

| Etapa | O que faz | Exemplos |
|---|---|---|
| **Instrumentação** | Faz o código emitir métricas, logs e traces | SDKs do **OpenTelemetry**, bibliotecas de cliente do Prometheus |
| **Coleta** | Recebe os dados, enriquece (acrescenta o nome do Pod, por exemplo), filtra, amostra e envia | **OpenTelemetry Collector**, Grafana Alloy, Fluent Bit |
| **Armazenamento** | Guarda cada sinal em um banco otimizado para ele | **Prometheus** (métricas), **Loki** ou Elasticsearch (logs), **Tempo** ou Jaeger (traces) |
| **Consulta e alerta** | Painéis, buscas, navegação entre sinais, alertas | **Grafana**, Alertmanager |

Há também plataformas comerciais que fazem as quatro etapas (Datadog, New Relic, Dynatrace, Honeycomb, Grafana Cloud) e serviços dos provedores de nuvem (na AWS, CloudWatch e X-Ray; em versões gerenciadas, *Amazon Managed Service for Prometheus* e *Amazon Managed Grafana*). Esta pasta usa a pilha aberta da figura, porque ela roda em qualquer lugar, inclusive na sua máquina, e os conceitos são os mesmos em todas.

O [próximo capítulo](02_telemetria_e_opentelemetry.md) trata das duas primeiras etapas, e de por que o **OpenTelemetry** se tornou o padrão para elas.

---

## 7. De quem é a responsabilidade

*Software Architecture: The Hard Parts* chama logging, monitoramento, descoberta de serviços e autenticação de **acoplamento operacional**: tudo aquilo que todos os serviços precisam ter, de forma consistente, e que não é regra de negócio. O problema que Richards e Ford levantam em *Fundamentals of Software Architecture* é prático: se cada equipe implementa o monitoramento do próprio serviço do seu jeito, **como garantir que todas façam**, e que façam da mesma forma? E quem cuida de atualizar a ferramenta de monitoramento em cinquenta serviços?

A resposta da arquitetura é padronizar a parte comum: com o **padrão sidecar** (do [capítulo de Compose e produção](../../02-containers/docker/04_compose_e_producao.md)), com bibliotecas compartilhadas ou, como esta pasta mostra, com um **padrão aberto de telemetria** (o OpenTelemetry) e um **coletor comum**. A parte que não pode ser padronizada, o que é importante medir **naquele domínio** (pagamentos aprovados, pedidos com estoque insuficiente), continua sendo de quem conhece o domínio.

Isso fecha o argumento do movimento DevOps do [capítulo de fundamentos de infraestrutura](../../01-infraestrutura/fundamentos/01_o_que_e_infraestrutura.md): quem constrói o serviço é quem melhor sabe o que ele precisa contar sobre si mesmo. *You build it, you run it* (quem constrói, opera) só é viável se quem constrói também torna o sistema observável.

> **Confusão comum:** "quanto mais dados coletarmos, mais observável o sistema fica". ✅ **Mais preciso:** observabilidade depende da **qualidade** dos dados, não da quantidade. Logs de nível DEBUG em produção, métricas com milhões de combinações de rótulos e 100% dos traces guardados podem custar mais que a própria infraestrutura da aplicação, deixar as consultas lentas e esconder o sinal útil no meio do ruído. Um sistema bem observável emite **poucos dados bem escolhidos e bem conectados**: métricas que cobrem os sinais de ouro e o negócio, logs estruturados com o `trace_id` e com o contexto que importa (qual pedido, qual motivo), e traces amostrados de forma inteligente (todos os erros e as requisições lentas, e só uma fração das normais). Cada capítulo de sinal desta pasta tem uma seção sobre o que **não** coletar.

---

## Próximo passo

Este capítulo apresentou o problema, a diferença entre monitorar e observar, os sinais e o caminho da telemetria. Antes de aprofundar cada sinal, falta entender **como o código passa a emiti-los**: se cada ferramenta exige uma biblioteca diferente, trocar de ferramenta significaria reescrever a instrumentação de todos os serviços.

O [02_telemetria_e_opentelemetry.md](02_telemetria_e_opentelemetry.md) apresenta o OpenTelemetry, o padrão aberto que resolve esse problema: suas peças (API, SDK, instrumentação automática, Collector, o protocolo OTLP) e as convenções de nomes que fazem os sinais de serviços diferentes conversarem entre si.
