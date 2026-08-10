# Escalabilidade e Sistemas Distribuídos na Prática

## O que é escalabilidade?

Em `arquitetura.md`, vimos que **escalabilidade** é uma das características de arquitetura ("-ilities") que todo sistema precisa equilibrar contra as demais. Este arquivo aprofunda especificamente essa característica: quais técnicas concretas existem para levar um sistema de "atende um usuário" a "atende milhões de usuários", e quais trade-offs cada uma delas exige.

É importante separar dois conceitos que se confundem:

- **Arquitetura** (visto em `arquitetura.md`) decide a *forma* do sistema — monolito, microsserviços, orientado a eventos.
- **Escalabilidade** (este arquivo) decide a *capacidade* dentro dessa forma — quantas requisições, usuários e dados o sistema aguenta antes de degradar.

Um monolito bem escalado (com réplicas, cache e balanceamento) pode aguentar mais carga do que microsserviços mal escalados. As duas coisas são complementares, não a mesma decisão.

---

## 1. Escala Vertical vs. Escala Horizontal

Existem apenas dois caminhos fundamentais para dar a um sistema mais capacidade:

| | Escala Vertical (*scale up*) | Escala Horizontal (*scale out*) |
|---|---|---|
| **O que faz** | Adiciona mais poder (CPU, RAM) a um servidor existente | Adiciona mais servidores ao conjunto |
| **Simplicidade** | Alta — nada muda na arquitetura da aplicação | Baixa — exige balanceamento de carga e, geralmente, que a aplicação seja *stateless* |
| **Limite** | Físico — existe um teto de CPU/RAM que um único servidor pode ter | Praticamente ilimitado — adicione mais máquinas |
| **Tolerância a falhas** | Nenhuma — se o servidor cai, o sistema cai inteiro | Alta — outras instâncias continuam respondendo |

> **Regra prática:** comece vertical (é mais simples e mais barato em baixa escala), mas planeje para horizontal assim que disponibilidade se tornar um requisito — porque escala vertical, por definição, é sempre um único ponto de falha.

---

## 2. Load Balancer

Assim que existe mais de um servidor atendendo o mesmo tráfego, alguém precisa decidir para qual servidor cada requisição vai. Essa é a função do **load balancer (LB)**: um componente que fica na frente dos servidores, recebe todo o tráfego em um único IP público, e distribui as requisições entre as instâncias saudáveis do pool.

```
                        ┌─────────────────┐
   Usuários ──────────▶ │  Load Balancer  │
                        │   (IP público)  │
                        └────────┬────────┘
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                  ▼
        ┌───────────┐      ┌───────────┐      ┌───────────┐
        │ Servidor 1 │      │ Servidor 2 │      │ Servidor 3 │
        │ (IP privado)│     │ (IP privado)│     │ (IP privado)│
        └───────────┘      └───────────┘      └───────────┘
```

**O que o LB resolve, além de distribuir carga:**
- **Failover:** se o Servidor 1 cair, o LB para de rotear tráfego para ele e redistribui entre os que continuam saudáveis (via *health checks* periódicos).
- **Escala transparente:** adicionar um Servidor 4 ao pool é suficiente — o LB começa a rotear tráfego para ele automaticamente, sem qualquer mudança no lado do cliente.

### Algoritmos de balanceamento

| Algoritmo | Como decide | Quando usar |
|---|---|---|
| **Round Robin** | Distribui em sequência circular (1, 2, 3, 1, 2, 3...) | Quando todos os servidores têm capacidade equivalente |
| **Least Connections** | Envia para o servidor com menos conexões ativas no momento | Quando as requisições têm duração muito variável (algumas rápidas, outras lentas) |
| **IP Hash** | Calcula um hash do IP do cliente para sempre mandá-lo ao mesmo servidor | Quando é preciso *sticky sessions* — o mesmo cliente sempre cair no mesmo servidor |

### Camada 4 vs. Camada 7

- **L4 (transporte):** decide o roteamento olhando só IP e porta, sem inspecionar o conteúdo da requisição. Mais rápido, porque não decodifica o protocolo de aplicação.
- **L7 (aplicação):** entende HTTP — pode rotear por path (`/api/pagamentos` → serviço de pagamentos), por header, por cookie. É o que um **API Gateway** faz por baixo, e por isso os dois componentes frequentemente se confundem (a distinção conceitual entre eles está detalhada em `api_gateway.md`, seção 3).

---

## 3. Replicação de Banco de Dados

Um load balancer resolve a camada de aplicação, mas o banco de dados continua sendo um ponto único de falha se existir uma só instância. A técnica padrão para resolver isso é a **replicação master-slave**:

```
                    ┌──────────────┐
   Escritas ───────▶│  Master DB    │
                    └──────┬───────┘
                           │ replica mudanças
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ Slave DB 1│ │ Slave DB 2│ │ Slave DB 3│  ◀── Leituras
        └──────────┘ └──────────┘ └──────────┘
```

- O **master** aceita escritas (`INSERT`, `UPDATE`, `DELETE`) e replica cada mudança para os **slaves**.
- Os **slaves** só aceitam leituras (`SELECT`) — como a maioria das aplicações lê muito mais do que escreve, isso distribui a maior parte da carga.

**O que acontece quando um nó cai:**
- **Um slave cai:** as leituras são redirecionadas para os slaves saudáveis restantes (ou temporariamente para o master); um novo slave é provisionado para substituí-lo.
- **O master cai:** um dos slaves é promovido a novo master. Em produção, isso é mais delicado do que parece — o slave promovido pode não ter recebido as últimas mudanças replicadas, exigindo um processo de recuperação de dados.

> **Ligação com o Teorema CAP:** replicação síncrona (o master espera confirmação dos slaves antes de responder) favorece consistência às custas de disponibilidade; replicação assíncrona favorece disponibilidade às custas de consistência. É o mesmo trade-off C-vs-A discutido em `arquitetura.md`, seção 5 — e o mecanismo de **quorum** (`W + R > N`) apresentado lá é exatamente a ferramenta usada para ajustar esse equilíbrio por operação, em vez de fixá-lo para o banco inteiro.

---

## 4. CDN (Content Delivery Network)

Uma CDN é uma rede de servidores geograficamente distribuídos que armazena cópias de conteúdo estático (imagens, vídeos, CSS, JavaScript) fisicamente mais perto do usuário final, evitando que toda requisição precise viajar até o servidor de origem.

```
Usuário em SP ──▶ CDN (nó em SP)  ──▶ [cache miss] ──▶ Servidor de origem
                        │                                     │
                        └──────────── guarda em cache ◀───────┘
Usuário em RJ ──▶ CDN (nó no RJ)  ──▶ [cache hit, resposta imediata]
```

**Fluxo (modelo *pull*, o mais comum):**
1. O usuário pede um recurso via URL da CDN (ex.: `https://cdn.meusite.com/logo.png`).
2. Se o nó da CDN mais próximo não tem o arquivo em cache, ele busca na origem uma única vez.
3. A resposta da origem inclui um TTL (Time-to-Live) dizendo por quanto tempo a CDN pode servir essa cópia sem consultar a origem de novo.
4. Todos os usuários próximos daquele nó recebem a versão em cache até o TTL expirar.

**Pontos de atenção:**
- **Custo:** você paga por transferência de dados na CDN — cachear ativos raramente acessados não compensa.
- **TTL bem calibrado:** curto demais gera recarregamentos desnecessários da origem; longo demais serve conteúdo desatualizado depois de um deploy.
- **Fallback:** a aplicação deve lidar com uma eventual indisponibilidade da CDN sem quebrar por completo.

---

## 5. Sharding e Consistent Hashing

Replicação (seção 3) resolve disponibilidade de leitura, mas não resolve o problema de um dataset grande demais para caber — ou ser escrito rápido o bastante — em um único servidor. A solução é o **sharding**: particionar os dados entre vários servidores, cada um responsável por um subconjunto das chaves.

### O problema do hash ingênuo

A forma mais óbvia de decidir em qual servidor uma chave mora é:

```
servidor = hash(chave) % N   (N = número de servidores)
```

Isso funciona bem enquanto `N` é fixo. O problema aparece assim que um servidor é **adicionado ou removido**: como `N` muda, o resultado de `hash(chave) % N` muda para *quase todas* as chaves — não só as que pertenciam ao servidor que saiu. Na prática, isso significa que remover um único servidor de um cluster de cache pode invalidar quase todo o cache de uma vez, gerando uma avalanche de cache miss simultânea no banco de dados.

### A solução: hash ring

O **consistent hashing** resolve isso trocando o módulo por um **anel de hash**. Tanto os servidores quanto as chaves são posicionados no mesmo anel (usando a mesma função de hash, ex. SHA-1), e a regra de busca passa a ser: *"a partir da posição da chave, ande no sentido horário até encontrar o primeiro servidor"*.

```
                    servidor A
                   ╱
        chave 3   ╱
             ╲    ╱
              ╲  ╱
   servidor D ──●── servidor B
              ╱  ╲
             ╱    ╲
        chave 1    chave 2
                   ╲
                    servidor C
```

**Por que isso resolve o problema:** ao adicionar ou remover um servidor do anel, apenas as chaves que estavam entre o servidor afetado e seu vizinho anterior no anel precisam ser redistribuídas — todas as outras chaves continuam apontando para o mesmo servidor de sempre. Na prática, adicionar/remover um nó de um cluster de `N` servidores redistribui, em média, apenas `1/N` das chaves — em vez de quase 100% delas.

### Nós virtuais (virtual nodes)

Um anel com poucos servidores tende a distribuir chaves de forma desigual (por acaso, os servidores podem cair muito próximos uns dos outros no anel, deixando um trecho enorme do anel — e portanto muitas chaves — para um único servidor). A correção é dar a cada servidor físico **múltiplas posições no anel** (nós virtuais): em vez de `servidor_A` aparecer uma vez, ele aparece como `servidor_A_0`, `servidor_A_1`, `servidor_A_2`... Isso distribui a carga de forma muito mais uniforme, ao custo de mais memória para guardar o mapeamento.

> **Onde isso aparece na prática:** é exatamente o mecanismo usado pelo **Redis Cluster** (via *hash slots*, uma variante do mesmo princípio — ver `redis.md`, seção 6), pelo Apache Cassandra, pelo DynamoDB da Amazon e por CDNs como a Akamai. Sempre que você lida com um sistema que precisa **particionar dados entre múltiplos nós sem uma reorganização catastrófica a cada mudança de topologia**, é consistent hashing por trás.

---

## 6. Algoritmos de Rate Limiting

Limitar quantas requisições um cliente pode fazer em um intervalo de tempo é essencial para proteger uma API de abuso — seja um ataque intencional (DoS) ou simplesmente um bug em um cliente que entra em loop. Existem cinco algoritmos clássicos, cada um com um trade-off diferente entre precisão, uso de memória e permissão de picos (*bursts*) de tráfego.

| Algoritmo | Como funciona | Permite burst? | Uso de memória | Ressalva |
|---|---|---|---|---|
| **Token Bucket** | Um "balde" com capacidade fixa recebe tokens a uma taxa constante; cada requisição consome 1 token; sem token, a requisição é recusada | Sim, até a capacidade do balde | Baixo | Dois parâmetros para calibrar (capacidade e taxa de reposição) |
| **Leaky Bucket** | Requisições entram em uma fila (FIFO) de tamanho fixo e são processadas a uma taxa constante; fila cheia = requisição descartada | Não — a taxa de saída é sempre constante | Baixo | Um pico de tráfego enche a fila com requisições antigas, atrasando as mais recentes |
| **Fixed Window Counter** | Divide o tempo em janelas fixas (ex.: a cada minuto) e conta requisições por janela | Sim — **e esse é o problema** | Muito baixo | Um cliente pode enviar 2x o limite permitido concentrando requisições na borda entre duas janelas (ver abaixo) |
| **Sliding Window Log** | Guarda o timestamp de cada requisição individual; a cada nova requisição, descarta os timestamps fora da janela e conta os que sobraram | Não — é o mais preciso de todos | Alto — um timestamp por requisição | Precisão máxima, mas caro em memória sob alto volume |
| **Sliding Window Counter** | Híbrido: combina o contador da janela atual com uma fração ponderada do contador da janela anterior | Atenuado — suaviza picos na borda | Baixo | Aproximação (assume distribuição uniforme dentro da janela anterior), mas o erro é pequeno na prática (~0,003% segundo experimentos da Cloudflare) |

### O problema de borda do Fixed Window Counter

Vale entender em detalhe por que o algoritmo mais simples (Fixed Window) tem uma falha real, porque é o algoritmo implementado na prática em `redis.md`, seção 4.4:

```
Limite: 5 requisições por minuto

Janela 1 [2:00:00 - 2:01:00]: 5 requisições, todas entre 2:00:30 e 2:01:00
Janela 2 [2:01:00 - 2:02:00]: 5 requisições, todas entre 2:01:00 e 2:01:30
                                       │
                                       ▼
        Entre 2:00:30 e 2:01:30 (uma janela de 1 minuto real)
        passaram 10 requisições — o DOBRO do limite configurado.
```

O contador reseta exatamente na virada do minuto, então um cliente malicioso pode enviar o limite inteiro nos últimos segundos de uma janela, e o limite inteiro de novo nos primeiros segundos da próxima — dobrando efetivamente a taxa permitida por um curto período. O **Sliding Window Log** ou o **Sliding Window Counter** eliminam esse problema, ao custo de mais memória ou de uma pequena aproximação, respectivamente.

> **Por que `redis.md` ainda usa Fixed Window, então?** Porque para a maioria das APIs, esse burst temporário na borda da janela é um risco aceitável frente à simplicidade e ao baixíssimo custo de memória do algoritmo — a escolha certa depende do quanto a precisão importa para o seu caso de uso. Sistemas de pagamento ou de prevenção a fraude tendem a preferir Sliding Window; um blog ou uma API interna geralmente não precisam desse rigor.

---

## Resumo

| Problema | Técnica |
|---|---|
| Um servidor não aguenta mais tráfego | Escala horizontal + Load Balancer |
| Um banco não aguenta mais leituras | Replicação master-slave |
| Conteúdo estático demora para chegar a usuários distantes | CDN |
| Um dataset não cabe (ou não escreve rápido o suficiente) em um servidor | Sharding + Consistent Hashing |
| Uma API precisa se proteger de abuso | Rate Limiting (Token Bucket, Sliding Window...) |

Essas técnicas não competem entre si — sistemas de grande escala normalmente usam todas ao mesmo tempo, em camadas diferentes. O próximo arquivo (`redis.md`) mostra como o Redis atua como peça de infraestrutura para várias delas (cache, rate limiting, sessões) na prática.
