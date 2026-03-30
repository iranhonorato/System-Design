# Hands-On: Continuous Integration com GitHub Actions

## Objetivo

Configurar um pipeline de **Continuous Integration (CI)** que, a cada `git push` ou Pull Request, automaticamente:

1. Instala as dependências do projeto
2. Executa a análise de qualidade de código (lint)
3. Roda os testes automatizados
4. Reporta o resultado (sucesso ou falha) para o desenvolvedor

```
git push
    ↓
GitHub Actions detecta o evento
    ↓
Runner Ubuntu é criado
    ↓
Instala dependências
    ↓
Executa lint (flake8)
    ↓
Executa testes (pytest)
    ↓
✅ Pipeline verde → PR pode ser mergeado
❌ Pipeline vermelho → desenvolvedor corrige antes do merge
```

---

## Contexto: Por que CI antes de CD?

O CI é a **fundação** do CI/CD. Sem CI funcionando, fazer CD é perigoso — você pode estar deployando código com bugs.

A ordem importa:
1. **CI garante** que o código está correto
2. **CD pega** esse código correto e o entrega ao ambiente

Neste hands-on, construímos apenas o CI. O CD (deploy automático) é o próximo passo.

---

## Estrutura do Projeto

Para este exemplo, usaremos uma API Python simples com Flask:

```
meu-projeto/
├── .github/
│   └── workflows/
│       └── ci.yml          ← pipeline de CI
├── app.py                  ← aplicação Flask
├── test_app.py             ← testes com pytest
├── requirements.txt        ← dependências
└── Dockerfile              ← para containerização
```

### app.py (exemplo)

```python
from flask import Flask, jsonify

app = Flask(__name__)

@app.route("/")
def home():
    return jsonify({"message": "Hello, World!", "status": "ok"})

@app.route("/health")
def health():
    return jsonify({"status": "healthy"})

@app.route("/soma")
def soma():
    a = int(request.args.get("a", 0))
    b = int(request.args.get("b", 0))
    return jsonify({"resultado": a + b})

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

### test_app.py (testes)

```python
import pytest
from app import app

@pytest.fixture
def client():
    app.config["TESTING"] = True
    with app.test_client() as client:
        yield client

def test_home_retorna_200(client):
    """Testa se a rota raiz responde com status 200."""
    response = client.get("/")
    assert response.status_code == 200

def test_home_retorna_mensagem(client):
    """Testa se a resposta contém a mensagem esperada."""
    response = client.get("/")
    data = response.get_json()
    assert data["message"] == "Hello, World!"
    assert data["status"] == "ok"

def test_health_check(client):
    """Testa o endpoint de health check."""
    response = client.get("/health")
    assert response.status_code == 200
    data = response.get_json()
    assert data["status"] == "healthy"

def test_soma(client):
    """Testa a operação de soma."""
    response = client.get("/soma?a=3&b=4")
    data = response.get_json()
    assert data["resultado"] == 7
```

### requirements.txt

```
flask==3.0.0
pytest==7.4.0
flake8==6.1.0
```

---

## PASSO 1 — Criar o Arquivo de Workflow

Crie o arquivo `.github/workflows/ci.yml`:

```yaml
name: CI - Continuous Integration

# Define quando o pipeline executa
on:
  push:
    branches: [ "main", "develop" ]   # em qualquer push para main ou develop
  pull_request:
    branches: [ "main" ]              # em todo PR direcionado ao main

jobs:
  ci:
    name: Lint e Testes
    runs-on: ubuntu-latest            # máquina virtual Linux fornecida pelo GitHub

    steps:
      # Step 1: Baixa o código do repositório
      - name: 📥 Checkout do código
        uses: actions/checkout@v4

      # Step 2: Configura o Python na versão desejada
      - name: 🐍 Configurar Python 3.10
        uses: actions/setup-python@v4
        with:
          python-version: "3.10"
          cache: "pip"               # cacheia dependências para execuções mais rápidas

      # Step 3: Instala as dependências
      - name: 📦 Instalar dependências
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt

      # Step 4: Análise estática com flake8
      - name: 🔍 Lint com flake8
        run: |
          # Para o pipeline em erros de sintaxe e imports indefinidos
          flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
          # Avisa (sem parar) em problemas de estilo
          flake8 . --count --exit-zero --max-complexity=10 --max-line-length=127 --statistics

      # Step 5: Executa os testes com pytest
      - name: 🧪 Executar testes com pytest
        run: |
          pytest test_app.py -v --tb=short
```

### Entendendo cada configuração

**`on: push / pull_request`:** define os gatilhos. O pipeline executa em pushes para `main` e `develop`, e também em qualquer PR direcionado ao `main`. Isso garante que código problemático seja detectado antes do merge.

**`cache: "pip"`:** o GitHub Actions cacheia a pasta de pacotes do pip entre execuções. Se o `requirements.txt` não mudou, as dependências são restauradas do cache em segundos em vez de minutos.

**`flake8 --select=E9,F63,F7,F82`:** seleciona apenas erros críticos que realmente quebram o código (sintaxe inválida, variáveis indefinidas). Não falha em problemas estéticos — isso fica como aviso.

**`pytest -v --tb=short`:** `-v` (verbose) mostra o nome de cada teste e seu resultado. `--tb=short` mostra traceback reduzido quando um teste falha.

---

## PASSO 2 — Fazer Push e Observar

```bash
git add .github/workflows/ci.yml
git commit -m "ci: adicionar pipeline de continuous integration"
git push origin main
```

No GitHub, vá em **Actions**. Você verá o pipeline executando. Clique para ver os logs em tempo real.

### O que uma execução bem-sucedida parece

```
✅ 📥 Checkout do código         2s
✅ 🐍 Configurar Python 3.10    8s
✅ 📦 Instalar dependências     25s
✅ 🔍 Lint com flake8            3s
✅ 🧪 Executar testes com pytest 5s
```

Dentro do step de pytest, você verá algo como:

```
========================= test session starts ==========================
collected 4 items

test_app.py::test_home_retorna_200          PASSED   [ 25%]
test_app.py::test_home_retorna_mensagem     PASSED   [ 50%]
test_app.py::test_health_check              PASSED   [ 75%]
test_app.py::test_soma                      PASSED   [100%]

========================== 4 passed in 0.45s ==========================
```

---

## PASSO 3 — Proteger o Branch Main

Com o pipeline funcionando, configure o GitHub para **bloquear merges quando o CI falha**:

```
Repositório → Settings → Branches → Add branch protection rule

Branch name pattern: main

☑️ Require a pull request before merging
☑️ Require status checks to pass before merging
   → Busque e selecione: "Lint e Testes" (nome do job)
☑️ Require branches to be up to date before merging
```

Agora, nenhum PR pode ser mergeado para `main` enquanto o CI estiver vermelho.

---

## PASSO 4 — Testando o Pipeline com um Erro Intencional

Veja o pipeline funcionando como guardião do código. Introduza um erro intencional:

```python
# Em app.py, adicione um erro de sintaxe proposital
def funcao_quebrada(
    print("erro de sintaxe")
```

```bash
git add app.py
git commit -m "test: verificar se CI detecta erro de sintaxe"
git push origin main
```

O pipeline falhará no step de lint:

```
❌ 🔍 Lint com flake8

app.py:15:5: E999 SyntaxError: invalid syntax
1 error in 1 file

Process completed with exit code 1.
```

O PR não pode ser mergeado. Corrija o erro, faça novo push e o pipeline volta a ficar verde.

---

## PASSO 5 — Pipeline com Matrix (Testando Múltiplas Versões)

Uma funcionalidade poderosa: testar o código em múltiplas versões do Python simultaneamente.

```yaml
jobs:
  ci:
    name: Testes em Python ${{ matrix.python-version }}
    runs-on: ubuntu-latest

    strategy:
      matrix:
        python-version: ["3.9", "3.10", "3.11"]  # executa 3 jobs em paralelo

    steps:
      - uses: actions/checkout@v4

      - name: Configurar Python ${{ matrix.python-version }}
        uses: actions/setup-python@v4
        with:
          python-version: ${{ matrix.python-version }}

      - name: Instalar dependências
        run: pip install -r requirements.txt

      - name: Executar testes
        run: pytest test_app.py -v
```

Isso cria três jobs paralelos — um para Python 3.9, outro para 3.10 e outro para 3.11. Se o código quebrar em alguma versão específica, você descobre imediatamente.

---

## PASSO 6 — Adicionando Relatório de Cobertura

A **cobertura de testes** mede que percentual do código está sendo exercitado pelos testes. Um arquivo com 0% de cobertura não tem nenhum teste verificando seu comportamento.

```yaml
      - name: Instalar pytest-cov
        run: pip install pytest-cov

      - name: Executar testes com cobertura
        run: |
          pytest test_app.py -v --cov=app --cov-report=term-missing

      # Opcional: falhar se cobertura < 80%
      - name: Verificar cobertura mínima
        run: |
          pytest test_app.py --cov=app --cov-fail-under=80
```

A saída mostrará algo como:

```
---------- coverage: platform linux, python 3.10 ----------
Name      Stmts   Miss  Cover   Missing
-----------------------------------------
app.py       18      2    89%   15-16
-----------------------------------------
TOTAL         18      2    89%
```

As linhas 15-16 não têm cobertura — você sabe exatamente onde precisa adicionar testes.

---

## Visão Geral: CI Completo

O pipeline final integra todas as verificações:

```yaml
name: CI - Continuous Integration

on:
  push:
    branches: [ "main", "develop" ]
  pull_request:
    branches: [ "main" ]

jobs:
  ci:
    name: Qualidade e Testes
    runs-on: ubuntu-latest

    steps:
      - name: 📥 Checkout
        uses: actions/checkout@v4

      - name: 🐍 Python 3.10
        uses: actions/setup-python@v4
        with:
          python-version: "3.10"
          cache: "pip"

      - name: 📦 Dependências
        run: pip install -r requirements.txt

      - name: 🔍 Lint (erros críticos)
        run: flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics

      - name: 🔍 Lint (estilo)
        run: flake8 . --count --exit-zero --max-line-length=127 --statistics

      - name: 🧪 Testes + Cobertura
        run: pytest test_app.py -v --cov=app --cov-report=term-missing

      - name: 📊 Cobertura mínima (80%)
        run: pytest test_app.py --cov=app --cov-fail-under=80
```

---

## Próximos Passos

Com o CI funcionando, você tem a base para construir o CD (Continuous Delivery):

```
CI (este guia)     →    CD (próximo guia)
  ✅ Código válido   →    🐳 Build Docker
  ✅ Testes passam   →    📤 Push Docker Hub
                    →    🚀 Deploy na EC2
```

O CD pega o código validado pelo CI e o entrega automaticamente ao ambiente de produção.
