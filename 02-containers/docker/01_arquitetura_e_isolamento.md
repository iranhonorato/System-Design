# Docker por Dentro

**Pré-requisito:** [../../01-infraestrutura/fundamentos/02_virtualizacao_containers_e_iac.md](../../01-infraestrutura/fundamentos/02_virtualizacao_containers_e_iac.md)

> Os livros de arquitetura usados neste repositório tratam o container como um dado, uma unidade de implantação, sem detalhar seu funcionamento. Este capítulo segue a documentação oficial do Docker e as páginas de manual do kernel Linux (`namespaces(7)`, `cgroups(7)`).

---

## Por que este arquivo existe

O capítulo anterior terminou com uma afirmação forte: um container não é uma máquina, é um processo comum do Linux com restrições. Este capítulo mostra isso na prática. A pergunta central é: *o que acontece, peça por peça, entre você digitar `docker run` e a aplicação estar rodando?*

Entender esse caminho tem utilidade direta. Quando um container não sobe, não alcança a rede ou é morto sem explicação, a causa quase sempre está em uma das peças descritas aqui.

---

## 1. O que é o Docker

Docker é uma plataforma de **conteinerização**: ela empacota uma aplicação junto com tudo de que ela precisa (runtime, bibliotecas, configurações) em uma unidade portátil e isolada, o **container**.

O princípio que guia a ferramenta é **"build once, run anywhere"**: você constrói o pacote uma vez, e ele roda de forma idêntica no notebook do desenvolvedor, no servidor de testes e na instância EC2 de produção.

Isso ataca diretamente o problema de ambiente que vinha desde a era do bare metal:

- Dev: "Funciona na minha máquina."
- QA: "Aqui dá erro."
- Ops: "Em produção quebrou."

Cada ambiente tinha uma versão diferente de Python, Node ou Java, bibliotecas de sistema diferentes e configurações diferentes. O Docker resolve isso **tornando o ambiente parte do código**: o mesmo arquivo que descreve o ambiente de desenvolvimento é o que vai para produção.

---

## 2. Instalação

### Windows e Mac: Docker Desktop

1. Baixe o Docker Desktop em [docker.com](https://www.docker.com).
2. No Windows, aceite a instalação do **WSL 2** (Windows Subsystem for Linux). Ele fornece o kernel Linux de que os containers precisam (veja a primeira "confusão comum" do [capítulo anterior](../../01-infraestrutura/fundamentos/02_virtualizacao_containers_e_iac.md)).
3. Aguarde o ícone da baleia aparecer na barra do sistema, indicando que o Docker está pronto.

### Linux (Ubuntu)

```bash
# Script oficial de instalação do Docker Engine
curl -fsSL https://get.docker.com | sh

# Adiciona seu usuário ao grupo docker, para não precisar de sudo em todo comando
sudo usermod -aG docker $USER

# Aplica o novo grupo na sessão atual (ou faça logout e login)
newgrp docker
```

### Verificando

```bash
docker --version
# Docker version 27.x.x, build ...

docker run hello-world
# Hello from Docker!
# This message shows that your installation appears to be working correctly.
```

> **Confusão comum:** "adicionar meu usuário ao grupo `docker` é só um atalho para não digitar `sudo`". ✅ **Mais preciso:** pertencer ao grupo `docker` **equivale a ter acesso de root na máquina**. Quem pode falar com o Docker pode criar um container que monta o disco inteiro do host (`-v /:/host`) e altera qualquer arquivo dele como root. Em um notebook pessoal isso é aceitável. Em um servidor compartilhado, adicionar alguém ao grupo `docker` é o mesmo que entregar a senha de root. Para esses casos existe o **modo rootless** do Docker, em que o próprio daemon roda sem privilégios.

---

## 3. A arquitetura: cliente, daemon e registry

Quando você digita um comando `docker`, quem executa o trabalho **não é** o programa que você chamou. O Docker usa uma arquitetura **cliente-servidor**:

```
┌────────────────┐        API REST        ┌─────────────────────┐
│  Docker Client │ ─────────────────────▶ │    Docker Daemon    │
│  (comando      │ ◀───────────────────── │    (dockerd)        │
│   `docker`)    │                        └──────────┬──────────┘
└────────────────┘                                   │ baixa imagens
                                          ┌──────────▼──────────┐
                                          │  Registry           │
                                          │  (ex.: Docker Hub)  │
                                          └─────────────────────┘
```

- **Docker Client:** a linha de comando `docker`. Ela só traduz o que você digitou em uma requisição para a API do daemon.
- **Docker Daemon (`dockerd`):** o processo que roda em segundo plano, como serviço do sistema, e faz o trabalho pesado: baixa imagens, cria containers, configura redes e volumes.
- **Registry:** o repositório de onde o daemon baixa as imagens que não tem localmente. O registry público padrão é o **Docker Hub**.

Uma consequência prática dessa separação: se o daemon não estiver rodando, todo comando falha, mesmo que o cliente esteja instalado. É o erro clássico `Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?`.

### O que acontece em um `docker run hello-world`

```bash
docker run hello-world
```

```
Unable to find image 'hello-world:latest' locally     ← ② o daemon procurou e não achou
latest: Pulling from library/hello-world              ← ③ baixando do Docker Hub
...
Hello from Docker!                                     ← ⑤ o container rodou
```

1. O cliente envia ao daemon o pedido "rode a imagem `hello-world`".
2. O daemon verifica se a imagem existe localmente. Na primeira vez, não existe.
3. O daemon baixa (*pull*) a imagem do Docker Hub.
4. O daemon cria um container a partir da imagem.
5. O container executa seu único programa, imprime a mensagem e termina. Como o programa terminou, o container para.

Rode o comando uma segunda vez: as linhas `Unable to find` e `Pulling` não aparecem mais, porque a imagem já está no disco.

> **Confusão comum:** "Docker e container são a mesma coisa". ✅ **Mais preciso:** o Docker é **uma** das ferramentas que criam containers, e por baixo dele existem outras peças. O `dockerd` não cria o container sozinho: ele delega ao **containerd**, que gerencia o ciclo de vida, e este chama o **runc**, um programa pequeno que efetivamente configura os namespaces e cgroups e inicia o processo. Os formatos de imagem e de execução são padronizados pela **OCI** (*Open Container Initiative*). Por isso uma imagem construída com Docker roda no Podman, no containerd puro ou no Kubernetes (que, desde 2022, nem usa mais o Docker internamente, e mesmo assim roda as mesmas imagens).

---

## 4. Namespaces: o que o container enxerga

O Docker não inventou um mecanismo de isolamento. Ele usa os **namespaces** do kernel Linux, que criam visões isoladas de partes do sistema. Um processo dentro de um namespace só enxerga o que está naquele namespace.

| Namespace | O que isola |
|---|---|
| `pid` | Processos. O container só vê os próprios, e o primeiro deles é o PID 1. |
| `net` | Interfaces de rede, endereços IP, tabelas de roteamento e portas. |
| `mnt` | Sistema de arquivos. Cada container enxerga apenas o seu. |
| `uts` | Hostname. O container tem um nome de máquina próprio. |
| `user` | Usuários e grupos (o root do container pode ser mapeado para um usuário comum do host). |
| `ipc` | Memória compartilhada e filas de mensagens entre processos. |

A tabela fica mais clara com os comandos a seguir. Todos usam a imagem `alpine`, um Linux mínimo de cerca de 8 MB, e a flag `--rm`, que remove o container assim que ele termina.

### Isolamento de processos (`pid`)

```bash
docker run --rm alpine ps
```

```
PID   USER     TIME  COMMAND
    1 root      0:00 ps
```

Dentro do container existe **um único processo**, o próprio `ps`, e ele é o PID 1. No host há centenas de processos rodando, mas o container não enxerga nenhum.

### Isolamento de hostname (`uts`)

```bash
hostname                          # no host
# meu-notebook

docker run --rm alpine hostname   # no container
# 3f9a1c2b7d4e
```

O container tem seu próprio nome de máquina. Por padrão, o Docker usa os 12 primeiros caracteres do ID do container.

### Isolamento de rede (`net`)

```bash
docker run --rm alpine ip addr
```

```
1: lo: <LOOPBACK,UP,LOWER_UP> ...
    inet 127.0.0.1/8 scope host lo
2: eth0@if12: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
```

O container tem sua própria interface `eth0`, com um IP de uma rede interna do Docker (`172.17.x.x`), e seu próprio `localhost`. Esse último detalhe é a origem de uma das confusões mais comuns com Docker, tratada no [03_volumes_e_redes.md](03_volumes_e_redes.md).

### Do outro lado: o container visto pelo host

Os namespaces escondem o host do container, mas **não** escondem o container do host:

```bash
docker run -d --name dorminhoco alpine sleep 1000   # inicia em segundo plano

ps aux | grep "sleep 1000"                          # no host
# root   48213  0.0  0.0   1612   4 ?   Ss  14:02  0:00 sleep 1000

docker rm -f dorminhoco                             # limpeza
```

O `sleep` aparece na lista de processos do host como qualquer outro programa, com outro número de PID. É a demonstração mais direta de que um container é um processo comum. (No Docker Desktop, inclusive no Linux, os containers rodam dentro de uma VM, e por isso o processo não aparece no `ps` do host. A demonstração funciona com o Docker Engine instalado direto no Linux.)

---

## 5. cgroups: quanto o container pode consumir

Namespaces controlam o que o container **vê**. Os **cgroups** (*control groups*) controlam o quanto ele **consome**: memória, CPU, operações de disco.

Sem limites, um container com vazamento de memória consumiria toda a RAM do servidor e derrubaria os outros containers junto. Com cgroups, o problema fica contido no próprio container.

### Definindo limites

```bash
# No máximo 512 MB de RAM e meia CPU
docker run -d --name limitado --memory="512m" --cpus="0.5" nginx

docker stats --no-stream limitado
```

```
CONTAINER ID   NAME       CPU %   MEM USAGE / LIMIT   MEM %
a1b2c3d4e5f6   limitado   0.00%   3.2MiB / 512MiB     0.62%
```

A coluna `LIMIT` mostra o teto aplicado pelo kernel. Sem a flag `--memory`, ela mostraria a memória total da máquina.

### O que acontece quando o limite é ultrapassado

```bash
# Um container com 50 MB tentando alocar 200 MB
docker run --rm --memory="50m" python:3.12-slim \
  python -c "x = bytearray(200 * 1024 * 1024)"

echo $?
# 137
```

O kernel encerra o processo (o chamado *OOM kill*, de *out of memory*), e o código de saída **137** indica que ele foi morto à força (128 + o sinal 9, `SIGKILL`). Quando um container em produção "morre sem deixar log", verificar se o código de saída foi 137 é um dos primeiros passos de investigação. O `docker inspect` confirma:

```bash
docker inspect --format '{{.State.OOMKilled}}' nome-do-container
# true
```

```bash
docker rm -f limitado   # limpeza
```

---

## 6. O primeiro servidor: Nginx e o mapeamento de portas

Até aqui os containers rodaram um comando e terminaram. Um servidor web é diferente: ele fica rodando, esperando requisições.

```bash
docker run -d -p 8080:80 --name meu-nginx nginx
```

| Parâmetro | Significado |
|---|---|
| `-d` | *Detached*: roda em segundo plano e devolve o terminal |
| `-p 8080:80` | Mapeia a porta **8080 do host** para a porta **80 do container** |
| `--name meu-nginx` | Dá um nome legível ao container (sem ele, o Docker sorteia um) |
| `nginx` | A imagem a ser usada |

Abra `http://localhost:8080` no navegador e a página de boas-vindas do Nginx aparece.

**Por que o `-p` é necessário.** Por causa do namespace de rede (seção 4), o container tem sua própria rede, e a porta 80 dele **não é** a porta 80 da sua máquina. A flag `-p` abre uma passagem entre as duas:

```
Navegador → localhost:8080 → Docker → container:80 → Nginx
            (porta do host)           (porta do container)
```

A ordem é sempre `-p HOST:CONTAINER`. Por isso dá para rodar três Nginx ao mesmo tempo, todos ouvindo na porta 80 dentro dos próprios containers, mas publicados em portas diferentes do host:

```bash
docker run -d -p 8081:80 nginx
docker run -d -p 8082:80 nginx
docker run -d -p 8083:80 nginx
```

> **Confusão comum:** "a imagem do Nginx tem `EXPOSE 80` no Dockerfile, então a porta já fica acessível". ✅ **Mais preciso:** `EXPOSE` é **documentação**. Ele informa qual porta a aplicação usa, mas não abre nada. Rode `docker run -d nginx` sem `-p` e tente acessar `localhost:80`: não funciona. Quem publica a porta no host é sempre o `-p` (ou o `ports:` do Compose, no capítulo [Docker: Compose e produção](04_compose_e_producao.md)). O `EXPOSE` só tem efeito prático com a flag `-P` maiúscula, que publica automaticamente todas as portas declaradas em portas aleatórias do host.

> **Confusão comum:** "`-p 8080:80` só deixa a porta acessível na minha máquina". ✅ **Mais preciso:** por padrão, o `-p` publica a porta em **todas** as interfaces de rede do host (`0.0.0.0`). Em um servidor com IP público, isso significa que a porta fica aberta para a internet, e no Linux o Docker insere suas próprias regras no `iptables`, que podem **passar por cima** de um firewall como o `ufw`. Para publicar apenas localmente, indique a interface: `-p 127.0.0.1:8080:80`. Isso é especialmente importante para bancos de dados, que nunca deveriam ficar expostos.

```bash
# Limpeza
docker rm -f meu-nginx
```

---

## Próximo passo

Neste capítulo, todos os containers vieram de imagens prontas do Docker Hub: `alpine`, `nginx`, `python`. Mas o objetivo do Docker é empacotar **a sua** aplicação. O que exatamente é uma imagem, como ela é montada em camadas e como escrever uma do zero com um Dockerfile?

O [02_imagens_e_dockerfile.md](02_imagens_e_dockerfile.md) responde a essas perguntas.
