# Testes de Software

## O Problema que os Testes Resolvem

Antes de definir o que são testes, é mais fácil entender por que eles existem partindo de um cenário sem eles.

Imagine um time de desenvolvimento entregando uma nova funcionalidade de carrinho de compras. O código parece certo, o desenvolvedor testou manualmente no próprio computador, e o deploy foi feito. Duas horas depois, o suporte recebe reclamações: clientes estão sendo cobrados o dobro do valor. O que aconteceu? Uma mudança no módulo de pagamento afetou silenciosamente o cálculo de desconto do carrinho — ninguém percebeu porque ninguém testou essa interação.

```
Sem testes:

  Dev escreve código
        │
        │  "Testei aqui e funcionou"
        ▼
     Deploy
        │
        ▼
   Bug em produção
        │
        ▼
   Clientes afetados → Suporte sobrecarregado → Receita perdida → Reputação danificada
```

```
Com testes automatizados:

  Dev escreve código
        │
        ├─→ Testes rodam automaticamente
        │         │
        │         ├── ✅ Tudo passa → segue para deploy
        │         └── ❌ Falhou → dev corrige antes de sair da própria máquina
        ▼
     Deploy confiante
        │
        ▼
   Produção estável
```

A diferença fundamental: com testes, os erros são encontrados **cedo**, quando corrigir é barato. Sem testes, os erros são encontrados **tarde**, quando corrigir é caro.

> **Regra dos 10x:** Um bug corrigido durante o desenvolvimento custa 1x. O mesmo bug encontrado em testes de integração custa 10x. Em produção, custa 100x — em tempo de engenharia, reputação e potencialmente dinheiro de clientes.

---

## 1. O que são Testes de Software?

Um teste de software é um **código que verifica se outro código se comporta como esperado**. Você escreve um teste que descreve uma situação (entrada) e define qual deve ser o resultado (saída esperada). O framework de testes executa essa verificação automaticamente.

```python
# Função que queremos testar
def calcular_desconto(preco, percentual):
    return preco - (preco * percentual / 100)

# Teste que verifica o comportamento
def test_desconto_de_10_por_cento():
    resultado = calcular_desconto(100.00, 10)
    assert resultado == 90.00   # se isso falhar, o teste falha
```

Quando você roda esse teste, uma de duas coisas acontece:

```
✅ PASSOU: calcular_desconto(100, 10) retornou 90.00 — comportamento correto

❌ FALHOU: calcular_desconto(100, 10) retornou 95.00 — algo está errado
           AssertionError: 95.0 != 90.0
```

A beleza dos testes automatizados é que esse mesmo código de verificação pode ser executado em milissegundos, milhares de vezes, por qualquer máquina — sem intervenção humana.

---

## 2. A Pirâmide de Testes

Não existe um único tipo de teste. Cada tipo opera em um nível diferente do sistema, com velocidade, custo e abrangência distintos. O modelo mais adotado para organizar isso é a **Pirâmide de Testes**, proposta por Mike Cohn.

```
                        ╱‾‾‾‾‾‾‾‾‾‾‾╲
                       ╱    E2E /     ╲
                      ╱   UI Tests     ╲
                     ╱  (poucos, lentos)╲
                    ╱─────────────────────╲
                   ╱                       ╲
                  ╱    Testes de Integração  ╲
                 ╱       (quantidade média)   ╲
                ╱─────────────────────────────────╲
               ╱                                   ╲
              ╱         Testes Unitários             ╲
             ╱    (muitos, rápidos, baratos)          ╲
            ╱─────────────────────────────────────────────╲

Quanto mais abaixo na pirâmide:
  → Mais rápido para executar
  → Mais barato para escrever e manter
  → Mais isolado (testa menos partes ao mesmo tempo)
  → Mais fácil de identificar onde está o erro

Quanto mais acima:
  → Mais lento para executar
  → Mais caro para manter
  → Testa mais partes integradas
  → Mais próximo da experiência real do usuário
```

A pirâmide não é uma regra rígida — é um guia de proporção. A ideia central é: **tenha muitos testes unitários, alguns de integração, e poucos end-to-end**.

---

## 3. Tipos de Testes

### 3.1 Testes Unitários

Testam a **menor unidade isolável** do código — geralmente uma única função ou método — completamente separada de qualquer dependência externa (banco de dados, rede, outros serviços).

```
O que um teste unitário isola:

  ┌──────────────────────────────────────────────────┐
  │  FUNÇÃO sendo testada                            │
  │                                                  │
  │  calcular_desconto(preco, percentual)            │
  │       │                                          │
  │       └── só usa parâmetros recebidos            │
  │           sem tocar em banco ou rede             │
  └──────────────────────────────────────────────────┘

  Dependências externas: NENHUMA → velocidade máxima
```

**Características:**
- Executam em milissegundos
- Completamente determinísticos (mesmo input → mesmo output, sempre)
- Identificam o erro com precisão cirúrgica
- Não dependem de infraestrutura (banco, Redis, APIs externas)

**Exemplo:**

```python
# Testa casos normais e casos extremos (edge cases)
def test_desconto_zero():
    assert calcular_desconto(200.00, 0) == 200.00

def test_desconto_total():
    assert calcular_desconto(200.00, 100) == 0.00

def test_preco_com_centavos():
    assert calcular_desconto(99.90, 10) == 89.91
```

**Quando usar:** para toda lógica de negócio, cálculos, transformações de dados, validações, e qualquer função pura.

---

### 3.2 Testes de Integração

Testam como **dois ou mais componentes funcionam juntos**. Aqui, as dependências externas reais são usadas — banco de dados, Redis, APIs internas.

```
O que um teste de integração verifica:

  ┌──────────────────────────────────────────────────┐
  │  Serviço de Pedidos                              │
  │       │                                          │
  │       ├── chama Repositório de Pedidos           │
  │       │        │                                 │
  │       │        └── conecta ao PostgreSQL real    │
  │       │                                          │
  │       └── chama Serviço de Cache                 │
  │                │                                 │
  │                └── conecta ao Redis real         │
  └──────────────────────────────────────────────────┘

  Testa: os componentes se comunicam corretamente?
         o dado salvo no banco pode ser recuperado?
         o cache é invalidado após atualização?
```

**Características:**
- Mais lentos que unitários (envolvem I/O real)
- Revelam problemas de contrato entre componentes
- Precisam de infraestrutura (banco de testes, Redis de testes)
- Geralmente rodam em containers Docker descartáveis

**Quando usar:** para repositórios e acesso a dados, integrações entre serviços internos, fluxos que envolvem múltiplas camadas da aplicação.

---

### 3.3 Testes End-to-End (E2E)

Testam o **sistema inteiro**, da interface do usuário até o banco de dados, simulando exatamente o que um usuário real faria. Um robô controla o navegador.

```
Fluxo de um teste E2E:

  [Robô (Playwright/Cypress)]
         │
         │  1. Abre o navegador
         │  2. Navega para /login
         │  3. Preenche email e senha
         │  4. Clica em "Entrar"
         │  5. Verifica redirecionamento para /dashboard
         │  6. Clica em "Novo Pedido"
         │  7. Adiciona produto ao carrinho
         │  8. Finaliza compra
         │  9. Verifica e-mail de confirmação recebido
         ▼
  [Sistema completo rodando: Frontend + API + Banco + Redis + Fila]
```

**Características:**
- Os mais lentos (segundos a minutos por teste)
- Os mais caros de manter (qualquer mudança visual pode quebrar)
- Os mais próximos da realidade do usuário
- Frágeis: dependem de timing, ordem de elementos na tela, dados de seed

**Quando usar:** para os fluxos mais críticos do negócio (login, checkout, cadastro). Tenha poucos, mas estratégicos.

---

### 3.4 Outros Tipos Relevantes

**Testes de Contrato (Contract Tests):**
Em microsserviços, garantem que a "promessa" entre um serviço e seus consumidores não foi quebrada. Se o Serviço de Pagamentos muda o formato do JSON de resposta, o teste de contrato falha antes do deploy.

```
Serviço A (consumidor)  ←─ contrato ─→  Serviço B (provedor)

  Contrato define:
  - Endpoint: POST /pagamentos
  - Body esperado: { valor, moeda, cartao_token }
  - Resposta esperada: { id, status, timestamp }

  Se Serviço B mudar "status" para "estado":
  → Teste de contrato falha
  → Deploy bloqueado antes de chegar em produção
```

**Testes de Performance / Carga:**
Verificam como o sistema se comporta sob alto volume de requisições. Ferramentas como k6 ou Locust simulam milhares de usuários simultâneos.

```
k6 simulando carga:
  100 usuários simultâneos → sistema responde em < 200ms ✅
  500 usuários simultâneos → sistema responde em < 500ms ✅
  1000 usuários simultâneos → sistema responde em 3000ms ❌ → gargalo identificado
```

**Testes de Regressão:**
Não são um tipo separado, mas uma prática: toda vez que um bug é corrigido, escreve-se um teste que reproduz esse bug. Isso garante que o bug nunca volte.

---

## 4. Onde os Testes Entram na Pipeline de CI/CD

Os testes são o coração da CI/CD. Sem eles, o pipeline automatizado seria apenas um script de deploy — arriscado e cego. Com eles, cada mudança de código passa por uma **bateria de verificações automáticas** antes de chegar em produção.

```
PIPELINE COMPLETA DE CI/CD COM TESTES

  git push
      │
      ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  STAGE 1 — Validação Rápida (< 2 min)                      │
  │                                                             │
  │  ├─→ Lint e formatação (código segue os padrões?)          │
  │  ├─→ Type check (TypeScript/mypy — tipos estão corretos?)  │
  │  └─→ Análise estática (vulnerabilidades óbvias no código?) │
  └─────────────────────────────┬───────────────────────────────┘
                                │ ✅ passou
                                ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  STAGE 2 — Testes Unitários (< 5 min)                      │
  │                                                             │
  │  └─→ Roda toda a suíte de testes unitários                 │
  │       (centenas ou milhares de testes em segundos)         │
  └─────────────────────────────┬───────────────────────────────┘
                                │ ✅ passou
                                ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  STAGE 3 — Testes de Integração (5–15 min)                 │
  │                                                             │
  │  ├─→ Sobe containers de banco e Redis via Docker           │
  │  ├─→ Roda testes de integração contra infraestrutura real  │
  │  └─→ Derruba os containers ao terminar                     │
  └─────────────────────────────┬───────────────────────────────┘
                                │ ✅ passou
                                ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  STAGE 4 — Build e Publicação (3–10 min)                   │
  │                                                             │
  │  ├─→ Build da imagem Docker                                │
  │  └─→ Push para o registry (Docker Hub, ECR, GCR)          │
  └─────────────────────────────┬───────────────────────────────┘
                                │ ✅ passou
                                ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  STAGE 5 — Deploy em Staging (automático)                  │
  │                                                             │
  │  ├─→ Atualiza ambiente de staging                          │
  │  └─→ Roda testes E2E contra o ambiente de staging          │
  └─────────────────────────────┬───────────────────────────────┘
                                │ ✅ passou
                                ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  STAGE 6 — Deploy em Produção                              │
  │                                                             │
  │  ├─→ Continuous Delivery: aguarda aprovação manual         │
  │  └─→ Continuous Deployment: vai automaticamente            │
  └─────────────────────────────────────────────────────────────┘
```

**Por que os stages são ordenados assim?**

A lógica é **falhar rápido e barato**. Os testes mais rápidos e baratos rodam primeiro. Se o código nem compila ou tem erros óbvios de lint, não faz sentido subir containers de banco para rodar integração:

```
Custo de cada falha (em tempo de pipeline):

  Lint falha:           pipeline para em 30 segundos   → feedback imediato
  Unitário falha:       pipeline para em 3 minutos     → rápido
  Integração falha:     pipeline para em 10 minutos    → aceitável
  E2E em staging falha: pipeline para em 25 minutos    → lento, mas evita produção
  Produção falha:       bug chega ao usuário           → custo máximo
```

---

## 5. Implementação Passo a Passo

Vamos construir uma API de e-commerce em Python com FastAPI e implementar uma suíte completa de testes — unitários, de integração e a configuração do pipeline de CI. O projeto é simples o suficiente para focar nos testes, mas realista o suficiente para mostrar os desafios reais.

### Estrutura do Projeto

```
ecommerce-api/
├── .github/
│   └── workflows/
│       └── ci.yml                ← Pipeline de CI
├── app/
│   ├── __init__.py
│   ├── main.py                   ← Aplicação FastAPI
│   ├── models.py                 ← Modelos de dados
│   ├── database.py               ← Configuração do banco
│   └── routers/
│       ├── __init__.py
│       └── pedidos.py            ← Endpoints de pedidos
├── tests/
│   ├── __init__.py
│   ├── conftest.py               ← Fixtures compartilhadas
│   ├── unit/
│   │   ├── __init__.py
│   │   └── test_calculos.py      ← Testes unitários
│   └── integration/
│       ├── __init__.py
│       └── test_pedidos_api.py   ← Testes de integração
├── requirements.txt
└── docker-compose.test.yml       ← Infra para testes
```

---

### PASSO 1 — Dependências

**requirements.txt:**

```
# Aplicação
fastapi==0.111.0
uvicorn==0.29.0
sqlalchemy==2.0.30
psycopg2-binary==2.9.9
pydantic==2.7.1

# Testes
pytest==8.2.0
pytest-asyncio==0.23.6    # suporte a testes assíncronos
httpx==0.27.0              # cliente HTTP para testar a API
pytest-cov==5.0.0          # relatório de cobertura de código
faker==25.2.0              # geração de dados falsos realistas
```

---

### PASSO 2 — O Código da Aplicação

Primeiro, o código que vamos testar. Não é necessário entendê-lo linha por linha — o foco aqui é nos testes.

**app/models.py:**

```python
from pydantic import BaseModel, field_validator
from typing import Optional


class ItemPedido(BaseModel):
    produto_id: int
    quantidade: int
    preco_unitario: float

    @field_validator("quantidade")
    @classmethod
    def quantidade_deve_ser_positiva(cls, v):
        if v <= 0:
            raise ValueError("Quantidade deve ser maior que zero")
        return v

    @field_validator("preco_unitario")
    @classmethod
    def preco_deve_ser_positivo(cls, v):
        if v <= 0:
            raise ValueError("Preço deve ser maior que zero")
        return v


class CriarPedidoRequest(BaseModel):
    cliente_id: int
    itens: list[ItemPedido]
    cupom_desconto: Optional[str] = None


class PedidoResponse(BaseModel):
    id: int
    cliente_id: int
    subtotal: float
    desconto: float
    total: float
    status: str
```

**app/main.py — A lógica de negócio que mais importa testar:**

```python
from typing import Optional


def calcular_subtotal(itens: list[dict]) -> float:
    """Soma preco_unitario * quantidade de cada item."""
    return sum(item["preco_unitario"] * item["quantidade"] for item in itens)


def calcular_desconto(subtotal: float, cupom: Optional[str]) -> float:
    """
    Aplica desconto baseado no cupom.
    Retorna o valor do desconto (não o total final).
    """
    CUPONS = {
        "PROMO10": 10,   # 10% de desconto
        "PROMO20": 20,   # 20% de desconto
        "BEMVINDO": 15,  # 15% para novos clientes
    }

    if not cupom:
        return 0.0

    cupom_upper = cupom.upper()
    if cupom_upper not in CUPONS:
        raise ValueError(f"Cupom '{cupom}' inválido ou expirado")

    percentual = CUPONS[cupom_upper]
    return round(subtotal * percentual / 100, 2)


def calcular_total(subtotal: float, desconto: float) -> float:
    """Total = subtotal - desconto. Nunca negativo."""
    return max(0.0, round(subtotal - desconto, 2))
```

---

### PASSO 3 — Testes Unitários

Testes unitários focam na **lógica pura**, sem banco, sem rede, sem containers.

**tests/unit/test_calculos.py:**

```python
import pytest
from app.main import calcular_subtotal, calcular_desconto, calcular_total


# ──────────────────────────────────────────
#  Testes de calcular_subtotal
# ──────────────────────────────────────────

class TestCalcularSubtotal:
    """
    Agrupa todos os testes relacionados ao cálculo de subtotal.
    Classes são opcionais, mas organizam bem testes relacionados.
    """

    def test_item_unico(self):
        itens = [{"produto_id": 1, "quantidade": 2, "preco_unitario": 50.00}]
        assert calcular_subtotal(itens) == 100.00

    def test_multiplos_itens(self):
        itens = [
            {"produto_id": 1, "quantidade": 2, "preco_unitario": 50.00},   # 100.00
            {"produto_id": 2, "quantidade": 1, "preco_unitario": 30.00},   # 30.00
            {"produto_id": 3, "quantidade": 3, "preco_unitario": 10.00},   # 30.00
        ]
        assert calcular_subtotal(itens) == 160.00

    def test_lista_vazia(self):
        """Edge case: pedido sem itens deve retornar zero."""
        assert calcular_subtotal([]) == 0.0

    def test_preco_com_centavos(self):
        """Verifica precisão com valores decimais."""
        itens = [{"produto_id": 1, "quantidade": 3, "preco_unitario": 9.99}]
        assert calcular_subtotal(itens) == 29.97


# ──────────────────────────────────────────
#  Testes de calcular_desconto
# ──────────────────────────────────────────

class TestCalcularDesconto:

    def test_sem_cupom(self):
        """Sem cupom, desconto deve ser zero."""
        assert calcular_desconto(200.00, None) == 0.0

    def test_cupom_promo10(self):
        assert calcular_desconto(200.00, "PROMO10") == 20.00

    def test_cupom_promo20(self):
        assert calcular_desconto(200.00, "PROMO20") == 40.00

    def test_cupom_case_insensitive(self):
        """Cupons devem funcionar independente de maiúsculas/minúsculas."""
        assert calcular_desconto(200.00, "promo10") == 20.00
        assert calcular_desconto(200.00, "Promo10") == 20.00
        assert calcular_desconto(200.00, "PROMO10") == 20.00

    def test_cupom_invalido_levanta_excecao(self):
        """
        pytest.raises verifica que uma exceção foi lançada.
        Isso testa comportamento de erro, não só caminho feliz.
        """
        with pytest.raises(ValueError) as exc_info:
            calcular_desconto(200.00, "CUPOM_INEXISTENTE")

        # Verifica também a mensagem da exceção
        assert "inválido ou expirado" in str(exc_info.value)

    def test_desconto_arredondado(self):
        """Desconto deve ser arredondado para 2 casas decimais."""
        # 15% de 99.90 = 14.985 → deve arredondar para 14.99
        resultado = calcular_desconto(99.90, "BEMVINDO")
        assert resultado == 14.99


# ──────────────────────────────────────────
#  Testes parametrizados — evitam repetição
# ──────────────────────────────────────────

@pytest.mark.parametrize("subtotal, desconto, esperado", [
    (100.00, 10.00,  90.00),   # caso normal
    (100.00,  0.00, 100.00),   # sem desconto
    (100.00, 100.00,  0.00),   # desconto total
    (100.00, 150.00,  0.00),   # desconto maior que subtotal → nunca negativo
    ( 49.90,   5.00, 44.90),   # valores com centavos
])
def test_calcular_total(subtotal, desconto, esperado):
    """
    @pytest.mark.parametrize executa o mesmo teste com diferentes inputs.
    Ao invés de 5 funções separadas, temos uma só com 5 cenários.
    """
    assert calcular_total(subtotal, desconto) == esperado
```

**Rodando os testes unitários:**

```bash
# Roda todos os testes
pytest tests/unit/ -v

# -v (verbose): mostra o nome de cada teste e se passou ou falhou
# Saída esperada:
# tests/unit/test_calculos.py::TestCalcularSubtotal::test_item_unico PASSED
# tests/unit/test_calculos.py::TestCalcularSubtotal::test_multiplos_itens PASSED
# tests/unit/test_calculos.py::TestCalcularDesconto::test_cupom_invalido_levanta_excecao PASSED
# ...
# 12 passed in 0.08s  ← rapidíssimo
```

**Rodando com relatório de cobertura:**

```bash
pytest tests/unit/ -v --cov=app --cov-report=term-missing

# Saída do relatório de cobertura:
# Name                Stmts   Miss  Cover   Missing
# -------------------------------------------------
# app/main.py            18      0   100%
# app/models.py          14      2    86%   linha 12, 18
# -------------------------------------------------
# TOTAL                  32      2    94%

# "Missing" mostra quais linhas nunca foram executadas por nenhum teste.
# Isso aponta onde você ainda precisa de mais testes.
```

> **Cobertura de código (code coverage):** indica a porcentagem das linhas do seu código que foram executadas durante os testes. 100% de cobertura não significa que o código está correto — significa que todas as linhas foram executadas pelo menos uma vez. É uma métrica útil para encontrar código não testado, não uma garantia de qualidade.

---

### PASSO 4 — Testes de Integração

Agora testamos a API completa com banco de dados real. Precisamos de infraestrutura descartável.

**docker-compose.test.yml — banco só para testes:**

```yaml
version: "3.9"

services:
  postgres-test:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: teste
      POSTGRES_PASSWORD: teste
      POSTGRES_DB: ecommerce_test
    ports:
      - "5433:5432"   # porta diferente para não conflitar com banco de dev
    # sem volume → banco descartável, recriado a cada execução
    tmpfs:
      - /var/lib/postgresql/data   # dados em memória → ainda mais rápido
```

**tests/conftest.py — fixtures compartilhadas entre todos os testes:**

```python
"""
conftest.py é um arquivo especial do pytest.
Fixtures definidas aqui ficam disponíveis para todos os testes
sem precisar importar — o pytest as injeta automaticamente.
"""
import pytest
import pytest_asyncio
from httpx import AsyncClient, ASGITransport
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

from app.main import app
from app.database import Base, get_db

# Banco de dados de teste — separado completamente do banco de desenvolvimento
DATABASE_URL_TESTE = "postgresql://teste:teste@localhost:5433/ecommerce_test"

engine_teste = create_engine(DATABASE_URL_TESTE)
SessaoTeste = sessionmaker(bind=engine_teste)


@pytest.fixture(scope="session", autouse=True)
def criar_tabelas():
    """
    scope="session": roda UMA VEZ por sessão de testes (não a cada teste).
    autouse=True: aplicada automaticamente, sem precisar declarar nos testes.

    Cria todas as tabelas no banco de teste antes de qualquer teste rodar,
    e as destrói ao final da sessão.
    """
    Base.metadata.create_all(bind=engine_teste)
    yield                                          # testes rodam aqui
    Base.metadata.drop_all(bind=engine_teste)      # limpeza ao final


@pytest.fixture(autouse=True)
def limpar_banco():
    """
    scope padrão (function): roda antes E depois de CADA teste.
    Garante que cada teste começa com banco limpo — sem dados de testes anteriores.
    Isso torna os testes independentes entre si.
    """
    yield
    # Após cada teste: limpa todas as tabelas
    db = SessaoTeste()
    for tabela in reversed(Base.metadata.sorted_tables):
        db.execute(tabela.delete())
    db.commit()
    db.close()


@pytest_asyncio.fixture
async def client():
    """
    Cliente HTTP que faz requisições direto para o app FastAPI
    sem precisar subir um servidor real — mais rápido e controlado.
    """
    # Sobrescreve a dependência de banco para usar o banco de teste
    def override_get_db():
        db = SessaoTeste()
        try:
            yield db
        finally:
            db.close()

    app.dependency_overrides[get_db] = override_get_db

    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test"
    ) as ac:
        yield ac

    app.dependency_overrides.clear()
```

**tests/integration/test_pedidos_api.py:**

```python
import pytest
from faker import Faker

fake = Faker("pt_BR")  # gera dados falsos em português


@pytest.mark.asyncio
class TestCriarPedido:

    async def test_criar_pedido_simples(self, client):
        """
        Testa o caminho feliz: criar um pedido válido deve retornar 201
        com os dados calculados corretamente.
        """
        payload = {
            "cliente_id": 1,
            "itens": [
                {"produto_id": 10, "quantidade": 2, "preco_unitario": 49.90},
                {"produto_id": 20, "quantidade": 1, "preco_unitario": 99.00},
            ]
        }

        resposta = await client.post("/pedidos", json=payload)

        assert resposta.status_code == 201

        dados = resposta.json()
        assert dados["cliente_id"] == 1
        assert dados["subtotal"] == 198.80   # (2 * 49.90) + (1 * 99.00)
        assert dados["desconto"] == 0.0
        assert dados["total"] == 198.80
        assert dados["status"] == "pendente"
        assert "id" in dados                 # ID gerado pelo banco

    async def test_criar_pedido_com_cupom_valido(self, client):
        """Verifica que o desconto é aplicado corretamente no total."""
        payload = {
            "cliente_id": 2,
            "itens": [{"produto_id": 1, "quantidade": 1, "preco_unitario": 200.00}],
            "cupom_desconto": "PROMO10"
        }

        resposta = await client.post("/pedidos", json=payload)

        assert resposta.status_code == 201
        dados = resposta.json()
        assert dados["subtotal"] == 200.00
        assert dados["desconto"] == 20.00   # 10% de 200
        assert dados["total"] == 180.00

    async def test_cupom_invalido_retorna_400(self, client):
        """Cupom inexistente deve retornar 400 Bad Request, não 500."""
        payload = {
            "cliente_id": 1,
            "itens": [{"produto_id": 1, "quantidade": 1, "preco_unitario": 100.00}],
            "cupom_desconto": "CUPOM_FALSO"
        }

        resposta = await client.post("/pedidos", json=payload)

        assert resposta.status_code == 400
        assert "inválido" in resposta.json()["detail"].lower()

    async def test_quantidade_zero_retorna_422(self, client):
        """
        422 Unprocessable Entity: validação do Pydantic falhou.
        O dado chegou mas não passou na validação de esquema.
        """
        payload = {
            "cliente_id": 1,
            "itens": [{"produto_id": 1, "quantidade": 0, "preco_unitario": 100.00}]
        }

        resposta = await client.post("/pedidos", json=payload)

        assert resposta.status_code == 422

    async def test_pedido_persistido_no_banco(self, client):
        """
        Diferencial do teste de integração: verifica não só a resposta HTTP,
        mas também que os dados foram REALMENTE gravados no banco.
        """
        payload = {
            "cliente_id": 99,
            "itens": [{"produto_id": 5, "quantidade": 1, "preco_unitario": 150.00}]
        }

        # Cria o pedido
        resposta_criacao = await client.post("/pedidos", json=payload)
        assert resposta_criacao.status_code == 201
        pedido_id = resposta_criacao.json()["id"]

        # Busca o pedido criado (GET)
        resposta_busca = await client.get(f"/pedidos/{pedido_id}")
        assert resposta_busca.status_code == 200

        dados = resposta_busca.json()
        assert dados["id"] == pedido_id
        assert dados["cliente_id"] == 99
        assert dados["total"] == 150.00

    async def test_pedidos_sao_isolados_entre_testes(self, client):
        """
        Verifica que o banco está limpo — a fixture 'limpar_banco'
        do conftest garante que dados de outros testes não vazam aqui.
        """
        resposta = await client.get("/pedidos")
        assert resposta.status_code == 200
        assert resposta.json() == []   # banco deve estar vazio


@pytest.mark.asyncio
class TestListarPedidos:

    async def test_listar_pedidos_de_cliente(self, client):
        """Testa que a listagem retorna apenas pedidos do cliente correto."""
        # Cria pedidos para dois clientes diferentes
        for cliente_id in [1, 1, 2]:   # cliente 1 tem 2 pedidos, cliente 2 tem 1
            await client.post("/pedidos", json={
                "cliente_id": cliente_id,
                "itens": [{"produto_id": 1, "quantidade": 1, "preco_unitario": 10.00}]
            })

        # Busca pedidos do cliente 1
        resposta = await client.get("/pedidos?cliente_id=1")
        assert resposta.status_code == 200
        pedidos = resposta.json()
        assert len(pedidos) == 2
        assert all(p["cliente_id"] == 1 for p in pedidos)
```

**Rodando os testes de integração:**

```bash
# 1. Sobe o banco de testes
docker compose -f docker-compose.test.yml up -d

# 2. Aguarda o banco estar pronto (importante!)
sleep 3

# 3. Roda os testes de integração
pytest tests/integration/ -v

# Saída:
# tests/integration/test_pedidos_api.py::TestCriarPedido::test_criar_pedido_simples PASSED
# tests/integration/test_pedidos_api.py::TestCriarPedido::test_criar_pedido_com_cupom_valido PASSED
# tests/integration/test_pedidos_api.py::TestCriarPedido::test_cupom_invalido_retorna_400 PASSED
# ...
# 7 passed in 4.32s

# 4. Derruba o banco ao terminar
docker compose -f docker-compose.test.yml down
```

---

### PASSO 5 — Pipeline de CI com GitHub Actions

Agora automatizamos tudo. A cada `git push`, o GitHub executa todos os testes automaticamente.

**.github/workflows/ci.yml:**

```yaml
name: CI — Testes e Validação

# Quando este pipeline deve rodar:
on:
  push:
    branches: ["main", "develop"]     # push direto nessas branches
  pull_request:
    branches: ["main"]                # todo PR abrido para main

jobs:
  # ──────────────────────────────────────────
  #  JOB 1: Validação rápida (lint e tipos)
  # ──────────────────────────────────────────
  validacao:
    name: Lint e Type Check
    runs-on: ubuntu-latest

    steps:
      - name: Baixa o código
        uses: actions/checkout@v4

      - name: Configura Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: "pip"              # cacheia dependências entre execuções

      - name: Instala dependências de validação
        run: pip install ruff mypy

      - name: Ruff — lint e formatação
        # ruff é um linter extremamente rápido (substitui flake8 + isort + black)
        run: ruff check app/ tests/

      - name: Mypy — verificação de tipos
        run: mypy app/ --ignore-missing-imports


  # ──────────────────────────────────────────
  #  JOB 2: Testes unitários
  # ──────────────────────────────────────────
  testes-unitarios:
    name: Testes Unitários
    runs-on: ubuntu-latest
    needs: validacao              # só roda se validacao passar

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: "pip"

      - name: Instala dependências
        run: pip install -r requirements.txt

      - name: Roda testes unitários com cobertura
        run: |
          pytest tests/unit/ \
            -v \
            --cov=app \
            --cov-report=xml \
            --cov-fail-under=80     # falha se cobertura cair abaixo de 80%

      - name: Publica relatório de cobertura
        # Envia o relatório para o Codecov (serviço gratuito de visualização)
        uses: codecov/codecov-action@v4
        with:
          file: ./coverage.xml
          fail_ci_if_error: false   # não quebra o CI se o Codecov estiver fora


  # ──────────────────────────────────────────
  #  JOB 3: Testes de integração
  # ──────────────────────────────────────────
  testes-integracao:
    name: Testes de Integração
    runs-on: ubuntu-latest
    needs: testes-unitarios       # só roda se unitários passarem

    # GitHub Actions permite subir serviços (containers) junto ao job
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: teste
          POSTGRES_PASSWORD: teste
          POSTGRES_DB: ecommerce_test
        ports:
          - 5433:5432
        # Aguarda o banco estar realmente pronto antes de continuar
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: "pip"

      - name: Instala dependências
        run: pip install -r requirements.txt

      - name: Roda testes de integração
        env:
          DATABASE_URL: postgresql://teste:teste@localhost:5433/ecommerce_test
        run: pytest tests/integration/ -v

  # ──────────────────────────────────────────
  #  JOB 4: Build da imagem Docker
  #  (só roda se todos os testes passarem)
  # ──────────────────────────────────────────
  build:
    name: Build Docker
    runs-on: ubuntu-latest
    needs: [testes-unitarios, testes-integracao]

    steps:
      - uses: actions/checkout@v4

      - name: Build da imagem
        run: docker build -t ecommerce-api:${{ github.sha }} .

      - name: Verifica se a imagem sobe corretamente
        run: |
          docker run -d --name teste-container ecommerce-api:${{ github.sha }}
          sleep 3
          docker ps | grep teste-container   # falha se container não estiver rodando
          docker stop teste-container
```

**Como o pipeline aparece no GitHub:**

```
Commit: "feat: adiciona cupom BEMVINDO"

  ✅ Lint e Type Check          (22s)
       │
       ▼
  ✅ Testes Unitários           (1m 04s)  — cobertura: 94%
       │
       ▼
  ✅ Testes de Integração       (3m 17s)
       │
       ▼
  ✅ Build Docker               (2m 45s)

  Total: 7m 28s — Pronto para merge
```

```
Commit: "fix: corrige cálculo de desconto" (com bug)

  ✅ Lint e Type Check          (20s)
       │
       ▼
  ❌ Testes Unitários           (48s)
     FAILED tests/unit/test_calculos.py::TestCalcularDesconto::test_cupom_promo10
     AssertionError: 25.0 != 20.0
       │
       ✗ Testes de Integração  (não executou)
       ✗ Build Docker          (não executou)

  Merge bloqueado — PR não pode entrar em main
```

---

### PASSO 6 — Boas Práticas de Testes

**Cada teste deve ser independente:**

```python
# ✗ Ruim — Teste B depende de dados criados pelo Teste A
def test_A():
    criar_usuario("maria@email.com")

def test_B():
    usuario = buscar_usuario("maria@email.com")   # falha se A não rodou antes
    assert usuario is not None

# ✓ Bom — cada teste cria seus próprios dados
def test_B(client):
    criar_usuario("maria@email.com")               # cria o que precisa
    usuario = buscar_usuario("maria@email.com")
    assert usuario is not None
```

**Nomeie testes como documentação:**

```python
# ✗ Ruim — nome não diz o que está sendo testado
def test_desconto_1():
    ...

# ✓ Bom — nome descreve o comportamento esperado
def test_desconto_nao_pode_resultar_em_total_negativo():
    ...

def test_cupom_invalido_levanta_valor_error_com_mensagem_descritiva():
    ...
```

**Teste os casos de erro, não só o caminho feliz:**

```python
# Caminho feliz (obrigatório, mas insuficiente):
def test_criar_pedido_valido():
    ...

# Casos de erro (onde os bugs costumam se esconder):
def test_criar_pedido_sem_itens_retorna_400():
    ...

def test_criar_pedido_com_cliente_inexistente_retorna_404():
    ...

def test_criar_pedido_com_preco_negativo_retorna_422():
    ...
```

**Use o padrão AAA (Arrange, Act, Assert):**

```python
def test_cupom_promo20_aplicado_corretamente(self, client):
    # ARRANGE — prepara os dados
    payload = {
        "cliente_id": 1,
        "itens": [{"produto_id": 1, "quantidade": 1, "preco_unitario": 300.00}],
        "cupom_desconto": "PROMO20"
    }

    # ACT — executa a ação sendo testada
    resposta = await client.post("/pedidos", json=payload)

    # ASSERT — verifica o resultado
    assert resposta.status_code == 201
    assert resposta.json()["desconto"] == 60.00
    assert resposta.json()["total"] == 240.00
```

---

## Resumo

```
┌──────────────────────────────────────────────────────────────────────┐
│                   TESTES DE SOFTWARE — VISÃO GERAL                   │
├───────────────────────┬──────────────────────────────────────────────┤
│ O que são             │ Código que verifica se outro código funciona  │
│                       │ como esperado, de forma automática            │
├───────────────────────┼──────────────────────────────────────────────┤
│ Por que existem       │ Encontrar bugs cedo (quando corrigir é barato)│
│                       │ e dar confiança para mudar o código           │
├───────────────────────┼──────────────────────────────────────────────┤
│ Unitários             │ Testam funções isoladas. Rápidos. Muitos.     │
├───────────────────────┼──────────────────────────────────────────────┤
│ Integração            │ Testam componentes juntos. Banco real. Médios.│
├───────────────────────┼──────────────────────────────────────────────┤
│ End-to-End            │ Testam o sistema inteiro. Lentos. Poucos.     │
├───────────────────────┼──────────────────────────────────────────────┤
│ Na pipeline de CI/CD  │ Rodam automaticamente a cada push.            │
│                       │ Bloqueiam merge se falharem.                  │
│                       │ Unitários → Integração → E2E → Deploy.        │
├───────────────────────┼──────────────────────────────────────────────┤
│ Ferramentas (Python)  │ pytest, httpx, pytest-cov, Faker              │
├───────────────────────┼──────────────────────────────────────────────┤
│ Regra de ouro         │ Testes que nunca falham não protegem nada.    │
│                       │ Teste os casos de erro, não só o caminho feliz│
└───────────────────────┴──────────────────────────────────────────────┘
```

Testes não são burocracia nem perda de tempo — são a diferença entre um time que **tem medo de mudar o código** e um time que **muda com confiança**. Cada teste escrito é uma rede de segurança que permite evoluir o sistema sem medo de quebrar o que já funciona.