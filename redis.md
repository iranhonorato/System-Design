# Redis

## O que é Redis?

Redis (Remote Dictionary Server) é um **banco de dados em memória**, de código aberto, que armazena dados como pares de chave-valor. Criado em 2009 por Salvatore Sanfilippo, ele foi projetado com um objetivo central: ser **absurdamente rápido**.

A diferença fundamental entre o Redis e bancos de dados tradicionais como PostgreSQL ou MySQL é onde os dados vivem:

```
Banco Relacional (PostgreSQL):
  Requisição → Lê do DISCO → Processa → Retorna
  Latência típica: 1ms a 100ms+

Redis:
  Requisição → Lê da RAM → Retorna
  Latência típica: < 1ms (microssegundos)
```

O disco magnético ou SSD é ordens de magnitude mais lento que a memória RAM. O Redis explora isso para entregar velocidade que banco de dados em disco simplesmente não consegue alcançar.

> **Analogia:** Pense em um banco relacional como uma biblioteca enorme — você precisa ir até a estante certa, procurar o livro na prateleira, e trazer até a mesa. O Redis é como ter os livros mais usados sempre em cima da sua mesa, prontos para consulta imediata.

---

## 1. Finalidade e Casos de Uso

O Redis não foi feito para substituir seu banco de dados principal. Ele é uma **camada complementar** que resolve problemas específicos de performance e funcionalidades que bancos relacionais não foram projetados para resolver.

### Para que o Redis serve?

| Caso de Uso | Problema que resolve | Exemplo |
|---|---|---|
| **Cache** | Consultas lentas ao banco | Resultado de uma query complexa armazenado por 5 minutos |
| **Sessões de usuário** | Armazenar estado de login sem sobrecarregar o banco | Token JWT + dados do usuário logado |
| **Filas de mensagens** | Comunicação assíncrona entre serviços | Fila de envio de e-mails, processamento de pagamentos |
| **Rate Limiting** | Controlar quantidade de requisições por usuário | Máximo de 100 requests por minuto por IP |
| **Leaderboards** | Rankings ordenados em tempo real | Top 10 jogadores de um game |
| **Pub/Sub** | Notificações em tempo real | Chat ao vivo, feeds de notificações |
| **Contadores** | Incrementos atômicos e rápidos | Número de visualizações de um post |
| **Locks distribuídos** | Evitar condições de corrida em sistemas distribuídos | Garantir que só um processo processe um pedido (algoritmo Redlock) |
| **Streams** | Log de eventos persistente com consumers groups | Fila de eventos entre microsserviços com reprocessamento |

### O que o Redis NÃO é

- **Não é um substituto para banco relacional:** se você precisa de joins complexos, integridade referencial e transações ACID completas, use PostgreSQL.
- **Não é permanente por padrão:** os dados vivem na RAM. Se o servidor cair sem persistência configurada, os dados somem.
- **Não escala para datasets enormes:** a RAM é cara. Não coloque terabytes de dados no Redis.

---

## 2. Como o Redis Funciona

### 2.1 Arquitetura de Memória

O Redis é um processo single-threaded para operações de dados (a partir da versão 6.0, operações de I/O de rede são multi-threaded, mas o processamento de comandos ainda é serializado). Isso pode parecer uma limitação, mas é deliberado:

```
┌─────────────────────────────────────────────────┐
│                  PROCESSO REDIS                 │
│                                                 │
│  Thread principal (single-threaded):            │
│  ┌─────────────────────────────────────────┐    │
│  │  Fila de comandos (event loop)          │    │
│  │  cmd1 → cmd2 → cmd3 → cmd4 → ...       │    │
│  └─────────────────────────────────────────┘    │
│                    │                            │
│                    ▼                            │
│  ┌─────────────────────────────────────────┐    │
│  │  Memória RAM (hash table global)        │    │
│  │  "user:1"  → { nome, email, plano }    │    │
│  │  "produto:99" → { nome, preco }        │    │
│  │  "contador" → 4821                     │    │
│  └─────────────────────────────────────────┘    │
└─────────────────────────────────────────────────┘
```

Por ser single-threaded para dados, o Redis elimina a necessidade de locks e mutexes. Cada operação é atômica por natureza — dois clientes não podem modificar a mesma chave simultaneamente de forma corrompida. Isso simplifica enormemente o modelo mental e elimina uma classe inteira de bugs de concorrência.

### 2.2 Event Loop (Reator de Eventos)

O Redis usa um padrão chamado **Reactor Pattern** com multiplexação de I/O:

```
Clientes conectados:
  Cliente A ──┐
  Cliente B ──┤─→ [ Event Loop ] ──→ Processa um comando por vez
  Cliente C ──┘        │
                        └──→ Responde ao cliente via socket
```

O event loop fica monitorando todos os sockets de clientes ao mesmo tempo. Quando um cliente envia um comando, o Redis o coloca na fila, processa rapidamente (memória é rápida) e passa para o próximo. Como cada operação leva microssegundos, o throughput alcança centenas de milhares de operações por segundo mesmo sendo single-threaded.

### 2.3 Persistência (Opcional)

Apesar de ser in-memory, o Redis oferece dois mecanismos de persistência no disco:

**RDB (Redis Database Snapshots):**
```
RAM ──[a cada N minutos ou N mudanças]──→ Arquivo .rdb no disco
                                           (snapshot completo)

Vantagem: arquivo compacto, ideal para backups
Desvantagem: pode perder dados entre snapshots
```

**AOF (Append Only File):**
```
Cada comando de escrita ──→ Logado no arquivo appendonly.aof
                             (log de operações)

Na reinicialização: Redis replays todos os comandos do arquivo

Vantagem: durabilidade quase total (perde no máximo 1 segundo de dados)
Desvantagem: arquivo maior, reinicialização mais lenta
```

**Recomendação para produção:** usar ambos juntos — AOF para durabilidade, RDB para backups rápidos.

### 2.4 Expiração Automática de Chaves (TTL)

Uma das funcionalidades mais valiosas do Redis é poder definir um **TTL (Time To Live)** para qualquer chave. Após o tempo expirar, a chave é automaticamente removida:

```
SET session:user123 "{dados}" EX 3600   ← expira em 1 hora

                    ┌──────────────────────┐
Tempo:  0s          │ session:user123 ✓   │
Tempo: 1800s        │ session:user123 ✓   │
Tempo: 3600s        │ session:user123 ✗   │ ← deletada automaticamente
                    └──────────────────────┘
```

O Redis usa dois mecanismos para isso:
- **Lazy expiration:** checa se a chave expirou quando ela é acessada.
- **Active expiration:** periodicamente, amostra chaves aleatórias e deleta as expiradas.

---

## 3. Estruturas de Dados

O Redis não é apenas um dicionário de strings. Ele suporta estruturas de dados ricas, cada uma otimizada para casos de uso específicos.

### 3.1 String

A estrutura mais básica. Pode armazenar texto, números ou dados binários (até 512MB por valor).

```
SET nome "João"
GET nome          → "João"

SET contador 0
INCR contador     → 1
INCR contador     → 2
INCRBY contador 5 → 7
```

**Casos de uso:** cache de HTML renderizado, contadores, flags de feature.

### 3.2 Hash

Um mapa de campos e valores dentro de uma chave. Perfeito para representar objetos.

```
HSET usuario:1 nome "Maria" email "maria@email.com" plano "premium"
HGET usuario:1 email          → "maria@email.com"
HGETALL usuario:1             → { nome: "Maria", email: "...", plano: "premium" }
HSET usuario:1 plano "basic"  → atualiza só o campo plano
```

**Vantagem vs string:** você pode atualizar um campo sem reescrever o objeto inteiro.

### 3.3 List

Lista encadeada de strings. Suporta inserção/remoção eficiente nas duas pontas.

```
RPUSH fila:emails "email1" "email2" "email3"  → adiciona ao final
LPOP fila:emails                              → remove e retorna "email1"
LRANGE fila:emails 0 -1                       → lista todos os elementos
```

**Casos de uso:** filas de processamento (FIFO com RPUSH + LPOP), histórico de atividades.

### 3.4 Set

Conjunto não ordenado de strings únicas. Sem duplicatas.

```
SADD tags:post:42 "redis" "backend" "performance"
SADD tags:post:42 "redis"     → ignorado (já existe)
SMEMBERS tags:post:42         → {"redis", "backend", "performance"}
SISMEMBER tags:post:42 "redis" → 1 (existe)

# Operações de conjunto:
SINTER tags:post:42 tags:post:10   → interseção
SUNION tags:post:42 tags:post:10   → união
```

**Casos de uso:** tags, amigos em comum, controle de itens únicos visitados.

### 3.5 Sorted Set (ZSet)

Como o Set, mas cada membro tem um **score numérico** que determina a ordenação.

```
ZADD ranking 4250 "jogador_A"
ZADD ranking 8100 "jogador_B"
ZADD ranking 5500 "jogador_C"

ZRANGE ranking 0 -1 WITHSCORES   → [jogador_A:4250, jogador_C:5500, jogador_B:8100]
ZREVRANGE ranking 0 2            → top 3 (ordem decrescente)
ZRANK ranking "jogador_A"        → posição no ranking (0-indexed)
```

**Casos de uso:** leaderboards, feeds cronológicos, sistemas de prioridade.

---

## 4. Aplicações Práticas — Implementação Passo a Passo

Vamos construir uma aplicação real de e-commerce em Python com FastAPI e Redis, implementando os casos de uso mais comuns. O stack escolhido é intencional — os conceitos se traduzem diretamente para Node.js, Go, Java ou qualquer outra linguagem.

### Estrutura do Projeto

```
ecommerce-api/
├── docker-compose.yml
├── requirements.txt
├── app/
│   ├── main.py
│   ├── redis_client.py
│   ├── routers/
│   │   ├── produtos.py
│   │   ├── usuarios.py
│   │   └── ranking.py
│   └── middleware/
│       └── rate_limit.py
```

---

### 4.1 Configuração do Ambiente

**docker-compose.yml** — sobe a API e o Redis juntos:

```yaml
version: "3.9"

services:
  redis:
    image: redis:7.2-alpine
    container_name: redis
    ports:
      - "6379:6379"
    command: >
      redis-server
      --appendonly yes
      --appendfsync everysec
      --maxmemory 256mb
      --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data

  api:
    build: .
    container_name: ecommerce-api
    ports:
      - "8000:8000"
    depends_on:
      - redis
    environment:
      - REDIS_URL=redis://redis:6379

volumes:
  redis_data:
```

**Explicando as configurações do Redis:**
- `appendonly yes` — ativa AOF para durabilidade
- `appendfsync everysec` — sincroniza com disco a cada 1 segundo (equilíbrio performance/durabilidade)
- `maxmemory 256mb` — limite de RAM que o Redis pode usar
- `maxmemory-policy allkeys-lru` — quando atingir o limite, remove as chaves menos usadas recentemente (LRU = Least Recently Used). Isso é fundamental para usar o Redis como cache.

**requirements.txt:**

```
fastapi==0.111.0
uvicorn==0.29.0
redis==5.0.3
python-jose==3.3.0
passlib==1.7.4
httpx==0.27.0
```

**app/redis_client.py** — cliente Redis centralizado:

```python
import redis.asyncio as redis
import os

# Pool de conexões: reutiliza conexões ao invés de criar uma nova por requisição
# Isso é crítico para performance em produção
redis_pool = redis.ConnectionPool.from_url(
    os.getenv("REDIS_URL", "redis://localhost:6379"),
    max_connections=20,           # máximo de conexões simultâneas no pool
    decode_responses=True,        # retorna strings ao invés de bytes
)

def get_redis() -> redis.Redis:
    """Retorna um cliente Redis usando o pool de conexões compartilhado."""
    return redis.Redis(connection_pool=redis_pool)
```

> **Por que pool de conexões?** Criar uma nova conexão TCP com o Redis a cada requisição seria desperdício de tempo e recursos. O pool mantém conexões abertas e reutilizáveis — uma requisição "pega" uma conexão do pool, usa, e "devolve". Isso reduz latência e overhead.

---

### 4.2 Cache de Produtos

**O problema sem cache:**

```
Usuário A requisita produto #42
  → API consulta PostgreSQL (50ms de query + JOIN de categorias)
  → Retorna resultado

Usuário B requisita produto #42 (1 segundo depois)
  → API consulta PostgreSQL DE NOVO (outros 50ms)
  → Retorna o mesmo resultado

1000 usuários por minuto requisitam produto #42:
  → 1000 queries idênticas no banco
  → Banco sobrecarregado, latência aumenta para todos
```

**Com cache Redis:**

```
Usuário A requisita produto #42
  → Redis: MISS (não tem no cache)
  → API consulta PostgreSQL (50ms)
  → Armazena resultado no Redis por 5 minutos
  → Retorna resultado

Usuário B requisita produto #42 (1 segundo depois)
  → Redis: HIT (< 1ms)
  → Retorna resultado SEM tocar no banco

1000 usuários por minuto requisitam produto #42:
  → 1 query no banco (quando o cache expirar)
  → 999 respostas servidas pelo Redis
```

**app/routers/produtos.py:**

```python
import json
from fastapi import APIRouter, HTTPException
from app.redis_client import get_redis

router = APIRouter(prefix="/produtos", tags=["produtos"])

# Simulação de banco de dados (substituiria por SQLAlchemy + PostgreSQL)
PRODUTOS_DB = {
    "42": {"id": 42, "nome": "Teclado Mecânico", "preco": 349.90, "estoque": 150},
    "99": {"id": 99, "nome": "Monitor 4K", "preco": 2199.00, "estoque": 30},
}

CACHE_TTL = 300  # 5 minutos em segundos


@router.get("/{produto_id}")
async def buscar_produto(produto_id: str):
    r = get_redis()
    cache_key = f"produto:{produto_id}"

    # 1. Tenta buscar no cache primeiro
    cached = await r.get(cache_key)
    if cached:
        print(f"[CACHE HIT] produto:{produto_id}")
        return {"source": "cache", "data": json.loads(cached)}

    # 2. Cache miss — busca no banco de dados
    print(f"[CACHE MISS] produto:{produto_id} — consultando banco")
    produto = PRODUTOS_DB.get(produto_id)
    if not produto:
        raise HTTPException(status_code=404, detail="Produto não encontrado")

    # 3. Armazena no cache com TTL de 5 minutos
    await r.setex(
        name=cache_key,
        time=CACHE_TTL,
        value=json.dumps(produto)  # Redis só armazena strings — serializa para JSON
    )

    return {"source": "database", "data": produto}


@router.put("/{produto_id}")
async def atualizar_produto(produto_id: str, dados: dict):
    """Ao atualizar um produto, invalida o cache para evitar dados desatualizados."""
    if produto_id not in PRODUTOS_DB:
        raise HTTPException(status_code=404, detail="Produto não encontrado")

    PRODUTOS_DB[produto_id].update(dados)

    # Invalida o cache: próxima leitura buscará do banco e regenerará o cache
    r = get_redis()
    await r.delete(f"produto:{produto_id}")
    print(f"[CACHE INVALIDADO] produto:{produto_id}")

    return {"message": "Produto atualizado e cache invalidado"}
```

> **Cache Invalidation:** Um dos problemas mais difíceis em computação. A estratégia usada acima é **invalidação por evento** — quando os dados mudam, o cache é deletado imediatamente. Alternativas incluem TTL curto (aceita dados levemente desatualizados) ou write-through (atualiza cache e banco ao mesmo tempo).

---

### 4.3 Sessões de Usuário

Armazenar sessões no Redis ao invés do banco de dados tem duas vantagens: velocidade na verificação de autenticação em cada requisição, e facilidade para invalidar sessões (logout, banimento, expiração).

**app/routers/usuarios.py:**

```python
import uuid
import json
from fastapi import APIRouter, HTTPException, Header
from app.redis_client import get_redis

router = APIRouter(prefix="/auth", tags=["autenticação"])

SESSAO_TTL = 3600  # 1 hora

# Simulação de usuários cadastrados
USUARIOS_DB = {
    "maria@email.com": {"id": "user:1", "nome": "Maria", "plano": "premium"},
    "joao@email.com":  {"id": "user:2", "nome": "João",  "plano": "basic"},
}


@router.post("/login")
async def login(email: str, senha: str):
    # Em produção: valida senha com hash bcrypt no banco de dados
    usuario = USUARIOS_DB.get(email)
    if not usuario:
        raise HTTPException(status_code=401, detail="Credenciais inválidas")

    r = get_redis()

    # Gera um token de sessão único e imprevisível
    session_token = str(uuid.uuid4())
    session_key = f"session:{session_token}"

    # Armazena dados da sessão como Hash no Redis
    # HSET permite atualizar campos individualmente depois
    await r.hset(session_key, mapping={
        "user_id": usuario["id"],
        "nome":    usuario["nome"],
        "plano":   usuario["plano"],
        "email":   email,
    })

    # Define expiração — sessão expira automaticamente após 1 hora de inatividade
    await r.expire(session_key, SESSAO_TTL)

    return {"session_token": session_token, "expires_in": SESSAO_TTL}


@router.get("/me")
async def perfil_atual(authorization: str = Header(...)):
    """Verifica sessão e retorna dados do usuário logado."""
    # Header esperado: "Bearer <session_token>"
    token = authorization.replace("Bearer ", "")
    r = get_redis()

    session_key = f"session:{token}"
    sessao = await r.hgetall(session_key)

    if not sessao:
        raise HTTPException(status_code=401, detail="Sessão inválida ou expirada")

    # Renova o TTL a cada acesso (sessão deslizante — expira N horas após último uso)
    await r.expire(session_key, SESSAO_TTL)

    return {"usuario": sessao}


@router.post("/logout")
async def logout(authorization: str = Header(...)):
    """Invalida a sessão imediatamente."""
    token = authorization.replace("Bearer ", "")
    r = get_redis()

    deletadas = await r.delete(f"session:{token}")
    if not deletadas:
        raise HTTPException(status_code=401, detail="Sessão não encontrada")

    return {"message": "Logout realizado com sucesso"}
```

**Por que não usar JWT puro para sessões?**

```
JWT puro:
  Token gerado → válido até expirar → NÃO dá para invalidar antes do prazo
  Problema: usuário deletado/banido ainda tem token válido por horas

Redis + token opaco:
  Token gerado → armazenado no Redis → DELETE imediato invalida a sessão
  Logout real, banimento imediato, sem aguardar expiração
```

---

### 4.4 Rate Limiting

Limitar requisições por IP ou usuário é fundamental para proteger APIs de abuso. O Redis é a escolha natural por sua atomicidade e velocidade.

**app/middleware/rate_limit.py:**

```python
from fastapi import Request, HTTPException
from app.redis_client import get_redis


async def rate_limit_middleware(request: Request, call_next):
    """
    Middleware que aplica rate limiting por IP usando o algoritmo Fixed Window.
    Permite até LIMITE requisições por janela de JANELA_SEGUNDOS.
    """
    LIMITE = 100         # máximo de requisições
    JANELA_SEGUNDOS = 60 # por janela de 60 segundos

    # Identificador único: IP do cliente
    ip_cliente = request.client.host
    chave = f"rate_limit:{ip_cliente}"

    r = get_redis()

    # Pipeline: executa múltiplos comandos em uma única ida ao Redis
    # Isso reduz a latência de rede (1 roundtrip ao invés de 2)
    async with r.pipeline(transaction=True) as pipe:
        pipe.incr(chave)                    # incrementa contador
        pipe.expire(chave, JANELA_SEGUNDOS)  # garante expiração
        resultado = await pipe.execute()

    contador_atual = resultado[0]

    # Adiciona headers informativos (boa prática de API)
    if contador_atual > LIMITE:
        raise HTTPException(
            status_code=429,
            detail=f"Muitas requisições. Tente novamente em {JANELA_SEGUNDOS} segundos.",
            headers={
                "X-RateLimit-Limit": str(LIMITE),
                "X-RateLimit-Remaining": "0",
                "Retry-After": str(JANELA_SEGUNDOS),
            }
        )

    response = await call_next(request)
    response.headers["X-RateLimit-Limit"] = str(LIMITE)
    response.headers["X-RateLimit-Remaining"] = str(max(0, LIMITE - contador_atual))
    return response
```

**Como funciona o algoritmo Fixed Window:**

```
Janela de 60s para IP 192.168.1.10:

  t=0s:   rate_limit:192.168.1.10 = 1     (INCR cria a chave, TTL = 60s)
  t=5s:   rate_limit:192.168.1.10 = 2
  t=10s:  rate_limit:192.168.1.10 = 100
  t=11s:  rate_limit:192.168.1.10 = 101   → BLOQUEADO (429)
  t=60s:  chave expira automaticamente
  t=61s:  rate_limit:192.168.1.10 = 1     (nova janela, contador zerado)
```

> **Por que INCR é seguro sem locks?** O Redis é single-threaded para operações de dados. `INCR` é atômico — não existe condição de corrida onde dois processos leem "100", ambos somam 1 e escrevem "101" (problema clássico de concorrência). Um processo executa primeiro, o outro espera.

**Registrando o middleware no app/main.py:**

```python
from fastapi import FastAPI
from app.routers import produtos, usuarios, ranking
from app.middleware.rate_limit import rate_limit_middleware

app = FastAPI(title="E-commerce API com Redis")

# Registra o middleware de rate limiting globalmente
app.middleware("http")(rate_limit_middleware)

app.include_router(produtos.router)
app.include_router(usuarios.router)
app.include_router(ranking.router)
```

---

### 4.5 Leaderboard em Tempo Real

Rankings com Sorted Sets — atualização em O(log N) e leitura do top-N em O(log N + N).

**app/routers/ranking.py:**

```python
from fastapi import APIRouter
from app.redis_client import get_redis

router = APIRouter(prefix="/ranking", tags=["ranking"])

RANKING_KEY = "ranking:vendedores"


@router.post("/venda")
async def registrar_venda(vendedor_id: str, valor: float):
    """Registra uma venda e atualiza o score do vendedor no ranking."""
    r = get_redis()

    # ZINCRBY: incrementa o score de um membro
    # Se o membro não existir, é criado com score = valor
    novo_score = await r.zincrby(RANKING_KEY, valor, vendedor_id)

    return {
        "vendedor": vendedor_id,
        "total_vendas": novo_score,
        "message": "Venda registrada no ranking"
    }


@router.get("/top/{n}")
async def top_vendedores(n: int = 10):
    """Retorna os N melhores vendedores, do maior para o menor score."""
    r = get_redis()

    # ZRANGE com REV=True: ordem decrescente (maior score primeiro) — Redis 6.2+
    # zrevrange() ainda funciona mas foi depreciado no Redis 7.0
    top = await r.zrange(RANKING_KEY, 0, n - 1, desc=True, withscores=True)

    resultado = [
        {"posicao": i + 1, "vendedor": vendedor, "total": score}
        for i, (vendedor, score) in enumerate(top)
    ]

    return {"top": resultado}


@router.get("/posicao/{vendedor_id}")
async def posicao_vendedor(vendedor_id: str):
    """Retorna a posição atual de um vendedor específico no ranking."""
    r = get_redis()

    # ZREVRANK: posição no ranking decrescente (0 = primeiro lugar)
    # Alternativa moderna: r.zrank(RANKING_KEY, vendedor_id, desc=True)
    posicao = await r.zrevrank(RANKING_KEY, vendedor_id)
    score = await r.zscore(RANKING_KEY, vendedor_id)

    if posicao is None:
        return {"message": "Vendedor não encontrado no ranking"}

    return {
        "vendedor": vendedor_id,
        "posicao": posicao + 1,   # converte para 1-indexed
        "total_vendas": score
    }
```

**Visualizando o Sorted Set internamente:**

```
RANKING_KEY = "ranking:vendedores"

  Score   │  Membro
──────────┼─────────────────
  12500   │  "vendedor_A"   ← ZREVRANGE posição 0 (1º lugar)
   9800   │  "vendedor_C"   ← ZREVRANGE posição 1 (2º lugar)
   7200   │  "vendedor_B"   ← ZREVRANGE posição 2 (3º lugar)
   3100   │  "vendedor_D"

Após ZINCRBY ranking:vendedores 5000 "vendedor_B":
  Score   │  Membro
──────────┼─────────────────
  12500   │  "vendedor_A"
  12200   │  "vendedor_B"   ← subiu para 2º lugar automaticamente
   9800   │  "vendedor_C"
   3100   │  "vendedor_D"
```

---

### 4.6 Fila de Processamento Assíncrono

Envio de e-mails, geração de relatórios e outros processos lentos não devem bloquear a resposta da API. O padrão é: a API enfileira o trabalho e retorna imediatamente; um worker processa em background.

**Produtor — enfileira tarefas (adicionar em app/routers/usuarios.py):**

```python
import json
from app.redis_client import get_redis

FILA_EMAILS = "fila:emails"

async def enfileirar_email_boas_vindas(usuario_email: str, usuario_nome: str):
    """
    Adiciona uma tarefa de envio de e-mail à fila.
    A API retorna imediatamente — o e-mail é enviado por um worker separado.
    """
    r = get_redis()

    tarefa = {
        "tipo": "email_boas_vindas",
        "destinatario": usuario_email,
        "nome": usuario_nome,
        "tentativas": 0,
    }

    # RPUSH: adiciona ao final da fila (Right Push)
    # Worker usará LPOP para retirar do início (Left Pop) — padrão FIFO
    await r.rpush(FILA_EMAILS, json.dumps(tarefa))
    print(f"[FILA] E-mail enfileirado para {usuario_email}")
```

**Worker — processa a fila (worker.py na raiz do projeto):**

```python
import asyncio
import json
import redis.asyncio as redis

FILA_EMAILS = "fila:emails"
FILA_EMAILS_MORTOS = "fila:emails:dead_letter"  # tarefas que falharam várias vezes
MAX_TENTATIVAS = 3


async def enviar_email(tarefa: dict):
    """Simulação do envio de e-mail. Em produção: integração com SendGrid, SES, etc."""
    print(f"[WORKER] Enviando e-mail de boas-vindas para {tarefa['destinatario']}...")
    await asyncio.sleep(2)  # simula latência do serviço de e-mail
    print(f"[WORKER] E-mail enviado para {tarefa['destinatario']} ✓")


async def processar_fila():
    r = redis.Redis.from_url("redis://localhost:6379", decode_responses=True)
    print("[WORKER] Iniciado. Aguardando tarefas...")

    while True:
        # BLPOP: bloqueia até ter um item na fila (evita polling com sleep)
        # timeout=0 significa esperar indefinidamente
        resultado = await r.blpop(FILA_EMAILS, timeout=0)

        if resultado:
            _, dados_raw = resultado
            tarefa = json.loads(dados_raw)

            try:
                await enviar_email(tarefa)
            except Exception as e:
                tarefa["tentativas"] += 1
                print(f"[WORKER] Erro ao processar tarefa: {e}. Tentativa {tarefa['tentativas']}")

                if tarefa["tentativas"] < MAX_TENTATIVAS:
                    # Re-enfileira para tentar novamente
                    await r.rpush(FILA_EMAILS, json.dumps(tarefa))
                else:
                    # Move para dead letter queue — requer atenção manual
                    await r.rpush(FILA_EMAILS_MORTOS, json.dumps(tarefa))
                    print(f"[WORKER] Tarefa movida para dead letter queue após {MAX_TENTATIVAS} falhas")


if __name__ == "__main__":
    asyncio.run(processar_fila())
```

**Fluxo completo:**

```
Usuário faz cadastro
        │
        ▼
  API (FastAPI)
        │
        ├── Salva usuário no banco ──→ Retorna 201 Created (rápido)
        │
        └── RPUSH fila:emails {...}
                    │
                    ▼ (assíncrono, não bloqueia a API)
             [Worker separado]
                    │
                    └── BLPOP fila:emails → processa → envia e-mail
```

---

### 4.7 Testando a Aplicação

**Iniciando o ambiente:**

```bash
# Sobe Redis e API
docker compose up -d

# Em outro terminal: inicia o worker de e-mails
python worker.py
```

**Testando com curl:**

```bash
# 1. Login e obtenção de sessão
curl -X POST "http://localhost:8000/auth/login?email=maria@email.com&senha=qualquer"
# Retorna: { "session_token": "550e8400-e29b-...", "expires_in": 3600 }

# 2. Buscar produto (primeiro acesso: CACHE MISS)
curl "http://localhost:8000/produtos/42"
# Retorna: { "source": "database", "data": {...} }

# 3. Buscar produto novamente (CACHE HIT — muito mais rápido)
curl "http://localhost:8000/produtos/42"
# Retorna: { "source": "cache", "data": {...} }

# 4. Registrar vendas para o ranking
curl -X POST "http://localhost:8000/ranking/venda?vendedor_id=vendedor_A&valor=5000"
curl -X POST "http://localhost:8000/ranking/venda?vendedor_id=vendedor_B&valor=8500"
curl -X POST "http://localhost:8000/ranking/venda?vendedor_id=vendedor_A&valor=3200"

# 5. Consultar top 3
curl "http://localhost:8000/ranking/top/3"
# Retorna: [{ posição: 1, vendedor: "vendedor_B", total: 8500 }, ...]
```

**Inspecionando o Redis diretamente:**

```bash
# Acessa o CLI do Redis dentro do container
docker exec -it redis redis-cli

# Lista todas as chaves (nunca use em produção com milhões de chaves — use SCAN)
KEYS *

# Verifica o TTL de uma sessão
TTL session:550e8400-e29b-...

# Inspeciona o hash de uma sessão
HGETALL session:550e8400-e29b-...

# Monitora todos os comandos em tempo real (útil para debug)
MONITOR
```

---

## 5. Boas Práticas e Armadilhas Comuns

### 5.1 Nomenclatura de Chaves

Adote uma convenção hierárquica com `:` como separador. Isso facilita filtragem e entendimento:

```
✓  usuario:42:perfil
✓  session:abc123
✓  produto:99:cache
✓  rate_limit:192.168.1.1
✓  ranking:vendedores:2024

✗  usuario42perfil
✗  SESS_abc123
✗  prod99
```

### 5.2 Evite Chaves Sem TTL em Cache

Todo dado de cache deve ter um TTL. Dados sem TTL ficam para sempre até serem explicitamente deletados:

```python
# ✗ Ruim — dado fica preso na memória para sempre
await r.set("produto:42", json.dumps(produto))

# ✓ Bom — expira automaticamente
await r.setex("produto:42", 300, json.dumps(produto))
```

### 5.3 Use Pipelines para Operações em Lote

```python
# ✗ Ruim — 3 roundtrips de rede
await r.set("chave1", "valor1")
await r.set("chave2", "valor2")
await r.set("chave3", "valor3")

# ✓ Bom — 1 roundtrip de rede
async with r.pipeline() as pipe:
    pipe.set("chave1", "valor1")
    pipe.set("chave2", "valor2")
    pipe.set("chave3", "valor3")
    await pipe.execute()
```

### 5.4 Nunca Use KEYS * em Produção

```bash
# ✗ NUNCA em produção — bloqueia o Redis inteiro para varrer todas as chaves
KEYS produto:*

# ✓ Use SCAN — itera em lotes sem bloquear
SCAN 0 MATCH produto:* COUNT 100
```

### 5.5 Configure maxmemory e a Política de Eviction

Sem `maxmemory`, o Redis consome RAM indefinidamente até o servidor ficar sem memória (e travar). Configure sempre:

```
maxmemory 512mb

# Escolha a política conforme o caso de uso:
maxmemory-policy allkeys-lru    # cache geral — remove o menos usado recentemente
maxmemory-policy volatile-lru   # remove apenas chaves com TTL (preserva dados permanentes)
maxmemory-policy noeviction     # rejeita escritas quando cheio (para dados críticos)
```

---

## 6. Redis em Arquitetura de Microsserviços

Em um ambiente com múltiplos serviços, o Redis funciona como **infraestrutura compartilhada**:

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Serviço de │    │  Serviço de │    │  Serviço de │
│   Pedidos   │    │  Produtos   │    │   Usuários  │
└──────┬──────┘    └──────┬──────┘    └──────┬──────┘
       │                  │                  │
       └──────────────────┼──────────────────┘
                          │
                   ┌──────▼──────┐
                   │    REDIS    │
                   │             │
                   │  Sessions   │
                   │  Cache      │
                   │  Pub/Sub    │
                   │  Queues     │
                   │  Rate Limit │
                   └─────────────┘
```

**Pub/Sub para comunicação entre serviços (exemplo):**

```python
# Serviço de Pedidos — publica evento quando pedido é confirmado
async def confirmar_pedido(pedido_id: str):
    r = get_redis()
    await r.publish("eventos:pedidos", json.dumps({
        "tipo": "pedido_confirmado",
        "pedido_id": pedido_id,
    }))

# Serviço de Estoque — assina e reage ao evento
async def ouvir_pedidos():
    r = get_redis()
    pubsub = r.pubsub()
    await pubsub.subscribe("eventos:pedidos")

    async for mensagem in pubsub.listen():
        if mensagem["type"] == "message":
            evento = json.loads(mensagem["data"])
            if evento["tipo"] == "pedido_confirmado":
                await decrementar_estoque(evento["pedido_id"])
```

> **Limitação crítica do Pub/Sub:** mensagens publicadas enquanto um subscriber está offline são **perdidas permanentemente** — o Pub/Sub não persiste mensagens. Se resiliência e reprocessamento são necessários, use **Redis Streams** (`XADD`/`XREAD`) ou uma solução dedicada como Kafka/RabbitMQ.

**Redis Streams — alternativa persistente ao Pub/Sub:**

```python
# Produtor: adiciona evento ao stream (persiste no Redis)
await r.xadd("stream:pedidos", {
    "tipo": "pedido_confirmado",
    "pedido_id": pedido_id,
})

# Consumidor: lê eventos a partir de um ID (pode reler eventos perdidos)
entradas = await r.xread({"stream:pedidos": "$"}, block=0)
for stream, mensagens in entradas:
    for msg_id, campos in mensagens:
        await processar_evento(campos)
        await r.xack("stream:pedidos", "grupo-estoque", msg_id)
```

Streams também suportam **Consumer Groups** — múltiplos consumidores dividindo o trabalho, com confirmação de processamento e reentrega automática de mensagens não confirmadas.

---

## Resumo

```
┌─────────────────────────────────────────────────────────┐
│                    REDIS — VISÃO GERAL                  │
├─────────────────┬───────────────────────────────────────┤
│ O que é         │ Banco de dados em memória chave-valor  │
├─────────────────┼───────────────────────────────────────┤
│ Por que é rápido│ Dados na RAM, single-threaded, sem I/O │
├─────────────────┼───────────────────────────────────────┤
│ Estruturas      │ String, Hash, List, Set, Sorted Set    │
├─────────────────┼───────────────────────────────────────┤
│ Casos de uso    │ Cache, Sessões, Filas, Rate Limit,     │
│                 │ Leaderboards, Pub/Sub, Contadores      │
├─────────────────┼───────────────────────────────────────┤
│ Persistência    │ RDB (snapshots) + AOF (log de ops)     │
├─────────────────┼───────────────────────────────────────┤
│ O que não é     │ Substituto do banco relacional,        │
│                 │ storage para dados frios ou enormes    │
└─────────────────┴───────────────────────────────────────┘
```

O Redis resolve um problema específico e o resolve excepcionalmente bem: **acesso ultrarrápido a dados que precisam de velocidade de memória**. Usado nos lugares certos, ele é a diferença entre uma aplicação que engatinha sob carga e uma que escala com elegância.