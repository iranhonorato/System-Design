# API Gateway

## O que é um API Gateway?

Um API Gateway é um **ponto de entrada único e centralizado** para todas as requisições externas que chegam a um sistema composto por múltiplos serviços. Ele atua como intermediário entre os clientes (browsers, apps mobile, outros serviços) e o conjunto de APIs internas da sua arquitetura.

Em termos práticos: em vez de os clientes conhecerem e chamarem diretamente cada microsserviço, eles chamam apenas o Gateway — e ele decide para onde cada requisição vai, aplica regras de segurança, transforma dados e muito mais.

> **Analogia:** Pense em um hotel grande. Quando você chega, não vai direto para a cozinha pedir comida, para a lavanderia pedir roupa limpa ou para a manutenção trocar uma lâmpada. Você fala com a **recepção**, e ela direciona, coordena e controla tudo. O API Gateway é a recepção do seu sistema.

### O problema que ele resolve

Sem um Gateway, em uma arquitetura de microsserviços, a situação fica assim:

```
                        Cliente Mobile
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
         ▼                   ▼                   ▼
  Serviço de Auth     Serviço de Pedidos    Serviço de Produtos
  :8001                :8002                :8003
         │
         ▼
  Serviço de Usuários
  :8004
```

O cliente precisa:
- Conhecer o endereço de cada serviço
- Autenticar separadamente em cada um
- Lidar com CORS em cada chamada
- Combinar respostas de múltiplos serviços
- Ser reescrito toda vez que um serviço muda de endereço

Com um Gateway:

```
                        Cliente Mobile
                             │
                             ▼
                      ┌─────────────┐
                      │  API Gateway │
                      │  (porta 443) │
                      └──────┬──────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
         ▼                   ▼                   ▼
  Serviço de Auth     Serviço de Pedidos    Serviço de Produtos
  (interno)           (interno)             (interno)
```

O cliente conhece apenas **um endereço**. O Gateway resolve tudo o que está por trás.

---

## 1. Para que um API Gateway Serve?

O Gateway não é apenas um "roteador glorificado". Ele centraliza uma série de responsabilidades transversais que, sem ele, teriam que ser reimplementadas em cada microsserviço:

### Responsabilidades principais

| Responsabilidade | O que faz | Problema que evita |
|---|---|---|
| **Roteamento** | Direciona `/pedidos/*` para o Serviço de Pedidos | Clientes precisarem conhecer endereços internos |
| **Autenticação** | Valida tokens JWT antes de deixar a requisição passar | Cada serviço reimplementar validação de auth |
| **Rate Limiting** | Limita a 100 req/min por usuário | DDoS, abuso de API, custos explosivos |
| **SSL Termination** | Decripta HTTPS e repassa HTTP internamente | Cada serviço gerenciar certificados TLS |
| **Load Balancing** | Distribui requisições entre instâncias do mesmo serviço | Sobrecarga em uma única instância |
| **Transformação** | Converte formatos (JSON → XML), renomeia campos | Incompatibilidade de contratos entre versões |
| **Cache** | Armazena respostas de requisições repetidas | Carga desnecessária nos serviços internos |
| **Logging centralizado** | Registra todas as requisições em um único lugar | Logs dispersos em dezenas de serviços |
| **CORS** | Trata políticas de origem cruzada | Cada serviço configurar seus próprios headers |
| **Versionamento** | `/v1/users` e `/v2/users` para serviços diferentes | Breaking changes afetando todos os clientes |
| **Circuit Breaker** | Para de chamar um serviço que está falhando | Falha em cascata derrubando o sistema inteiro |

### O que o Gateway NÃO deve fazer

- **Lógica de negócio:** o Gateway não deve saber que "um pedido aprovado notifica o estoque". Isso é responsabilidade dos serviços.
- **Acesso direto ao banco:** ele não consulta dados, apenas roteia e aplica políticas.
- **Ser um monolito disfarçado:** se o Gateway acumula lógica demais, você criou um ponto único de falha e um gargalo arquitetural.

---

## 2. Como o API Gateway Funciona

### 2.1 O Ciclo de Vida de uma Requisição

Toda requisição que passa pelo Gateway percorre um **pipeline de processamento**:

```
  Requisição externa
         │
         ▼
┌────────────────────────────────────────────────────────────┐
│                      API GATEWAY                           │
│                                                            │
│  ① Recepção          → Aceita a conexão TCP/TLS           │
│         │                                                  │
│  ② SSL Termination   → Decripta HTTPS                     │
│         │                                                  │
│  ③ Autenticação      → Valida JWT / API Key               │
│         │                                                  │
│  ④ Rate Limiting     → Conta requisições por cliente      │
│         │                                                  │
│  ⑤ Roteamento        → Identifica qual serviço responde   │
│         │                                                  │
│  ⑥ Transformação     → Modifica headers, body, path       │
│         │                                                  │
│  ⑦ Cache Check       → Retorna resposta cacheada se houver│
│         │                                                  │
│  ⑧ Proxy Forward     → Encaminha para o serviço interno   │
│         │                                                  │
│  ⑨ Circuit Breaker   → Monitora falhas do serviço         │
│         │                                                  │
│  ⑩ Resposta          → Devolve ao cliente (com logs)      │
└────────────────────────────────────────────────────────────┘
         │
         ▼
  Resposta para o cliente
```

Cada etapa pode **modificar, interromper ou enriquecer** a requisição. Se a autenticação falha na etapa ③, a requisição nunca chega ao serviço interno.

### 2.2 Roteamento: Como o Gateway Decide Para Onde Enviar

O roteamento funciona através de **regras declarativas** — você configura quais padrões de URL mapeiam para quais serviços:

```
Requisição: GET /api/v1/pedidos/123

Gateway verifica as regras na ordem:
  Regra 1: /api/v1/auth/*      → serviço-auth:3000     ✗ não casa
  Regra 2: /api/v1/pedidos/*   → serviço-pedidos:3001  ✓ casa!
  Regra 3: /api/v1/produtos/*  → serviço-produtos:3002 (ignorada)

Gateway reescreve o path:
  /api/v1/pedidos/123  →  /123  (remove o prefixo)

Encaminha para:
  http://serviço-pedidos:3001/123
```

O roteamento pode ser baseado em:
- **Path:** `/users/*` vai para o serviço de usuários
- **Método HTTP:** `GET /produto` vai para serviço de leitura, `POST /produto` vai para serviço de escrita
- **Header:** `X-API-Version: 2` roteia para a versão 2 do serviço
- **Host:** `api.empresa.com` vs `admin.empresa.com` para serviços diferentes

### 2.3 Autenticação Centralizada

O Gateway pode validar tokens JWT sem consultar o Auth0 a cada requisição (a validação é local, via chave pública):

```
  Requisição com header:
  Authorization: Bearer eyJhbGciOiJSUzI1...
         │
         ▼
  ┌─────────────────────────────────────────┐
  │  Gateway valida o token:               │
  │                                         │
  │  - Assinatura válida? (chave pública)  │
  │  - Token expirou? (campo exp)          │
  │  - Audience correto? (campo aud)       │
  │  - Issuer esperado? (campo iss)        │
  └─────────────────────────────────────────┘
         │                    │
         ▼                    ▼
     Token válido         Token inválido
         │                    │
         ▼                    ▼
  Encaminha requisição   Retorna 401
  + injeta headers:      Unauthorized
  X-User-ID: 42
  X-User-Roles: admin
```

Os serviços internos recebem os dados do usuário **já verificados** via headers — eles não precisam reimplementar a lógica de autenticação.

### 2.4 Rate Limiting

O Gateway controla quantas requisições um cliente pode fazer em um período:

```
Algoritmo: Token Bucket (mais comum)

Configuração: 100 requests/minuto por IP

  T=0s   IP 10.0.0.1 faz req → bucket tem 100 tokens → consome 1 → OK (99)
  T=1s   IP 10.0.0.1 faz req → bucket tem 99 tokens  → consome 1 → OK (98)
  ...
  T=50s  IP 10.0.0.1 faz req → bucket tem 0 tokens   → BLOQUEADO → 429

  Headers na resposta 429:
    X-RateLimit-Limit: 100
    X-RateLimit-Remaining: 0
    X-RateLimit-Reset: 1716912000  (timestamp Unix quando reseta)
```

O estado do rate limiting pode ser armazenado em **Redis** para funcionar mesmo com múltiplas instâncias do Gateway.

### 2.5 Circuit Breaker

Se um serviço interno começa a falhar, o Circuit Breaker evita que o Gateway continue enviando requisições para ele (e causando timeouts em cascata):

```
Estado: FECHADO (operação normal)
  → Gateway roteia normalmente para o serviço
  → Monitora taxa de falhas

Taxa de erros ultrapassa o threshold (ex: 50% em 10s):
  → Circuit Breaker ABRE

Estado: ABERTO
  → Gateway retorna erro imediatamente (sem chamar o serviço)
  → Serviço tem tempo para se recuperar

Após timeout de recovery (ex: 30s):
  → Estado: MEIO-ABERTO
  → Gateway permite 1 requisição de teste

  Se funcionar → volta para FECHADO
  Se falhar    → volta para ABERTO
```

Isso evita o **efeito dominó**: um serviço lento ou fora do ar sobrecarregando os outros com requisições acumuladas.

---

## 3. Gateway vs Conceitos Similares

Uma dúvida comum é diferenciar o API Gateway de outras peças de infraestrutura:

| Componente | O que faz | Onde vive | Quem usa |
|---|---|---|---|
| **API Gateway** | Ponto de entrada, auth, rate limit, roteamento por regras de negócio | Borda da rede (edge) | Externo → seus serviços |
| **Load Balancer** | Distribui carga entre instâncias idênticas de um serviço | Camada de rede | Qualquer tráfego |
| **Reverse Proxy** | Encaminha requisições para servidores internos, oculta topologia | Frente de um servidor | Geralmente HTTP |
| **Service Mesh** | Comunicação segura e observável *entre* serviços internos (sidecar) | Interno ao cluster | Serviço → Serviço |

```
Internet
   │
   ▼
[API Gateway]      ← autenticação, rate limit, roteamento por negócio
   │
   ▼
[Load Balancer]    ← distribui entre instâncias do mesmo serviço
   │
   ├──▶ [Instância A]
   ├──▶ [Instância B]    ← cada instância tem um sidecar (Service Mesh)
   └──▶ [Instância C]
```

Na prática, muitos gateways (Kong, AWS API Gateway) também fazem load balancing. Mas conceitualmente são responsabilidades distintas.

---

## 4. Padrão BFF (Backend for Frontend)

Em sistemas com múltiplos tipos de clientes, uma variação popular é criar **um Gateway especializado por tipo de cliente**:

```
             ┌─────────────────┐
             │  App Mobile     │
             └────────┬────────┘
                      │
                      ▼
             ┌─────────────────┐
             │  BFF Mobile     │  ← dados condensados, otimizados para 4G
             └────────┬────────┘
                      │
         ┌────────────┼────────────┐
         ▼            ▼            ▼
   Serv. Users   Serv. Pedidos   Serv. Produtos


             ┌─────────────────┐
             │  Web App        │
             └────────┬────────┘
                      │
                      ▼
             ┌─────────────────┐
             │  BFF Web        │  ← dados ricos, múltiplas chamadas agregadas
             └────────┬────────┘
                      │
         ┌────────────┼────────────┐
         ▼            ▼            ▼
   Serv. Users   Serv. Pedidos   Serv. Produtos
```

**Por que BFF?**
- O mobile precisa de respostas menores (economizar banda e bateria)
- O web pode se dar ao luxo de dados mais ricos e múltiplas chamadas agregadas
- Um único Gateway genérico acaba servindo mal todos os clientes

---

## 5. Como Implementar

### 5.1 Tutorial: Criando seu Próprio API Gateway com FastAPI

Antes de usar Kong, AWS ou Nginx como gateway, é valioso construir um do zero. Isso desmistifica o que cada ferramenta faz internamente e permite personalização total quando as soluções prontas não atendem.

#### Por que FastAPI?

FastAPI é assíncrono por padrão (baseado em `asyncio`), o que é ideal para um gateway: a maior parte do trabalho é I/O (esperar respostas dos serviços internos), não processamento de CPU. Um gateway síncrono bloquearia a thread enquanto espera — o FastAPI não bloqueia.

#### Estrutura do projeto

```
meu-gateway/
├── gateway/
│   ├── __init__.py
│   ├── main.py          ← ponto de entrada, monta a app FastAPI
│   ├── config.py        ← mapa de rotas (qual path vai para qual serviço)
│   ├── proxy.py         ← lógica de encaminhar a requisição
│   ├── auth.py          ← validação de JWT
│   ├── rate_limit.py    ← controle de requisições por cliente
│   └── circuit.py       ← circuit breaker simples
├── .env
├── requirements.txt
└── docker-compose.yml
```

#### Instalação das dependências

```bash
pip install fastapi uvicorn httpx python-jose[cryptography] python-dotenv
```

- `fastapi` — framework web assíncrono
- `uvicorn` — servidor ASGI (roda a app FastAPI)
- `httpx` — cliente HTTP assíncrono (para encaminhar requisições aos serviços)
- `python-jose` — validação de JWT
- `python-dotenv` — lê variáveis de ambiente do arquivo `.env`

---

#### Passo 1 — config.py: o mapa de rotas

Este arquivo é o "cérebro" do gateway — ele define quais URLs externas mapeiam para quais serviços internos.

```python
# gateway/config.py
import os
from dotenv import load_dotenv

load_dotenv()  # carrega variáveis do arquivo .env

# Cada entrada define uma rota do gateway.
# "prefix"  → prefixo de URL que o cliente usa
# "target"  → endereço base do serviço interno
# "strip"   → se True, remove o prefixo antes de encaminhar
# "public"  → se True, não exige autenticação
ROUTES = [
    {
        "prefix": "/api/v1/auth",
        "target": os.getenv("AUTH_SERVICE_URL", "http://localhost:8001"),
        "strip": True,   # /api/v1/auth/login → /login no serviço de auth
        "public": True,  # login e registro não exigem token
    },
    {
        "prefix": "/api/v1/usuarios",
        "target": os.getenv("USUARIOS_SERVICE_URL", "http://localhost:8002"),
        "strip": True,
        "public": False,  # exige JWT válido
    },
    {
        "prefix": "/api/v1/pedidos",
        "target": os.getenv("PEDIDOS_SERVICE_URL", "http://localhost:8003"),
        "strip": True,
        "public": False,
    },
]

# Configurações de rate limiting
RATE_LIMIT_REQUESTS = int(os.getenv("RATE_LIMIT_REQUESTS", "100"))
RATE_LIMIT_WINDOW_SECONDS = int(os.getenv("RATE_LIMIT_WINDOW", "60"))

# Chave pública para verificar tokens JWT
# Em produção, busque do endpoint JWKS do seu auth server
JWT_SECRET = os.getenv("JWT_SECRET", "chave-secreta-aqui")
JWT_ALGORITHM = os.getenv("JWT_ALGORITHM", "HS256")
```

> Separar a configuração em um arquivo próprio é fundamental: quando um serviço muda de endereço, você altera apenas o `.env`, sem tocar na lógica do gateway.

---

#### Passo 2 — auth.py: validação de JWT

```python
# gateway/auth.py
from fastapi import HTTPException, Request
from jose import jwt, JWTError
from gateway.config import JWT_SECRET, JWT_ALGORITHM

def extrair_token(request: Request) -> str:
    """
    Lê o header Authorization e extrai o token Bearer.
    Lança 401 se o header não existir ou tiver formato errado.
    """
    auth_header = request.headers.get("Authorization")

    # Verifica se o header existe
    if not auth_header:
        raise HTTPException(status_code=401, detail="Header Authorization ausente")

    # O formato esperado é "Bearer <token>"
    partes = auth_header.split(" ")
    if len(partes) != 2 or partes[0].lower() != "bearer":
        raise HTTPException(status_code=401, detail="Formato inválido. Use: Bearer <token>")

    return partes[1]  # retorna apenas o token, sem o "Bearer"


def validar_token(token: str) -> dict:
    """
    Decodifica e valida o JWT.
    Retorna o payload (claims) se válido.
    Lança 401 se inválido ou expirado.
    """
    try:
        # jose.jwt.decode verifica:
        #   - a assinatura (usando JWT_SECRET)
        #   - se o token expirou (campo "exp" no payload)
        #   - o algoritmo usado (HS256 por padrão)
        payload = jwt.decode(token, JWT_SECRET, algorithms=[JWT_ALGORITHM])
        return payload

    except JWTError as e:
        # JWTError cobre: token adulterado, expirado, assinatura inválida
        raise HTTPException(status_code=401, detail=f"Token inválido: {str(e)}")


def autenticar(request: Request) -> dict:
    """
    Função de conveniência: extrai + valida em um só passo.
    Retorna o payload do JWT (contém user_id, roles, etc).
    """
    token = extrair_token(request)
    return validar_token(token)
```

**Por que retornar o payload?**
O payload do JWT contém informações como `sub` (user ID), `roles`, `email`. Após validar, o gateway injeta esses dados nos headers que encaminha ao serviço interno — assim o serviço não precisa ler nem validar o JWT de novo.

---

#### Passo 3 — rate_limit.py: controle de requisições

```python
# gateway/rate_limit.py
import time
from collections import defaultdict
from fastapi import HTTPException, Request
from gateway.config import RATE_LIMIT_REQUESTS, RATE_LIMIT_WINDOW_SECONDS

# Dicionário em memória: chave = IP do cliente, valor = lista de timestamps
# defaultdict(list) cria automaticamente uma lista vazia para IPs novos
_janelas: dict[str, list[float]] = defaultdict(list)

def checar_rate_limit(request: Request) -> None:
    """
    Implementa o algoritmo Sliding Window (janela deslizante).

    Guarda o timestamp de cada requisição e descarta as mais antigas
    que a janela de tempo. Se o número de requisições recentes exceder
    o limite, lança 429 Too Many Requests.
    """
    # Identifica o cliente pelo IP (em produção, considere usar o JWT sub)
    client_ip = request.client.host
    agora = time.time()

    # Remove timestamps antigos (fora da janela de tempo)
    # Exemplo: se a janela é 60s, descarta tudo antes de "agora - 60s"
    _janelas[client_ip] = [
        ts for ts in _janelas[client_ip]
        if agora - ts < RATE_LIMIT_WINDOW_SECONDS
    ]

    # Conta quantas requisições existem dentro da janela atual
    requisicoes_na_janela = len(_janelas[client_ip])

    if requisicoes_na_janela >= RATE_LIMIT_REQUESTS:
        # Calcula quando a janela vai resetar (o timestamp mais antigo + janela)
        reset_em = int(_janelas[client_ip][0] + RATE_LIMIT_WINDOW_SECONDS)
        raise HTTPException(
            status_code=429,
            detail="Limite de requisições excedido",
            headers={
                "X-RateLimit-Limit": str(RATE_LIMIT_REQUESTS),
                "X-RateLimit-Remaining": "0",
                "X-RateLimit-Reset": str(reset_em),
                "Retry-After": str(reset_em - int(agora)),
            }
        )

    # Registra a requisição atual
    _janelas[client_ip].append(agora)
```

> **Limitação desta implementação:** o estado fica em memória — se o gateway tiver múltiplas instâncias, cada uma tem seu próprio contador. Para produção com múltiplas instâncias, substitua o dicionário em memória por Redis.

---

#### Passo 4 — circuit.py: circuit breaker simples

```python
# gateway/circuit.py
import time
from enum import Enum

class Estado(Enum):
    FECHADO = "fechado"      # operação normal
    ABERTO = "aberto"        # serviço falhando, rejeita requisições
    MEIO_ABERTO = "meio_aberto"  # testando se o serviço se recuperou

class CircuitBreaker:
    def __init__(
        self,
        servico: str,
        limite_falhas: int = 5,       # quantas falhas antes de abrir
        timeout_recovery: int = 30,   # segundos até tentar novamente
    ):
        self.servico = servico
        self.limite_falhas = limite_falhas
        self.timeout_recovery = timeout_recovery

        self.estado = Estado.FECHADO
        self.contagem_falhas = 0
        self.ultimo_aberto_em: float | None = None

    def pode_passar(self) -> bool:
        """Retorna True se a requisição pode ser encaminhada ao serviço."""

        if self.estado == Estado.FECHADO:
            return True  # operação normal

        if self.estado == Estado.ABERTO:
            # Verifica se já passou tempo suficiente para tentar novamente
            if time.time() - self.ultimo_aberto_em >= self.timeout_recovery:
                self.estado = Estado.MEIO_ABERTO
                return True  # deixa uma requisição de teste passar
            return False  # ainda em recuperação, rejeita

        if self.estado == Estado.MEIO_ABERTO:
            return True  # uma requisição de teste

    def registrar_sucesso(self) -> None:
        """Chamado quando a requisição ao serviço retornou com sucesso."""
        self.contagem_falhas = 0
        self.estado = Estado.FECHADO  # serviço se recuperou

    def registrar_falha(self) -> None:
        """Chamado quando a requisição ao serviço falhou."""
        self.contagem_falhas += 1

        if self.estado == Estado.MEIO_ABERTO:
            # Teste falhou — volta a ficar aberto
            self.estado = Estado.ABERTO
            self.ultimo_aberto_em = time.time()
            return

        if self.contagem_falhas >= self.limite_falhas:
            self.estado = Estado.ABERTO
            self.ultimo_aberto_em = time.time()

# Um circuit breaker por serviço, criados na inicialização
_breakers: dict[str, CircuitBreaker] = {}

def obter_breaker(servico_url: str) -> CircuitBreaker:
    if servico_url not in _breakers:
        _breakers[servico_url] = CircuitBreaker(servico=servico_url)
    return _breakers[servico_url]
```

---

#### Passo 5 — proxy.py: o coração do gateway

Este é o arquivo mais importante — ele encaminha a requisição original para o serviço interno e devolve a resposta ao cliente.

```python
# gateway/proxy.py
import httpx
from fastapi import Request, HTTPException
from fastapi.responses import Response
from gateway.circuit import obter_breaker

# Cliente HTTP assíncrono compartilhado entre todas as requisições.
# httpx.AsyncClient mantém um pool de conexões — muito mais eficiente
# do que criar um novo cliente a cada requisição.
_cliente_http = httpx.AsyncClient(timeout=10.0)


async def encaminhar(request: Request, url_destino: str) -> Response:
    """
    Encaminha a requisição recebida para o serviço interno.

    Preserva:
    - método HTTP (GET, POST, PUT, DELETE, etc.)
    - headers originais
    - query parameters (?filtro=ativo&pagina=2)
    - corpo da requisição (body JSON, form data, etc.)
    """
    breaker = obter_breaker(url_destino)

    # Verifica o circuit breaker antes de tentar
    if not breaker.pode_passar():
        raise HTTPException(
            status_code=503,
            detail=f"Serviço temporariamente indisponível (circuit breaker aberto)"
        )

    # Lê o corpo da requisição original
    # await é necessário porque a leitura do body é assíncrona
    body = await request.body()

    # Copia os headers da requisição original, exceto "host"
    # O header "host" deve ser o do serviço interno, não o do gateway
    headers = {
        chave: valor
        for chave, valor in request.headers.items()
        if chave.lower() != "host"
    }

    try:
        # Faz a requisição ao serviço interno
        resposta_interna = await _cliente_http.request(
            method=request.method,       # GET, POST, etc.
            url=url_destino,             # URL completa já montada
            headers=headers,             # headers originais do cliente
            params=request.query_params, # query string (?foo=bar)
            content=body,                # corpo da requisição
        )
        breaker.registrar_sucesso()

    except httpx.TimeoutException:
        breaker.registrar_falha()
        raise HTTPException(status_code=504, detail="Timeout ao chamar serviço interno")

    except httpx.ConnectError:
        breaker.registrar_falha()
        raise HTTPException(status_code=502, detail="Não foi possível conectar ao serviço")

    # Devolve a resposta do serviço interno diretamente ao cliente
    # Preserva: status code, headers de resposta e body
    return Response(
        content=resposta_interna.content,
        status_code=resposta_interna.status_code,
        headers=dict(resposta_interna.headers),
        media_type=resposta_interna.headers.get("content-type"),
    )
```

---

#### Passo 6 — main.py: juntando tudo

```python
# gateway/main.py
import logging
import time
from fastapi import FastAPI, Request
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import JSONResponse

from gateway.config import ROUTES
from gateway.auth import autenticar
from gateway.rate_limit import checar_rate_limit
from gateway.proxy import encaminhar

# Configuração de logging — cada requisição gera uma linha no log
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s %(levelname)s %(message)s"
)
logger = logging.getLogger("gateway")

app = FastAPI(title="Meu API Gateway", version="1.0.0")

# ── CORS ──────────────────────────────────────────────────────────────────────
# Middleware do FastAPI que adiciona os headers de CORS em todas as respostas.
# "allow_origins" define quais domínios podem fazer requisições ao gateway.
# Em produção, substitua o "*" pelos domínios reais do seu frontend.
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],          # domínios permitidos
    allow_credentials=True,       # permite cookies e headers de autenticação
    allow_methods=["*"],          # GET, POST, PUT, DELETE, etc.
    allow_headers=["*"],          # todos os headers são permitidos
)


# ── Middleware de logging e tempo de resposta ─────────────────────────────────
# Middleware é uma função que envolve TODAS as requisições.
# "call_next" chama o próximo handler na cadeia (a rota propriamente dita).
@app.middleware("http")
async def logging_middleware(request: Request, call_next):
    inicio = time.time()

    # Executa o handler da rota
    response = await call_next(request)

    duracao_ms = (time.time() - inicio) * 1000

    logger.info(
        "%s %s → %d (%.1fms) | IP: %s",
        request.method,
        request.url.path,
        response.status_code,
        duracao_ms,
        request.client.host,
    )

    # Adiciona o tempo de resposta no header para facilitar debug
    response.headers["X-Response-Time"] = f"{duracao_ms:.1f}ms"
    return response


# ── Rota catchall: captura QUALQUER path e QUALQUER método ───────────────────
# {caminho:path} é um path parameter especial do FastAPI que captura
# qualquer string, incluindo barras (/usuarios/42/pedidos).
# methods=["GET","POST","PUT","DELETE","PATCH"] cobre todos os métodos REST.
@app.api_route(
    "/{caminho:path}",
    methods=["GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS", "HEAD"],
)
async def gateway_handler(caminho: str, request: Request):
    """
    Handler principal do gateway.
    Recebe TODAS as requisições e decide o que fazer com cada uma.
    """

    # 1. Rate limiting — independente de autenticação
    checar_rate_limit(request)

    # 2. Encontra qual rota do config.py corresponde ao path atual
    rota_encontrada = None
    for rota in ROUTES:
        if request.url.path.startswith(rota["prefix"]):
            rota_encontrada = rota
            break

    if rota_encontrada is None:
        return JSONResponse(
            status_code=404,
            content={"error": f"Rota '{request.url.path}' não encontrada no gateway"}
        )

    # 3. Autenticação — só para rotas não públicas
    user_payload = None
    if not rota_encontrada["public"]:
        user_payload = autenticar(request)  # lança 401 se inválido

    # 4. Monta a URL de destino
    # strip=True: remove o prefixo antes de encaminhar
    # Ex: /api/v1/pedidos/123 → /123 no serviço de pedidos
    if rota_encontrada["strip"]:
        path_sem_prefixo = request.url.path[len(rota_encontrada["prefix"]):]
        path_sem_prefixo = path_sem_prefixo or "/"  # garante que não fique vazio
    else:
        path_sem_prefixo = request.url.path

    url_destino = rota_encontrada["target"].rstrip("/") + path_sem_prefixo

    # 5. Injeta informações do usuário autenticado nos headers
    # Os serviços internos podem ler esses headers sem precisar validar o JWT
    if user_payload:
        request.headers.__dict__["_list"].append(
            (b"x-user-id",    str(user_payload.get("sub", "")).encode())
        )
        request.headers.__dict__["_list"].append(
            (b"x-user-roles", str(user_payload.get("roles", "")).encode())
        )

    # 6. Encaminha para o serviço e retorna a resposta
    return await encaminhar(request, url_destino)
```

> **Por que uma rota catchall?**
> Em vez de declarar uma rota para cada endpoint de cada serviço, o gateway captura tudo e decide dinamicamente. Se você registrar um novo serviço no `config.py`, não precisa alterar o `main.py`.

---

#### Passo 7 — .env: variáveis de ambiente

```bash
# .env
AUTH_SERVICE_URL=http://localhost:8001
USUARIOS_SERVICE_URL=http://localhost:8002
PEDIDOS_SERVICE_URL=http://localhost:8003

JWT_SECRET=minha-chave-secreta-muito-longa-e-segura
JWT_ALGORITHM=HS256

RATE_LIMIT_REQUESTS=100
RATE_LIMIT_WINDOW=60
```

---

#### Passo 8 — Executando e testando

```bash
# Rodar o gateway
uvicorn gateway.main:app --port 8000 --reload

# Testar: rota pública (sem token)
curl http://localhost:8000/api/v1/auth/login \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"email": "user@teste.com", "password": "123"}'

# Testar: rota protegida (com token)
curl http://localhost:8000/api/v1/pedidos \
  -H "Authorization: Bearer SEU_TOKEN_JWT_AQUI"

# Testar: rota inválida
curl http://localhost:8000/api/v1/inexistente
# → 404 {"error": "Rota '/api/v1/inexistente' não encontrada no gateway"}

# Testar rate limit: execute 101 vezes rapidamente
for i in $(seq 1 101); do
  curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8000/api/v1/auth/ping
done
# Os primeiros 100 retornam 200, o 101° retorna 429
```

---

#### Passo 9 — docker-compose.yml

```yaml
version: '3.8'

services:
  gateway:
    build: .
    ports:
      - "8000:8000"
    env_file: .env
    depends_on:
      - servico-auth
      - servico-usuarios
      - servico-pedidos

  servico-auth:
    build: ./servico-auth
    expose:
      - "8001"

  servico-usuarios:
    build: ./servico-usuarios
    expose:
      - "8002"

  servico-pedidos:
    build: ./servico-pedidos
    expose:
      - "8003"
```

**Dockerfile do gateway:**
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY gateway/ ./gateway/
COPY .env .
CMD ["uvicorn", "gateway.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

#### Fluxo completo de uma requisição autenticada

```
Cliente: GET /api/v1/pedidos/42
         Authorization: Bearer eyJ...

         │
         ▼
  gateway_handler()
         │
  ① checar_rate_limit()  → IP tem tokens disponíveis? Continua.
         │
  ② loop em ROUTES       → /api/v1/pedidos casa com a 3ª rota
         │
  ③ autenticar()         → JWT válido? Extrai sub="user-99", roles="cliente"
         │
  ④ monta URL            → http://localhost:8003/42
         │
  ⑤ injeta headers       → X-User-ID: user-99 / X-User-Roles: cliente
         │
  ⑥ encaminhar()         → Circuit breaker: FECHADO, pode passar
         │
  ⑦ httpx.request()      → GET http://localhost:8003/42
         │
  ⑧ Serviço de pedidos   → recebe a requisição, lê X-User-ID do header
         │
         ▼
      Response 200 {"id": 42, "status": "em_entrega"}
         │
         ▼
  logging_middleware()   → "GET /api/v1/pedidos/42 → 200 (12.3ms)"
         │
         ▼
  Cliente recebe a resposta
```

## 6. Integração com Auth0

O Gateway e o Auth0 trabalham juntos de forma complementar:

```
  Cliente
     │
     │  1. Usuário faz login
     ▼
  Auth0 ──────────────────────────────────────────────────────┐
     │  Emite Access Token (JWT)                              │
     │                                                        │
     │  2. Cliente usa o token nas requisições                │
     ▼                                                        │
  API Gateway                                                 │
     │  3. Gateway valida o token                             │
     │     usando a chave pública do Auth0 (JWKS endpoint)   │
     │     ↑                                                  │
     └─────┘  https://SEU_TENANT.auth0.com/.well-known/jwks.json
     │
     │  4. Token válido → injeta headers + encaminha
     │     X-User-ID: 42
     │     X-User-Roles: admin,editor
     ▼
  Serviço Interno
     │  5. Serviço confia nos headers injetados pelo Gateway
     │     (não valida o token novamente)
     ▼
  Resposta
```

Os serviços internos **não precisam conhecer o Auth0** — eles apenas leem os headers que o Gateway injetou. Isso simplifica enormemente cada serviço.

---

## 7. Diagrama Completo da Arquitetura

```
                              INTERNET
                                 │
                          ┌──────▼──────┐
                          │   DNS /     │
                          │   CDN       │
                          └──────┬──────┘
                                 │
                          ┌──────▼──────────────────────────────────┐
                          │           API GATEWAY                    │
                          │                                          │
                          │  ┌─────────┐  ┌──────────┐             │
                          │  │  Auth   │  │  Rate    │             │
                          │  │  JWT    │  │  Limit   │             │
                          │  └─────────┘  └──────────┘             │
                          │  ┌─────────┐  ┌──────────┐             │
                          │  │  CORS   │  │  Cache   │             │
                          │  └─────────┘  └──────────┘             │
                          │  ┌─────────────────────────┐           │
                          │  │       Roteamento        │           │
                          │  └─────────────────────────┘           │
                          └────────────────┬─────────────────────────┘
                                           │
              ┌────────────────────────────┼────────────────────────────┐
              │                            │                            │
       ┌──────▼──────┐             ┌───────▼─────┐             ┌───────▼─────┐
       │  Serviço de  │             │  Serviço de │             │  Serviço de │
       │  Usuários    │             │  Pedidos    │             │  Produtos   │
       │              │             │             │             │             │
       │  ┌─────────┐ │             │ ┌─────────┐ │             │ ┌─────────┐ │
       │  │   DB    │ │             │ │   DB    │ │             │ │   DB    │ │
       │  └─────────┘ │             │ └─────────┘ │             │ └─────────┘ │
       └──────────────┘             └─────────────┘             └─────────────┘
              │                            │
              └──────────────┬─────────────┘
                             │
                      ┌──────▼──────┐
                      │    Redis    │ ← Rate limit state
                      │   Cache     │   Session cache
                      └─────────────┘
```

---

## 8. Pontos de Atenção

| Situação | Recomendação |
|---|---|
| **Gateway como ponto único de falha** | Rode múltiplas instâncias do Gateway com load balancer na frente |
| **Latência adicionada** | O Gateway adiciona ~1-5ms por requisição — aceitável, mas monitore |
| **Lógica de negócio entrando no Gateway** | Resista à tentação. Mantenha o Gateway como camada de infraestrutura |
| **JWKS caching** | Faça cache da chave pública do Auth0 — buscar a cada request é lento e desnecessário |
| **Timeouts** | Configure timeouts agressivos no Gateway para não deixar conexões penduradas se um serviço travar |
| **Observabilidade** | O Gateway é o lugar ideal para coletar métricas: latência por rota, taxa de erro, uso por cliente |
| **Versões de API** | Use o Gateway para gerenciar `/v1` e `/v2` de forma transparente para os serviços internos |
