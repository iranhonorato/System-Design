# Qualidade e Segurança no Pipeline

**Pré-requisito:** [02_github_actions_na_pratica.md](02_github_actions_na_pratica.md)

> A seção sobre *fitness functions* resume o Capítulo 6 (*Measuring and Governing Architecture Characteristics*) de **Fundamentals of Software Architecture** (Mark Richards & Neal Ford). As ferramentas de segurança seguem suas documentações oficiais e o guia *OWASP DevSecOps*. Os ataques e vulnerabilidades citados são tratados em [Arquitetura de Software: seguranca.md](https://github.com/iranhonorato/Arquitetura-de-Software/blob/main/system-design-pt1/seguranca.md).

---

## Por que este arquivo existe

O [capítulo anterior](02_github_actions_na_pratica.md) terminou com um pipeline que funciona: lint, testes unitários, testes de integração e build da imagem, em menos de 3 minutos. Mas o [capítulo de fundamentos](01_fundamentos_de_integracao_continua.md) avisou: um pipeline verde significa apenas que nenhuma das verificações **existentes** encontrou problema.

Este capítulo trata de três perguntas que um pipeline maduro precisa responder:

1. **Os testes merecem confiança?** Cobertura, testes instáveis e a proporção certa entre as camadas.
2. **O código é seguro?** Vulnerabilidades no próprio código, nas dependências, na imagem, e segredos esquecidos.
3. **O pipeline continua rápido à medida que cresce?** Cada verificação nova custa tempo, e o tempo é um requisito.

---

## 1. Cobertura de testes: o que ela mede

O capítulo anterior mostrou como medir a cobertura com o pytest-cov. O número em si é fácil de obter. Difícil é saber o que ele significa.

**Cobertura de linhas** responde a uma única pergunta: *esta linha foi executada por algum teste?* Ela não diz se o teste **verificou** alguma coisa. Compare:

```python
def test_frete_executa():
    calcular_frete(2, 100)          # executa todas as linhas de calcular_frete...
                                    # ...e não verifica nada

def test_frete_correto():
    assert calcular_frete(2, 100) == 23.0
```

Os dois testes produzem **100% de cobertura** de `calcular_frete`. Só o segundo encontraria um erro na fórmula.

Duas variações dão uma imagem mais fiel:

- **Cobertura de ramos** (`--cov-branch`): verifica se **cada caminho** de um `if` foi percorrido, e não só se a linha rodou. Um `if peso <= 0 or distancia < 0` com só o caso válido testado tem 100% de linhas e 50% de ramos.
- **Cobertura das linhas alteradas** (*diff coverage*): mede só as linhas que o PR mudou. Ferramentas como `diff-cover` permitem a regra "todo código novo precisa de pelo menos 80% de cobertura", sem exigir que a equipe escreva testes para todo o código antigo de uma vez.

```yaml
      - run: pytest --cov=app --cov-branch --cov-report=xml
      - run: |
          pip install diff-cover
          git fetch origin main
          diff-cover coverage.xml --compare-branch=origin/main --fail-under=80
```

> **Confusão comum:** "precisamos chegar a 100% de cobertura para ter um código confiável". ✅ **Mais preciso:** a cobertura é ótima para mostrar o que **não** é testado (0% em um módulo é um aviso claro), e fraca para provar que algo **é** bem testado, como o exemplo acima mostrou. Uma meta alta e rígida costuma produzir testes que executam o código sem verificar nada, só para subir o número: quando uma métrica vira meta, ela deixa de medir o que media. Na prática, um patamar mínimo razoável para o projeto (70–85% é comum), cobertura de ramos e a exigência de cobertura nas **linhas alteradas** dão muito mais resultado do que perseguir 100%. Para saber se os testes realmente detectam erros, existe o **teste de mutação** (ferramentas como `mutmut`): ele altera o código de propósito (troca um `+` por `-`, um `<=` por `<`) e verifica se algum teste falha. Se nenhum falhar, aquele trecho está coberto, mas não testado.

---

## 2. Testes instáveis (*flaky tests*)

Um **teste instável** é aquele que às vezes passa e às vezes falha, **sem que o código mude**. É um dos problemas mais corrosivos de um pipeline, porque ensina a equipe a ignorar o vermelho.

### As causas mais comuns

| Causa | Exemplo | Correção |
|---|---|---|
| **Dependência de tempo** | `assert pedido.criado_em.day == hoje.day`, rodando à meia-noite | Injetar o relógio ou usar uma biblioteca de congelar o tempo (`freezegun`) |
| **Ordem entre testes** | Um teste deixa dados no banco, e o seguinte depende de eles **não** estarem lá | Cada teste prepara e limpa o próprio estado (como a fixture `tabela_limpa` do capítulo anterior) |
| **Espera fixa** | `time.sleep(2)` esperando um serviço subir | Esperar por uma **condição** (tentar até responder), com tempo máximo |
| **Rede externa** | O teste chama a API real dos Correios | Mock no unitário; serviço local (container) no de integração |
| **Concorrência** | Duas threads disputando um recurso, e o resultado depende de quem chega primeiro | Tornar a ordem determinística no teste, ou testar a sincronização explicitamente |
| **Aleatoriedade** | Dados gerados aleatoriamente que às vezes caem em um caso não tratado | Fixar a semente; ou, melhor, transformar o caso encontrado em um teste fixo |

### Como lidar

1. **Detectar:** rodar a suíte várias vezes (`pytest --count=20`, com o plugin `pytest-repeat`) revela os instáveis. Muitas plataformas de CI marcam automaticamente os testes que falham e passam na mesma execução.
2. **Isolar:** um teste comprovadamente instável vai para **quarentena** (por exemplo, uma marca `@pytest.mark.quarentena`, excluída do pipeline principal), com uma tarefa aberta para corrigi-lo. Assim, ele não bloqueia a equipe, e também não some.
3. **Corrigir a causa**, usando a tabela acima.

> **Confusão comum:** "o teste falha de vez em quando, então configuramos o pipeline para tentar de novo até três vezes". ✅ **Mais preciso:** a nova tentativa automática **esconde** o sintoma e preserva a causa. Às vezes a instabilidade está no teste, e a nova tentativa só atrasa o feedback. Mas às vezes ela revela um defeito **real** na aplicação: uma condição de corrida, um timeout curto demais, um recurso que não é liberado. Esse defeito vai acontecer também em produção, com usuários de verdade, só que sem uma nova tentativa automática para salvá-los. Novas tentativas fazem sentido para falhas de **infraestrutura** do pipeline (a VM perdeu a rede, o registry ficou fora do ar por um instante), e não para asserções de teste. Teste instável é um defeito com uma tarefa aberta, não uma configuração.

---

## 3. Análise estática além do lint

### 3.1 Checagem de tipos

Com anotações de tipo, o **mypy** encontra uma classe inteira de erros antes de o código rodar:

```python
def calcular_frete(peso_kg: float, distancia_km: float) -> float: ...

calcular_frete("2", 100)
# error: Argument 1 to "calcular_frete" has incompatible type "str"; expected "float"
```

```yaml
      - run: pip install mypy && mypy app/
```

### 3.2 Levando as verificações para antes do push: pre-commit

O caminho mais rápido de feedback não é o pipeline: é a própria máquina de quem desenvolve. O **pre-commit** executa verificações a cada `git commit` e bloqueia o commit se alguma falhar:

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.17.2
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.21.2
    hooks:
      - id: gitleaks          # bloqueia commits com segredos (seção 4.3)
```

```bash
pip install pre-commit
pre-commit install            # instala o hook no repositório local
```

O pre-commit **não substitui** o pipeline (ele pode ser pulado com `git commit --no-verify`, e nem todos o instalam), mas corta o ciclo "push, esperar, ver o erro, corrigir, push de novo" para os erros mais simples.

Essa ideia de mover as verificações para o mais cedo possível no ciclo de desenvolvimento tem um nome: ***shift left*** (deslocar para a esquerda, pensando em uma linha do tempo que vai da escrita do código, à esquerda, até a produção, à direita). Ela retoma diretamente a curva de custo do [capítulo de fundamentos](01_fundamentos_de_integracao_continua.md): quanto mais à esquerda o problema é encontrado, mais barato ele é.

---

## 4. Segurança no pipeline

Aplicar o *shift left* à segurança é o que se costuma chamar de **DevSecOps**: em vez de uma auditoria de segurança antes do lançamento, verificações automáticas em cada PR. São quatro frentes, cada uma com seu tipo de ferramenta:

```
┌──────────────────────────────────────────────────────────────────────┐
│  O QUE ESTÁ SENDO VERIFICADO           TIPO DE VERIFICAÇÃO           │
│                                                                      │
│  ① o código que você escreveu    ───▶  SAST (análise estática)       │
│  ② as bibliotecas que você usa   ───▶  SCA (análise de dependências) │
│  ③ segredos no repositório       ───▶  secret scanning               │
│  ④ a imagem que vai rodar        ───▶  scan de imagem                │
└──────────────────────────────────────────────────────────────────────┘
```

### 4.1 SAST: falhas no seu código

**SAST** (*Static Application Security Testing*) procura padrões perigosos no código-fonte: SQL montado com concatenação de strings, `eval` sobre dados do usuário, desserialização insegura, senha fixa no código.

```python
# O tipo de código que o SAST aponta:
conn.execute(f"SELECT * FROM pedidos WHERE produto = '{produto}'")   # SQL Injection
```

```yaml
      - run: pip install bandit && bandit -r app/ -ll     # -ll: só severidade média ou alta
```

Para Python, o **Bandit** é a opção mais simples. O **Semgrep** e o **CodeQL** (gratuito para repositórios públicos no GitHub) cobrem várias linguagens e analisam o fluxo dos dados: eles conseguem dizer "este valor veio da requisição HTTP e chegou ao SQL sem tratamento".

Repare que o código da API do capítulo anterior usa `%s` com parâmetros separados (`conn.execute("... VALUES (%s, %s)", (produto, quantidade))`), e não uma f-string. É a forma correta, e é justamente o que o SAST verifica.

### 4.2 SCA: vulnerabilidades nas dependências

A maior parte do código que vai para produção **não foi escrita pela equipe**: são as bibliotecas e as dependências delas. **SCA** (*Software Composition Analysis*) compara as versões instaladas com bancos públicos de vulnerabilidades conhecidas (CVEs):

```yaml
      - run: pip install pip-audit && pip-audit -r requirements.txt
```

```
Found 1 known vulnerability in 1 package
Name     Version ID                  Fix Versions
-------- ------- ------------------- ------------
exemplo  1.2.0   GHSA-xxxx-xxxx-xxxx 1.2.4
```

Complementando a verificação no pipeline, o **Dependabot** (embutido no GitHub) abre PRs automaticamente quando sai uma correção de segurança para uma dependência, e também quando sai uma versão nova de uma action (o problema da seção 9.3 do capítulo anterior):

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: pip
    directory: /
    schedule:
      interval: weekly
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
  - package-ecosystem: docker          # a imagem base do Dockerfile
    directory: /
    schedule:
      interval: weekly
```

Cada PR do Dependabot passa pelo próprio pipeline de CI, e é aqui que uma boa suíte de testes paga o investimento: com testes confiáveis, atualizar uma dependência vira um merge de rotina, e não um projeto com medo de quebrar tudo.

### 4.3 Secret scanning: senhas e chaves no código

O [capítulo de IAM](../../01-infraestrutura/aws/03_iam_identidades_e_permissoes.md) mostrou o que acontece com uma access key da AWS em um repositório: robôs a encontram em minutos. A defesa tem três camadas:

1. **No commit:** o hook do **gitleaks** no pre-commit (seção 3.2) bloqueia o commit localmente.
2. **No push:** o **push protection** do GitHub recusa pushes que contenham padrões de segredos conhecidos (chaves da AWS, tokens do GitHub, do Stripe, do Slack...).
3. **No pipeline:** o gitleaks também roda sobre o histórico, pegando o que escapou das duas primeiras:

```yaml
  segredos:
    name: Busca de segredos
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0                  # o histórico completo, não só o último commit
      - uses: gitleaks/gitleaks-action@v3
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

(Em repositórios de **organizações**, essa action pede uma chave de licença gratuita do projeto gitleaks. A alternativa é instalar o binário do gitleaks no job e rodar `gitleaks detect` diretamente.)

Se um segredo chegar ao repositório, **remover o arquivo não resolve**: ele continua no histórico. O procedimento correto é sempre o mesmo: **revogar** o segredo imediatamente (gerar uma chave nova e desativar a antiga), e só depois, se fizer sentido, limpar o histórico.

### 4.4 Scan da imagem

A imagem Docker contém um sistema operacional mínimo e seus pacotes (a `python:3.12-slim` tem uma base Debian). Esses pacotes também acumulam vulnerabilidades. O **Trivy** verifica a imagem construída:

```yaml
      - uses: docker/build-push-action@v7
        with:
          context: .
          load: true                      # deixa a imagem disponível no Docker local do runner
          tags: minha-api:${{ github.sha }}

      - uses: aquasecurity/trivy-action@v0.36.0
        with:
          image-ref: minha-api:${{ github.sha }}
          severity: CRITICAL,HIGH
          ignore-unfixed: true            # ignora o que ainda não tem correção disponível
          exit-code: "1"                  # falha o job se encontrar algo
```

Este scan é o que torna concretas as recomendações do [capítulo de Compose e produção](../../02-containers/docker/04_compose_e_producao.md): uma imagem `-slim` e multi-stage tem **muito** menos pacotes, e portanto muito menos achados, do que uma imagem completa com compiladores.

O Trivy também gera um **SBOM** (*Software Bill of Materials*): a lista de tudo o que existe dentro da imagem, com versões. Quando uma vulnerabilidade nova é anunciada, o SBOM permite responder em minutos "quais das nossas imagens usam essa biblioteca?".

> **Confusão comum:** "o scanner encontrou 140 vulnerabilidades na imagem, então ela não pode ir para produção". ✅ **Mais preciso:** a lista bruta de um scanner mistura situações muito diferentes. Muitas vulnerabilidades estão em pacotes que a aplicação nunca executa, muitas não têm correção disponível e outras exigem condições que não existem no seu ambiente. Um portão que bloqueia por **qualquer** achado acaba desligado em uma semana, porque nada passa. Um portão útil é **calibrado**: bloqueia vulnerabilidades críticas e altas **com correção disponível** (as opções `severity` e `ignore-unfixed` acima), e as exceções conscientes ficam registradas (no Trivy, em um arquivo `.trivyignore`, com o motivo). A forma mais eficaz de reduzir os achados não é filtrá-los, e sim encolher a imagem: base `-slim` ou *distroless*, multi-stage e atualizações frequentes da imagem base.

---

## 5. Fitness functions: testando a arquitetura

Richards e Ford dedicam o Capítulo 6 de *Fundamentals of Software Architecture* a um problema que nenhum teste unitário resolve: como garantir que as **características da arquitetura** (modularidade, desempenho, escalabilidade) continuem sendo respeitadas conforme o código cresce? Os autores observam que a busca por automação que criou a integração contínua acabou levando à **governança automatizada da arquitetura**, e usam o termo **fitness function**, emprestado da computação evolutiva, para os mecanismos dessa governança:

> **Fitness function de arquitetura:** qualquer mecanismo que forneça uma avaliação objetiva da integridade de uma característica da arquitetura (ou de uma combinação delas).

O ponto central é o "qualquer mecanismo": fitness functions não são uma ferramenta nova, e sim um novo **uso** de ferramentas que já existem (testes, métricas, monitores), voltado para a arquitetura e não para a lógica de negócio. E o lugar natural para elas é o pipeline, porque ali elas rodam a cada mudança, sem depender de alguém lembrar de verificar.

### Exemplo: proteger as camadas da aplicação

O exemplo clássico do livro é detectar **dependências cíclicas** entre componentes, e regras de camadas, que as IDEs ajudam a quebrar ao oferecer "importar automaticamente". Em Python, o **import-linter** transforma uma regra de arquitetura em uma verificação do pipeline:

```ini
# .importlinter
[importlinter]
root_package = app

[importlinter:contract:camadas]
name = A API depende do domínio, e nunca o contrário
type = layers
layers =
    app.main
    app.frete
```

```yaml
      - run: pip install import-linter && lint-imports
```

Se alguém fizer `app/frete.py` importar algo de `app/main.py`, a regra de negócio passa a depender da camada HTTP, e o pipeline falha com uma mensagem que diz qual regra de arquitetura foi violada. Uma decisão de design deixa de ser um conselho em um documento e vira uma verificação automática.

### Outros exemplos de fitness functions no pipeline

| Característica | Fitness function |
|---|---|
| **Modularidade** | Regras de dependência entre pacotes (import-linter, ArchUnit em Java) |
| **Desempenho** | Um teste de carga curto (k6, Locust) que falha se o p95 de `/frete` passar de 200 ms |
| **Tamanho da imagem** | Falhar se a imagem passar de 250 MB (`docker image inspect` + uma comparação no script) |
| **Segurança** | O scan de imagem da seção 4.4, com um limite de severidade |
| **Compatibilidade** | Testes de contrato que verificam se a API continua compatível com os clientes atuais |

---

## 6. Mantendo o pipeline rápido

Cada seção deste capítulo acrescentou verificações, e cada uma custa tempo. Sem cuidado, o pipeline de 3 minutos vira um de 20, e o [capítulo de fundamentos](01_fundamentos_de_integracao_continua.md) mostrou o que isso faz com a prática da equipe. Algumas técnicas:

**Ordem certa e paralelismo.** Verificações independentes rodam em jobs paralelos (lint, unitários, SAST, busca de segredos); as caras (integração, build, scan de imagem) vêm depois, só se as baratas passarem.

**Cache em tudo o que se repete.** Dependências (`cache: pip`), camadas da imagem (`cache-from: type=gha`), bancos de vulnerabilidades do Trivy.

**Rodar só o que a mudança afeta.** Um PR que só altera a documentação não precisa rodar os testes de integração:

```yaml
on:
  pull_request:
    paths-ignore:
      - "**.md"
      - "docs/**"
```

(Cuidado: um check obrigatório que não roda fica "esperando" para sempre na regra de proteção. Para filtros por caminho em checks obrigatórios, prefira um job que sempre roda e decide internamente o que executar.)

**Dividir a suíte de testes.** Com centenas de testes lentos, o `pytest-xdist` distribui os testes entre os núcleos da máquina (`pytest -n auto`), e uma matriz pode dividir a suíte entre várias máquinas.

**Mover o que é lento para fora do PR.** Testes E2E completos, testes de carga longos e scans profundos podem rodar na `main` depois do merge, ou em um agendamento noturno, desde que alguém seja avisado quando falham:

```yaml
on:
  schedule:
    - cron: "0 3 * * *"        # todo dia às 03:00 UTC
```

**Medir.** A página de cada execução mostra quanto cada job levou. Olhar esses números de tempos em tempos, e atacar o job mais lento, é o que mantém o pipeline no limite de 10 minutos.

---

## 7. Resumo: os portões de qualidade

Juntando os três capítulos de CI, um pipeline maduro tem estes **portões** (*quality gates*):

| Portão | Ferramenta (Python) | Quando roda | Bloqueia o merge? |
|---|---|---|---|
| Lint e formatação | Ruff | Pre-commit e todo PR | Sim |
| Checagem de tipos | mypy | Todo PR | Sim |
| Testes unitários + cobertura mínima | pytest, pytest-cov | Todo PR | Sim |
| Cobertura das linhas alteradas | diff-cover | Todo PR | Sim (ou só aviso, no começo) |
| Testes de integração | pytest + service container | Todo PR | Sim |
| Regras de arquitetura | import-linter | Todo PR | Sim |
| SAST | Bandit, Semgrep, CodeQL | Todo PR | Sim, para severidade alta |
| Dependências vulneráveis | pip-audit, Dependabot | Todo PR e semanalmente | Sim, para severidade alta com correção |
| Segredos | gitleaks, push protection | Commit, push e PR | Sim |
| Build da imagem | Docker Buildx | Todo PR | Sim |
| Scan da imagem | Trivy | Todo PR | Sim, para crítico/alto com correção |
| E2E e carga | Playwright, k6 | Na `main` ou agendado | Não bloqueia o PR; abre um alerta |

Não é preciso implantar tudo de uma vez. A ordem que costuma dar mais retorno é: testes confiáveis, lint, busca de segredos e dependências vulneráveis, depois o scan da imagem, e por fim as fitness functions específicas da sua arquitetura.

---

## Próximo passo

A CI está completa: cada mudança é verificada em várias dimensões, e o resultado é uma imagem com uma tag única que carrega todas essas garantias. Mas essa imagem ainda não foi a lugar nenhum. Ela foi construída e descartada no fim do job.

O bloco de CD começa no [../cd/01_entrega_e_implantacao_continua.md](../cd/01_entrega_e_implantacao_continua.md), que explica como levar esse artefato até produção, a diferença entre entrega contínua e implantação contínua, e por que a imagem testada no CI deve ser exatamente a mesma que roda em produção.
