# Docker Compose e Imagens de Produção

**Pré-requisito:** [03_volumes_e_redes.md](03_volumes_e_redes.md)

> A seção sobre Kubernetes retoma o Prefácio de **Fundamentals of Software Architecture** (Mark Richards & Neal Ford). A seção sobre sidecar e service mesh resume o Capítulo 8 de **Software Architecture: The Hard Parts** (Neal Ford, Mark Richards, Pramod Sadalage & Zhamak Dehghani). O restante segue a documentação oficial do Docker e do Compose.

---

## Por que este arquivo existe

O capítulo anterior terminou com um sistema funcionando, mas montado à mão: cinco comandos longos, digitados na ordem certa, com nomes que precisavam bater entre si. Esse jeito de trabalhar tem os mesmos defeitos da configuração manual de servidores descrita no capítulo [Virtualização, containers e IaC](../../01-infraestrutura/fundamentos/02_virtualizacao_containers_e_iac.md): não é versionado, não é reproduzível e depende da memória de quem digitou.

Este capítulo resolve isso em duas etapas:

1. **Docker Compose:** descrever o sistema inteiro (serviços, volumes, redes, dependências) em um único arquivo, e subi-lo com um comando.
2. **Imagens de produção:** o que muda quando a imagem sai do notebook e vai para um servidor (tamanho, segurança, configuração), e o que vem depois do Compose quando um servidor só não basta.

---

## 1. Do `docker run` ao `docker-compose.yml`

O **Docker Compose** lê um arquivo YAML que descreve todos os containers de uma aplicação e cria tudo o que for preciso para que eles funcionem juntos.

O melhor jeito de entendê-lo é traduzir o sistema do capítulo anterior. Estes eram os comandos:

```bash
docker network create minha-rede
docker volume create pgdata
docker run -d --name banco --network minha-rede \
  -e POSTGRES_PASSWORD=senha123 \
  -v pgdata:/var/lib/postgresql/data postgres:16
docker run -d --name api --network minha-rede -p 8000:8000 \
  -e DATABASE_URL="postgresql://postgres:senha123@banco:5432/postgres" \
  minha-api:1.0
```

E este é o mesmo sistema em Compose:

```yaml
# docker-compose.yml
services:
  banco:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: senha123
    volumes:
      - pgdata:/var/lib/postgresql/data

  api:
    image: minha-api:1.0
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql://postgres:senha123@banco:5432/postgres

volumes:
  pgdata:
```

Cada flag do `docker run` tem uma chave correspondente no arquivo:

| `docker run` | `docker-compose.yml` |
|---|---|
| `--name banco` | o nome do serviço (`banco:`) |
| imagem no fim do comando | `image:` |
| `-p 8000:8000` | `ports:` |
| `-e CHAVE=valor` | `environment:` |
| `-v pgdata:/caminho` | `volumes:` dentro do serviço + `volumes:` no fim do arquivo |
| `--network minha-rede` | **nada**: o Compose cria uma rede automaticamente |

A última linha é a mais interessante. O Compose cria uma rede própria para cada projeto e coloca todos os serviços nela. O nome de cada serviço vira um hostname, exatamente como na rede criada à mão no capítulo *Docker: volumes e redes*. Por isso o `DATABASE_URL` continua usando `banco`.

Para subir tudo:

```bash
docker compose up -d
```

```
[+] Running 4/4
 ✔ Network minha-api_default    Created
 ✔ Volume "minha-api_pgdata"    Created
 ✔ Container minha-api-banco-1  Started
 ✔ Container minha-api-api-1    Started
```

O Compose criou a rede, o volume e os dois containers, na ordem certa e com os nomes prefixados pelo nome da pasta do projeto (`minha-api`). Cinco comandos viraram um, e o arquivo vai para o Git junto com o código.

---

## 2. Um sistema completo: API, Postgres e Redis

Agora o caso realista: uma API que constrói a própria imagem, um Postgres e um Redis para cache.

```yaml
# docker-compose.yml
services:
  api:
    build: .                          # constrói a imagem a partir do Dockerfile desta pasta
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}
      REDIS_URL: redis://redis:6379
    depends_on:
      postgres:
        condition: service_healthy    # espera o banco estar PRONTO (ver seção 3)
      redis:
        condition: service_started
    restart: unless-stopped           # reinicia sozinho se o processo cair

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 5s
      timeout: 3s
      retries: 5
    restart: unless-stopped

  redis:
    image: redis:7.2-alpine
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
```

```text
# .env (na mesma pasta; NÃO vai para o Git)
POSTGRES_USER=app
POSTGRES_PASSWORD=troque-esta-senha
POSTGRES_DB=loja
```

O que há de novo em relação à versão simples:

- **`build: .`** no lugar de `image:`. O Compose roda o `docker build` por você. Depois de mudar o Dockerfile ou o código, use `docker compose up -d --build`.
- **`${POSTGRES_PASSWORD}`**: o Compose substitui essas variáveis pelos valores do arquivo `.env`. As senhas saem do `docker-compose.yml`, que pode ir para o Git, e ficam em um arquivo que não vai.
- **Postgres e Redis não têm `ports:`**. Como no capítulo *Docker: volumes e redes*, só a API recebe tráfego de fora. Os outros dois só são alcançáveis pela rede interna.
- **`command:`** substitui o `CMD` da imagem do Redis. Aqui ele liga a persistência em disco (`--appendonly yes`) e limita a memória a 256 MB, descartando as chaves menos usadas quando o limite é atingido (`allkeys-lru`). Esses parâmetros são explicados em [Arquitetura de Software: redis.md](https://github.com/iranhonorato/Arquitetura-de-Software/blob/main/system-design-pt1/redis.md).
- **`restart: unless-stopped`** é uma **política de reinício**: se o processo morrer (até por um *OOM kill*, como no capítulo *Docker: arquitetura e isolamento*), o Docker o sobe de novo, a não ser que você o tenha parado de propósito.

Antes de subir, dá para conferir se o arquivo está correto e ver como ele fica com as variáveis substituídas:

```bash
docker compose config
```

---

## 3. `depends_on` e healthchecks: esperar o serviço ficar pronto

A API precisa do banco para funcionar. Se ela subir primeiro, a conexão falha e a aplicação cai. O `depends_on` existe para controlar essa ordem, mas tem uma armadilha.

> **Confusão comum:** "com `depends_on: [postgres]`, a API só sobe quando o banco estiver pronto". ✅ **Mais preciso:** na forma simples, o `depends_on` espera apenas o container do banco **iniciar**, e iniciar não é estar pronto. O Postgres leva alguns segundos entre o processo começar e aceitar conexões (na primeira vez, ele ainda precisa criar os arquivos do banco). Nesse intervalo a API sobe, tenta conectar e falha. A solução são duas peças juntas:
> 1. um **`healthcheck`** no serviço do banco, que diz ao Docker *como* testar se ele está pronto (aqui, o `pg_isready`);
> 2. **`condition: service_healthy`** no `depends_on` da API, que diz para esperar esse teste passar.

O ciclo fica assim:

```
docker compose up
       │
       ▼
  postgres inicia ──▶ status: starting
       │
       │  a cada 5s: pg_isready ... → falhou (ainda inicializando)
       │  a cada 5s: pg_isready ... → falhou
       │  a cada 5s: pg_isready ... → ok!
       ▼
  postgres: healthy ──▶ só agora a api é iniciada
```

O estado aparece no `docker compose ps`:

```
NAME                  SERVICE    STATUS
minha-api-postgres-1  postgres   Up 12 seconds (healthy)
minha-api-redis-1     redis      Up 12 seconds
minha-api-api-1       api        Up 4 seconds
```

O `depends_on` resolve a **inicialização**. Se o banco cair com o sistema já rodando, a API precisa saber reconectar sozinha. Por isso aplicações robustas também implementam novas tentativas de conexão no próprio código.

---

## 4. Comandos essenciais do Compose

```bash
# Sobe todos os serviços em segundo plano
docker compose up -d

# Sobe reconstruindo as imagens (depois de mudar o código ou o Dockerfile)
docker compose up -d --build

# Estado dos serviços
docker compose ps

# Logs de todos os serviços, ou de um só, acompanhando em tempo real
docker compose logs -f
docker compose logs -f api

# Executa um comando dentro de um serviço que já está rodando
docker compose exec api python manage.py migrate
docker compose exec postgres psql -U app -d loja

# Para e remove os containers e a rede (os volumes ficam)
docker compose down

# Para e remove os containers, a rede E os volumes
docker compose down -v
```

> **Confusão comum:** "`docker compose down` apaga o banco". ✅ **Mais preciso:** o `down` remove containers e redes, mas **preserva os volumes nomeados**. Ao rodar `up` de novo, os containers novos se conectam aos mesmos volumes e os dados continuam lá. Quem apaga os volumes é o `down -v`. Ele é útil para recomeçar do zero em desenvolvimento, e perigoso se usado no servidor errado. Antes de rodar `-v`, confira com `docker compose ls` em qual projeto você está.

> **Confusão comum:** "tirei as senhas do `docker-compose.yml` e coloquei no `.env`, então elas estão protegidas". ✅ **Mais preciso:** o `.env` resolve um problema específico, que é **não versionar segredos no Git**. Mas as variáveis continuam visíveis para qualquer um com acesso ao Docker naquela máquina (`docker inspect` lista todas em texto claro), aparecem no `docker compose config` e são herdadas por qualquer processo dentro do container. Para desenvolvimento, `.env` no `.gitignore` é suficiente. Em produção, os segredos devem vir de um gerenciador próprio, como os *Secrets* do GitHub Actions (usados em [../../03-entrega-continua/cd/03_pipeline_de_cd_na_pratica.md](../../03-entrega-continua/cd/03_pipeline_de_cd_na_pratica.md)), o AWS Secrets Manager ou os `secrets:` do Compose, que entregam o valor como arquivo dentro do container em vez de variável de ambiente.

---

## 5. Multi-stage builds: imagens de produção enxutas

### O problema

Muitas dependências precisam ser **compiladas** durante a instalação, e para isso exigem compiladores (`gcc`) e cabeçalhos de bibliotecas. Projetos em TypeScript precisam do compilador TypeScript. Nada disso é necessário para **rodar** a aplicação, mas, em um Dockerfile comum, tudo vai para a imagem final. O resultado é uma imagem grande (lenta para transferir e para iniciar em um servidor novo) e com mais software instalado, ou seja, mais pontos para explorar em um ataque.

O capítulo *Docker: imagens e Dockerfile* mostrou que um `RUN rm` posterior não resolve isso, porque as camadas antigas continuam lá.

### A solução: um estágio para construir, outro para rodar

Um **multi-stage build** usa mais de um `FROM` no mesmo Dockerfile. Cada `FROM` começa um estágio novo, a partir do zero, e só a **última** imagem é entregue. Os estágios anteriores servem como oficina: você constrói neles e copia para o estágio final **apenas o resultado**.

```
┌──────────────── Estágio "builder" ─────────────────┐
│ python:3.12 completo (gcc, headers, ferramentas)   │
│ pip install → gera /opt/venv com as dependências   │──┐
└────────────────────────────────────────────────────┘  │ COPY --from=builder
                                                        │ (só o /opt/venv)
┌──────────────── Estágio "runtime" ─────────────────┐  │
│ python:3.12-slim (sem compiladores)                │◀─┘
│ /opt/venv + seu código                             │  ← só esta vai para produção
└────────────────────────────────────────────────────┘
```

### Exemplo em Python

```dockerfile
# ── Estágio 1: builder ─────────────────────────────────────
# Imagem completa: tem gcc e os cabeçalhos para compilar dependências
FROM python:3.12 AS builder

WORKDIR /app

# Cria um ambiente virtual isolado, fácil de copiar inteiro
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# ── Estágio 2: runtime ─────────────────────────────────────
# Imagem mínima: só o necessário para rodar
FROM python:3.12-slim AS runtime

WORKDIR /app

# Traz do builder apenas as dependências já instaladas
COPY --from=builder /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Copia apenas o código da aplicação
COPY app/ ./app/

# Cria e usa um usuário sem privilégios (ver seção 6)
RUN useradd --create-home appuser
USER appuser

EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```bash
docker images minha-api
```

```
REPOSITORY   TAG            SIZE
minha-api    single-stage   1.1GB    ← tudo construído e executado sobre python:3.12
minha-api    multi-stage    190MB    ← python:3.12-slim + dependências + código
```

Os números exatos dependem das dependências do projeto, mas a ordem de grandeza da diferença é essa.

### Exemplo em Node.js (TypeScript)

O mesmo princípio, com o compilador TypeScript ficando no estágio de build:

```dockerfile
# Estágio 1: compila o TypeScript para JavaScript
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
# Compila de src/ (TypeScript) para dist/ (JavaScript)
RUN npm run build

# Estágio 2: só o JavaScript compilado e as dependências de produção
FROM node:20-alpine AS runtime
WORKDIR /app
COPY package*.json ./
# Instala sem as dependências de desenvolvimento (TypeScript, linters, testes)
RUN npm ci --omit=dev
COPY --from=builder /app/dist ./dist
# A imagem oficial do Node já traz um usuário sem privilégios chamado "node"
USER node
CMD ["node", "dist/main.js"]
```

---

## 6. Boas práticas para imagens de produção

**Use bases pequenas.** `python:3.12-slim` tem cerca de 130 MB; `python:3.12` passa de 1 GB. Imagens menores são transferidas mais rápido, sobem mais rápido em servidores novos e têm menos pacotes que podem conter vulnerabilidades. As variantes `-alpine` são ainda menores, mas usam uma biblioteca C diferente (musl), e algumas dependências Python com partes em C precisam ser compiladas do zero nela. Para Python, `-slim` costuma ser o melhor equilíbrio.

**Não rode como root.** Por padrão, o processo dentro do container roda como root. Se a aplicação for comprometida, o atacante tem root dentro do container, e qualquer falha de isolamento do kernel (capítulo *Virtualização, containers e IaC*) vira root no host. Crie um usuário sem privilégios:

```dockerfile
RUN useradd --create-home appuser
USER appuser
```

**Uma responsabilidade por container.** Cada container deve ter um único processo principal. Não coloque a API e o banco no mesmo container: eles precisam ser atualizados, escalados e reiniciados de forma independente, e o Docker só monitora o PID 1 (capítulo *Docker: imagens e Dockerfile*).

**Configuração por variáveis de ambiente.** A mesma imagem deve rodar em desenvolvimento, homologação e produção, mudando só a configuração. Nunca escreva senhas, URLs de banco ou chaves no Dockerfile:

```bash
docker run -e DATABASE_URL="postgresql://..." minha-api:1.4.2
```

**Use `.dockerignore` e tags explícitas.** Ambos vistos no capítulo *Docker: imagens e Dockerfile*: o primeiro mantém `.git`, `.env` e `node_modules` fora da imagem; o segundo garante que você saiba exatamente qual versão está rodando.

---

## 7. Quando o Compose não basta: Kubernetes

O Compose orquestra containers em **uma máquina**. Isso cobre desenvolvimento local e muitas aplicações pequenas em produção. Quando o sistema cresce para dezenas de serviços e centenas de containers espalhados por várias máquinas, surgem perguntas que o Compose não responde: em qual máquina cada container deve rodar? O que acontece se uma máquina inteira cair? Como atualizar 50 réplicas sem tirar o sistema do ar?

O **Kubernetes** (K8s) é o padrão da indústria para responder a essas perguntas. Richards e Ford usam justamente sua ascensão como exemplo de como o ecossistema de software muda depressa: ele não existia uma década antes da publicação do livro e hoje tem conferências inteiras dedicadas a ele.

```
┌────────────────────────────────────────────────────────┐
│                  CLUSTER KUBERNETES                    │
│                                                        │
│  Control Plane (o "cérebro")                           │
│  ┌──────────────────────────────────────────────────┐  │
│  │  API Server  │  Scheduler  │  etcd (estado)      │  │
│  └──────────────────────────────────────────────────┘  │
│                                                        │
│  Worker Nodes (as máquinas que rodam os containers)    │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │   Node 1    │  │   Node 2    │  │   Node 3    │     │
│  │  [Pod][Pod] │  │  [Pod][Pod] │  │  [Pod][Pod] │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└────────────────────────────────────────────────────────┘
```

Você declara o estado desejado ("quero 5 réplicas da API, cada uma com 512 MB"), e o Kubernetes trabalha continuamente para manter o cluster nesse estado. É a mesma ideia de estado desejado e idempotência do Terraform e do Ansible (capítulo *Virtualização, containers e IaC*), aplicada a containers. Na prática, ele oferece:

| Recurso | O que faz |
|---|---|
| **Self-healing** | Reinicia containers que falham e os move para outra máquina se um node cair |
| **Auto-scaling** | Aumenta ou diminui o número de réplicas conforme a carga |
| **Rolling updates** | Troca as réplicas pela versão nova aos poucos, sem downtime ([../../03-entrega-continua/cd/02_estrategias_de_deploy.md](../../03-entrega-continua/cd/02_estrategias_de_deploy.md)) |
| **Service discovery** | Serviços se encontram pelo nome, como no Compose, mas entre várias máquinas |
| **Load balancing** | Distribui o tráfego entre as réplicas |

Os provedores de nuvem oferecem Kubernetes gerenciado, em que eles operam o control plane: **AWS EKS**, **Google GKE** e **Azure AKS**.

| | Docker Compose | Kubernetes |
|---|---|---|
| **Escopo** | Uma máquina | Um cluster de várias máquinas |
| **Se a máquina cair** | Tudo cai | Os containers são realocados em outros nodes |
| **Curva de aprendizado** | Baixa | Alta |
| **Uso típico** | Desenvolvimento local, sistemas pequenos | Produção em escala, muitos serviços |

Não comece pelo Kubernetes. Ele resolve problemas de escala e disponibilidade que a maioria dos projetos ainda não tem, e cobra um custo operacional permanente. O caminho natural é o desta trilha: container → Compose → um servidor na nuvem → Kubernetes, quando a necessidade aparecer.

### Uma ideia que nasce com containers: o sidecar

Quando um sistema tem muitos serviços, cada um precisa de recursos operacionais iguais: enviar logs, expor métricas, aplicar timeouts, criptografar o tráfego entre serviços. Implementar isso em cada serviço, possivelmente em linguagens diferentes, gera duplicação e inconsistência.

*Software Architecture: The Hard Parts* descreve a solução que se consolidou com containers: o **padrão sidecar**. Os recursos operacionais saem do código do serviço e vão para um **segundo container** que roda ao lado dele (como o carrinho lateral de uma moto), compartilhando a mesma rede. O serviço cuida só do domínio; o sidecar cuida do resto, de forma idêntica em todos os serviços. Quando os sidecars de todos os serviços se conectam entre si e passam a ser controlados por um painel central, o conjunto forma uma **service mesh** (Istio e Linkerd são exemplos), uma camada operacional comum a toda a arquitetura.

---

## Próximo passo

Os capítulos de Docker cobriram o caminho completo do empacotamento: como o Docker funciona por dentro, como construir imagens, como dar persistência e rede a elas e como descrever um sistema inteiro em um arquivo. O limite que ficou claro na seção 7 é que tudo isso acontece em **uma máquina**.

O próximo passo da trilha é o [../kubernetes/01_fundamentos_e_arquitetura.md](../kubernetes/01_fundamentos_e_arquitetura.md), que abre a seção 7 deste capítulo por completo: o que é um cluster, seus componentes, Pods e o loop de reconciliação. Depois do Kubernetes, a trilha passa pela AWS ([../../01-infraestrutura/aws/](../../01-infraestrutura/SUMARIO.md): EC2, VPC e IAM) e termina automatizando tudo com CI/CD ([../../03-entrega-continua/](../../03-entrega-continua/SUMARIO.md)).

Se o seu objetivo imediato é colocar este Compose em um servidor e automatizar o deploy, sem Kubernetes, o caminho mais curto é: [../../01-infraestrutura/aws/01_ec2_servidores_na_nuvem.md](../../01-infraestrutura/aws/01_ec2_servidores_na_nuvem.md) → [../../03-entrega-continua/ci/01_fundamentos_de_integracao_continua.md](../../03-entrega-continua/ci/01_fundamentos_de_integracao_continua.md) → [../../03-entrega-continua/cd/03_pipeline_de_cd_na_pratica.md](../../03-entrega-continua/cd/03_pipeline_de_cd_na_pratica.md).

---

## Apêndice: referência de comandos

### Imagens

```bash
docker images                              # lista as imagens locais
docker pull nginx:1.27                     # baixa uma imagem do registry
docker build -t nome:tag .                 # constrói a partir do Dockerfile da pasta atual
docker history nome:tag                    # mostra as camadas da imagem
docker tag nome:tag usuario/nome:tag       # adiciona um novo nome à imagem
docker push usuario/nome:tag               # envia para o Docker Hub
docker rmi nome:tag                        # remove a imagem
```

### Containers

```bash
docker run [opções] imagem                 # cria e inicia um container
docker ps                                  # lista os containers em execução
docker ps -a                               # lista todos, inclusive os parados
docker stop nome                           # para (envia SIGTERM, espera, depois SIGKILL)
docker start nome                          # inicia um container parado
docker restart nome                        # para e inicia de novo
docker rm nome                             # remove (precisa estar parado)
docker rm -f nome                          # para e remove de uma vez
```

Opções mais usadas do `docker run`:

| Opção | Efeito |
|---|---|
| `-d` | Roda em segundo plano |
| `-p HOST:CONTAINER` | Publica uma porta |
| `-v VOLUME:CAMINHO` / `-v PASTA:CAMINHO` | Monta um volume / bind mount |
| `-e CHAVE=valor` | Define uma variável de ambiente |
| `--name nome` | Dá um nome ao container |
| `--network rede` | Conecta a uma rede |
| `--rm` | Remove o container quando ele terminar |
| `--memory=512m --cpus=0.5` | Limita recursos (cgroups) |
| `-it` | Terminal interativo |

### Inspeção e debug

```bash
docker logs -f nome                        # logs em tempo real
docker exec -it nome bash                  # abre um terminal dentro do container
docker exec -it nome sh                    # idem, para imagens mínimas sem bash
docker inspect nome                        # todos os detalhes do container (JSON)
docker inspect --format '{{.State.OOMKilled}}' nome   # o container foi morto por falta de memória?
docker stats                               # uso de CPU e memória em tempo real
```

### Volumes e redes

```bash
docker volume create nome                  # cria um volume
docker volume ls                           # lista os volumes
docker volume rm nome                      # remove um volume (apaga os dados)
docker network create nome                 # cria uma rede
docker network ls                          # lista as redes
docker network inspect nome                # mostra os containers conectados
```

### Compose

```bash
docker compose up -d [--build]             # sobe (e reconstrói) os serviços
docker compose ps                          # estado dos serviços
docker compose logs -f [serviço]           # logs
docker compose exec serviço comando        # executa um comando em um serviço
docker compose config                      # valida e mostra o arquivo final
docker compose down [-v]                   # remove containers e redes (e volumes, com -v)
```

### Limpeza

```bash
docker container prune                     # remove os containers parados
docker image prune                         # remove as imagens sem nome e sem uso
docker system prune -a                     # remove tudo o que não está em uso (containers, imagens, redes)
docker system prune -a --volumes           # idem, incluindo volumes: apaga dados!
docker system df                           # mostra quanto espaço o Docker está usando
```
