# Imagens e Dockerfile

**Pré-requisito:** [01_arquitetura_e_isolamento.md](01_arquitetura_e_isolamento.md)

> Segue a documentação oficial do Docker (*Dockerfile reference* e *Build cache*). Os livros de arquitetura deste repositório não cobrem o formato de imagens.

---

## Por que este arquivo existe

No capítulo anterior, todo container nasceu de uma imagem pronta. Para empacotar a sua própria aplicação, você precisa construir uma imagem, e para isso precisa entender duas coisas:

1. **O que é uma imagem por dentro.** Ela é feita de camadas empilhadas, e essa estrutura explica por que alguns builds levam 2 segundos e outros 2 minutos, e por que algumas imagens têm 100 MB e outras 1 GB.
2. **Como descrevê-la.** O Dockerfile é a receita da imagem, e a ordem das instruções importa mais do que parece.

Ao final deste capítulo, você terá uma API Python rodando a partir de uma imagem construída por você.

---

## 1. Imagem e container

Uma **imagem** é um pacote **estático e imutável** com tudo de que a aplicação precisa para rodar: um sistema de arquivos mínimo, o runtime (Python, Node, Java), as bibliotecas e o seu código. Uma vez construída, ela não muda.

Um **container** é uma **instância em execução** de uma imagem: o processo isolado que o capítulo anterior dissecou.

```
Imagem  ──── docker run ────▶  Container
(estática, no disco)            (processo em execução)

Analogia:
Receita de bolo ──── assar ────▶ Bolo na mesa
```

A analogia tem um detalhe importante: **com uma receita, dá para assar quantos bolos você quiser**. Da mesma forma, uma imagem pode originar vários containers ao mesmo tempo, cada um independente:

```bash
docker run -d --name web1 nginx
docker run -d --name web2 nginx
docker run -d --name web3 nginx

docker ps --format "table {{.Names}}\t{{.Image}}"
# NAMES   IMAGE
# web3    nginx
# web2    nginx
# web1    nginx

docker rm -f web1 web2 web3   # limpeza
```

Três containers, uma única imagem no disco.

---

## 2. Imagens são feitas de camadas

Uma imagem não é um bloco único. Ela é uma **pilha de camadas** (*layers*), em que cada camada guarda apenas o que mudou em relação à de baixo:

```
┌─────────────────────────────────┐
│  Camada 4: seu código           │  ← muda a cada commit
├─────────────────────────────────┤
│  Camada 3: suas dependências    │  ← muda quando o requirements.txt muda
├─────────────────────────────────┤
│  Camada 2: Python 3.12          │  ← muda raramente
├─────────────────────────────────┤
│  Camada 1: Debian (base)        │  ← quase nunca muda
└─────────────────────────────────┘
```

Dá para ver as camadas de qualquer imagem:

```bash
docker history nginx
```

```
IMAGE          CREATED      CREATED BY                                      SIZE
a8758716bb6a   2 weeks ago  CMD ["nginx" "-g" "daemon off;"]                0B
<missing>      2 weeks ago  EXPOSE map[80/tcp:{}]                           0B
<missing>      2 weeks ago  RUN /bin/sh -c set -x && groupadd --system ...  112MB
...
```

Cada linha é uma instrução do Dockerfile que construiu a imagem. Repare que `CMD` e `EXPOSE` ocupam 0 B: são metadados. Só as instruções que alteram arquivos (`RUN`, `COPY`, `ADD`) geram camadas com conteúdo.

### Camadas de leitura e a camada de escrita

As camadas da imagem são **somente leitura**. Quando um container é criado, o Docker empilha por cima delas uma camada fina **de escrita**, exclusiva daquele container. Tudo o que o container altera em tempo de execução (um log gravado, um arquivo temporário, um pacote instalado com `apt`) vai para essa camada.

```
         Container A              Container B
┌─────────────────────┐   ┌─────────────────────┐
│ camada de escrita A │   │ camada de escrita B │  ← uma por container, temporária
└──────────┬──────────┘   └──────────┬──────────┘
           └────────────┬────────────┘
           ┌────────────▼────────────┐
           │  Camada 3 (código)      │
           ├─────────────────────────┤
           │  Camada 2 (Python)      │  ← camadas da imagem: somente leitura,
           ├─────────────────────────┤     compartilhadas por todos os containers
           │  Camada 1 (Debian)      │
           └─────────────────────────┘
```

O mecanismo que faz essa pilha parecer um único sistema de arquivos para o processo se chama **sistema de arquivos em união** (*union file system*). No Linux, o driver usado hoje é o **overlay2**. Dois efeitos práticos:

- **Economia de disco:** dez containers da mesma imagem Python não guardam dez cópias do Python. As camadas da imagem existem **uma vez só** no disco, e cada container acrescenta apenas sua camada de escrita.
- **Dados temporários:** quando o container é removido, a camada de escrita vai junto. Qualquer dado gravado ali **se perde**. O [03_volumes_e_redes.md](03_volumes_e_redes.md) mostra como guardar dados de forma permanente.

---

## 3. O Dockerfile, instrução por instrução

O **Dockerfile** é o arquivo de texto que descreve como construir uma imagem. Ele se chama literalmente `Dockerfile`, sem extensão, e fica na raiz do projeto.

### O projeto de exemplo

Uma API mínima em FastAPI:

```
minha-api/
├── app.py
├── requirements.txt
├── .dockerignore
└── Dockerfile
```

```python
# app.py
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def raiz():
    return {"mensagem": "Olá de dentro do container!"}
```

```text
# requirements.txt
fastapi==0.115.0
uvicorn==0.30.6
```

### O Dockerfile

```dockerfile
# 1. Imagem base: o ponto de partida (Debian mínimo + Python 3.12)
FROM python:3.12-slim

# 2. Diretório de trabalho dentro da imagem (criado se não existir)
WORKDIR /app

# 3. Copia SÓ o arquivo de dependências primeiro (o motivo está na seção 4)
COPY requirements.txt .

# 4. Instala as dependências (--no-cache-dir evita guardar o cache do pip na imagem)
RUN pip install --no-cache-dir -r requirements.txt

# 5. Copia o restante do código
COPY . .

# 6. Documenta a porta usada pela aplicação (não publica nada; veja o capítulo *Docker: arquitetura e isolamento*)
EXPOSE 8000

# 7. Comando executado quando o container inicia
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

| Instrução | Quando executa | O que faz |
|---|---|---|
| `FROM` | Build | Define a imagem base. Todo Dockerfile começa por ela. |
| `WORKDIR` | Build | Define o diretório onde as instruções seguintes rodam. |
| `COPY` | Build | Copia arquivos do seu projeto para dentro da imagem. Gera uma camada. |
| `RUN` | Build | Executa um comando durante a construção. Gera uma camada. |
| `EXPOSE` | (metadado) | Documenta a porta. Não abre nada. |
| `ENV` | Build e execução | Define variáveis de ambiente. |
| `CMD` | **Execução** | Define o comando padrão do container. Não roda no build. |

A distinção entre **tempo de build** e **tempo de execução** é a mais importante da tabela. `RUN pip install` acontece uma vez, quando a imagem é construída. `CMD` acontece toda vez que um container é iniciado.

### Construindo e rodando

```bash
# Constrói a imagem a partir do Dockerfile na pasta atual (o ".")
docker build -t minha-api:1.0 .

# Roda um container a partir dela
docker run -d -p 8000:8000 --name minha-api minha-api:1.0

curl http://localhost:8000
# {"mensagem":"Olá de dentro do container!"}
```

A flag `-t minha-api:1.0` dá à imagem um **nome** (`minha-api`) e uma **tag** (`1.0`). A tag funciona como um rótulo de versão.

> **Confusão comum:** "minha API funciona fora do Docker, mas dentro do container o `curl` dá *connection refused*, mesmo com `-p 8000:8000`". ✅ **Mais preciso:** quase sempre a aplicação está ouvindo em `127.0.0.1`, que é o padrão de muitos servidores de desenvolvimento (uvicorn, Flask, Vite). Dentro do container, `127.0.0.1` é o `localhost` **do próprio container** (lembre do namespace de rede do capítulo *Docker: arquitetura e isolamento*), e o tráfego que chega pela porta publicada vem pela interface `eth0`, não pelo `localhost`. A aplicação precisa ouvir em **`0.0.0.0`** (todas as interfaces), como no `--host 0.0.0.0` do `CMD` acima.

---

## 4. O cache de build: por que a ordem importa

Ao construir uma imagem, o Docker executa as instruções de cima para baixo e, para cada uma, verifica se já tem aquela camada pronta de um build anterior. Se a instrução e os arquivos envolvidos não mudaram, ele **reaproveita** a camada (*cache hit*). A regra que governa tudo:

> **Quando uma camada muda, ela e todas as camadas acima dela são reconstruídas.**

### Na prática: a ordem certa

Com o Dockerfile da seção 3, altere uma linha do `app.py` e reconstrua:

```bash
docker build -t minha-api:1.1 .
```

```
 => [1/5] FROM docker.io/library/python:3.12-slim
 => CACHED [2/5] WORKDIR /app
 => CACHED [3/5] COPY requirements.txt .
 => CACHED [4/5] RUN pip install --no-cache-dir -r requirements.txt
 => [5/5] COPY . .
```

As etapas 2, 3 e 4 aparecem como `CACHED`. Só a cópia do código foi refeita, e o build leva segundos.

### Na prática: a ordem errada

Agora imagine o Dockerfile escrito de forma "mais simples", copiando tudo de uma vez:

```dockerfile
FROM python:3.12-slim
WORKDIR /app
# ↓ copia tudo, inclusive o app.py
COPY . .
# ↓ está acima do COPY, então depende dele
RUN pip install --no-cache-dir -r requirements.txt
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

Ao mudar uma linha do `app.py`, a camada do `COPY . .` muda. Como o `pip install` está **acima** dela, ele é invalidado e **todas as dependências são baixadas e instaladas de novo**, a cada mudança de código. Em projetos com muitas dependências, isso transforma um build de 5 segundos em um de vários minutos.

### O princípio geral

Ordene as instruções **da que muda menos para a que muda mais**:

```
FROM python          ← quase nunca muda
WORKDIR
COPY requirements    ← muda quando você adiciona uma biblioteca
RUN pip install
COPY . .             ← muda a cada commit
CMD
```

> **Confusão comum:** "adicionei `RUN rm -rf /tmp/arquivos-grandes` no fim do Dockerfile, então a imagem ficou menor". ✅ **Mais preciso:** cada instrução gera uma camada **nova**, e camadas são somente leitura. O `rm` em uma camada posterior apenas **esconde** os arquivos. Eles continuam guardados na camada anterior, e a imagem tem o mesmo tamanho (ou um pouco mais). Para que a remoção reduza o tamanho, ela precisa acontecer **na mesma instrução** que criou os arquivos:
> ```dockerfile
> RUN apt-get update && apt-get install -y gcc \
>     && rm -rf /var/lib/apt/lists/*
> ```
> Uma solução ainda melhor para esse problema são os **multi-stage builds** do capítulo [Docker: Compose e produção](04_compose_e_producao.md).

---

## 5. `.dockerignore`: o que não deve entrar na imagem

O `COPY . .` copia **tudo** o que está na pasta do projeto, e isso inclui coisas que não deveriam estar na imagem. O `.dockerignore` funciona como o `.gitignore`:

```text
# .dockerignore
.git
__pycache__
*.pyc
.venv
node_modules
.env
```

Ele resolve três problemas de uma vez:

- **Tamanho:** pastas como `.git`, `.venv` e `node_modules` podem ter centenas de MB.
- **Cache:** sem o `.dockerignore`, qualquer mudança dentro de `.git` (como um commit) altera o resultado do `COPY . .` e invalida o cache.
- **Segurança:** o `.env` costuma conter senhas e chaves de API. Se ele entrar na imagem e a imagem for publicada, os segredos vão junto.

---

## 6. `CMD` e `ENTRYPOINT`

As duas instruções definem o que o container executa ao iniciar, mas com papéis diferentes:

- **`CMD`** define o **comando padrão**, que pode ser **substituído** facilmente na hora do `docker run`.
- **`ENTRYPOINT`** define o **executável fixo** do container. O que vier depois no `docker run` (ou no `CMD`) vira **argumento** dele.

```dockerfile
# Só CMD: o comando inteiro pode ser trocado
CMD ["uvicorn", "app:app", "--host", "0.0.0.0"]
```
```bash
docker run minha-api            # roda o uvicorn
docker run minha-api python -V  # substitui o CMD: roda "python -V"
```

```dockerfile
# ENTRYPOINT + CMD: executável fixo, argumentos padrão
ENTRYPOINT ["ping"]
CMD ["localhost"]
```
```bash
docker run pinger               # roda "ping localhost"
docker run pinger google.com    # roda "ping google.com" (só o CMD foi trocado)
```

Na maioria das aplicações web, `CMD` sozinho é suficiente. `ENTRYPOINT` é útil quando a imagem funciona como uma ferramenta de linha de comando.

### Forma *exec* e forma *shell*

As duas instruções aceitam dois formatos, e a diferença tem consequências reais:

```dockerfile
CMD ["uvicorn", "app:app"]   # forma exec (lista JSON): recomendada
CMD uvicorn app:app          # forma shell (texto): evite
```

Na forma *shell*, o Docker executa `/bin/sh -c "uvicorn app:app"`. O PID 1 do container passa a ser o **shell**, e não a aplicação. Quando você roda `docker stop`, o Docker envia o sinal `SIGTERM` ao PID 1, e o shell **não repassa** esse sinal para o uvicorn. A aplicação não fica sabendo que precisa encerrar, o Docker espera 10 segundos e então a mata com `SIGKILL`, sem dar chance de fechar conexões ou terminar requisições em andamento. Na forma *exec*, a aplicação é o PID 1 e recebe o sinal diretamente.

---

## 7. Docker Hub: publicando e reutilizando imagens

O **Docker Hub** é o registry público padrão, uma espécie de loja de aplicativos para imagens. Ele traz **imagens oficiais** mantidas e auditadas para quase tudo:

| Imagem | O que é |
|---|---|
| `python:3.12-slim` | Python oficial sobre Debian mínimo |
| `node:20-alpine` | Node.js oficial sobre Alpine |
| `postgres:16` | PostgreSQL oficial |
| `nginx` | Nginx oficial |

**Prefira as imagens oficiais como base.** Elas recebem atualizações de segurança regulares, e construir uma base do zero raramente compensa.

### Publicando sua imagem

Para compartilhar sua imagem (ou para que um pipeline de CI/CD a envie para produção, como em [../../03-entrega-continua/cd/03_pipeline_de_cd_na_pratica.md](../../03-entrega-continua/cd/03_pipeline_de_cd_na_pratica.md)), o nome dela precisa começar com seu usuário do Docker Hub:

```bash
docker login                                         # autentica no Docker Hub

docker tag minha-api:1.0 seu-usuario/minha-api:1.0   # cria um novo nome para a mesma imagem
docker push seu-usuario/minha-api:1.0                # envia para o registry

# Em qualquer outra máquina:
docker pull seu-usuario/minha-api:1.0
docker run -d -p 8000:8000 seu-usuario/minha-api:1.0
```

O `docker tag` não copia nada: ele só adiciona um segundo nome que aponta para a mesma imagem.

> **Confusão comum:** "a tag `latest` sempre aponta para a versão mais recente". ✅ **Mais preciso:** `latest` é apenas a tag **usada quando nenhuma é informada**. `docker build -t minha-api .` cria `minha-api:latest`, e `docker pull nginx` baixa `nginx:latest`. Nada atualiza essa tag automaticamente: se alguém publicar a versão `2.0` sem também publicar como `latest`, a `latest` continua apontando para a versão antiga. Pior, ela é **mutável**: o mesmo `latest` pode significar uma imagem hoje e outra amanhã. Em produção, sempre use tags explícitas (`minha-api:1.4.2`, ou o hash do commit) para saber exatamente o que está rodando e conseguir voltar para a versão anterior se algo der errado.

---

## Próximo passo

A API do exemplo funciona, mas é uma API sem memória: não grava nada e não conversa com nenhum outro serviço. Sistemas reais têm banco de dados, e é aí que aparecem dois problemas novos: se a camada de escrita de um container é temporária, **onde ficam os dados do banco?** E se cada container tem sua própria rede, **como a API encontra o banco?**

O [03_volumes_e_redes.md](03_volumes_e_redes.md) responde às duas perguntas.
