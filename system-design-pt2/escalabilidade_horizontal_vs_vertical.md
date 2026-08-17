# Escalabilidade Horizontal vs. Escalabilidade Vertical

**Pré-requisito:** `../system-design-pt1/arquitetura.md`, `../system-design-pt1/escalabilidade.md`

> Baseado no Capítulo 1 (*Scale from Zero to Millions of Users*) de **System Design Interview — An Insider's Guide** (Alex Xu) e no Capítulo 5 (*Identifying Architectural Characteristics*) de **Fundamentals of Software Architecture** (Mark Richards & Neal Ford), que separa formalmente escalabilidade de elasticidade.

---

## Por que este arquivo existe

O arquivo `../system-design-pt1/escalabilidade.md` apresentou a tabela comparativa entre escala vertical e horizontal em meia página — o suficiente para seguir adiante e falar de CDN, sharding e rate limiting. Este arquivo volta a esse ponto e o abre por completo, porque **essa é a decisão que condiciona todas as outras**: quase todo componente que aparece nos arquivos seguintes desta trilha (load balancer, Nginx, API Gateway, réplicas de banco) só existe porque alguém decidiu escalar horizontalmente.

A pergunta central não é "qual das duas é melhor". É: *o que exatamente cada uma exige do sistema em troca da capacidade que oferece?*

---

## 1. O que significa "não aguentar mais"

Antes de escolher como escalar, é preciso entender o que se está tentando consertar. Um sistema sob carga crescente não passa de "funciona" para "não funciona" de forma abrupta — ele **degrada**, e degrada em uma curva bem característica:

```
Tempo de
resposta
    │                                          ╱
    │                                        ╱
    │                                     ╱
    │                              ╱ ╱ ╱          ← "joelho": a partir daqui,
    │                    ╱ ╱ ╱                       cada usuário novo custa
    │ ─────── ╱ ╱                                    desproporcionalmente mais
    │
    └──────────────────────────────────────────▶ Usuários simultâneos
      zona confortável    │  saturação  │  colapso
```

O que acontece nesse "joelho" é sempre a mesma coisa: **algum recurso finito saturou**. Pode ser CPU, memória, I/O de disco, banda de rede, número de conexões abertas no banco, ou o pool de threads da aplicação. Escalar é, na prática, **dar mais desse recurso escasso ao sistema** — e existem exatamente dois caminhos para isso.

> **Confusão comum:** "meu sistema está lento, então preciso escalar". ✅ **Mais preciso:** lentidão e falta de escala são sintomas diferentes, com causas diferentes. Se uma única requisição, feita por um único usuário em um sistema ocioso, já demora 3 segundos, o problema é de **performance** — uma query sem índice, um algoritmo ruim, uma chamada externa serial — e adicionar servidores vai simplesmente replicar essa lentidão em mais máquinas, a um custo maior. Escalabilidade só é a resposta quando a requisição é rápida sozinha e fica lenta **na presença de outras**. Medir a latência sob carga baixa antes de escalar é o que separa os dois diagnósticos.

---

## 2. Escala Vertical (*scale up*)

Escalar verticalmente é **trocar a máquina por uma maior**: mais núcleos de CPU, mais RAM, disco mais rápido (NVMe no lugar de SSD SATA), placa de rede de maior capacidade.

```
   ANTES                          DEPOIS

┌────────────┐                ┌──────────────────┐
│  Servidor   │                │    Servidor      │
│  4 vCPU     │   ───────▶     │    32 vCPU       │
│  8 GB RAM   │                │    256 GB RAM    │
└────────────┘                └──────────────────┘

A arquitetura da aplicação não muda em nada.
```

### O que a torna atraente

- **Custo zero de arquitetura.** Nenhuma linha de código muda. Não há balanceador para configurar, nem sessão para externalizar, nem cache local para reconciliar entre instâncias. Na nuvem, é literalmente trocar o tipo da instância e reiniciar.
- **Nenhuma nova classe de bug.** Sistemas distribuídos introduzem problemas que simplesmente não existem em uma máquina só: consistência entre nós, partições de rede, relógios dessincronizados, requisições parcialmente processadas. Escala vertical não traz nada disso.
- **O teto é mais alto do que a intuição sugere.** Alex Xu registra que a AWS oferece instâncias de banco de dados com **24 TB de RAM**, e que o Stack Overflow — com mais de 10 milhões de visitantes únicos mensais em 2013 — operava com **um único banco de dados master**. A maioria absoluta dos sistemas nunca chega perto de esgotar uma máquina moderna bem dimensionada.

### O que ela cobra

| Limitação | Por quê |
|---|---|
| **Teto físico rígido** | Existe um maior servidor disponível no mundo. Chegando nele, não há para onde ir — e a partir daí a única saída é uma reescrita arquitetural feita sob pressão, no pior momento possível. |
| **Custo superlinear** | O preço por unidade de capacidade **cresce** conforme a máquina fica maior. Dobrar a capacidade normalmente custa bem mais que o dobro, porque hardware de ponta é um mercado de nicho. |
| **Ponto único de falha (SPOF)** | Não existe redundância. Se aquela máquina cai — falha de hardware, kernel panic, manutenção do provedor — o sistema inteiro cai junto. |
| **Downtime para escalar** | Redimensionar uma instância exige, na prática, reiniciá-la. A operação de "escalar" é ela própria uma indisponibilidade programada. |

> **Confusão comum:** "escala vertical é uma abordagem ultrapassada — sistemas sérios escalam horizontalmente". ✅ **Mais preciso:** escala vertical é o **primeiro passo correto** para praticamente todo sistema, e continua sendo a escolha certa por muito mais tempo do que a cultura de engenharia costuma admitir. Ela é mais barata, mais simples de operar, mais fácil de depurar e não introduz nenhuma das falhas de sistemas distribuídos. Adotar escala horizontal cedo demais é uma forma clássica de complexidade acidental: você paga o custo permanente de operar um sistema distribuído (observabilidade distribuída, consistência, deploy coordenado) para resolver um problema de capacidade que ainda não existe. O critério para migrar não é "quantos usuários eu tenho", é: **disponibilidade virou um requisito de negócio, ou o teto da máquina está próximo?**

---

## 3. Escala Horizontal (*scale out*)

Escalar horizontalmente é **adicionar mais máquinas ao conjunto**, mantendo cada uma do mesmo tamanho.

```
   ANTES                          DEPOIS

┌────────────┐                      ┌─────────────────┐
│  Servidor   │                      │  Load Balancer  │
│  4 vCPU     │   ───────▶           └────────┬────────┘
│  8 GB RAM   │              ┌────────────────┼────────────────┐
└────────────┘               ▼                ▼                ▼
                       ┌──────────┐    ┌──────────┐    ┌──────────┐
                       │ 4 vCPU   │    │ 4 vCPU   │    │ 4 vCPU   │
                       │ 8 GB RAM │    │ 8 GB RAM │    │ 8 GB RAM │
                       └──────────┘    └──────────┘    └──────────┘

A arquitetura muda: surge um componente novo na frente,
e as instâncias precisam ser intercambiáveis entre si.
```

### O que ela entrega

- **Teto praticamente inexistente.** Sempre é possível adicionar mais uma máquina — o limite passa a ser orçamento e coordenação, não física.
- **Tolerância a falhas embutida.** Se uma instância cai, o load balancer para de rotear tráfego para ela e as demais continuam atendendo. A queda de uma máquina deixa de ser um incidente e vira um evento operacional rotineiro.
- **Elasticidade viável.** Com instâncias intercambiáveis, é possível ligar e desligar máquinas conforme a demanda (*auto scaling*) — algo impossível com uma única máquina grande.
- **Custo linear.** Dez máquinas pequenas custam aproximadamente dez vezes uma máquina pequena, e não o preço de nicho de uma máquina dez vezes maior.

### O que ela cobra: o pré-requisito escondido

O preço da escala horizontal **não é o load balancer** — esse é o custo visível e barato. O preço real é que **as instâncias precisam ser intercambiáveis**, e isso raramente é verdade por acidente. Alex Xu descreve isso como a diferença entre arquitetura *stateful* e *stateless*:

```
ARQUITETURA STATEFUL (não escala horizontalmente sem dor)

  Usuário A ──▶ Servidor 1  [sessão do A na memória, foto do A no disco local]
  Usuário B ──▶ Servidor 2  [sessão do B na memória]
  Usuário C ──▶ Servidor 3  [sessão do C na memória]

  Toda requisição do A PRECISA cair no Servidor 1. Se cair no 2,
  a autenticação falha — o Servidor 2 nunca ouviu falar do usuário A.


ARQUITETURA STATELESS (escala horizontalmente)

  Usuário A ──┐
  Usuário B ──┼──▶ [qualquer servidor] ──▶ ┌──────────────────────┐
  Usuário C ──┘                            │ Armazenamento comum   │
                                           │ (Redis / banco / S3)  │
                                           └──────────────────────┘

  Nenhum servidor guarda nada exclusivo. Qualquer um atende qualquer
  requisição. Adicionar, remover ou perder instâncias é trivial.
```

Na prática, tornar uma aplicação *stateless* significa auditar e mover para fora de cada instância:

| Estado escondido | Onde ele costuma estar | Para onde vai |
|---|---|---|
| Sessão de usuário | Memória do processo | Redis, banco, ou token assinado no cliente (JWT) |
| Arquivos enviados por usuários | Disco local (`/uploads`) | Object storage (S3, MinIO) |
| Cache em memória do processo | Dicionário local | Redis compartilhado |
| Jobs agendados (`cron`) | Cada instância roda o seu | Um agendador com *lock* distribuído, ou serviço dedicado |
| Contadores e rate limit | Variável local | Redis (ver `../system-design-pt1/redis.md`) |
| Conexões WebSocket | Ligadas a um processo específico | *Pub/Sub* entre instâncias, ou roteamento por afinidade |

> **Confusão comum:** "escalar horizontalmente é só subir mais instâncias atrás de um load balancer". ✅ **Mais preciso:** subir instâncias é a parte trivial; o trabalho de verdade é **eliminar o estado local**, e a tabela acima mostra por quê. É comum uma equipe adicionar a segunda instância, ver tudo funcionar nos testes, e só descobrir o problema em produção — usuários deslogados aleatoriamente (sessão em memória), imagens que aparecem para metade dos visitantes (upload em disco local), e-mails de cobrança enviados em duplicata (o `cron` agora roda em duas máquinas). Nenhum desses bugs existe com uma instância só, e todos aparecem juntos com a segunda.

> **Confusão comum:** "*sticky sessions* resolvem o problema de estado — o load balancer manda o usuário sempre para o mesmo servidor". ✅ **Mais preciso:** resolvem o sintoma imediato e **preservam todos os problemas de fundo**. Com afinidade de sessão, o servidor 2 continua sendo o único que conhece o usuário B — então se ele cair, o usuário B perde a sessão de qualquer forma; a carga fica desbalanceada (um servidor pode acumular usuários pesados enquanto outro fica ocioso); remover uma instância do pool passa a ter custo real; e *auto scaling* fica prejudicado, porque instâncias novas só recebem usuários novos. Alex Xu é direto ao classificar *sticky sessions* como "sobrecarga adicional". É uma ponte útil para uma migração gradual — não um destino.

---

## 4. Escalabilidade não é performance, disponibilidade nem elasticidade

Essas quatro características de arquitetura (as "-ilities" de `../system-design-pt1/arquitetura.md`) são frequentemente tratadas como sinônimos em conversas informais, e não são. *Fundamentals of Software Architecture* as separa explicitamente:

| Característica | Pergunta que responde | Como se mede |
|---|---|---|
| **Performance** | Quão rápido o sistema responde a **uma** requisição? | Latência (p50, p95, p99) sob carga controlada |
| **Escalabilidade** | O sistema mantém o desempenho conforme o número de **usuários simultâneos cresce**? | Curva de latência/throughput vs. usuários concorrentes |
| **Elasticidade** | O sistema aguenta **picos súbitos** de tráfego? | Tempo de reação a uma rampa abrupta de carga |
| **Disponibilidade** | Que fração do tempo o sistema está no ar? | *Uptime* (99,9%, 99,99%...) |

A distinção entre escalabilidade e elasticidade é a menos óbvia e a mais cara de errar. O livro usa dois exemplos que a tornam concreta:

```
ESCALÁVEL, MAS NÃO ELÁSTICO — sistema de reservas de hotel

Carga │ ────────────────────────────────
      │ Muitos usuários, mas volume previsível e estável.
      └──────────────────────────────────▶ tempo


ELÁSTICO — venda de ingressos de show

Carga │                    █
      │                    █
      │                    █
      │ ─────────────      █      ─────────────
      │ Ocioso...   ingressos abrem   ...ocioso de novo
      └──────────────────────────────────▶ tempo
```

Um sistema pode ser perfeitamente escalável (sustenta 100 mil usuários simultâneos) e ainda assim ruir em um pico, porque o problema do pico não é o **volume**, é a **derivada**: a carga chega mais rápido do que a capacidade consegue subir.

> **Confusão comum:** "*auto scaling* resolve picos de tráfego". ✅ **Mais preciso:** *auto scaling* resolve **crescimento**, não **pico**. O ciclo completo — a métrica ultrapassar o limiar, o alarme disparar, a instância ser provisionada, o sistema operacional subir, a aplicação inicializar, o *health check* passar, o load balancer começar a rotear — leva de dezenas de segundos a vários minutos. Um pico de venda de ingressos satura o sistema em segundos: quando as instâncias novas ficam prontas, o estrago já aconteceu. Elasticidade real exige medidas que agem *antes* — capacidade pré-aquecida, fila de admissão (*message queue*, ver Xu), rate limiting agressivo (`../system-design-pt1/escalabilidade.md`, seção 6), sala de espera, ou cache pesado na borda via CDN.

> **Confusão comum:** "escalar horizontalmente deixa o sistema mais rápido". ✅ **Mais preciso:** para o usuário individual em um sistema ocioso, escalar horizontalmente deixa o sistema **marginalmente mais lento** — surge um salto de rede a mais (o load balancer), e potencialmente uma consulta a um armazenamento de sessão externo que antes estava na memória local. O ganho aparece apenas sob concorrência: a latência do usuário 5.000 melhora enormemente, porque ele deixou de disputar CPU com os outros 4.999. Escalabilidade protege a latência **sob carga**; ela não melhora a latência mínima.

---

## 5. Por que dobrar as máquinas não dobra a capacidade

Existe uma expectativa intuitiva de que 10 servidores atendem 10 vezes mais que 1. Na prática, a curva real é bem diferente:

```
Capacidade
 real     │                    ╭──────────────  ← platô: o gargalo
          │                 ╭──╯                  compartilhado saturou
          │             ╭───╯
          │        ╭────╯       ← ideal (linear): ╱
          │    ╭───╯                              ╱
          │╭───╯                            ╱ ╱
          └──────────────────────────────────────▶ nº de servidores
```

Dois efeitos derrubam a linearidade:

1. **Recursos compartilhados que não escalaram junto.** Dez instâncias de aplicação apontando para o mesmo banco de dados não multiplicam a capacidade do sistema — elas multiplicam a **pressão sobre o banco**. O gargalo não desapareceu; apenas se moveu de camada. É exatamente por isso que a seção seguinte desta trilha trata de replicação de banco: escalar a camada de aplicação sem escalar a de dados só antecipa o próximo colapso.

2. **Custo de coordenação.** Quanto mais nós, mais esforço para mantê-los coerentes: invalidação de cache entre instâncias, *locks* distribuídos, sincronização de estado, consenso. Esse custo cresce com o número de nós — em certos regimes, adicionar mais uma máquina pode *reduzir* a capacidade total do sistema.

> **Confusão comum:** "escalar horizontalmente elimina o ponto único de falha". ✅ **Mais preciso:** elimina o SPOF **daquela camada específica**, e frequentemente cria um novo. Três instâncias de aplicação atrás de **um** load balancer: o load balancer virou o novo SPOF. Três instâncias apontando para **um** banco: o banco é o SPOF. Alta disponibilidade não é propriedade de um componente, é propriedade do **caminho inteiro** da requisição — e o caminho é tão disponível quanto seu elo mais frágil. Mapear o fluxo completo (DNS → load balancer → aplicação → cache → banco) e perguntar "o que acontece se este item cair?" em cada etapa é o exercício que revela onde a redundância realmente falta.

---

## 6. Escalando a camada de dados

A camada de aplicação é a fácil: ela é (ou pode se tornar) *stateless*, e instâncias *stateless* são intercambiáveis por definição. O banco de dados é o oposto — ele **é** o estado. Por isso as duas estratégias assumem formas próprias nessa camada:

| Estratégia | O que faz | Escala o quê | Onde está detalhado |
|---|---|---|---|
| **Vertical** | Máquina de banco maior | Leitura e escrita, até o teto do hardware | Esta seção |
| **Replicação (master/slave)** | Cópias que servem leitura | Apenas **leitura**; a escrita continua em um nó só | `arquitetura_master_slave.md` |
| **Sharding** | Particiona os dados entre servidores | Leitura, escrita **e** volume de dados | `../system-design-pt1/escalabilidade.md`, seção 5 |
| **Cache** | Evita a ida ao banco | Leitura, reduzindo a carga na origem | `../system-design-pt1/redis.md` |

> **Confusão comum:** "replicação e sharding são duas formas de fazer a mesma coisa (distribuir o banco entre máquinas)". ✅ **Mais preciso:** resolvem problemas **opostos**, e a maioria dos sistemas grandes usa os dois ao mesmo tempo. Na **replicação**, cada nó guarda uma cópia **completa** dos dados — o objetivo é redundância e capacidade de leitura. No **sharding**, cada nó guarda uma fatia **distinta** dos dados — o objetivo é caber e escrever mais rápido do que uma máquina permite. Repare que replicação **não resolve** "os dados não cabem" (toda réplica precisa caber inteira em cada nó) e **não resolve** "escrevo demais" (toda escrita passa pelo master e ainda é reaplicada em cada réplica). Sharding, por sua vez, não dá redundância sozinho — perder um shard significa perder aquela fatia dos dados. Daí a combinação padrão: **sharding para dividir, replicação dentro de cada shard para não perder nada.**

---

## 7. Como decidir, na prática

A escolha raramente é binária. A sequência que a maioria dos sistemas reais percorre é:

```
1. Uma máquina só
        │  "está apertado"
        ▼
2. Máquina maior (vertical)                    ← barato, rápido, sem reescrita
        │  "disponibilidade virou requisito"   ← este é o gatilho real
        ▼
3. Tornar a aplicação stateless                ← o trabalho de verdade
        │
        ▼
4. Load balancer + N instâncias (horizontal)   ← load_balancer.md
        │  "o banco agora é o gargalo"
        ▼
5. Réplicas de leitura + cache                 ← arquitetura_master_slave.md
        │  "os dados não cabem / escrita saturou"
        ▼
6. Sharding
```

Perguntas que orientam a decisão em cada etapa:

- **A indisponibilidade tem custo de negócio real?** Se sim, escala horizontal deixa de ser sobre capacidade e passa a ser sobre disponibilidade — e o argumento de "ainda cabe em uma máquina maior" perde a força.
- **O tráfego é constante ou tem picos?** Tráfego com picos exige elasticidade, e elasticidade exige instâncias intercambiáveis. Uma máquina grande não encolhe de madrugada.
- **Quanto custa a máquina maior vs. N máquinas menores?** Compare o preço real. Acima de certo porte, a máquina grande perde por larga margem.
- **A equipe consegue operar um sistema distribuído?** *Logging* centralizado, *tracing* distribuído, deploy coordenado, gestão de configuração. Sem isso, escala horizontal troca um problema de capacidade por um problema de operação — e o segundo costuma ser pior.

> **Confusão comum:** "as duas estratégias são mutuamente exclusivas — escolho uma". ✅ **Mais preciso:** sistemas reais quase sempre combinam as duas, e em camadas diferentes. O padrão mais comum em produção é **horizontal na aplicação, vertical no banco**: N instâncias pequenas e intercambiáveis servindo requisições, apontando para um banco único e propositalmente robusto. E mesmo dentro da escala horizontal, existe uma decisão vertical embutida: *qual o tamanho de cada nó do pool?* Muitas máquinas pequenas dão granularidade fina e falhas de baixo impacto, ao custo de mais overhead por nó; poucas máquinas grandes reduzem o overhead, ao custo de perder mais capacidade quando uma cai.

---

## Resumo do arquivo

| Dimensão | Vertical (*scale up*) | Horizontal (*scale out*) |
|---|---|---|
| **O que muda** | O tamanho da máquina | O número de máquinas |
| **Muda a arquitetura?** | Não | Sim — exige load balancer e aplicação *stateless* |
| **Teto** | Físico, rígido | Praticamente ilimitado |
| **Custo** | Superlinear | Aproximadamente linear |
| **Tolerância a falha** | Nenhuma (SPOF) | Alta, se todo o caminho for redundante |
| **Elasticidade** | Impossível | Possível (com ressalvas de tempo de resposta) |
| **Complexidade operacional** | Baixa | Alta |
| **Custo escondido** | O dia em que o teto chega | Eliminar todo o estado local |

- Escalar é **dar mais do recurso que saturou**; antes de escolher como, é preciso saber qual recurso é esse — e confirmar que o problema é de escala, não de performance.
- **Vertical** é o primeiro passo correto para quase todo sistema: mais barato, mais simples, sem novas classes de bug. Seu limite real não é técnico, é a ausência de redundância.
- **Horizontal** é o caminho quando disponibilidade vira requisito. O custo visível é o load balancer; o custo real é **tornar a aplicação *stateless***.
- **Escalabilidade, performance, elasticidade e disponibilidade são características distintas.** Escalar não deixa uma requisição mais rápida, e *auto scaling* não protege contra picos súbitos.
- Dobrar as máquinas não dobra a capacidade: o gargalo **se move de camada** (tipicamente para o banco) em vez de desaparecer.
- Na camada de dados, **replicação** (cópias completas, escala leitura) e **sharding** (fatias distintas, escala escrita e volume) resolvem problemas opostos e costumam ser usados juntos.

**Próxima leitura:** `load_balancer.md` — o componente que a escala horizontal torna obrigatório, e sem o qual "N instâncias" é só um conjunto de máquinas que ninguém sabe alcançar.
