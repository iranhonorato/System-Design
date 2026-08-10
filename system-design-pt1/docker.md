# Docker

## 1. O que é Docker?

Docker é uma plataforma de **conteinerização** que permite empacotar uma aplicação junto com todas as suas dependências (bibliotecas, configurações, runtime) em uma unidade portátil e isolada chamada **container**.

O princípio central é: **"Build once, run anywhere"** — você constrói a imagem uma vez e ela roda de forma idêntica no notebook do desenvolvedor, no servidor de staging e na instância EC2 de produção.

### Por que o Docker importa?

Antes do Docker, um dos maiores pesadelos do desenvolvimento era o **problema de ambiente**:

- Dev: "Funciona na minha máquina"
- QA: "Aqui dá erro"
- Ops: "Em produção quebrou"

Cada ambiente tinha versões diferentes de Python, Node, Java, bibliotecas de sistema e configurações. O Docker elimina esse problema ao **fazer o ambiente parte do código** — o mesmo Dockerfile que você usa no desenvolvimento é o que vai para produção.

---

## 2. Conceitos Fundamentais

### 2.1 Imagem (Image)

Uma imagem é um **arquivo estático e imutável** que contém tudo que sua aplicação precisa para rodar: o sistema operacional simplificado, o runtime (Python, Node, Java), as bibliotecas e o seu código.

Pense na imagem como uma **fotografia de um ambiente pronto**. Uma vez criada, ela não muda. Isso garante que se você enviar essa imagem para qualquer lugar, o ambiente será exatamente o mesmo.

**Imagens são compostas por camadas (layers):**

```
┌─────────────────────────────────┐
│  Camada 4: seu código           │  ← sua camada
├─────────────────────────────────┤
│  Camada 3: suas dependências    │  ← pip install / npm install
├─────────────────────────────────┤
│  Camada 2: Python 3.11          │  ← runtime
├─────────────────────────────────┤
│  Camada 1: Ubuntu 22.04         │  ← base
└─────────────────────────────────┘
```

**Por que camadas importam?** O Docker usa um sistema de cache inteligente. Se você mudar apenas o seu código (camada 4), o Docker reaproveita as camadas 1, 2 e 3 do cache — o rebuild fica muito mais rápido. Só a camada alterada e as acima dela são reconstruídas.

### 2.2 Container

O container é a **instância em execução de uma imagem**. Quando você manda o Docker rodar uma imagem, ele cria um container — um processo isolado com seu próprio sistema de arquivos, rede e recursos.

```
Imagem  →  docker run  →  Container
(estático)               (em execução)

Analogia:
Receita de bolo → Processo de assar → Bolo pronto na mesa
```

Múltiplos containers podem ser criados a partir da mesma imagem, rodando simultaneamente e de forma completamente independente.

### 2.3 Dockerfile

O Dockerfile é o **arquivo de instruções** que descreve como construir uma imagem. É um arquivo de texto simples, sem extensão, que fica na raiz do projeto.

```dockerfile
# 1. Imagem base (ponto de partida)
FROM python:3.11-slim

# 2. Diretório de trabalho dentro do container
WORKDIR /app

# 3. Copia arquivos de dependências primeiro (otimiza cache)
COPY requirements.txt .

# 4. Instala dependências
RUN pip install --no-cache-dir -r requirements.txt

# 5. Copia o restante do código
COPY . .

# 6. Porta que a aplicação vai usar (documentação)
EXPOSE 5000

# 7. Comando de inicialização
CMD ["python", "app.py"]
```

> **Por que copiar o requirements.txt antes do código?** Se você copiar tudo de uma vez e depois mudar uma linha do código, o Docker invalida o cache do `pip install` e reinstala todas as dependências. Separando em dois passos, uma mudança no código apenas invalida o `COPY . .` — o `pip install` permanece cacheado.

### 2.4 Docker Hub

O Docker Hub é o **registry público** de imagens — uma espécie de "App Store" para imagens Docker. Lá você encontra imagens oficiais e verificadas de praticamente tudo:

- `python:3.11` — Python oficial
- `node:20` — Node.js oficial
- `postgres:16` — PostgreSQL oficial
- `nginx:latest` — Nginx oficial

**Por que usar imagens oficiais?** Elas são auditadas, mantidas e atualizadas regularmente com patches de segurança. Evite construir suas imagens do zero quando existe uma base confiável disponível.

Você também pode criar uma conta gratuita e publicar suas próprias imagens no Docker Hub — o que é essencial para pipelines de CI/CD.

---

## 3. Como o Docker Funciona por Dentro

### 3.1 Arquitetura: Cliente e Daemon

O Docker usa uma arquitetura **cliente-servidor**:

```
┌────────────────┐        API REST        ┌─────────────────────┐
│  Docker Client │ ─────────────────────→ │    Docker Daemon     │
│   (terminal)   │ ←───────────────────── │    (dockerd)         │
└────────────────┘                        └──────────┬──────────┘
                                                     │
                                          ┌──────────▼──────────┐
                                          │   Docker Hub /       │
                                          │   Registry           │
                                          └──────────────────────┘
```

- **Docker Client:** a interface de linha de comando (`docker`) que você usa. Quando você digita `docker run`, o cliente envia uma requisição para o daemon.
- **Docker Daemon (dockerd):** o processo em background que faz o trabalho pesado: cria containers, gerencia imagens, configura redes. Roda como serviço no SO.
- **Registry:** repositório de imagens. O daemon busca imagens aqui quando você não as tem localmente.

### 3.2 Isolamento com Tecnologias do Kernel Linux

O Docker não inventa um mecanismo de isolamento novo — ele usa recursos que já existem no kernel Linux:

**Namespaces** — criam "visões isoladas" do sistema para cada container:

| Namespace | O que isola |
|---|---|
| `pid` | Processos (o container acha que seu processo é o PID 1) |
| `net` | Interface de rede, tabelas de roteamento, portas |
| `mnt` | Sistema de arquivos (cada container enxerga apenas o seu) |
| `uts` | Hostname (o container tem seu próprio hostname) |
| `user` | Usuários e grupos |
| `ipc` | Memória compartilhada entre processos |

**Control Groups (cgroups)** — limitam o consumo de recursos:

```bash
# Exemplo: container limitado a 512MB de RAM e 0.5 CPU
docker run --memory="512m" --cpus="0.5" minha-imagem
```

Sem cgroups, um container mal-comportado poderia consumir toda a memória do servidor e derrubar os outros containers.

### 3.3 Sistema de Arquivos em Camadas (Union File System)

As imagens Docker usam um sistema de arquivos em camadas sobrepostas. Cada instrução do Dockerfile cria uma nova camada — e as camadas são **somente leitura**. Quando um container roda, o Docker adiciona uma **camada de escrita** no topo (Container Layer), onde todas as modificações feitas em runtime ficam armazenadas.

```
┌─────────────────────┐  ← Container Layer (escrita - temporária)
├─────────────────────┤
│  Camada 3 (código)  │
├─────────────────────┤  ← Image Layers (somente leitura)
│  Camada 2 (Python)  │
├─────────────────────┤
│  Camada 1 (Ubuntu)  │
└─────────────────────┘
```

**Vantagem do compartilhamento:** se você tem 10 containers baseados na mesma imagem Python, as camadas da imagem existem **uma única vez no disco** e são compartilhadas entre todos os containers. Cada container tem apenas sua própria camada de escrita.

> **Atenção:** a camada de escrita do container é **temporária**. Quando o container é removido, tudo que foi escrito nela é perdido. Para persistir dados, use **Volumes**.

---

## 4. Instalação

### Docker Desktop (Windows e Mac)
1. Acesse [docker.com](https://www.docker.com) e baixe o Docker Desktop
2. No Windows, aceite a instalação do **WSL 2** (Windows Subsystem for Linux) — é o que permite o Docker rodar com performance nativa
3. Após instalar, aguarde o ícone da baleia verde no menu do sistema

### Linux (Ubuntu)
```bash
# Método recomendado: script oficial
curl -fsSL https://get.docker.com | sh

# Adicionar seu usuário ao grupo docker (evita usar sudo a cada comando)
sudo usermod -aG docker $USER

# Relogar para aplicar o grupo
newgrp docker
```

**Verificando a instalação:**
```bash
docker --version
docker run hello-world
```

---

## 5. Primeiros Passos

### 5.1 Hello World

```bash
docker run hello-world
```

O que acontece nos bastidores:
1. Docker Client envia o pedido ao Daemon
2. Daemon verifica se a imagem `hello-world` existe localmente → não existe
3. Daemon faz pull do Docker Hub
4. Daemon cria um container a partir da imagem
5. Container executa, imprime a mensagem e para

### 5.2 Rodando um Servidor Web

```bash
docker run -d -p 8080:80 --name meu-nginx nginx
```

Detalhando os parâmetros:

| Parâmetro | Significado |
|---|---|
| `-d` | Detached: roda em background (você continua usando o terminal) |
| `-p 8080:80` | Mapeia porta 8080 do host → porta 80 do container |
| `--name meu-nginx` | Nome amigável para o container |
| `nginx` | Nome da imagem |

Acesse `http://localhost:8080` e verá a página do Nginx.

**Entendendo o mapeamento de portas:**
```
Seu navegador → localhost:8080 → Docker → container:80 → Nginx
```
O container está "fechado" — a flag `-p` cria uma "janela" que redireciona tráfego da porta do host para dentro do container.

### 5.3 Construindo sua Própria Imagem

Com um projeto Python simples:

```
meu-projeto/
├── app.py
├── requirements.txt
└── Dockerfile
```

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["python", "app.py"]
```

```bash
# Construir a imagem
docker build -t meu-app:1.0 .

# Rodar o container
docker run -d -p 5000:5000 --name meu-app meu-app:1.0
```

---

## 6. Volumes: Persistindo Dados

Como a camada de escrita do container é temporária, dados importantes precisam ser armazenados em **Volumes** — espaços de armazenamento gerenciados pelo Docker que persistem independentemente do ciclo de vida do container.

```bash
# Criar um volume
docker volume create meus-dados

# Usar o volume no container
docker run -d \
  -v meus-dados:/app/data \
  --name minha-api \
  minha-imagem
```

O diretório `/app/data` dentro do container agora está mapeado para o volume `meus-dados` no host. Mesmo que o container seja removido e recriado, os dados persistem.

**Bind mounts** — mapear uma pasta local para dentro do container (útil para desenvolvimento):

```bash
docker run -d \
  -v $(pwd)/codigo:/app \  # pasta local → pasta no container
  -p 5000:5000 \
  minha-imagem
```

Agora mudanças no código local refletem imediatamente dentro do container — sem precisar rebuild.

---

## 7. Redes Docker

Por padrão, containers em redes diferentes não se enxergam. Para que containers se comuniquem (ex: API + banco de dados), eles precisam estar na mesma rede:

```bash
# Criar uma rede
docker network create minha-rede

# Rodar banco de dados na rede
docker run -d \
  --name banco \
  --network minha-rede \
  -e POSTGRES_PASSWORD=senha123 \
  postgres:16

# Rodar a API na mesma rede
docker run -d \
  --name api \
  --network minha-rede \
  -p 5000:5000 \
  minha-api-imagem
```

Dentro da rede Docker, containers se comunicam pelo **nome do container** como hostname:

```python
# Na API, conectar ao banco usando o nome do container
DATABASE_URL = "postgresql://user:senha@banco:5432/mydb"
#                                          ↑
#                                  nome do container
```

---

## 8. Referência de Comandos

### Imagens

```bash
# Listar imagens locais
docker images

# Baixar imagem do registry
docker pull nginx:latest

# Construir imagem a partir do Dockerfile na pasta atual
docker build -t nome-da-imagem:tag .

# Remover imagem
docker rmi nome-da-imagem

# Enviar imagem para o Docker Hub
docker push seu-usuario/nome-da-imagem:tag
```

### Containers

```bash
# Criar e iniciar container
docker run [opções] nome-da-imagem

# Listar containers em execução
docker ps

# Listar todos os containers (incluindo parados)
docker ps -a

# Parar container graciosamente
docker stop nome-do-container

# Iniciar container parado
docker start nome-do-container

# Remover container (deve estar parado)
docker rm nome-do-container

# Parar e remover em um comando
docker rm -f nome-do-container
```

### Inspeção e Debug

```bash
# Ver logs do container (em tempo real com -f)
docker logs -f nome-do-container

# Entrar no container com terminal interativo
docker exec -it nome-do-container bash
# ou para imagens minimalistas sem bash:
docker exec -it nome-do-container sh

# Inspecionar detalhes do container (JSON completo)
docker inspect nome-do-container

# Ver uso de recursos em tempo real
docker stats
```

### Limpeza

```bash
# Remover todos os containers parados
docker container prune

# Remover imagens sem uso
docker image prune

# Limpeza completa (containers, imagens, volumes, redes sem uso)
docker system prune -a
```

### Volumes e Redes

```bash
# Criar volume
docker volume create nome-do-volume

# Listar volumes
docker volume ls

# Criar rede
docker network create nome-da-rede

# Listar redes
docker network ls

# Inspecionar rede (ver containers conectados)
docker network inspect nome-da-rede
```

---

## 9. Docker Compose

Gerenciar containers individualmente com `docker run` fica inviável quando a aplicação tem vários serviços (API + banco + Redis + worker). O **Docker Compose** resolve isso com um único arquivo declarativo que descreve todos os serviços.

```yaml
# docker-compose.yml
services:
  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://user:senha@postgres:5432/mydb
      - REDIS_URL=redis://redis:6379
    depends_on:
      postgres:
        condition: service_healthy   # espera o banco estar pronto (não apenas iniciado)
      redis:
        condition: service_started

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: senha
      POSTGRES_DB: mydb
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d mydb"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7.2-alpine
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

**Comandos essenciais do Compose:**

```bash
# Sobe todos os serviços em background
docker compose up -d

# Sobe e reconstrói imagens (após mudanças no Dockerfile)
docker compose up -d --build

# Ver logs de todos os serviços (ou de um específico)
docker compose logs -f
docker compose logs -f api

# Para e remove containers (volumes persistem)
docker compose down

# Para e remove containers E volumes (útil para reset completo)
docker compose down -v

# Roda um comando em um serviço sem criar novo container
docker compose exec api python manage.py migrate

# Ver status dos serviços
docker compose ps
```

> **depends_on com condition:** o `depends_on` simples apenas espera o container *iniciar*, não estar *pronto*. Use `condition: service_healthy` com um `healthcheck` para garantir que o banco aceita conexões antes de a API tentar conectar.

---

## 10. Multi-Stage Builds

Imagens de produção devem ser o menor possível — sem ferramentas de build, compiladores ou dependências de desenvolvimento. **Multi-stage builds** resolvem isso: você usa um container para compilar/construir e outro apenas para rodar.

**Exemplo com aplicação Python:**

```dockerfile
# ── Stage 1: Builder ──────────────────────────────────────
FROM python:3.12-slim AS builder

WORKDIR /app

# Instala dependências em um diretório isolado
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# ── Stage 2: Runtime ──────────────────────────────────────
FROM python:3.12-slim AS runtime

WORKDIR /app

# Copia apenas as dependências já instaladas do stage anterior
COPY --from=builder /install /usr/local

# Copia apenas o código da aplicação
COPY app/ ./app/

# Usuário não-root para segurança
RUN useradd -m appuser
USER appuser

EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```
Tamanho comparativo:
  Sem multi-stage:  ~1.2 GB  (inclui pip, compiladores, cache)
  Com multi-stage:  ~180 MB  (apenas runtime + código + deps)
```

**Exemplo com Node.js (TypeScript):**

```dockerfile
# Stage 1: Compila o TypeScript
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build   # compila TS → JS em /dist

# Stage 2: Imagem de produção (sem TypeScript, sem node_modules de dev)
FROM node:20-alpine AS runtime
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev   # só dependências de produção
COPY --from=builder /app/dist ./dist
USER node
CMD ["node", "dist/main.js"]
```

---

## 11. Boas Práticas

**Use imagens base slim/alpine:** `python:3.12-slim` é muito menor que `python:3.12`. Imagens menores = transferência mais rápida, menos superfície de ataque.

**Não rode como root:** adicione um usuário não-privilegiado no Dockerfile. Um container comprometido rodando como root pode escalar privilégios para o host:
```dockerfile
RUN useradd -m appuser
USER appuser
```

**Use .dockerignore:** similar ao .gitignore, evita copiar arquivos desnecessários para a imagem:
```
.git
__pycache__
*.pyc
.env
node_modules
```

**Uma responsabilidade por container:** containers devem ter um único processo principal. Não coloque o banco de dados e a API no mesmo container.

**Variáveis de ambiente para configuração:** nunca coloque senhas hardcoded no Dockerfile. Use variáveis de ambiente:
```bash
docker run -e DATABASE_URL="postgresql://..." minha-imagem
```
