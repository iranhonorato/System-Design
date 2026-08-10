# Threads e Sockets

Dois conceitos fundamentais que aparecem em quase toda discussão sobre sistemas — servidores web, bancos de dados, jogos online, chats. Entender o que são e como se relacionam é essencial para compreender por que softwares são projetados da forma que são.

---

## Parte 1 — Sockets

### O que é um Socket?

Um socket é um **ponto de comunicação entre dois programas através de uma rede** (ou até na mesma máquina). Ele é a abstração que permite que dois processos troquem dados, da mesma forma que uma tomada elétrica é a abstração que permite conectar um aparelho à rede elétrica.

> **Analogia:** Pense em um socket como a **ficha telefônica de uma ligação**. Quando você liga para alguém, a operadora cria um canal exclusivo entre os dois aparelhos. Enquanto a ligação está ativa, tudo que você fala vai direto para o outro lado e vice-versa. O socket é exatamente isso — um canal bidirecional aberto entre dois pontos.

### O Problema que o Socket Resolve

Programas rodam isolados na memória do computador. Um processo não pode simplesmente "acessar" a memória de outro processo em outra máquina. O sistema operacional precisa de um mecanismo para que dados saiam de um programa, viajem pela rede, e entrem em outro programa no destino.

```
Sem socket — impossível comunicar diretamente:

  [Seu navegador]          [Servidor da Netflix]
   (processo A)               (processo B)
       │                           │
       │  ← memórias isoladas →    │
       │                           │
       ✗ não há canal de comunicação
```

```
Com socket — comunicação possível:

  [Seu navegador]          [Servidor da Netflix]
   (processo A)               (processo B)
       │                           │
   socket A ←────rede────→  socket B
       │                           │
       └── dados fluem nos dois sentidos ──┘
```

### Como um Socket Funciona na Prática

Quando você digita `netflix.com` no navegador, a seguinte sequência acontece nos bastidores:

```
1. SEU NAVEGADOR
   ├── Resolve "netflix.com" para um IP (ex: 54.230.1.100)
   └── Pede ao sistema operacional: "Crie um socket para 54.230.1.100, porta 443"

2. SISTEMA OPERACIONAL (na sua máquina)
   └── Cria um socket local na porta efêmera (ex: porta 52341)
       └── Envia pacote SYN para 54.230.1.100:443  ← "quero me conectar"

3. SERVIDOR DA NETFLIX
   ├── Está escutando na porta 443 (HTTPS)
   ├── Recebe o SYN
   └── Responde SYN-ACK  ← "conexão aceita"

4. SEU NAVEGADOR
   └── Responde ACK  ← "recebi a confirmação"

5. CONEXÃO ESTABELECIDA
   Seu socket 52341 ←────────→ Socket Netflix 443
   Canal bidirecional aberto, pronto para trocar dados
```

Esse processo (SYN → SYN-ACK → ACK) é chamado de **TCP Three-Way Handshake** — o "aperto de mão" que estabelece a conexão antes de qualquer dado ser enviado.

### Socket no Lado do Servidor

Um servidor não tem apenas um socket — ele tem **um socket de escuta** e **um socket por cliente conectado**:

```
SERVIDOR (porta 8000)

  Socket de Escuta
  ┌─────────────────────────────────────┐
  │  "Estou na porta 8000.              │
  │   Me avise quando alguém conectar" │
  └──────────────────┬──────────────────┘
                     │ novo cliente conecta
                     ▼
         Sistema operacional cria
         um socket dedicado para ele

  Socket Cliente A ←───→ [Cliente A]
  Socket Cliente B ←───→ [Cliente B]
  Socket Cliente C ←───→ [Cliente C]
```

É por isso que o Redis, ao receber uma resposta de uma operação, pode enviar de volta especificamente para o cliente que fez aquela requisição — cada cliente tem seu próprio socket, seu próprio canal exclusivo.

### Porta: o "Ramal" do Socket

O IP identifica a máquina. A porta identifica o programa dentro da máquina. São como o número do prédio (IP) e o número do apartamento (porta):

```
Endereço completo de um socket:
  54.230.1.100 : 443
       │           │
      IP          Porta
  (a máquina)  (o programa)

Portas conhecidas (convenção universal):
  80   → HTTP
  443  → HTTPS
  5432 → PostgreSQL
  6379 → Redis
  22   → SSH
  25   → SMTP (e-mail)
```

---

## Parte 2 — Threads

### O que é uma Thread?

Para entender threads, precisamos primeiro entender o que é um **processo**.

**Processo** é um programa em execução. Quando você abre o Chrome, o sistema operacional cria um processo para ele — aloca memória, atribui recursos, e começa a executar o código. Cada processo tem sua própria memória isolada.

**Thread** é uma **unidade de execução dentro de um processo**. Um processo pode ter várias threads, e todas elas **compartilham a mesma memória** do processo.

> **Analogia:** Pense num restaurante.
> - O **restaurante** é o processo — tem sua própria cozinha, mesas, e recursos.
> - Os **garçons** são as threads — cada um trabalha de forma independente, mas todos compartilham a mesma cozinha, os mesmos utensílios e as mesmas mesas.
> - Se um garçom quebra um prato, o barulho afeta todos na mesma cozinha. Se um processo corrompe a memória, pode afetar todas as threads.

```
PROCESSO (programa em execução)
┌─────────────────────────────────────────────────┐
│  Memória compartilhada (heap, código, arquivos) │
│                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│  │ Thread 1 │  │ Thread 2 │  │ Thread 3 │      │
│  │          │  │          │  │          │      │
│  │ Stack    │  │ Stack    │  │ Stack    │      │
│  │ própria  │  │ própria  │  │ própria  │      │
│  └──────────┘  └──────────┘  └──────────┘      │
│                                                 │
│  Cada thread tem sua pilha de execução,         │
│  mas acessa a mesma memória do processo         │
└─────────────────────────────────────────────────┘
```

Cada thread tem sua própria **pilha de execução (stack)** — o histórico das funções que está chamando no momento. Mas todas acessam o mesmo **heap** — onde ficam os objetos, variáveis compartilhadas e dados.

### O que uma Thread Faz?

Uma thread executa uma sequência de instruções. Em qualquer instante, ela está fazendo exatamente uma coisa. O que muda entre single-thread e multi-thread é quantas dessas sequências acontecem simultaneamente.

---

## Parte 3 — Single-Thread vs Multi-Thread

### Single-Thread — Uma fila, um caixa

Um programa single-threaded tem exatamente uma thread de execução. Tudo acontece em sequência — uma instrução de cada vez, uma tarefa de cada vez.

```
PROGRAMA SINGLE-THREADED

Fila de tarefas:
  [Tarefa A] → [Tarefa B] → [Tarefa C] → [Tarefa D]

Execução:
  Thread única:
  ──[Tarefa A]──[Tarefa B]──[Tarefa C]──[Tarefa D]──→ tempo
```

**O problema do bloqueio:**

```
Servidor web single-threaded recebendo requisições:

  t=0ms:  Requisição do Cliente A chega
          Thread inicia: consulta banco de dados...
          ⏳ aguardando resposta do banco (50ms)

  t=10ms: Requisição do Cliente B chega
          ❌ Thread está ocupada com A — B fica esperando

  t=30ms: Requisição do Cliente C chega
          ❌ Thread ainda ocupada com A — C também espera

  t=50ms: Banco responde para A
          Thread processa A, responde
          Agora processa B (já esperou 40ms desnecessariamente)
          Depois C...

Resultado: clientes B e C sofreram latência extra sem necessidade
```

**Por que o Redis é single-threaded e funciona tão bem?**

O Redis consegue ser single-threaded e ainda assim extremamente performático porque suas operações são de memória — elas completam em microssegundos. Não há espera por disco ou rede durante o processamento. O event loop nunca fica bloqueado esperando:

```
Redis single-threaded (event loop):

  t=0μs:   Comando GET produto:42  → lê da RAM → responde em 50μs
  t=50μs:  Comando SET sessao:abc  → escreve na RAM → responde em 30μs
  t=80μs:  Comando INCR contador   → soma na RAM → responde em 20μs

Throughput: 100.000+ operações/segundo mesmo sendo single-threaded
porque cada operação é muito rápida
```

### Multi-Thread — Várias filas, vários caixas

Um programa multi-threaded tem várias threads executando em paralelo (ou de forma concorrente). Cada thread pode processar uma tarefa independentemente das outras.

```
PROGRAMA MULTI-THREADED

  Thread 1: ──[Tarefa A]──────────────────────────→
  Thread 2: ──────[Tarefa B]──────────────────────→
  Thread 3: ──────────[Tarefa C]──────────────────→
  Thread 4: ────────────────[Tarefa D]────────────→

Tempo total ≈ duração da tarefa mais longa (ao invés da soma de todas)
```

**Servidor web multi-threaded:**

```
  t=0ms:  Requisição do Cliente A → Thread 1 assume
          Thread 1: consulta banco... (50ms de espera)

  t=10ms: Requisição do Cliente B → Thread 2 assume
          Thread 2: consulta banco... (50ms de espera)

  t=30ms: Requisição do Cliente C → Thread 3 assume
          Thread 3: consulta banco... (50ms de espera)

  t=50ms: Thread 1 recebe resposta do banco → responde A
  t=60ms: Thread 2 recebe resposta do banco → responde B
  t=80ms: Thread 3 recebe resposta do banco → responde C

Resultado: todos os clientes respondidos sem esperar uns pelos outros
```

### Comparação direta

```
                   SINGLE-THREAD          MULTI-THREAD
                 ┌──────────────────┬───────────────────────┐
  Complexidade   │  Simples         │  Complexo             │
  ───────────────┼──────────────────┼───────────────────────┤
  Concorrência   │  Não             │  Sim                  │
  ───────────────┼──────────────────┼───────────────────────┤
  Condição de    │  Impossível      │  Possível — requer    │
  corrida        │                  │  sincronização        │
  ───────────────┼──────────────────┼───────────────────────┤
  Uso de CPU     │  1 núcleo        │  Múltiplos núcleos    │
  ───────────────┼──────────────────┼───────────────────────┤
  Ideal para     │  Operações       │  Operações que        │
                 │  ultrarrápidas   │  esperam (I/O, rede,  │
                 │  (Redis, event   │  disco) ou que usam   │
                 │  loops isolados) │  CPU intensamente     │
                 └──────────────────┴───────────────────────┘
```

---

## Parte 4 — O Grande Problema do Multi-Thread: Condição de Corrida

Threads compartilham memória. Isso é poderoso, mas perigoso. Imagine duas threads incrementando um contador:

```python
contador = 0  # variável compartilhada na memória

# Thread 1 e Thread 2 executam isso ao mesmo tempo:
def incrementar():
    contador = contador + 1
```

Parece simples. Mas a CPU não executa `contador + 1` em uma instrução atômica. Ela faz:

```
1. Lê o valor de contador da memória para um registrador
2. Soma 1 ao registrador
3. Escreve o resultado de volta na memória
```

Se duas threads fizerem isso simultaneamente:

```
Esperado: contador vai de 0 para 2

O que pode acontecer:
  t=1: Thread 1 lê contador → obtém 0
  t=2: Thread 2 lê contador → obtém 0  (Thread 1 ainda não escreveu!)
  t=3: Thread 1 soma 1 → tem 1, escreve na memória → contador = 1
  t=4: Thread 2 soma 1 → tem 1, escreve na memória → contador = 1

Resultado: contador = 1 (perdemos um incremento)
```

Isso é uma **condição de corrida (race condition)** — o resultado depende da ordem imprevisível em que as threads executam. É um bug extremamente difícil de reproduzir e debugar porque depende do timing.

### Solução: Locks (Mutex)

Para proteger dados compartilhados, usamos **locks** (ou mutexes — mutual exclusion):

```python
import threading

contador = 0
lock = threading.Lock()

def incrementar():
    global contador
    with lock:           # adquire o lock — só uma thread passa por vez
        contador += 1    # seção crítica — protegida
                         # lock liberado automaticamente ao sair do bloco
```

```
Com lock:
  t=1: Thread 1 adquire o lock
  t=2: Thread 2 tenta adquirir o lock → BLOQUEADA, aguarda
  t=3: Thread 1 lê (0), soma, escreve → contador = 1
  t=4: Thread 1 libera o lock
  t=5: Thread 2 adquire o lock
  t=6: Thread 2 lê (1), soma, escreve → contador = 2
  t=7: Thread 2 libera o lock

Resultado correto: contador = 2
```

Locks resolvem o problema, mas introduzem novos desafios:

- **Deadlock:** Thread A espera o lock que Thread B segura; Thread B espera o lock que Thread A segura. Ambas esperam para sempre.
- **Contenção:** Muitas threads competindo pelo mesmo lock se tornam essencialmente single-threaded naquela seção, anulando os benefícios do paralelismo.

---

## Parte 5 — Paralelismo vs Concorrência

Dois termos frequentemente confundidos, mas com significados distintos.

### Concorrência — Intercalando tarefas

Concorrência é sobre **lidar com múltiplas tarefas**. Elas não precisam executar ao mesmo tempo — podem se intercalar. Uma thread pode pausar enquanto espera I/O e deixar outra thread executar.

```
CPU com 1 núcleo — concorrência via intercalamento:

  Thread A: ──██░░░░██░░░░██──  (██ = executando, ░ = esperando I/O)
  Thread B: ────░░██░░░░██────
  Thread C: ──────░░░░██░░░░──

  CPU real:  ──A──B──C──A──B──C──→ tempo
             (troca rapidamente entre threads)

O usuário percebe como se fossem simultâneas, mas a CPU
executa uma de cada vez, alternando rapidamente.
```

### Paralelismo — Realmente ao mesmo tempo

Paralelismo é sobre **executar múltiplas tarefas ao mesmo tempo físico**. Requer múltiplos núcleos de CPU.

```
CPU com 4 núcleos — paralelismo real:

  Núcleo 1: ──[Thread A processando]──────────────────→
  Núcleo 2: ──[Thread B processando]──────────────────→
  Núcleo 3: ──[Thread C processando]──────────────────→
  Núcleo 4: ──[Thread D processando]──────────────────→

As quatro threads executam fisicamente ao mesmo tempo.
```

```
Resumo visual:

  Concorrência:           Paralelismo:
  Uma cozinheira          Quatro cozinheiros
  fazendo três pratos     fazendo um prato cada
  (alterna entre eles)    (ao mesmo tempo)

  Mais eficiente que      Mais rápido que
  sequencial, mas         concorrência para
  ainda uma de cada vez   trabalho CPU-intensivo
```

---

## Parte 6 — Modelos na Prática

Sistemas reais combinam essas ideias de formas diferentes. Cada escolha tem razão de ser.

### Node.js — Single-Thread com Event Loop Assíncrono

Node.js usa uma única thread principal com um event loop, mas usa operações de I/O assíncronas. Quando uma operação de disco ou rede é iniciada, o Node não espera — ele registra um callback e continua processando outros eventos.

```
EVENT LOOP DO NODE.JS

  Thread única:
  ┌────────────────────────────────────────────────┐
  │  1. Recebe requisição do Cliente A             │
  │  2. Inicia consulta ao banco (assíncrona)      │
  │     → "me avise quando terminar"               │
  │  3. Recebe requisição do Cliente B (imediato!) │
  │  4. Inicia consulta ao banco para B            │
  │  5. Banco responde para A → executa callback A │
  │  6. Banco responde para B → executa callback B │
  └────────────────────────────────────────────────┘

Nunca bloqueia. Sempre processando o próximo evento disponível.
```

**O que acontece durante a espera do banco:**

```
Node.js delega o I/O para o sistema operacional (libuv).
O SO usa chamadas assíncronas (epoll no Linux, kqueue no macOS).
A thread do Node fica livre para processar outros eventos.
Quando o SO conclui o I/O, notifica o Node via event loop.
```

### Apache — Multi-Thread por Requisição

O Apache tradicional (modelo prefork/worker) cria uma nova thread (ou processo) para cada requisição recebida:

```
APACHE — THREAD POR REQUISIÇÃO

  Requisição A → Thread 1 criada → processa A do início ao fim → encerra
  Requisição B → Thread 2 criada → processa B do início ao fim → encerra
  Requisição C → Thread 3 criada → processa C do início ao fim → encerra

Problema: criar e destruir threads tem custo.
Com 10.000 conexões simultâneas → 10.000 threads → esgota a RAM.
```

### Nginx — Multi-Processo com Event Loop por Worker

O Nginx usa um modelo diferente do Apache: um **processo master** e múltiplos **processos worker** (geralmente um por núcleo de CPU). Cada worker é single-threaded e usa multiplexação de I/O para atender milhares de conexões sem criar uma thread por cliente.

```
NGINX — ARQUITETURA MULTI-PROCESSO

  Processo Master
  └── Gerencia workers, lê config, faz reload sem downtime

  Worker 1 (núcleo 0):            Worker 2 (núcleo 1):
  ┌──────────────────────────┐    ┌──────────────────────────┐
  │  epoll_wait(sockets...)  │    │  epoll_wait(sockets...)  │
  │  "Avise quando tiver     │    │  "Avise quando tiver     │
  │   dados prontos"         │    │   dados prontos"         │
  └────────────┬─────────────┘    └────────────┬─────────────┘
               │ socket pronto                  │ socket pronto
               ▼                               ▼
        processa conexão               processa conexão
```

Cada worker usa `epoll` (Linux) ou `kqueue` (macOS) para monitorar todos os seus sockets ao mesmo tempo — sem criar threads extras. O resultado: um servidor com 4 núcleos tem 4 workers, cada um atendendo dezenas de milhares de conexões. É por isso que o Nginx aguenta 10.000 conexões simultâneas com muito menos memória que o Apache tradicional.

> **Distinção importante:** o Nginx NÃO é single-threaded — é **multi-processo com event loop single-threaded por processo**. Cada worker é um processo independente com seu próprio event loop.

### Redis — Single-Thread com Multiplexação de I/O

O Redis é o caso genuinamente single-threaded para processamento de comandos: um único processo, uma única thread, um único event loop:

```
REDIS — MULTIPLEXAÇÃO (epoll/kqueue)

  Thread única monitorando N sockets:
  ┌─────────────────────────────────────┐
  │  epoll_wait([socket_A, socket_B,    │
  │              socket_C, ...])        │
  │  "Me avise quando QUALQUER um       │
  │   desses tiver dados prontos"       │
  └──────────────────┬──────────────────┘
                     │ socket_B ficou pronto
                     ▼
              processa socket_B (μs)
                     │ socket_A ficou pronto
                     ▼
              processa socket_A (μs)
                     │ ...

Sem threads extras. Sem locks. Sem condições de corrida.
100.000+ operações/segundo com um único processo.
```

O Redis consegue ser single-threaded e performático porque cada operação leva microssegundos — não há espera por disco durante o processamento. A partir da versão 6.0, o I/O de rede é multi-threaded, mas o processamento dos comandos continua serializado na thread principal.

### Python asyncio — Concorrência Assíncrona com Coroutines

Python moderno (3.5+) adota um modelo de concorrência baseado em **coroutines** e `async/await`, que é o que o FastAPI usa internamente:

```python
import asyncio

async def buscar_dados(cliente_id: int):
    # Quando o await é atingido, a coroutine "pausa"
    # e o event loop executa outras coroutines enquanto espera
    resultado = await banco.query(f"SELECT * FROM clientes WHERE id={cliente_id}")
    return resultado

async def main():
    # asyncio.gather executa as três coroutines de forma concorrente
    # (não paralela — ainda uma thread, mas intercaladas)
    resultado_a, resultado_b, resultado_c = await asyncio.gather(
        buscar_dados(1),
        buscar_dados(2),
        buscar_dados(3),
    )
```

```
EVENT LOOP DO asyncio:

  Thread única:
  t=0ms: inicia buscar_dados(1) → encontra await → pausa
  t=0ms: inicia buscar_dados(2) → encontra await → pausa
  t=0ms: inicia buscar_dados(3) → encontra await → pausa
  t=50ms: banco responde para (1) → retoma buscar_dados(1)
  t=51ms: banco responde para (2) → retoma buscar_dados(2)
  t=52ms: banco responde para (3) → retoma buscar_dados(3)

Total: ~52ms (vs ~150ms sequencial)
```

> **Regra de ouro:** `async/await` só é vantajoso para I/O-bound (banco, rede, disco). Para CPU-bound (compressão, ML, criptografia), use `multiprocessing` para paralelismo real — o GIL do Python bloqueia threads para código Python puro.

### Java / .NET — Thread Pool

Aplicações empresariais frequentemente usam um pool fixo de threads reutilizáveis:

```
THREAD POOL

  Pool (tamanho fixo = 10 threads):
  ┌──────────────────────────────────────┐
  │  T1  T2  T3  T4  T5  T6  T7  T8  T9  T10  │
  └──────────────────────────────────────┘
         ↑                    ↑
     processando          disponível

  Fila de requisições: [Req_11] [Req_12] [Req_13]...
  Ficam na fila até uma thread ficar disponível.

Vantagem: sem overhead de criar threads a cada requisição.
Desvantagem: pool esgotado → requisições esperam na fila.
```

---

## Resumo

```
┌─────────────────────────────────────────────────────────────────┐
│                        CONCEITOS-CHAVE                          │
├──────────────────┬──────────────────────────────────────────────┤
│ Socket           │ Canal de comunicação bidirecional entre       │
│                  │ dois programas via rede. IP + Porta           │
├──────────────────┼──────────────────────────────────────────────┤
│ Thread           │ Unidade de execução dentro de um processo.   │
│                  │ Compartilha memória com outras threads.       │
├──────────────────┼──────────────────────────────────────────────┤
│ Single-Thread    │ Uma sequência de execução. Simples, sem       │
│                  │ condições de corrida. Ideal quando operações  │
│                  │ são rápidas (Redis, event loops).             │
├──────────────────┼──────────────────────────────────────────────┤
│ Multi-Thread     │ Múltiplas execuções paralelas. Melhor uso    │
│                  │ de CPU multi-núcleo. Requer sincronização.    │
├──────────────────┼──────────────────────────────────────────────┤
│ Concorrência     │ Lidar com múltiplas tarefas intercalando-as. │
│                  │ Não precisa de múltiplos núcleos.             │
├──────────────────┼──────────────────────────────────────────────┤
│ Paralelismo      │ Executar múltiplas tarefas ao mesmo tempo    │
│                  │ físico. Requer múltiplos núcleos de CPU.      │
├──────────────────┼──────────────────────────────────────────────┤
│ Condição de      │ Bug causado por threads acessando memória    │
│ corrida          │ compartilhada sem sincronização.              │
├──────────────────┼──────────────────────────────────────────────┤
│ Lock / Mutex     │ Mecanismo que garante acesso exclusivo a uma │
│                  │ seção crítica — uma thread por vez.           │
└──────────────────┴──────────────────────────────────────────────┘
```

A escolha entre single-thread e multi-thread não é sobre qual é "melhor" — é sobre qual se encaixa no problema. O Redis escolheu single-thread porque suas operações são de memória e extremamente rápidas. O Java escolheu thread pools porque processa lógica de negócio complexa que pode durar segundos. Cada decisão tem uma razão arquitetural.