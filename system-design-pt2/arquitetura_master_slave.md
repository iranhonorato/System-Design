# Arquitetura Master/Slave de Banco de Dados

**Pré-requisito:** `escalabilidade_horizontal_vs_vertical.md`, `../system-design-pt1/arquitetura.md` (Teorema CAP), `../system-design-pt1/escalabilidade.md`

> Baseado na seção *Database replication* do Capítulo 1 de **System Design Interview** (Alex Xu) e no Capítulo 9 (*Data Ownership and Distributed Transactions* — BASE e **Eventual Consistency Patterns**) de **Software Architecture: The Hard Parts**.

---

## Por que este arquivo existe

Os arquivos anteriores desta trilha resolveram a camada de aplicação: instâncias *stateless*, load balancer, gateway. Todos partem da mesma premissa — **as instâncias são intercambiáveis porque não guardam nada**.

O banco de dados não tem essa saída. Ele **é** o estado. Não existe "banco *stateless*", e por isso a camada de dados exige uma família inteira de técnicas próprias. A primeira e mais fundamental delas é a **replicação master/slave**.

`../system-design-pt1/escalabilidade.md`, seção 3, apresentou o modelo em meia página. Este arquivo abre o que acontece por baixo: como a replicação de fato ocorre, o que é *replication lag* e por que ele produz bugs que não aparecem em nenhum teste local, como um failover realmente funciona e por que ele é mais delicado do que os diagramas sugerem.

---

## 1. O problema

Depois de escalar a aplicação horizontalmente, o desenho fica assim:

```
                 ┌─────────────────┐
   Usuários ────▶│  Load Balancer  │
                 └────────┬────────┘
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
   [Instância 1]    [Instância 2]    [Instância 3]
        │                │                │
        └────────────────┼────────────────┘
                         ▼
                ┌──────────────────┐
                │   BANCO ÚNICO    │  ◀── SPOF: se cair, TUDO cai
                │                  │  ◀── GARGALO: 3 instâncias
                └──────────────────┘        disputam o mesmo banco
```

Dois problemas distintos, que a replicação resolve em graus diferentes:

1. **Ponto único de falha.** Toda a redundância construída na camada de aplicação é inútil se existe um só banco.
2. **Gargalo de leitura.** A maioria absoluta das aplicações lê muito mais do que escreve — proporções de 10:1 a 1000:1 são comuns. Todo esse volume de leitura bate em uma única máquina.

---

## 2. O modelo

```
                    ESCRITAS (INSERT, UPDATE, DELETE)
                              │
                              ▼
                    ┌──────────────────┐
                    │    MASTER        │  única fonte de verdade
                    │  (fonte única    │  para escrita
                    │   de escrita)    │
                    └────────┬─────────┘
                             │ fluxo de replicação
                             │ (log de transações)
            ┌────────────────┼────────────────┐
            ▼                ▼                ▼
      ┌──────────┐     ┌──────────┐     ┌──────────┐
      │ SLAVE 1  │     │ SLAVE 2  │     │ SLAVE 3  │
      └──────────┘     └──────────┘     └──────────┘
            ▲                ▲                ▲
            └────────────────┴────────────────┘
                    LEITURAS (SELECT)
```

As regras são simples e rígidas:

- **O master aceita escritas** e é a única fonte de verdade. Toda operação que modifica dados vai para ele.
- **Os slaves recebem uma cópia completa** dos dados via fluxo de replicação e **só aceitam leituras**.
- Como as leituras são a maioria, **o número de slaves costuma ser maior que o de masters** — o sistema distribui o volume dominante entre várias máquinas.

Os ganhos, na formulação de Alex Xu: **desempenho** (leituras em paralelo entre vários nós), **confiabilidade** (os dados existem em várias máquinas, potencialmente em locais físicos diferentes) e **disponibilidade** (o sistema continua servindo leitura mesmo com um nó fora).

### Uma nota sobre a terminologia

O termo *master/slave* é o histórico e ainda é o mais reconhecível — por isso está no título deste arquivo. A indústria, no entanto, migrou para outros nomes, e você vai encontrar todos eles na documentação atual:

| Sistema | Terminologia atual |
|---|---|
| MySQL | **source / replica** (desde a versão 8.0.22) |
| PostgreSQL | **primary / standby** (ou *replica*) |
| MongoDB | **primary / secondary** |
| Redis | **primary / replica** (`replicaof` substituiu `slaveof` na versão 5) |
| Literatura de sistemas distribuídos | **leader / follower** |

Vale conhecer os dois vocabulários: o antigo aparece em material de estudo e em sistemas legados; o novo, na documentação e nas ferramentas atuais.

---

## 3. Como a replicação funciona por baixo

O mecanismo é sempre o mesmo, independentemente do banco: **o master já mantém um log sequencial de tudo que modifica**, porque precisa dele para durabilidade e recuperação de falha. A replicação simplesmente **envia esse log para outra máquina, que o reaplica**.

```
   MASTER                                    SLAVE

   1. Recebe UPDATE
   2. Aplica na sua cópia
   3. Grava no log de transações
      (binlog no MySQL,
       WAL no PostgreSQL)
              │
              │  ── envia o log pela rede ──▶  4. Recebe e grava localmente
              │                                5. Reaplica as operações
              │                                   na sua própria cópia
              ▼
   6. Responde ao cliente
      (quando, exatamente? → seção 4)
```

Duas variações importam na prática:

**Replicação física vs. lógica.** A física envia mudanças em nível de bytes/páginas do disco (é o padrão do *streaming replication* do PostgreSQL) — muito eficiente, mas exige versão e plataforma idênticas, e replica o banco inteiro. A lógica envia as mudanças em nível de linhas ou comandos — mais flexível (permite replicar apenas algumas tabelas, ou entre versões diferentes) e mais custosa.

**Baseada em comando vs. em linha.** Replicar o comando `UPDATE pedidos SET status='pago' WHERE id=42` é compacto, mas perigoso: comandos com `NOW()`, `RAND()` ou `UUID()` produzem resultados **diferentes** em cada nó. Replicar o resultado (linha 42 mudou para estes valores) é maior em volume e determinístico. Por isso a replicação baseada em linhas é hoje o padrão recomendado na maioria dos bancos.

---

## 4. Síncrona, assíncrona e semissíncrona

Este é o trade-off central, e é exatamente o **C vs. A** do Teorema CAP discutido em `../system-design-pt1/arquitetura.md`, seção 5, materializado em uma opção de configuração.

A pergunta é uma só: **em que momento o master responde "ok" ao cliente?**

```
ASSÍNCRONA                      SEMISSÍNCRONA              SÍNCRONA
─────────────────────────────────────────────────────────────────────────
Master grava                    Master grava               Master grava
Master responde OK ✓            Espera 1 slave CONFIRMAR   Espera slaves APLICAREM
Depois envia ao slave           que RECEBEU                Só então responde OK ✓
                                Só então responde OK ✓

Latência: mínima                Latência: + 1 ida e volta  Latência: + ida e volta
                                                             e escrita remota
Perda em falha do master:       Perda: praticamente nula   Perda: nenhuma
transações que ainda não
haviam sido enviadas

Disponível se os slaves         Bloqueia se nenhum slave   Bloqueia se um slave
caírem: SIM                     confirmar (há timeout)     estiver lento ou fora
```

| | Assíncrona | Semissíncrona | Síncrona |
|---|---|---|---|
| **Latência da escrita** | Menor | Média | Maior |
| **Risco de perder dados** | Real | Muito baixo | Nenhum |
| **Disponibilidade da escrita** | Máxima | Alta | Reduzida — depende dos slaves |
| **No CAP** | Privilegia **A** | Meio-termo | Privilegia **C** |
| **Uso típico** | Padrão da maioria dos sistemas | Sistemas com dado sensível | Financeiro, quando a perda é inaceitável |

O PostgreSQL torna esse espectro explícito no parâmetro `synchronous_commit`, que aceita desde `off` (nem espera o disco local) até `remote_apply` (espera o standby ter **aplicado** a mudança, não só recebido). O MySQL oferece o plugin de replicação semissíncrona. A escolha é por *cluster* — e, em alguns bancos, ajustável por transação, o que permite usar síncrona apenas nas operações que realmente exigem.

> **Confusão comum:** "escolhendo replicação síncrona, elimino todos os problemas de consistência". ✅ **Mais preciso:** você elimina a janela de perda de dados e paga por isso em **latência e disponibilidade** — e a conta pode ser alta. Com replicação síncrona, toda escrita passa a custar pelo menos uma ida e volta de rede até o slave, o que degrada o throughput de escrita de forma perceptível, ainda mais se as réplicas estiverem em outra zona de disponibilidade. Pior: se o slave síncrono ficar lento ou cair, o master **para de aceitar escritas** — a garantia de consistência transformou a réplica, criada para aumentar a disponibilidade, num novo ponto de falha para a escrita. É o Teorema CAP cobrando exatamente o que ele promete cobrar, e é por isso que a maioria dos sistemas em produção usa replicação assíncrona e trata as consequências (seção 5) na aplicação.

---

## 5. Replication lag: onde os bugs nascem

Com replicação assíncrona, existe sempre uma janela — normalmente de milissegundos, ocasionalmente de segundos ou minutos — em que **o slave está atrasado em relação ao master**. Esse atraso é o *replication lag*, e ele é a origem de uma classe inteira de bugs que **não aparece em desenvolvimento** (onde há um banco só) e **não aparece em testes** (onde não há carga).

### O bug clássico: *read-your-own-writes*

```
  t=0ms    Usuário salva o perfil        ──▶  MASTER  (escrita commitada)
  t=5ms    Aplicação responde 200 OK
  t=8ms    Frontend redireciona e recarrega o perfil
  t=10ms   Aplicação lê o perfil          ──▶  SLAVE   (ainda com o dado ANTIGO)
  t=12ms   Usuário vê os dados antigos.
           "Não salvou!" — e salva de novo.
  t=40ms   A replicação alcança o slave. Agora está certo.
```

O usuário acabou de fazer uma alteração e não a vê. Do ponto de vista dele, o sistema está quebrado — e do ponto de vista de qualquer log, tudo funcionou perfeitamente.

### Os outros dois padrões de anomalia

- **Leituras não-monotônicas:** duas leituras seguidas caem em slaves com atrasos diferentes, e o usuário vê o dado "voltar no tempo" — um comentário que aparece e depois some.
- **Violação de causalidade:** a resposta a uma pergunta chega a um slave antes da pergunta, e a interface exibe uma conversa fora de ordem.

### Soluções, da mais simples à mais sofisticada

| Estratégia | Como funciona | Custo |
|---|---|---|
| **Ler do master após escrever** | Depois de um `write`, aquele usuário lê do master por N segundos | Simples; concentra carga no master |
| **Afinidade de sessão a um slave** | Cada sessão sempre lê da mesma réplica | Garante monotonicidade, não *read-your-writes* |
| **Leitura por marca de progresso** | A aplicação guarda a posição do log no momento da escrita e só lê de réplicas que já a alcançaram (GTID no MySQL, LSN no PostgreSQL) | Correto e preciso; exige suporte no driver ou no proxy |
| **Rotas críticas sempre no master** | Saldo, checkout, estoque nunca vão para réplica | Trivial de implementar; exige classificar as rotas |
| **Monitorar e alarmar o lag** | Retirar do pool de leitura a réplica atrasada acima de um limiar | Complementa qualquer das anteriores |

> **Confusão comum:** "replicação assíncrona é praticamente instantânea, então na prática dá para tratar o slave como se estivesse sempre atualizado". ✅ **Mais preciso:** o atraso típico é mesmo de poucos milissegundos — e é justamente isso que torna o problema traiçoeiro, porque o bug **não reproduz** em desenvolvimento e passa em todos os testes. Ele aparece só em produção, de forma intermitente, para uma fração dos usuários, e tipicamente logo depois de uma ação de escrita — o cenário mais difícil de diagnosticar que existe. Pior: o lag **não é constante**. Ele explode exatamente nos piores momentos: durante uma carga em lote, uma migração de schema, um pico de escrita, uma reindexação, uma rede degradada. É comum ver o lag saltar de 5 ms para 30 segundos justamente na hora em que o sistema está sob pressão. Tratar o slave como atualizado é assumir que a pior hora do sistema nunca vai chegar.

> **Confusão comum:** "o `SELECT` do meu ORM lê da réplica, então a carga já está distribuída". ✅ **Mais preciso:** só se alguém tiver configurado isso — separação de leitura e escrita **não acontece sozinha**. E ela quebra em um caso específico e muito frequente: uma leitura **dentro de uma transação de escrita** precisa ir para o master, porque só ele enxerga as alterações ainda não commitadas daquela transação. ORMs configurados ingenuamente para mandar todo `SELECT` para a réplica produzem exatamente esse bug. Bibliotecas maduras (e proxies como ProxySQL ou Pgpool-II) tratam isso automaticamente: dentro de uma transação, tudo vai para o master.

---

## 6. Failover: quando o master cai

Este é o momento em que a arquitetura prova seu valor — e onde ela é mais complexa.

**Se um slave cai:** o caso fácil. As leituras são redirecionadas para os slaves saudáveis restantes (ou temporariamente para o master, se ele era o único) e um novo slave é provisionado. Nenhum impacto na escrita.

**Se o master cai:** um slave precisa ser **promovido** a novo master.

```
   1. DETECÇÃO      O master parou de responder aos heartbeats
                    ⚠ ou a REDE entre o monitor e o master falhou?
                       (não há como distinguir com certeza — este é
                        o problema fundamental)
                              │
                              ▼
   2. ESCOLHA       Qual slave promover? O mais atualizado — o que
                    tiver a maior posição no log de replicação.
                              │
                              ▼
   3. FENCING       Garantir que o master antigo NÃO volte a aceitar
                    escritas. Sem isso: split-brain.
                              │
                              ▼
   4. PROMOÇÃO      O slave escolhido vira master e passa a aceitar escritas.
                              │
                              ▼
   5. RECONFIGURAÇÃO Os demais slaves passam a replicar do novo master;
                    a aplicação precisa descobrir o novo endereço de escrita.
```

Duas dificuldades reais nesse fluxo:

**Perda de dados na promoção.** Com replicação assíncrona, o slave promovido pode não ter recebido as últimas transações commitadas pelo master. Alex Xu registra o ponto diretamente: em sistemas de produção, promover um novo master é mais complicado do que parece, porque os dados do slave podem não estar atualizados, e o que faltar precisa ser reconciliado com scripts de recuperação.

**Split-brain.** Se o master antigo não estava morto — apenas isolado por uma partição de rede — e volta a aceitar escritas enquanto o novo master também aceita, o sistema passa a ter **duas fontes de verdade divergentes**. Reconciliar isso depois é, no caso geral, impossível sem perder dados ou intervir manualmente. É por isso que sistemas sérios usam **quorum** (a promoção só acontece se a maioria dos nós concordar que o master caiu) e **fencing** (isolar ativamente o nó suspeito — desligá-lo, revogar seu acesso ao storage, remover seu IP virtual).

Ferramentas que automatizam esse fluxo: **Orchestrator** e **MHA** (MySQL), **Patroni** e **repmgr** (PostgreSQL), **Redis Sentinel** (Redis), e o failover automático dos serviços gerenciados (RDS Multi-AZ, Cloud SQL).

> **Confusão comum:** "failover automático é sempre melhor do que manual". ✅ **Mais preciso:** failover automático troca **tempo de indisponibilidade** por **risco de decisão errada**, e nem sempre é a troca certa. Um falso positivo — uma oscilação de rede de 10 segundos interpretada como queda do master — dispara uma promoção desnecessária, que custa uma janela de escrita indisponível, potencial perda das transações não replicadas e, no pior caso, split-brain. Sistemas com failover automático mal calibrado sofrem mais incidentes do que sistemas com failover manual bem monitorado. Os ingredientes que tornam a automação confiável são conhecidos: **quorum** (decisão por maioria, nunca por um único monitor), **fencing** (garantir que o nó antigo não escreve mais) e **limiares generosos** (tolerar oscilações curtas). Sem os três, automatizar failover é automatizar a produção de incidentes.

---

## 7. Como a aplicação roteia leitura e escrita

Saber que "escrita vai ao master e leitura ao slave" não diz **quem** toma essa decisão. Existem três lugares possíveis, com trade-offs claros:

| Onde | Como | Vantagem | Desvantagem |
|---|---|---|---|
| **No código** | Duas conexões explícitas; o desenvolvedor escolhe | Controle total, rota a rota | Fácil de errar; espalha a decisão por todo o código |
| **No driver/ORM** | A biblioteca roteia por tipo de comando | Transparente para o código de negócio | Heurística por `SELECT` erra dentro de transações |
| **Em um proxy** | ProxySQL, Pgpool-II, MaxScale, ou o *reader endpoint* do RDS/Aurora | A aplicação vê um endereço só; failover fica invisível | Mais um componente a operar (e mais um SPOF potencial) |

A opção do proxy é elegante porque **repete, na camada de dados, exatamente a ideia dos arquivos anteriores**: um componente intermediário que esconde a topologia real de quem consome. Um `reader endpoint` do Aurora é, conceitualmente, um load balancer com health check e failover — só que para réplicas de banco em vez de instâncias de aplicação.

Cabe uma distinção prática: **PgBouncer não faz roteamento leitura/escrita** — ele é um *pooler* de conexões, e resolve outro problema (o custo de abrir conexões no PostgreSQL). Para separação de leitura e escrita em PostgreSQL, a ferramenta é o Pgpool-II ou o roteamento na aplicação.

---

## 8. Os limites: o que master/slave não resolve

Esta é a seção mais importante do arquivo, porque contém a expectativa mais frequentemente equivocada.

```
                    ESCRITAS
                       │
                       ▼
              ┌────────────────┐
              │     MASTER     │  100% das escritas continuam aqui
              └───────┬────────┘
        ┌─────────────┼─────────────┐
        ▼             ▼             ▼
    [SLAVE 1]     [SLAVE 2]     [SLAVE 3]
        │             │             │
     aplica        aplica        aplica     ◀── cada slave reaplica
     100% das      100% das      100% das       TODAS as escritas.
     escritas      escritas      escritas       A escrita não é dividida
                                                 — é MULTIPLICADA.
```

**Replicação não escala escrita.** Toda escrita passa pelo master, e cada réplica reaplica 100% dela. Adicionar réplicas aumenta a capacidade de **leitura** e a redundância — e adiciona trabalho de replicação ao master. Se o gargalo é escrita, a única saída é **sharding** (`../system-design-pt1/escalabilidade.md`, seção 5).

**Replicação não resolve volume de dados.** Cada réplica guarda a base **inteira**. Se os dados não cabem em uma máquina, não vão caber em dez cópias da mesma máquina.

> **Confusão comum:** "adicionar mais réplicas sempre aumenta a capacidade de leitura do sistema". ✅ **Mais preciso:** aumenta até um ponto, e depois começa a piorar as coisas. Cada réplica nova precisa **reaplicar todas as escritas** — então, em um sistema com escrita intensa, chega um momento em que a réplica gasta a maior parte da sua capacidade apenas acompanhando o master, sobrando pouco para servir leituras. E cada réplica adicional consome banda e trabalho do master para enviar o log. Historicamente, a aplicação do log de replicação era single-threaded (versões modernas de MySQL e PostgreSQL oferecem aplicação paralela), o que criava um teto rígido: mesmo que o master escrevesse em paralelo com muitos núcleos, o slave reaplicava em série — e o lag crescia sem limite. Quando as réplicas não conseguem mais acompanhar, o problema não é falta de réplicas: é que o volume de **escrita** ultrapassou o modelo, e a resposta é sharding.

> **Confusão comum, e a mais cara deste arquivo:** "tenho réplicas do banco, então tenho backup". ✅ **Mais preciso:** replicação e backup resolvem problemas **opostos**, e confundi-los custa a empresa inteira. Replicação protege contra **falha de infraestrutura** — a máquina morre, e existe uma cópia viva em outro lugar. Backup protege contra **erro lógico** — alguém executou `DELETE FROM pedidos` sem `WHERE`, um bug apagou dados, um ransomware criptografou a base. E a diferença crucial: a replicação **propaga o erro lógico fielmente e em milissegundos**. Um `DROP TABLE` no master é replicado para todos os slaves quase instantaneamente; em segundos, você tem quatro cópias perfeitas de um banco destruído. Backup, por definição, é um **ponto no tempo ao qual se pode voltar** — e voltar no tempo é exatamente o que a replicação não permite. Um sistema completo precisa dos dois: replicação para disponibilidade, backups (testados e com restauração ensaiada) para recuperação, e idealmente *point-in-time recovery*, que combina um backup base com o log de transações para reconstruir o estado em qualquer instante.

> **Confusão comum:** "se o master cair, não perco nada, porque os dados estão nas réplicas". ✅ **Mais preciso:** com replicação **assíncrona** — a configuração padrão da maioria dos sistemas — as transações commitadas nos últimos instantes antes da queda **podem não ter sido enviadas a réplica nenhuma**, e se perdem na promoção. A janela costuma ser pequena (milissegundos a segundos), mas "pequena" não é "nenhuma": se essas transações eram confirmações de pagamento, a diferença é bem concreta. Quem precisa de perda zero paga com replicação síncrona ou semissíncrona, e paga em latência e disponibilidade de escrita — o trade-off da seção 4.

---

## 9. Variantes e para onde ir depois

| Modelo | Como funciona | Quando faz sentido |
|---|---|---|
| **Master/slave** | Um escritor, N leitores | O padrão; resolve leitura e disponibilidade |
| **Réplicas em cascata** | Slave replicando de outro slave | Muitas réplicas, para aliviar o master do custo de envio |
| **Multi-master** | Vários nós aceitam escrita | Escrita geograficamente distribuída — ao custo de **conflitos** de escrita concorrente, que exigem uma política de resolução |
| **Replicação circular** | Cada nó replica do seguinte, em anel | Citado por Alex Xu como alternativa; frágil e pouco usado hoje |
| **Sem líder (quorum)** | Escreve em W nós, lê de R, com `W + R > N` | Dynamo, Cassandra. Ver `../system-design-pt1/arquitetura.md`, seção 5 |
| **Sharding + replicação** | Dados particionados, cada shard com suas réplicas | O destino natural de sistemas grandes |
| **CQRS** | Modelos separados para leitura e escrita, sincronizados por eventos | Quando leitura e escrita têm formatos e requisitos muito diferentes |

O **multi-master** merece um alerta: ele parece resolver a limitação de escrita do modelo master/slave, mas troca esse problema por um pior. Se dois nós aceitam escritas simultâneas na mesma linha, alguém precisa decidir qual vence — e qualquer política ("o último timestamp ganha", "o maior nó ganha", "resolve na aplicação") **descarta dados que alguém commitou com sucesso**. Alex Xu classifica esses arranjos como "mais complicados" e fora do escopo do livro, o que é uma forma educada de dizer que multi-master é uma escolha de último recurso, não um upgrade natural.

> **Confusão comum:** "master/slave, quorum e sharding são três gerações da mesma ideia — a mais nova substitui a anterior". ✅ **Mais preciso:** são respostas a perguntas diferentes, e sistemas grandes usam as três ao mesmo tempo. **Sharding** responde "os dados não cabem / a escrita não escala" e divide os dados. **Replicação** responde "não posso perder um nó" e copia os dados. **Quorum** responde "quero ajustar consistência e disponibilidade por operação, em vez de fixá-las para o banco inteiro". A topologia típica de um sistema de grande escala é `sharding para dividir + replicação dentro de cada shard para não perder + quorum para calibrar as garantias` — três mecanismos empilhados, cada um resolvendo o que os outros não resolvem.

---

## Resumo do arquivo

| Problema | Master/slave resolve? |
|---|---|
| Banco é ponto único de falha | **Sim** — via promoção de um slave |
| Excesso de leituras | **Sim** — distribuídas entre as réplicas |
| Excesso de escritas | **Não** — todas passam pelo master e são reaplicadas em cada réplica |
| Dados não cabem em uma máquina | **Não** — cada réplica guarda a base inteira → sharding |
| Alguém apagou dados por engano | **Não** — replicado fielmente para todas as cópias → backup |
| Precisa de consistência forte | **Depende** — só com replicação síncrona, ao custo de latência e disponibilidade |

- O master é a **única fonte de escrita**; os slaves recebem uma cópia completa via **log de transações** e servem leitura.
- **Síncrona × assíncrona × semissíncrona** é o Teorema CAP virado parâmetro de configuração: menor latência e maior disponibilidade de um lado, garantia de não perder dados do outro.
- **Replication lag** é a fonte de uma classe de bugs que não reproduz em desenvolvimento — *read-your-own-writes*, leituras não-monotônicas, causalidade invertida. Existem soluções conhecidas; nenhuma é automática.
- **Failover** exige detecção, escolha, **fencing** e reconfiguração. Sem quorum e fencing, automatizar failover é automatizar split-brain.
- O roteamento leitura/escrita mora no **código**, no **driver** ou em um **proxy** — e um proxy de banco é, conceitualmente, o mesmo load balancer dos arquivos anteriores aplicado à camada de dados.
- **Réplica não é backup.** Replicação protege contra falha de máquina; backup protege contra erro humano e lógico — e a replicação propaga o erro humano com perfeição, em milissegundos.

**Fim da trilha da parte 2.** O caminho completo de uma requisição foi percorrido de ponta a ponta: da decisão de escalar (`escalabilidade_horizontal_vs_vertical.md`), passando pelo componente que distribui (`load_balancer.md`), pelo produto que o implementa (`nginx.md`), pelo papel que governa a API (`api_gateway.md`), pela distinção entre os três (`load_balancer_vs_nginx_vs_api_gateway.md`), até a única camada que não pode ser tornada *stateless* — o banco de dados.
