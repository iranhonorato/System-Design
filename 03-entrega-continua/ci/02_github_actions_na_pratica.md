# CI na Prática com GitHub Actions

**Pré-requisito:** [01_fundamentos_de_integracao_continua.md](01_fundamentos_de_integracao_continua.md), [../../02-containers/docker/02_imagens_e_dockerfile.md](../../02-containers/docker/02_imagens_e_dockerfile.md)

> Segue a documentação oficial do GitHub Actions (*Workflow syntax*, *About service containers*, *Security hardening for GitHub Actions*). Reescreve e amplia o antigo `ci_hannds_on.md` do repositório Arquitetura de Software, agora com FastAPI, testes de integração com banco real e build da imagem.

---

## Por que este arquivo existe

O capítulo anterior definiu a integração contínua como prática. Este a implementa. A pergunta central é: *como montar, do zero, um pipeline que verifique cada push e cada pull request de uma API Python, do lint até a imagem Docker, e que impeça código quebrado de entrar na `main`?*

O capítulo é construído em camadas. Começa com um pipeline de 20 linhas, que só roda o lint e os testes, e vai acrescentando uma peça por seção: proteção da `main`, jobs paralelos, matriz de versões, cache, testes de integração com Postgres, cobertura, build da imagem e as práticas de segurança. A seção 10 junta tudo no arquivo final.

---

## 1. O projeto de exemplo

Uma API da loja, com uma regra de negócio pura (o cálculo de frete) e um endpoint que grava no banco:

```
minha-api/
├── .github/
│   └── workflows/
│       └── ci.yml                 ← o pipeline (construído neste capítulo)
├── app/
│   ├── __init__.py
│   ├── frete.py                   ← regra de negócio pura
│   └── main.py                    ← a API
├── tests/
│   ├── unit/
│   │   ├── test_frete.py
│   │   └── test_api.py
│   └── integration/
│       └── test_pedidos.py        ← precisa de um Postgres
├── Dockerfile
├── pyproject.toml                 ← configuração do ruff e do pytest
├── requirements.txt               ← dependências da aplicação
└── requirements-dev.txt           ← dependências de desenvolvimento e testes
```

### O código

```python
# app/frete.py
def calcular_frete(peso_kg: float, distancia_km: float) -> float:
    """Frete = taxa fixa + valor por kg + valor por km."""
    if peso_kg <= 0 or distancia_km < 0:
        raise ValueError("peso deve ser positivo e distância não pode ser negativa")
    return round(10.0 + peso_kg * 1.5 + distancia_km * 0.1, 2)
```

```python
# app/main.py
import os

import psycopg
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

from app.frete import calcular_frete

DATABASE_URL = os.getenv("DATABASE_URL", "postgresql://app:app@localhost:5432/loja")

app = FastAPI()


class Pedido(BaseModel):
    produto: str
    quantidade: int


@app.get("/healthz")
def healthz():
    return {"status": "vivo"}


@app.get("/frete")
def frete(peso_kg: float, distancia_km: float):
    try:
        return {"valor": calcular_frete(peso_kg, distancia_km)}
    except ValueError as erro:
        raise HTTPException(status_code=422, detail=str(erro)) from erro


@app.post("/pedidos", status_code=201)
def criar_pedido(pedido: Pedido):
    with psycopg.connect(DATABASE_URL) as conn:
        (novo_id,) = conn.execute(
            "INSERT INTO pedidos (produto, quantidade) VALUES (%s, %s) RETURNING id",
            (pedido.produto, pedido.quantidade),
        ).fetchone()
    return {"id": novo_id, **pedido.model_dump()}
```

### Os testes

```python
# tests/unit/test_frete.py
import pytest

from app.frete import calcular_frete


@pytest.mark.parametrize(
    ("peso", "distancia", "esperado"),
    [
        (1, 0, 11.5),  # só taxa fixa + 1 kg
        (2, 100, 23.0),  # 10 + 3 + 10
        (0.5, 10, 11.75),
    ],
)
def test_calcula_frete(peso, distancia, esperado):
    assert calcular_frete(peso, distancia) == esperado


def test_peso_zero_e_invalido():
    with pytest.raises(ValueError):
        calcular_frete(0, 10)
```

```python
# tests/unit/test_api.py
from fastapi.testclient import TestClient

from app.main import app

cliente = TestClient(app)


def test_healthz():
    assert cliente.get("/healthz").json() == {"status": "vivo"}


def test_frete_invalido_devolve_422():
    resposta = cliente.get("/frete", params={"peso_kg": 0, "distancia_km": 5})
    assert resposta.status_code == 422
```

```python
# tests/integration/test_pedidos.py
import os

import psycopg
import pytest
from fastapi.testclient import TestClient

from app.main import app

pytestmark = pytest.mark.integration  # marca todos os testes deste arquivo
DATABASE_URL = os.getenv("DATABASE_URL", "postgresql://app:app@localhost:5432/loja")


@pytest.fixture(autouse=True)
def tabela_limpa():
    with psycopg.connect(DATABASE_URL) as conn:
        conn.execute(
            "CREATE TABLE IF NOT EXISTS pedidos ("
            "id SERIAL PRIMARY KEY, produto TEXT NOT NULL, quantidade INT NOT NULL)"
        )
        conn.execute("TRUNCATE pedidos")


def test_criar_pedido_grava_no_banco():
    resposta = TestClient(app).post("/pedidos", json={"produto": "caneca", "quantidade": 2})
    assert resposta.status_code == 201

    with psycopg.connect(DATABASE_URL) as conn:
        linha = conn.execute("SELECT produto, quantidade FROM pedidos").fetchone()
    assert linha == ("caneca", 2)
```

Repare que o `DATABASE_URL` do teste é lido com `os.getenv` e um valor padrão. Com `os.environ["DATABASE_URL"]`, o arquivo falharia ao ser **importado** em uma máquina sem a variável, e o pytest importa todos os arquivos de teste antes de aplicar o filtro `-m`. Até a execução só dos unitários quebraria.

O teste de integração usa um Postgres **de verdade**, e não um *mock*: é exatamente essa a diferença entre as camadas da pirâmide do [capítulo anterior](01_fundamentos_de_integracao_continua.md). A marca `integration` permite rodar as duas camadas separadamente.

### Configuração e dependências

```toml
# pyproject.toml
[tool.ruff]
line-length = 100

[tool.ruff.lint]
select = ["E", "F", "I", "B"]     # erros, pyflakes, ordenação de imports, bugbear

[tool.pytest.ini_options]
pythonpath = ["."]                # permite "from app..." nos testes
markers = ["integration: testes que precisam de um Postgres"]
```

```text
# requirements.txt
fastapi==0.162.0
uvicorn==0.54.0
psycopg[binary]==3.3.6
```

```text
# requirements-dev.txt
-r requirements.txt
pytest==9.1.1
pytest-cov==7.1.0
httpx2==2.16.0         # cliente HTTP usado pelo TestClient (sucessor do httpx no Starlette 1.x)
ruff==0.17.2
```

Antes de qualquer pipeline, tudo isso deve funcionar **na sua máquina**:

```bash
pip install -r requirements-dev.txt
ruff check . && ruff format --check .
pytest -m "not integration"                    # só os unitários: não precisa de banco

docker run -d --name pg-teste -p 5432:5432 \
  -e POSTGRES_USER=app -e POSTGRES_PASSWORD=app -e POSTGRES_DB=loja postgres:16-alpine
DATABASE_URL=postgresql://app:app@localhost:5432/loja pytest -m integration
docker rm -f pg-teste
```

Um pipeline automatiza o que já funciona manualmente. Se um passo não funciona na sua máquina, ele não vai funcionar no pipeline, e lá é bem mais lento de investigar.

---

## 2. Os conceitos do GitHub Actions

O GitHub Actions executa **workflows** (fluxos de trabalho) definidos em arquivos YAML dentro de `.github/workflows/`. Cada arquivo é um workflow independente.

```
 evento (push, pull_request, agendamento...)
     │
     ▼
┌─────────────────── WORKFLOW  (.github/workflows/ci.yml) ───────────────────┐
│                                                                            │
│  ┌──────── JOB "lint" ─────────┐      ┌──────── JOB "testes" ────────┐     │
│  │ runner: ubuntu-latest (VM)  │      │ runner: ubuntu-latest (VM)   │     │
│  │                             │      │                              │     │
│  │  step: checkout             │      │  step: checkout              │     │
│  │  step: instalar Python      │      │  step: instalar Python       │     │
│  │  step: ruff check           │      │  step: pytest                │     │
│  └─────────────────────────────┘      └──────────────────────────────┘     │
│             jobs rodam EM PARALELO, cada um em uma VM nova                 │
└────────────────────────────────────────────────────────────────────────────┘
```

| Conceito | O que é |
|---|---|
| **Workflow** | O pipeline inteiro, descrito em um arquivo YAML |
| **Evento** | O que dispara o workflow: um `push`, a abertura de um `pull_request`, um horário (`schedule`), um clique (`workflow_dispatch`) |
| **Job** | Um conjunto de steps que roda **em um runner**. Jobs diferentes rodam em paralelo, a não ser que um dependa de outro (`needs`) |
| **Step** | Uma etapa dentro de um job: um comando de shell (`run:`) ou uma action pronta (`uses:`) |
| **Action** | Um bloco reutilizável publicado por alguém, como `actions/checkout` (baixa o código) ou `actions/setup-python` (instala o Python) |
| **Runner** | A máquina que executa um job. Os hospedados pelo GitHub são **VMs novas a cada job**, destruídas no fim |

A última linha tem uma consequência que explica boa parte dos erros de iniciante: **cada job começa em uma máquina vazia**. Ela não tem o seu código (por isso todo job começa com `checkout`), não tem as dependências instaladas por outro job e não guarda nada da execução anterior. Para passar arquivos entre jobs, usam-se *artifacts* (seção 7); para reaproveitar downloads entre execuções, *cache* (seção 5).

---

## 3. O primeiro workflow

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  verificar:
    name: Lint e testes unitários
    runs-on: ubuntu-latest

    steps:
      - name: Baixar o código
        uses: actions/checkout@v7

      - name: Instalar o Python
        uses: actions/setup-python@v7
        with:
          python-version: "3.12"

      - name: Instalar dependências
        run: pip install -r requirements-dev.txt

      - name: Lint
        run: ruff check .

      - name: Formatação
        run: ruff format --check .

      - name: Testes unitários
        run: pytest -m "not integration" -v
```

Linha por linha:

- **`name: CI`**: o nome que aparece na aba *Actions* do repositório.
- **`on:`**: os eventos que disparam o workflow. Aqui, todo push na `main` e todo pull request **direcionado** à `main`. É assim que o código é verificado **antes** do merge (no PR) e **depois** (na `main`, para garantir que a combinação de vários merges continua funcionando).
- **`jobs.verificar`**: um job com o identificador `verificar`. O `name` é o rótulo exibido, e é por ele que o job aparece nas regras de proteção da seção 4.
- **`runs-on: ubuntu-latest`**: o tipo de runner, uma VM Linux mantida pelo GitHub.
- **`uses: actions/checkout@v7`**: baixa o repositório, no commit que disparou o evento, para dentro da VM. O `@v7` fixa a versão principal da action.
- **`with:`**: os parâmetros da action.
- **`run:`**: um comando de shell. Se o comando terminar com código diferente de zero, **o step falha, o job falha e os steps seguintes não rodam**. Esse é todo o mecanismo de "parar quando algo quebra": o pipeline confia no código de saída dos comandos, exatamente como o `docker run` e o kubelet confiavam nos códigos de saída dos containers.

### Rodando

```bash
git add .github/workflows/ci.yml
git commit -m "ci: adiciona pipeline de integração contínua"
git push
```

Na aba **Actions** do repositório, o workflow aparece em execução. Clicando nele, cada step mostra seu log:

```
✔ Baixar o código            1s
✔ Instalar o Python          2s
✔ Instalar dependências     14s
✔ Lint                       0s
✔ Formatação                 0s
✔ Testes unitários           2s

tests/unit/test_api.py::test_healthz PASSED                         [ 16%]
tests/unit/test_api.py::test_frete_invalido_devolve_422 PASSED      [ 33%]
tests/unit/test_frete.py::test_calcula_frete[1-0-11.5] PASSED       [ 50%]
tests/unit/test_frete.py::test_calcula_frete[2-100-23.0] PASSED     [ 66%]
tests/unit/test_frete.py::test_calcula_frete[0.5-10-11.75] PASSED   [ 83%]
tests/unit/test_frete.py::test_peso_zero_e_invalido PASSED          [100%]
==================== 6 passed, 1 deselected in 0.41s ====================
```

O `1 deselected` é o teste de integração, ignorado pelo filtro `-m "not integration"`. Ele ganha um job próprio na seção 6.

### Vendo o pipeline falhar

Um pipeline só prova seu valor quando falha. Crie uma branch, deixe um import sem uso no `app/main.py` e abra um PR:

```python
import json      # nunca usado
```

O step de lint falha, e o log aponta o problema (o formato exato varia entre versões do Ruff):

```
F401 [*] `json` imported but unused
 --> app/main.py:2:8
Found 1 error.
[*] 1 fixable with the `--fix` option.
Error: Process completed with exit code 1.
```

No PR, o check aparece em vermelho, com um link direto para o log. Os testes **não chegaram a rodar**: o lint vem antes, falha em segundos e interrompe o job. É o *fail fast* do capítulo anterior.

---

## 4. Protegendo a `main`

Um pipeline vermelho no PR é só um aviso: nada impede alguém de clicar em *Merge* mesmo assim. Para que o CI seja um **portão** e não um conselho, configure uma regra no repositório.

Em **Settings → Rules → Rulesets → New branch ruleset** (o modelo mais recente; o modelo clássico, em *Settings → Branches*, oferece as mesmas opções):

```
Ruleset name:        proteger-main
Enforcement status:  Active
Target branches:     Include default branch (main)

☑ Restrict deletions
☑ Block force pushes
☑ Require a pull request before merging
    Required approvals: 1
☑ Require status checks to pass
    ☑ Require branches to be up to date before merging
    Status checks: "Lint e testes unitários"     ← o `name` do job
```

A partir daí, código só entra na `main` por PR, com uma aprovação e com o pipeline verde. A opção *up to date* exige que a branch do PR esteja atualizada com a `main` antes do merge, de forma que os testes rodem sobre a combinação real que vai entrar.

Um detalhe importante: o nome exigido é o **nome do job**. Se você renomear o job no YAML, a regra passa a esperar por um check que nunca mais vai aparecer, e todo PR fica bloqueado. Ao renomear, atualize a regra junto.

---

## 5. Jobs paralelos, matriz e cache

### 5.1 Separando em jobs

Com tudo em um job, os passos rodam em sequência. Separados, eles rodam **em paralelo**, cada um em sua VM, e cada um aparece como um check independente no PR:

```yaml
jobs:
  lint:
    name: Lint e formatação
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-python@v7
        with:
          python-version: "3.12"
      - run: pip install ruff==0.17.2          # o lint não precisa das outras dependências
      - run: ruff check .
      - run: ruff format --check .

  testes-unitarios:
    name: Testes unitários
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-python@v7
        with:
          python-version: "3.12"
      - run: pip install -r requirements-dev.txt
      - run: pytest -m "not integration"
```

### 5.2 Dependências entre jobs: `needs`

Para que um job só comece depois de outros terminarem com sucesso, use `needs`:

```yaml
  testes-integracao:
    needs: [lint, testes-unitarios]      # só roda se os dois passarem
```

O GitHub desenha o grafo resultante na página da execução:

```
  lint ─────────────┐
                    ├──▶ testes-integracao ──▶ build-imagem
  testes-unitarios ─┘
```

### 5.3 Matriz: a mesma verificação em várias versões

Uma **matriz** executa o mesmo job várias vezes, uma para cada combinação de valores:

```yaml
  testes-unitarios:
    name: Testes unitários (Python ${{ matrix.python-version }})
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false                    # uma versão falhando não cancela as outras
      matrix:
        python-version: ["3.12", "3.13", "3.14"]
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-python@v7
        with:
          python-version: ${{ matrix.python-version }}
      - run: pip install -r requirements-dev.txt
      - run: pytest -m "not integration"
```

São três jobs paralelos. A sintaxe `${{ ... }}` é uma **expressão**: o GitHub substitui o valor antes de executar. A matriz é especialmente útil para **bibliotecas**, que precisam funcionar nas versões que os usuários delas têm. Para uma **aplicação**, que roda em uma única versão (a da imagem Docker), testar só essa versão costuma bastar, e economiza minutos de runner.

### 5.4 Cache de dependências

A cada execução, a VM nova baixa todas as dependências de novo. O `setup-python` pode guardar a pasta de downloads do pip entre execuções:

```yaml
      - uses: actions/setup-python@v7
        with:
          python-version: "3.12"
          cache: pip
          cache-dependency-path: requirements-dev.txt
```

A **chave** do cache é calculada a partir do conteúdo de `requirements-dev.txt`. Enquanto o arquivo não muda, as dependências vêm do cache; quando muda, o cache é refeito. É a mesma ideia do cache de camadas do [capítulo de imagens](../../02-containers/docker/02_imagens_e_dockerfile.md): reaproveitar o que não mudou.

---

## 6. Testes de integração com um Postgres real

### 6.1 Service containers

O teste de integração precisa de um Postgres. O GitHub Actions pode subir containers auxiliares ao lado de um job, os **service containers**:

```yaml
  testes-integracao:
    name: Testes de integração
    needs: [lint, testes-unitarios]
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: app
          POSTGRES_PASSWORD: app
          POSTGRES_DB: loja
        ports:
          - 5432:5432                     # publica a porta na VM do runner
        options: >-
          --health-cmd "pg_isready -U app -d loja"
          --health-interval 5s
          --health-timeout 3s
          --health-retries 10

    env:
      DATABASE_URL: postgresql://app:app@localhost:5432/loja

    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-python@v7
        with:
          python-version: "3.12"
          cache: pip
          cache-dependency-path: requirements-dev.txt
      - run: pip install -r requirements-dev.txt
      - run: pytest -m integration -v
```

Três detalhes que vêm diretamente da parte de Docker:

- **`options: --health-cmd ...`** é o mesmo healthcheck do [capítulo de Compose](../../02-containers/docker/04_compose_e_producao.md). O GitHub só começa os steps depois que o container fica `healthy`, que é o equivalente ao `depends_on: condition: service_healthy`.
- **`ports: 5432:5432`** publica a porta do container na VM, como o `-p` do [capítulo de arquitetura do Docker](../../02-containers/docker/01_arquitetura_e_isolamento.md).
- **`env:` no nível do job** vale para todos os steps. É assim que o `DATABASE_URL` chega ao pytest.

> **Confusão comum:** "o capítulo de redes do Docker disse que `localhost` não funciona entre containers, então este `DATABASE_URL` com `localhost` está errado". ✅ **Mais preciso:** depende de **onde o job roda**. Neste workflow, os steps rodam **direto na VM** do runner, e não em um container. O Postgres, sim, é um container, mas com a porta publicada na VM (`ports: 5432:5432`), exatamente como no seu notebook com `docker run -p 5432:5432`. Para quem está na VM, `localhost:5432` chega ao Postgres. Se o job rodasse **dentro** de um container (com a chave `container:` no job), a situação seria a do [capítulo de volumes e redes](../../02-containers/docker/03_volumes_e_redes.md): os dois containers ficariam em uma rede Docker criada pelo GitHub, e o endereço correto seria o **nome do serviço**, `postgres:5432`, sem precisar de `ports:`.

### 6.2 Por que não usar um *mock* do banco aqui

Os testes unitários de `test_api.py` não tocam no banco, e isso é proposital: eles verificam a lógica da API em milissegundos. Mas um mock não verifica se a query SQL está correta, se a coluna existe, se o tipo bate. Esses erros só aparecem diante de um Postgres de verdade, e é para isso que existe a camada de integração. Ela é mais lenta, por isso roda **depois** dos unitários (`needs`), e só se eles passarem.

---

## 7. Cobertura e relatórios

### 7.1 Cobertura de testes

O **pytest-cov** mede quais linhas do código foram executadas pelos testes:

```yaml
      - run: pytest -m "not integration" --cov=app --cov-report=term-missing --cov-report=xml
```

```
---------- coverage: platform linux, python 3.12 ----------
Name              Stmts   Miss  Cover   Missing
-----------------------------------------------
app/__init__.py       0      0   100%
app/frete.py          4      0   100%
app/main.py          22      3    86%   35-40
-----------------------------------------------
TOTAL                26      3    88%
```

A coluna `Missing` mostra exatamente quais linhas nenhum teste executou. Aqui, são as do `POST /pedidos`, que só é exercitado pelo teste de integração. Para medir as duas camadas juntas, rode a cobertura nos dois jobs e combine os relatórios, ou rode a cobertura apenas no job de integração com todos os testes.

Para transformar a cobertura em um portão, acrescente `--cov-fail-under=80`: o comando falha se a cobertura total ficar abaixo de 80%. O [próximo capítulo](03_qualidade_e_seguranca_no_pipeline.md) discute o que esse número mede, e o que ele não mede.

### 7.2 Artifacts: guardando arquivos da execução

A VM é destruída no fim do job, e os relatórios iriam junto. Um **artifact** é um arquivo guardado pelo GitHub ao fim da execução, disponível para download na página do workflow e para outros jobs:

```yaml
      - run: pytest -m "not integration" --cov=app --cov-report=xml --junitxml=relatorio.xml

      - name: Guardar relatórios
        if: always()                     # guarda até quando os testes falham
        uses: actions/upload-artifact@v7
        with:
          name: relatorios-testes
          path: |
            relatorio.xml
            coverage.xml
          retention-days: 14
```

O `if: always()` é uma **condição**: por padrão, um step só roda se todos os anteriores passaram, mas os relatórios são mais úteis justamente quando algo falhou. Outras condições comuns:

| Condição | Roda quando |
|---|---|
| `if: success()` | Tudo antes passou (o padrão) |
| `if: failure()` | Algum step anterior falhou (útil para notificações) |
| `if: always()` | Sempre |
| `if: github.ref == 'refs/heads/main'` | Só em execuções na `main` |
| `if: github.event_name == 'pull_request'` | Só em pull requests |

---

## 8. Build da imagem

O último job prova que a imagem Docker é construída com sucesso, usando o Dockerfile multi-stage do [capítulo de Compose e produção](../../02-containers/docker/04_compose_e_producao.md):

```yaml
  build-imagem:
    name: Build da imagem
    needs: testes-integracao
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7

      - uses: docker/setup-buildx-action@v4

      - uses: docker/build-push-action@v7
        with:
          context: .
          push: false                          # aqui só constrói; publicar é tarefa do CD
          tags: minha-api:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

- **`setup-buildx-action`** prepara o **BuildKit**, o motor de build moderno do Docker.
- **`tags: minha-api:${{ github.sha }}`** usa como tag o **hash do commit**. É a tag única recomendada nos capítulos de [imagens](../../02-containers/docker/02_imagens_e_dockerfile.md) e de [Kubernetes](../../02-containers/kubernetes/02_workloads_rede_e_escala.md): a imagem fica ligada, sem ambiguidade, ao código que a gerou.
- **`cache-from` / `cache-to: type=gha`** guarda as camadas da imagem no cache do GitHub Actions. Sem isso, cada execução reconstrói todas as camadas, porque a VM é nova. Com isso, o cache de camadas funciona entre execuções, como funcionava na sua máquina.

Por que construir a imagem no CI se ela não é publicada? Porque um Dockerfile quebrado (um arquivo que o `COPY` não encontra, uma dependência de sistema faltando) é um defeito como qualquer outro, e deve ser descoberto no PR, e não na hora do deploy. Publicar a imagem é o primeiro passo do [pipeline de CD](../cd/03_pipeline_de_cd_na_pratica.md).

---

## 9. Segurança do workflow

Um workflow executa código com acesso ao repositório e, frequentemente, a segredos. Algumas práticas reduzem bastante o risco.

### 9.1 Secrets e variáveis

Valores sensíveis (tokens, senhas) ficam em **Settings → Secrets and variables → Actions**, e são lidos no workflow com `${{ secrets.NOME }}`. O GitHub mascara os valores nos logs, substituindo-os por `***`. Valores não sensíveis (uma região, o nome de um registry) ficam em **variables** e são lidos com `${{ vars.NOME }}`.

```yaml
      - run: ./publicar.sh
        env:
          TOKEN_REGISTRY: ${{ secrets.TOKEN_REGISTRY }}
          REGIAO: ${{ vars.REGIAO_AWS }}
```

O pipeline de CI deste capítulo não precisa de nenhum segredo, e isso é bom sinal: verificar código não deveria exigir credenciais. Os segredos aparecem no CD.

### 9.2 Permissões mínimas do token

Toda execução recebe um token automático (`GITHUB_TOKEN`) para falar com a API do GitHub. Por padrão, dependendo da configuração da organização, ele pode ter permissão de escrita no repositório. Declare explicitamente só o que o workflow usa:

```yaml
permissions:
  contents: read            # só ler o código
```

É o princípio do menor privilégio do [capítulo de IAM](../../01-infraestrutura/aws/03_iam_identidades_e_permissoes.md), aplicado ao pipeline.

> **Confusão comum:** "um PR vindo de um fork recebe os secrets do repositório, então qualquer pessoa pode roubá-los abrindo um PR". ✅ **Mais preciso:** o GitHub protege esse caso por padrão. Workflows disparados por `pull_request` a partir de um **fork** rodam **sem acesso aos secrets** e com um `GITHUB_TOKEN` somente leitura, justamente porque o código do PR é de outra pessoa e pode ter sido escrito para vazá-los. O risco real está em contornar essa proteção. O evento `pull_request_target` roda **com** os secrets, no contexto do repositório de destino. Ele é seguro **apenas** se o workflow não executar código vindo do PR. Usar `pull_request_target` e, em seguida, fazer checkout do código do PR e rodar seus testes é a receita para um vazamento. Para a CI comum, `pull_request` é o evento certo.

### 9.3 Fixando as versões das actions

`uses: actions/checkout@v7` executa um código mantido por terceiros dentro do seu pipeline. A tag `v7` pode ser movida pelo autor para outro commit, e um atacante que comprometa uma action popular passa a executar código em todos os repositórios que a usam.

> **Confusão comum:** "usar `@main` ou `@v7` em uma action é seguro, porque ela é popular". ✅ **Mais preciso:** tags e branches são **mutáveis**. Em março de 2025, a action `tj-actions/changed-files`, usada por dezenas de milhares de repositórios, foi comprometida: as tags de versão passaram a apontar para um commit malicioso que imprimia os secrets dos pipelines nos logs, que em repositórios públicos ficam visíveis para qualquer um. A defesa é fixar a action pelo **hash completo do commit**, que é imutável, mantendo a versão em um comentário para leitura:
> ```yaml
> - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1  # v7.0.1
> ```
> Ferramentas como o **Dependabot** atualizam esses hashes automaticamente quando sai uma versão nova, com um PR que passa pelo próprio CI. Como regra prática: fixe por hash, no mínimo, as actions de terceiros, e prefira as mantidas por organizações conhecidas (`actions/`, `docker/`, `aws-actions/`).

### 9.4 Evitando execuções inúteis

Dois ajustes economizam minutos de runner e deixam o feedback mais rápido:

```yaml
concurrency:
  group: ci-${{ github.ref }}       # uma execução por branch
  cancel-in-progress: true          # um push novo cancela a execução anterior da mesma branch

jobs:
  lint:
    timeout-minutes: 5              # um job travado é morto em 5 minutos, não em 6 horas
```

---

## 10. O workflow completo

Juntando todas as seções:

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  lint:
    name: Lint e formatação
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-python@v7
        with:
          python-version: "3.12"
      - run: pip install ruff==0.17.2
      - run: ruff check .
      - run: ruff format --check .

  testes-unitarios:
    name: Testes unitários
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-python@v7
        with:
          python-version: "3.12"
          cache: pip
          cache-dependency-path: requirements-dev.txt
      - run: pip install -r requirements-dev.txt
      - run: >-
          pytest -m "not integration"
          --cov=app --cov-report=term-missing --cov-report=xml
          --junitxml=relatorio-unitarios.xml
      - uses: actions/upload-artifact@v7
        if: always()
        with:
          name: relatorios-unitarios
          path: |
            relatorio-unitarios.xml
            coverage.xml
          retention-days: 14

  testes-integracao:
    name: Testes de integração
    needs: [lint, testes-unitarios]
    runs-on: ubuntu-latest
    timeout-minutes: 10
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: app
          POSTGRES_PASSWORD: app
          POSTGRES_DB: loja
        ports:
          - 5432:5432
        options: >-
          --health-cmd "pg_isready -U app -d loja"
          --health-interval 5s
          --health-timeout 3s
          --health-retries 10
    env:
      DATABASE_URL: postgresql://app:app@localhost:5432/loja
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-python@v7
        with:
          python-version: "3.12"
          cache: pip
          cache-dependency-path: requirements-dev.txt
      - run: pip install -r requirements-dev.txt
      - run: pytest -m integration -v

  build-imagem:
    name: Build da imagem
    needs: testes-integracao
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v7
      - uses: docker/setup-buildx-action@v4
      - uses: docker/build-push-action@v7
        with:
          context: .
          push: false
          tags: minha-api:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

E o grafo de execução:

```
              ┌── lint (≈ 20s) ─────────────┐
 push / PR ───┤                             ├──▶ testes-integracao (≈ 1min) ──▶ build-imagem (≈ 1min)
              └── testes-unitarios (≈ 40s) ─┘
```

Com o cache aquecido, o pipeline inteiro fica abaixo de 3 minutos, bem dentro da meta de 10 do [capítulo anterior](01_fundamentos_de_integracao_continua.md). Na regra de proteção da seção 4, marque os quatro jobs como checks obrigatórios.

### Testando localmente

Para experimentar mudanças no workflow sem fazer dezenas de pushes, a ferramenta **act** executa workflows do GitHub Actions na sua máquina, usando Docker:

```bash
act pull_request -j lint
```

Ela não reproduz tudo com perfeição (o cache e algumas actions se comportam diferente), mas acelera bastante a fase de ajuste.

---

## Próximo passo

O pipeline deste capítulo verifica lint, formatação, testes em duas camadas e o build da imagem. Mas ele ainda deixa perguntas em aberto: 88% de cobertura é bom? O que fazer com um teste que às vezes falha sem motivo? As dependências instaladas têm vulnerabilidades conhecidas? Alguém deixou uma senha no código? E como manter o pipeline rápido quando ele crescer?

O [03_qualidade_e_seguranca_no_pipeline.md](03_qualidade_e_seguranca_no_pipeline.md) trata dessas perguntas.
