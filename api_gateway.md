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

### 5.1 Opções de Solução

| Solução | Tipo | Quando usar |
|---|---|---|
| **Kong Gateway** | Open-source / SaaS | Microsserviços, altamente extensível via plugins |
| **AWS API Gateway** | Managed (cloud) | Arquiteturas serverless (Lambda) na AWS |
| **Nginx** | Reverse proxy / manual | Controle total, configuração manual, baixa complexidade |
| **Traefik** | Open-source | Integração nativa com Docker e Kubernetes |
| **Azure API Management** | Managed (cloud) | Ecossistema Microsoft Azure |
| **Express Gateway** | Node.js | Projetos Node que querem um gateway customizável no próprio runtime |

### 5.2 Implementação com Nginx (Simples e Direto)

O Nginx é a forma mais básica de criar um gateway — um reverse proxy com roteamento por path:

**Estrutura do projeto:**
```
projeto/
├── nginx/
│   └── nginx.conf
├── servico-usuarios/
├── servico-pedidos/
└── docker-compose.yml
```

**nginx/nginx.conf:**
```nginx
upstream usuarios {
    server servico-usuarios:3001;
}

upstream pedidos {
    server servico-pedidos:3002;
}

upstream produtos {
    server servico-produtos:3003;
}

server {
    listen 80;

    # Rate limiting: define zona de memória compartilhada
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

    # Rota: /api/v1/usuarios/* → serviço de usuários
    location /api/v1/usuarios/ {
        limit_req zone=api burst=20 nodelay;

        rewrite ^/api/v1/usuarios/(.*) /$1 break;
        proxy_pass http://usuarios;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Rota: /api/v1/pedidos/* → serviço de pedidos
    location /api/v1/pedidos/ {
        limit_req zone=api burst=20 nodelay;

        rewrite ^/api/v1/pedidos/(.*) /$1 break;
        proxy_pass http://pedidos;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    }

    # Retorna 404 para rotas não mapeadas
    location / {
        return 404 '{"error": "Rota não encontrada"}';
        add_header Content-Type application/json;
    }
}
```

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  gateway:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - servico-usuarios
      - servico-pedidos

  servico-usuarios:
    build: ./servico-usuarios
    expose:
      - "3001"

  servico-pedidos:
    build: ./servico-pedidos
    expose:
      - "3002"
```

### 5.3 Implementação com Kong Gateway

Kong é um gateway de produção construído sobre o Nginx, com um sistema de plugins poderoso:

**docker-compose.yml com Kong:**
```yaml
version: '3.8'

services:
  kong-db:
    image: postgres:15
    environment:
      POSTGRES_DB: kong
      POSTGRES_USER: kong
      POSTGRES_PASSWORD: kong

  kong-migrations:
    image: kong:3.6
    command: kong migrations bootstrap
    environment:
      KONG_DATABASE: postgres
      KONG_PG_HOST: kong-db
      KONG_PG_PASSWORD: kong
    depends_on:
      - kong-db

  kong:
    image: kong:3.6
    environment:
      KONG_DATABASE: postgres
      KONG_PG_HOST: kong-db
      KONG_PG_PASSWORD: kong
      KONG_PROXY_ACCESS_LOG: /dev/stdout
      KONG_ADMIN_ACCESS_LOG: /dev/stdout
      KONG_PROXY_ERROR_LOG: /dev/stderr
      KONG_ADMIN_ERROR_LOG: /dev/stderr
      KONG_ADMIN_LISTEN: 0.0.0.0:8001
    ports:
      - "80:8000"      # proxy (requisições dos clientes)
      - "8001:8001"    # admin API (configuração)
    depends_on:
      - kong-migrations
```

**Configurando serviços e rotas via Admin API:**
```bash
# 1. Registrar o serviço interno
curl -X POST http://localhost:8001/services \
  --data name=servico-pedidos \
  --data url=http://servico-pedidos:3002

# 2. Criar rota para o serviço
curl -X POST http://localhost:8001/services/servico-pedidos/routes \
  --data "paths[]=/api/v1/pedidos" \
  --data "strip_path=true"

# 3. Habilitar plugin de autenticação JWT
curl -X POST http://localhost:8001/services/servico-pedidos/plugins \
  --data name=jwt

# 4. Habilitar rate limiting
curl -X POST http://localhost:8001/services/servico-pedidos/plugins \
  --data name=rate-limiting \
  --data config.minute=100 \
  --data config.policy=redis \
  --data config.redis_host=redis

# 5. Habilitar CORS
curl -X POST http://localhost:8001/plugins \
  --data name=cors \
  --data config.origins=https://meusite.com \
  --data config.methods=GET,POST,PUT,DELETE \
  --data config.headers=Authorization,Content-Type
```

**Configurando via arquivo declarativo (kong.yml):**
```yaml
# kong.yml — configuração como código (recomendado para produção)
_format_version: "3.0"

services:
  - name: servico-pedidos
    url: http://servico-pedidos:3002
    routes:
      - name: rota-pedidos
        paths:
          - /api/v1/pedidos
        strip_path: true
    plugins:
      - name: jwt
      - name: rate-limiting
        config:
          minute: 100
          policy: redis
          redis_host: redis

  - name: servico-usuarios
    url: http://servico-usuarios:3001
    routes:
      - name: rota-usuarios
        paths:
          - /api/v1/usuarios
        strip_path: true
    plugins:
      - name: jwt
      - name: rate-limiting
        config:
          minute: 200

plugins:
  - name: cors
    config:
      origins:
        - https://meusite.com
      methods:
        - GET
        - POST
        - PUT
        - DELETE
      headers:
        - Authorization
        - Content-Type
```

### 5.4 Implementação com Node.js / Express (Gateway Customizado)

Para casos onde você precisa de lógica específica no gateway, ou quer começar simples:

```javascript
// gateway.js
const express = require('express');
const { createProxyMiddleware } = require('http-proxy-middleware');
const rateLimit = require('express-rate-limit');
const jwt = require('jsonwebtoken');

const app = express();

// ── Middleware de Rate Limiting ──────────────────────────────
const limiter = rateLimit({
  windowMs: 60 * 1000,   // 1 minuto
  max: 100,              // 100 requisições por IP
  message: { error: 'Muitas requisições. Tente novamente em 1 minuto.' },
  standardHeaders: true,
  legacyHeaders: false,
});
app.use(limiter);

// ── Middleware de Autenticação ───────────────────────────────
function autenticar(req, res, next) {
  const authHeader = req.headers['authorization'];

  if (!authHeader?.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'Token não fornecido' });
  }

  const token = authHeader.split(' ')[1];

  try {
    const payload = jwt.verify(token, process.env.JWT_PUBLIC_KEY, {
      algorithms: ['RS256'],
      audience: process.env.JWT_AUDIENCE,
      issuer: process.env.JWT_ISSUER,
    });

    // Injeta dados do usuário para os serviços downstream
    req.headers['x-user-id']    = payload.sub;
    req.headers['x-user-roles'] = (payload.permissions || []).join(',');

    next();
  } catch (err) {
    return res.status(401).json({ error: 'Token inválido ou expirado' });
  }
}

// ── Rotas Públicas (sem autenticação) ───────────────────────
app.use('/api/v1/auth', createProxyMiddleware({
  target: process.env.AUTH_SERVICE_URL,
  changeOrigin: true,
  pathRewrite: { '^/api/v1/auth': '' },
}));

// ── Rotas Privadas (com autenticação) ───────────────────────
app.use('/api/v1/pedidos', autenticar, createProxyMiddleware({
  target: process.env.PEDIDOS_SERVICE_URL,
  changeOrigin: true,
  pathRewrite: { '^/api/v1/pedidos': '' },
  on: {
    error: (err, req, res) => {
      console.error('Erro no proxy:', err.message);
      res.status(502).json({ error: 'Serviço indisponível' });
    },
  },
}));

app.use('/api/v1/usuarios', autenticar, createProxyMiddleware({
  target: process.env.USUARIOS_SERVICE_URL,
  changeOrigin: true,
  pathRewrite: { '^/api/v1/usuarios': '' },
}));

// ── Rota não encontrada ──────────────────────────────────────
app.use((req, res) => {
  res.status(404).json({ error: `Rota ${req.path} não encontrada` });
});

app.listen(3000, () => console.log('Gateway rodando na porta 3000'));
```

**Variáveis de ambiente (.env):**
```bash
JWT_PUBLIC_KEY="-----BEGIN PUBLIC KEY-----\n..."
JWT_AUDIENCE=https://api.meuapp.com
JWT_ISSUER=https://meutenante.auth0.com/

AUTH_SERVICE_URL=http://servico-auth:4001
PEDIDOS_SERVICE_URL=http://servico-pedidos:4002
USUARIOS_SERVICE_URL=http://servico-usuarios:4003
```

### 5.5 Gateway com AWS API Gateway + Lambda

Para arquiteturas serverless na AWS, o API Gateway gerenciado é o caminho natural:

```
Cliente
   │
   ▼
AWS API Gateway
   │
   ├── GET  /pedidos      → Lambda: listPedidos
   ├── POST /pedidos      → Lambda: criarPedido
   ├── GET  /pedidos/{id} → Lambda: buscarPedido
   └── DELETE /pedidos/{id} → Lambda: deletarPedido
```

**Configuração via AWS CDK (TypeScript):**
```typescript
import * as cdk from 'aws-cdk-lib';
import * as apigateway from 'aws-cdk-lib/aws-apigateway';
import * as lambda from 'aws-cdk-lib/aws-lambda';

const api = new apigateway.RestApi(this, 'MinhaApi', {
  restApiName: 'minha-api',
  defaultCorsPreflightOptions: {
    allowOrigins: apigateway.Cors.ALL_ORIGINS,
    allowMethods: apigateway.Cors.ALL_METHODS,
  },
});

// Authorizer JWT (Auth0)
const authorizer = new apigateway.CognitoUserPoolsAuthorizer(this, 'Authorizer', {
  cognitoUserPools: [userPool],
});

// Lambda handler
const pedidosLambda = new lambda.Function(this, 'PedidosHandler', {
  runtime: lambda.Runtime.NODEJS_20_X,
  code: lambda.Code.fromAsset('lambda/pedidos'),
  handler: 'index.handler',
});

// Recurso e método com autenticação
const pedidos = api.root.addResource('pedidos');
pedidos.addMethod('GET', new apigateway.LambdaIntegration(pedidosLambda), {
  authorizer,
  authorizationType: apigateway.AuthorizationType.COGNITO,
});
```

---

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
