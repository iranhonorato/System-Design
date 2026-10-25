# Volumes e Redes

**Pré-requisito:** [02_imagens_e_dockerfile.md](02_imagens_e_dockerfile.md)

> Segue a documentação oficial do Docker (*Storage* e *Networking*). A arquitetura API + banco usada nos exemplos é a configuração inicial descrita no Capítulo 1 de **System Design Interview** (Alex Xu), em que o banco é separado do servidor web para que cada um possa escalar de forma independente.

---

## Por que este arquivo existe

Até aqui, cada container foi uma ilha: nasceu, rodou e foi removido sem deixar rastro. Isso é ótimo para uma API sem estado, mas um sistema real tem pelo menos um componente que **precisa** lembrar das coisas, o banco de dados. Ao colocar um banco em um container, dois problemas aparecem imediatamente:

1. **Persistência.** O [capítulo anterior](02_imagens_e_dockerfile.md) mostrou que a camada de escrita de um container é apagada junto com ele. Se o Postgres grava seus dados ali, atualizar a versão do container significa perder o banco.
2. **Comunicação.** O capítulo [Docker: arquitetura e isolamento](01_arquitetura_e_isolamento.md) mostrou que cada container tem sua própria rede. Como a API descobre o endereço do banco?

Este capítulo resolve o primeiro problema com **volumes** e o segundo com **redes**.

---

## 1. O problema: a camada de escrita é temporária

Primeiro, a demonstração do problema:

```bash
# Cria um container que fica rodando e grava um arquivo nele
docker run -d --name teste alpine sleep 3600
docker exec teste sh -c "echo 'dado importante' > /dados.txt"

docker exec teste cat /dados.txt
# dado importante

# Reiniciar (parar e iniciar de novo) NÃO apaga o arquivo
docker restart teste
docker exec teste cat /dados.txt
# dado importante

# Remover o container, sim
docker rm -f teste
docker run --rm alpine cat /dados.txt
# cat: can't open '/dados.txt': No such file or directory
```

O arquivo foi gravado na camada de escrita do container `teste`. Ela sobreviveu ao `restart`, mas foi removida junto com o container. O container novo começa limpo, a partir da imagem.

Repare no detalhe que a demonstração mostra: **parar** um container não apaga nada. A camada de escrita só some quando o container é **removido** (`docker rm`). O problema é que, na prática, containers são removidos o tempo todo: toda atualização de versão cria um container novo a partir da imagem nova. É assim que o Docker foi pensado. Containers são descartáveis, e os dados precisam morar fora deles.

---

## 2. Volumes: dados que sobrevivem ao container

Um **volume** é uma área de armazenamento **gerenciada pelo Docker**, que fica fora da camada de escrita e tem um ciclo de vida próprio. O container é removido, o volume continua. Você o conecta a um diretório dentro do container, e tudo o que for gravado nesse diretório vai, na verdade, para o volume.

```
┌──────────────────────────┐
│  Container (descartável) │
│                          │
│  /var/lib/postgresql/data ──────┐
└──────────────────────────┘      │  montado em
                                  ▼
                       ┌──────────────────────┐
                       │  Volume "pgdata"     │  ← persiste depois do docker rm
                       │  (gerenciado pelo    │
                       │   Docker no host)    │
                       └──────────────────────┘
```

### Demonstração: um banco que sobrevive à remoção

```bash
# 1. Cria um volume
docker volume create pgdata

# 2. Sobe o Postgres com o volume montado no diretório de dados dele
docker run -d --name banco \
  -e POSTGRES_PASSWORD=senha123 \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16

# 3. Espere alguns segundos (o Postgres demora para aceitar conexões na primeira vez),
#    depois crie uma tabela e insira um registro
docker exec -it banco psql -U postgres -c \
  "CREATE TABLE clientes (nome TEXT); INSERT INTO clientes VALUES ('Ana');"

# 4. Remove o container por completo
docker rm -f banco

# 5. Sobe um container NOVO, apontando para o MESMO volume
docker run -d --name banco \
  -e POSTGRES_PASSWORD=senha123 \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16

# 6. Os dados continuam lá
docker exec -it banco psql -U postgres -c "SELECT * FROM clientes;"
#  nome
# ------
#  Ana
```

É exatamente assim que se atualiza um banco em container: remove-se o container antigo e cria-se um novo (com a imagem nova) apontando para o mesmo volume.

A sintaxe `-v NOME:CAMINHO` funciona como a do `-p`: **o que está à esquerda é do host, o que está à direita é do container**.

> **Confusão comum:** "os dados estão em um volume, então estão seguros". ✅ **Mais preciso:** um volume protege os dados contra **a remoção do container**, e só contra isso. Ele continua sendo um diretório no disco do host. Se o disco falhar, se a máquina for destruída, ou se alguém rodar `docker volume rm` ou `docker compose down -v`, os dados se vão. Volume é persistência, **não é backup**. Para bancos em produção, você ainda precisa de backups regulares guardados em outro lugar (por exemplo, `pg_dump` enviado para o S3), ou de um banco gerenciado como o AWS RDS, que cuida disso por você.

---

## 3. Bind mounts: uma pasta sua dentro do container

Existe uma segunda forma de montar armazenamento: o **bind mount**, que conecta uma **pasta específica da sua máquina** a um diretório do container.

```bash
docker run -d -p 8000:8000 \
  -v "$(pwd)":/app \
  minha-api:1.0 \
  uvicorn app:app --host 0.0.0.0 --port 8000 --reload
```

Quando o lado esquerdo do `-v` é um **caminho** (começa com `/` ou `./`), o Docker entende que é um bind mount. Quando é um **nome**, entende que é um volume.

Aqui, a pasta atual do projeto aparece dentro do container em `/app`, por cima do código que foi copiado na imagem. Ao editar o `app.py` no seu editor, a mudança aparece **imediatamente** no container, e a flag `--reload` do uvicorn reinicia a aplicação sozinha. Não é preciso reconstruir a imagem a cada alteração.

| | Volume | Bind mount |
|---|---|---|
| **Sintaxe** | `-v pgdata:/caminho` | `-v /pasta/do/host:/caminho` |
| **Quem gerencia** | O Docker | Você |
| **Onde fica no host** | Diretório interno do Docker | Onde você indicar |
| **Uso típico** | Dados de bancos e serviços em produção | Código-fonte durante o desenvolvimento |

A regra prática: **volumes para dados, bind mounts para desenvolvimento**. Em produção, o código deve vir da imagem, que é imutável e versionada, e não de uma pasta do servidor que alguém pode editar à mão.

---

## 4. Redes: como os containers se encontram

### O problema

Com o banco rodando em um container, a API precisa de uma string de conexão. A primeira tentativa costuma ser:

```python
DATABASE_URL = "postgresql://postgres:senha123@localhost:5432/postgres"
```

E ela falha.

> **Confusão comum:** "o banco está rodando na minha máquina, então a API conecta em `localhost`". ✅ **Mais preciso:** dentro de um container, `localhost` é **o próprio container**, por causa do namespace de rede visto no capítulo [Docker: arquitetura e isolamento](01_arquitetura_e_isolamento.md). Quando a API (em um container) tenta `localhost:5432`, ela procura um Postgres dentro de si mesma, e não acha nenhum. Cada container é como uma máquina separada na rede. Para que um container alcance outro, os dois precisam estar em uma mesma **rede Docker** e o endereço tem que ser o **nome** do outro container. (O caso inverso também existe: para acessar, de dentro de um container, um serviço que roda direto na sua máquina, o Docker Desktop oferece o nome especial `host.docker.internal`. No Docker Engine do Linux, ele precisa ser ativado com `--add-host=host.docker.internal:host-gateway`.)

### A solução: uma rede criada por você

```bash
# 1. Cria uma rede
docker network create minha-rede

# 2. Sobe o banco nessa rede
docker run -d \
  --name banco \
  --network minha-rede \
  -e POSTGRES_PASSWORD=senha123 \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16

# 3. Sobe a API na mesma rede
docker run -d \
  --name api \
  --network minha-rede \
  -p 8000:8000 \
  -e DATABASE_URL="postgresql://postgres:senha123@banco:5432/postgres" \
  minha-api:1.0
```

Agora, dentro da rede `minha-rede`, **o nome do container funciona como hostname**:

```python
DATABASE_URL = "postgresql://postgres:senha123@banco:5432/postgres"
#                                              ↑
#                         nome do container, resolvido pelo DNS do Docker
```

Dá para conferir a resolução de nomes direto de um container na rede:

```bash
docker run --rm --network minha-rede alpine ping -c 2 banco
# PING banco (172.18.0.2): 56 data bytes
# 64 bytes from 172.18.0.2: seq=0 ttl=64 time=0.091 ms
# 64 bytes from 172.18.0.2: seq=1 ttl=64 time=0.104 ms
```

O Docker roda um servidor DNS interno que traduz `banco` para o IP atual daquele container. Isso é importante porque os IPs **mudam**: cada vez que o container do banco é recriado, ele pode ganhar um IP diferente. O nome permanece.

Note também o que **não** foi feito: o banco **não tem `-p`**. A porta 5432 não é publicada no host, e por isso o banco só é alcançável por quem está na rede `minha-rede`. Esse é um padrão de segurança importante: só o que precisa receber tráfego de fora (a API) publica porta.

```
           Internet / seu navegador
                     │
                     │ localhost:8000
┌────────────────────┼─────────────────────────────┐
│ rede "minha-rede"  ▼                             │
│              ┌──────────┐        ┌──────────┐    │
│              │   api    │ ─────▶ │  banco   │    │
│              │  :8000   │ banco: │  :5432   │    │
│              └──────────┘  5432  └──────────┘    │
│                                  (sem -p: não    │
│                                   aparece fora)  │
└──────────────────────────────────────────────────┘
```

### A rede padrão e por que evitá-la

Containers criados sem `--network` vão para a rede padrão, chamada `bridge`. Eles conseguem se comunicar por IP nela, mas **não por nome**: o DNS interno só funciona em redes criadas pelo usuário. Por isso a regra é sempre criar uma rede própria para cada aplicação, e isso tem um bônus: aplicações em redes diferentes ficam isoladas umas das outras.

```bash
docker network ls                      # lista as redes
docker network inspect minha-rede      # mostra quais containers estão nela
```

```bash
# Limpeza da demonstração
docker rm -f api banco
docker network rm minha-rede
docker volume rm pgdata
```

---

## Próximo passo

A aplicação deste capítulo já é um sistema de verdade: uma API, um banco, um volume e uma rede. Mas subir esse sistema exigiu cinco comandos longos, digitados na ordem certa, com nomes que precisam bater entre si (`banco` na string de conexão, `minha-rede` nos dois containers, `pgdata` no volume). Basta acrescentar um Redis e um worker para isso ficar impraticável, e frágil.

O [04_compose_e_producao.md](04_compose_e_producao.md) descreve o mesmo sistema em um único arquivo declarativo, e depois trata do que muda quando a imagem vai para produção.
