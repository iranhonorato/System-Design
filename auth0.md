# Auth0

## O que é Auth0?

Auth0 é uma **plataforma de identidade como serviço (IDaaS — Identity as a Service)** que resolve um dos problemas mais críticos e complexos do desenvolvimento de software: **autenticação e autorização de usuários**.

Em vez de você construir do zero toda a infraestrutura de login, registro, recuperação de senha, autenticação multifator, OAuth e controle de permissões, o Auth0 fornece isso tudo como um serviço gerenciado, via APIs e SDKs prontos para uso.

> **Analogia:** Imagine que você está construindo um prédio comercial. Você poderia contratar um especialista em segurança para instalar câmeras, controle de acesso e gerenciar credenciais dos funcionários — ou poderia construir um departamento de segurança do zero. O Auth0 é o especialista terceirizado: você delega o problema de identidade para quem nasceu para resolvê-lo.

### Por que não construir autenticação do zero?

Construir autenticação segura é incrivelmente difícil:

| Problema | O que pode dar errado |
|---|---|
| **Armazenamento de senhas** | Senhas em texto plano, hash fraco (MD5), sem salt |
| **Tokens de sessão** | Tokens previsíveis, sem expiração, vulneráveis a roubo |
| **OAuth/OIDC** | Implementação incorreta dos flows, vulnerabilidades em callbacks |
| **Brute force** | Sem rate limiting, bots conseguem testar senhas ilimitadamente |
| **MFA** | Implementação complexa, suporte a múltiplos métodos |
| **Compliance** | LGPD, GDPR, SOC 2 exigem controles específicos de identidade |

O custo de errar é alto: uma falha de autenticação pode expor todos os dados dos seus usuários. O Auth0 centraliza essa responsabilidade em uma plataforma especializada, auditada e constantemente atualizada.

---

## 1. Para que o Auth0 Serve?

O Auth0 não resolve apenas "login e senha". Ele é uma plataforma completa de gerenciamento de identidade:

### Casos de uso principais

| Funcionalidade | O que resolve |
|---|---|
| **Login social** | Entrar com Google, GitHub, Microsoft, Facebook sem gerenciar OAuth manualmente |
| **Single Sign-On (SSO)** | Um único login dá acesso a múltiplos sistemas da empresa |
| **Autenticação Multifator (MFA)** | SMS, TOTP (Google Authenticator), notificações push |
| **Autorização baseada em roles** | Controlar o que cada tipo de usuário pode fazer |
| **Login por magic link** | Acesso via e-mail sem senha |
| **Passwordless** | Autenticação sem senhas usando biometria ou links temporários |
| **B2B federado** | Empresas clientes trazem seus próprios provedores de identidade (SAML, LDAP) |
| **Gestão de usuários** | Dashboard para criar, bloquear, resetar senhas, ver logs de acesso |
| **Tokens padronizados** | Emissão de JWT (Access Token e ID Token) no padrão OpenID Connect |

### O que o Auth0 NÃO é

- **Não é um banco de dados de negócios:** armazena dados de identidade (e-mail, nome, roles), não dados do domínio da sua aplicação.
- **Não substitui autorização granular:** define *quem pode o quê* em nível de roles, mas regras de negócio complexas ainda ficam na sua aplicação.
- **Não é gratuito em escala:** a camada gratuita é limitada. Volumes altos de usuários ativos mensais têm custo.

---

## 2. Como o Auth0 Funciona

### 2.1 Os Conceitos Centrais

Antes de entender o fluxo, é essencial conhecer os atores:

```
┌─────────────┐     ┌──────────────────┐     ┌─────────────────┐
│             │     │                  │     │                 │
│   Usuário   │────▶│  Sua Aplicação   │────▶│     Auth0       │
│             │     │  (Client)        │     │  (Auth Server)  │
└─────────────┘     └──────────────────┘     └─────────────────┘
                                                      │
                                             ┌────────▼────────┐
                                             │  Sua API        │
                                             │  (Resource      │
                                             │   Server)       │
                                             └─────────────────┘
```

| Conceito | O que é |
|---|---|
| **Tenant** | Seu "espaço" isolado no Auth0 — uma instância dedicada à sua aplicação |
| **Application** | Representa seu frontend/backend registrado no Auth0 |
| **API** | Representa sua API protegida que valida tokens do Auth0 |
| **Connection** | Fonte de identidade: banco de dados próprio, Google, GitHub, SAML, etc. |
| **Access Token** | JWT que sua API valida para autorizar requisições |
| **ID Token** | JWT com informações do usuário autenticado (para o client) |
| **Refresh Token** | Token de longa duração para renovar Access Tokens expirados |

### 2.2 Os Protocolos por Baixo

O Auth0 implementa padrões abertos da indústria:

```
OpenID Connect (OIDC)  →  Quem é o usuário? (Autenticação)
       ↓
   Emite ID Token (JWT com perfil do usuário)

OAuth 2.0              →  O usuário permite que eu acesse X? (Autorização)
       ↓
   Emite Access Token (JWT para acessar APIs protegidas)
```

Você não precisa conhecer esses protocolos em detalhes para usar o Auth0 — os SDKs abstraem tudo isso. Mas entender que eles existem ajuda a depurar problemas e tomar decisões de arquitetura.

### 2.3 O Fluxo de Autenticação (Authorization Code Flow)

Este é o fluxo padrão para aplicações web com backend:

```
Usuário         Sua Aplicação        Auth0             Sua API
   │                  │                 │                  │
   │  Clica "Login"   │                 │                  │
   │─────────────────▶│                 │                  │
   │                  │  Redireciona    │                  │
   │                  │────────────────▶│                  │
   │                  │                 │                  │
   │◀────────────────────────────────── │                  │
   │  Tela de Login do Auth0            │                  │
   │                                    │                  │
   │  Insere credenciais                │                  │
   │──────────────────────────────────▶ │                  │
   │                                    │                  │
   │  Auth0 valida. Retorna "code"      │                  │
   │◀──────────────── ──────────────────│                  │
   │                  │                 │                  │
   │  Redireciona     │                 │                  │
   │─────────────────▶│                 │                  │
   │                  │  Troca code     │                  │
   │                  │  por tokens     │                  │
   │                  │────────────────▶│                  │
   │                  │◀────────────────│                  │
   │                  │  Access Token   │                  │
   │                  │  + ID Token     │                  │
   │                  │                 │                  │
   │                  │  Usa Access Token em requisições   │
   │                  │───────────────────────────────────▶│
   │                  │                 │  Valida JWT      │
   │                  │                 │◀─────────────────│
   │                  │◀───────────────────────────────────│
   │                  │  Resposta da API                   │
```

**Por que existe o "code" intermediário?**
Segurança. O `code` é de uso único, de curta duração, e a troca pelos tokens acontece de servidor para servidor (sem exposição no browser), evitando que tokens sejam interceptados.

### 2.4 Validação do Token na API

Quando sua API recebe uma requisição, ela valida o Access Token sem precisar consultar o Auth0 a cada chamada:

```
Requisição chega com header:
  Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
         │
         ▼
┌─────────────────────────────────────┐
│  Sua API valida o JWT:              │
│                                     │
│  1. Verifica a assinatura com a     │
│     chave pública do Auth0 (JWKS)   │
│                                     │
│  2. Verifica se não expirou (exp)   │
│                                     │
│  3. Verifica o audience (aud)       │
│     (esse token é pra minha API?)   │
│                                     │
│  4. Verifica as permissões (scope)  │
│     (esse token tem read:users?)    │
└─────────────────────────────────────┘
         │
         ▼
  Autoriza ou rejeita a requisição
```

A validação é **local** — sua API usa criptografia assimétrica (RS256) para verificar que o token foi emitido pelo Auth0, sem roundtrip de rede.

---

## 3. Como Implementar

### 3.1 Configuração Inicial no Auth0 Dashboard

Antes de escrever código, configure no painel do Auth0:

1. **Crie um Tenant** em [manage.auth0.com](https://manage.auth0.com)
2. **Crie uma Application** → escolha o tipo:
   - `Single Page Application` para React/Vue/Angular
   - `Regular Web Application` para apps com backend (Node, Python, Java)
   - `Machine to Machine` para comunicação entre serviços
3. **Registre sua API** → Applications → APIs → Create API
   - Defina um `identifier` (URI que identifica sua API, ex: `https://api.meuapp.com`)
4. **Configure URLs permitidas:**
   - `Allowed Callback URLs`: onde o Auth0 redireciona após login
   - `Allowed Logout URLs`: onde redireciona após logout
   - `Allowed Web Origins`: domínios que podem fazer requisições

### 3.2 Implementação no Frontend (React)

**Instalação:**
```bash
npm install @auth0/auth0-react
```

**Configuração do Provider:**
```jsx
// index.jsx
import { Auth0Provider } from '@auth0/auth0-react';

const root = ReactDOM.createRoot(document.getElementById('root'));
root.render(
  <Auth0Provider
    domain="SEU_DOMINIO.auth0.com"
    clientId="SEU_CLIENT_ID"
    authorizationParams={{
      redirect_uri: window.location.origin,
      audience: "https://api.meuapp.com",  // identifier da sua API
    }}
  >
    <App />
  </Auth0Provider>
);
```

**Usando autenticação nos componentes:**
```jsx
import { useAuth0 } from '@auth0/auth0-react';

function NavBar() {
  const { isAuthenticated, loginWithRedirect, logout, user } = useAuth0();

  if (isAuthenticated) {
    return (
      <div>
        <p>Olá, {user.name}</p>
        <button onClick={() => logout({ returnTo: window.location.origin })}>
          Sair
        </button>
      </div>
    );
  }

  return <button onClick={() => loginWithRedirect()}>Entrar</button>;
}
```

**Fazendo chamadas autenticadas para sua API:**
```jsx
import { useAuth0 } from '@auth0/auth0-react';

function Dashboard() {
  const { getAccessTokenSilently } = useAuth0();

  async function fetchData() {
    const token = await getAccessTokenSilently();

    const response = await fetch('https://api.meuapp.com/dados', {
      headers: {
        Authorization: `Bearer ${token}`,
      },
    });

    return response.json();
  }
}
```

### 3.3 Implementação no Backend (Node.js / Express)

**Instalação:**
```bash
npm install express-oauth2-jwt-bearer
```

**Configurando o middleware de validação:**
```javascript
// auth.middleware.js
const { auth } = require('express-oauth2-jwt-bearer');

const checkJwt = auth({
  audience: 'https://api.meuapp.com',
  issuerBaseURL: 'https://SEU_DOMINIO.auth0.com/',
});

module.exports = { checkJwt };
```

**Protegendo rotas:**
```javascript
// routes.js
const express = require('express');
const { checkJwt } = require('./auth.middleware');
const { checkRequiredPermissions } = require('./permissions.middleware');

const router = express.Router();

// Rota pública
router.get('/publico', (req, res) => {
  res.json({ mensagem: 'Acesso livre' });
});

// Rota autenticada (qualquer usuário logado)
router.get('/privado', checkJwt, (req, res) => {
  res.json({ mensagem: 'Você está autenticado!' });
});

// Rota com permissão específica
router.delete('/admin/usuario/:id',
  checkJwt,
  checkRequiredPermissions(['delete:users']),
  (req, res) => {
    res.json({ mensagem: 'Usuário removido' });
  }
);
```

**Verificando permissões (scopes):**
```javascript
// permissions.middleware.js
function checkRequiredPermissions(requiredPermissions) {
  return (req, res, next) => {
    const permissionCheck = requiredPermissions.every(permission =>
      req.auth.payload.permissions?.includes(permission)
    );

    if (!permissionCheck) {
      return res.status(403).json({ error: 'Permissão insuficiente' });
    }

    next();
  };
}
```

### 3.4 Implementação no Backend (Python / FastAPI)

**Instalação:**
```bash
pip install python-jose[cryptography] httpx
```

**Validação do token:**
```python
# auth.py
import httpx
from jose import jwt, JWTError
from fastapi import HTTPException, Security
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

AUTH0_DOMAIN = "SEU_DOMINIO.auth0.com"
API_AUDIENCE = "https://api.meuapp.com"
ALGORITHMS = ["RS256"]

security = HTTPBearer()

async def get_jwks():
    async with httpx.AsyncClient() as client:
        response = await client.get(f"https://{AUTH0_DOMAIN}/.well-known/jwks.json")
        return response.json()

async def verify_token(credentials: HTTPAuthorizationCredentials = Security(security)):
    token = credentials.credentials
    jwks = await get_jwks()

    try:
        unverified_header = jwt.get_unverified_header(token)
        rsa_key = next(
            {
                "kty": key["kty"],
                "kid": key["kid"],
                "n": key["n"],
                "e": key["e"],
            }
            for key in jwks["keys"]
            if key["kid"] == unverified_header["kid"]
        )

        payload = jwt.decode(
            token,
            rsa_key,
            algorithms=ALGORITHMS,
            audience=API_AUDIENCE,
            issuer=f"https://{AUTH0_DOMAIN}/",
        )
        return payload

    except JWTError:
        raise HTTPException(status_code=401, detail="Token inválido")
```

**Usando nas rotas:**
```python
# main.py
from fastapi import FastAPI, Depends
from auth import verify_token

app = FastAPI()

@app.get("/privado")
async def rota_privada(payload: dict = Depends(verify_token)):
    return {"usuario": payload.get("sub"), "mensagem": "Autenticado!"}
```

### 3.5 Variáveis de Ambiente

Nunca coloque credenciais do Auth0 diretamente no código:

```bash
# .env
AUTH0_DOMAIN=seu-tenant.auth0.com
AUTH0_CLIENT_ID=xxxxxxxxxxxxxxxxxxxxxxxx
AUTH0_CLIENT_SECRET=xxxxxxxxxxxxxxxxxxxxxxxx  # apenas no backend
AUTH0_AUDIENCE=https://api.meuapp.com
```

---

## 4. Recursos Avançados

### 4.1 Customizando o Token com Actions

O Auth0 permite injetar dados customizados no token via **Actions** (funções JavaScript serverless que executam em pontos do fluxo de autenticação):

```javascript
// Action: "Add user roles to token"
// Trigger: Login / Post Login

exports.onExecutePostLogin = async (event, api) => {
  const namespace = 'https://meuapp.com';

  // Adiciona roles do usuário ao Access Token
  if (event.authorization) {
    api.accessToken.setCustomClaim(`${namespace}/roles`, event.authorization.roles);
  }

  // Adiciona dados do app_metadata
  api.idToken.setCustomClaim(`${namespace}/plano`, event.user.app_metadata?.plano);
};
```

### 4.2 Controle de Acesso Baseado em Roles (RBAC)

```
Auth0 Dashboard:
  User Management → Roles → Create Role
    ├── "admin"       → permissões: read:users, write:users, delete:users
    ├── "editor"      → permissões: read:users, write:users
    └── "viewer"      → permissões: read:users

  Atribuir role ao usuário:
    User Management → Users → [usuário] → Roles → Assign Roles
```

Com RBAC habilitado na API (Settings → Enable RBAC), o Auth0 inclui as permissões do usuário diretamente no Access Token.

### 4.3 Single Sign-On (SSO) entre Aplicações

```
Usuário faz login na Aplicação A
          │
          ▼
   Auth0 cria sessão (cookie no domínio .auth0.com)
          │
          ▼
Usuário acessa Aplicação B (mesmo tenant)
          │
          ▼
Auth0 detecta sessão ativa → emite tokens sem pedir login novamente
```

Para SSO entre aplicações da mesma empresa, basta ter ambas registradas no mesmo tenant do Auth0.

---

## 5. Diagrama Completo de Arquitetura

```
                         ┌──────────────────────────────────┐
                         │            AUTH0 TENANT           │
                         │                                   │
                         │  ┌─────────────┐                 │
                         │  │  Universal   │                 │
                         │  │  Login Page  │                 │
                         │  └──────┬──────┘                 │
                         │         │                         │
                         │  ┌──────▼──────────────────────┐ │
                         │  │       Connections            │ │
                         │  │  ┌──────────┐  ┌──────────┐ │ │
                         │  │  │ Database │  │  Google  │ │ │
                         │  │  └──────────┘  └──────────┘ │ │
                         │  │  ┌──────────┐  ┌──────────┐ │ │
                         │  │  │  GitHub  │  │   SAML   │ │ │
                         │  │  └──────────┘  └──────────┘ │ │
                         │  └─────────────────────────────┘ │
                         │                                   │
                         │  ┌──────────────────────────────┐ │
                         │  │  Actions Pipeline            │ │
                         │  │  (enrich token, block user)  │ │
                         │  └──────────────────────────────┘ │
                         └──────────────────────────────────┘
                                         │
                    ┌────────────────────┼────────────────────┐
                    │                    │                     │
             ┌──────▼──────┐     ┌──────▼──────┐     ┌──────▼──────┐
             │  React SPA  │     │   Next.js   │     │  Mobile App │
             │  (Frontend) │     │  (Full-stack│     │  (iOS/Android│
             └──────┬──────┘     └──────┬──────┘     └──────┬──────┘
                    │                    │                     │
                    └────────────────────┼─────────────────────┘
                                         │  Access Token (JWT)
                                         ▼
                              ┌──────────────────────┐
                              │     Sua API REST      │
                              │  (valida JWT local)   │
                              └──────────────────────┘
```

---

## 6. Pontos de Atenção

| Situação | Recomendação |
|---|---|
| **Tokens expiram rápido** | Access Tokens com vida curta (1h) são intencional — use Refresh Tokens para renovar sem novo login |
| **Ambiente de desenvolvimento** | Crie um tenant separado para dev/staging — nunca teste com o tenant de produção |
| **CORS errors** | Verifique se o domínio do frontend está na lista `Allowed Web Origins` no dashboard |
| **Token muito grande** | Não coloque dados demais nas claims — prefira buscar dados na API usando o `sub` (user ID) |
| **Custo em escala** | O Auth0 cobra por MAU (Monthly Active Users) — avalie alternativas self-hosted (Keycloak, Supabase Auth) para volumes muito altos |
