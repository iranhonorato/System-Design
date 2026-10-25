# Jenkinsfile: Pipelines como Código

**Pré-requisito:** [01_fundamentos_do_jenkins.md](01_fundamentos_do_jenkins.md), [../ci/02_github_actions_na_pratica.md](../ci/02_github_actions_na_pratica.md)

> Segue a documentação oficial do Jenkins (*Pipeline Syntax*, *Using a Jenkinsfile*, *Using Docker with Pipeline*, *Extending with Shared Libraries*) e o aviso de segurança sobre interpolação de strings do *Credentials Binding*.

---

## Por que este arquivo existe

O capítulo anterior mostrou que o Jenkins tem dois jeitos de definir uma automação: formulários na interface (Freestyle) ou código (Pipeline), e explicou por que o segundo é o único que escala. Este capítulo é a referência desse código: o **Jenkinsfile**.

A pergunta central é: *como descrever, em um arquivo versionado no repositório, tudo o que o pipeline faz: onde roda, com quais ferramentas, com quais segredos, em que condições, em que ordem, e o que acontece quando dá certo ou errado?* Cada seção apresenta uma parte da linguagem com um exemplo curto, e a seção 14 mostra a correspondência com o GitHub Actions, para quem veio dos blocos de CI e CD. O [próximo capítulo](03_jenkins_na_pratica.md) junta tudo no pipeline completo da loja.

---

## 1. Declarativo e Scripted

O Jenkinsfile existe em duas sintaxes, ambas baseadas em **Groovy** (uma linguagem da JVM, parecida com Java, mas mais concisa):

```groovy
// DECLARATIVO: uma estrutura fixa, com blocos que têm significado definido
pipeline {
    agent any
    stages {
        stage('Testar') {
            steps {
                sh 'pytest'
            }
        }
    }
}
```

```groovy
// SCRIPTED: código Groovy livre, com chamadas a funções do Jenkins
node {
    stage('Testar') {
        checkout scm
        sh 'pytest'
    }
}
```

| | Declarativo | Scripted |
|---|---|---|
| **Estrutura** | Fixa: `pipeline { agent / stages / post }` | Livre: qualquer código Groovy |
| **Validação** | O Jenkins verifica a estrutura antes de executar | Erros aparecem durante a execução |
| **Legibilidade** | Alta, mesmo para quem não sabe Groovy | Depende de quem escreveu |
| **Recursos prontos** | `when`, `post`, `options`, `parameters`, `input`... | Precisam ser programados |
| **Quando usar** | **Quase sempre** | Lógica muito dinâmica, ou dentro de Shared Libraries (seção 13) |

Este capítulo usa o **declarativo**, que é a recomendação atual da documentação. Quando for preciso um trecho de lógica livre dentro dele, existe o bloco `script { }` (seção 4).

---

## 2. A estrutura de um Jenkinsfile declarativo

```groovy
pipeline {                              // ← obrigatório: todo pipeline declarativo começa aqui

    agent { label 'docker' }            // ← ONDE roda (seção 3)

    options { ... }                     // ← comportamento do pipeline: timeout, histórico (seção 6)
    parameters { ... }                  // ← entradas escolhidas ao iniciar um build (seção 7)
    triggers { ... }                    // ← o que dispara automaticamente (seção 7)
    environment { ... }                 // ← variáveis para todas as etapas (seção 5)

    stages {                            // ← obrigatório: a lista de etapas
        stage('Nome da etapa') {
            when { ... }                //   condição para a etapa rodar (seção 8)
            agent { ... }               //   agente próprio da etapa (opcional)
            environment { ... }         //   variáveis só desta etapa
            steps {                     //   os passos: comandos e chamadas
                sh '...'
            }
            post { ... }                //   ações ao fim desta etapa
        }
    }

    post { ... }                        // ← ações ao fim do pipeline inteiro (seção 11)
}
```

A ordem dos blocos de topo é fixa, e o Jenkins valida a estrutura **antes** de executar qualquer etapa. Um bloco no lugar errado gera um erro imediato, com a linha:

```
WorkflowScript: 5: Expected a stage @ line 5, column 9.
```

O arquivo fica na raiz do repositório, com o nome `Jenkinsfile` (sem extensão).

---

## 3. `agent`: onde o pipeline roda

O `agent` diz em qual máquina (ou em qual container) as etapas são executadas. Ele pode ser declarado no topo (vale para todas as etapas) ou em cada etapa.

```groovy
agent any                               // qualquer agente com executor livre
agent none                              // nenhum no topo: cada etapa declara o seu
agent { label 'linux && docker' }       // um agente com as duas labels
```

### Dentro de um container

Com o plugin *Docker Pipeline* e o Docker disponível no agente, as etapas podem rodar **dentro de um container**, e as ferramentas do build vêm da imagem, não do agente:

```groovy
pipeline {
    agent {
        docker {
            image 'python:3.12-slim'
            args  '-v pip-cache:/root/.cache/pip'   // opções extras do "docker run"
        }
    }
    stages {
        stage('Testar') {
            steps {
                sh 'pip install -r requirements-dev.txt && pytest'
            }
        }
    }
}
```

O Jenkins baixa a imagem, inicia um container com o **workspace montado dentro dele**, executa os passos ali e remove o container no fim. É o mesmo princípio do [capítulo de imagens](../../02-containers/docker/02_imagens_e_dockerfile.md): o ambiente faz parte do código. Um agente só precisa ter Docker; Python, Node ou Java vêm de cada pipeline. O volume `pip-cache` reaproveita os downloads entre builds, o equivalente ao `cache: pip` do GitHub Actions.

Também é possível construir o ambiente a partir de um Dockerfile do próprio repositório (`agent { dockerfile true }`), ou pedir um **Pod** no Kubernetes (`agent { kubernetes { yaml '...' } }`, no [capítulo prático](03_jenkins_na_pratica.md)).

### Agentes diferentes por etapa

```groovy
pipeline {
    agent none
    stages {
        stage('Testes') {
            agent { docker { image 'python:3.12-slim' } }
            steps { sh 'pytest' }
        }
        stage('Build da imagem') {
            agent { label 'docker' }          // precisa de um Docker de verdade para "docker build"
            steps { sh 'docker build -t minha-api:${GIT_COMMIT} .' }
        }
    }
}
```

Cada etapa pode cair em um agente **diferente**, com um workspace diferente. Arquivos gerados em uma etapa não aparecem automaticamente na outra: para isso existem `stash` e `unstash` (seção 12).

---

## 4. `steps`: o que cada etapa faz

Os passos mais usados:

```groovy
steps {
    echo 'Mensagem no log'

    sh 'pytest -v'                         // um comando de shell (Linux/Mac)
    bat 'dotnet test'                      // um comando no Windows

    sh '''
        set -e
        pip install -r requirements-dev.txt
        ruff check .
        pytest -m "not integration"
    '''                                    // várias linhas: aspas triplas

    dir('frontend') {                      // executa dentro de um subdiretório
        sh 'npm ci && npm test'
    }

    retry(3) {                             // tenta até 3 vezes (para falhas de rede, não de testes)
        sh 'curl -fsS https://registry.exemplo.com/health'
    }

    timeout(time: 5, unit: 'MINUTES') {    // falha se passar de 5 minutos
        sh './testes-lentos.sh'
    }
}
```

Como no GitHub Actions, um `sh` que termina com código diferente de zero **falha** a etapa e interrompe o pipeline. Para capturar a saída de um comando em uma variável, use `returnStdout`, dentro de um bloco `script`:

```groovy
steps {
    script {
        def versao = sh(script: 'git describe --tags --always', returnStdout: true).trim()
        echo "Versão: ${versao}"
        if (env.BRANCH_NAME == 'main') {
            currentBuild.description = "release ${versao}"
        }
    }
}
```

O bloco `script { }` é a "saída de emergência" do declarativo: dentro dele vale o Groovy livre do modo Scripted. Use com moderação: se o Jenkinsfile tiver mais `script` do que estrutura, a lógica provavelmente deveria estar em um script do repositório (`./scripts/deploy.sh`) ou em uma Shared Library.

---

## 5. Variáveis de ambiente e credenciais

### 5.1 Variáveis prontas

O Jenkins define várias variáveis em todo build:

| Variável | Exemplo | Significado |
|---|---|---|
| `BUILD_NUMBER` | `42` | O número do build |
| `BUILD_URL` | `http://jenkins:8080/job/minha-api/job/main/42/` | O link do build |
| `JOB_NAME` | `minha-api/main` | O nome do job |
| `WORKSPACE` | `/home/jenkins/workspace/minha-api_main` | O diretório de trabalho |
| `BRANCH_NAME` | `main`, `PR-142` | A branch (em Multibranch Pipelines) |
| `CHANGE_ID` | `142` | O número do PR (só em builds de PR) |
| `GIT_COMMIT` | `a1b2c3d4...` | O hash do commit (com o plugin Git) |

### 5.2 Variáveis próprias

```groovy
environment {
    REGISTRY = 'ghcr.io/seu-usuario'
    IMAGEM   = "${REGISTRY}/minha-api:${GIT_COMMIT}"
}
```

### 5.3 Credenciais

O [capítulo anterior](01_fundamentos_do_jenkins.md) mostrou como cadastrar credenciais. No Jenkinsfile, elas são pedidas **pelo ID**, de duas formas.

**No `environment`, com `credentials()`:**

```groovy
environment {
    SONAR_TOKEN = credentials('sonar-token')          // "Secret text" → uma variável
    REGISTRY    = credentials('registry-login')       // "Username with password" → TRÊS variáveis:
                                                      //   REGISTRY      = "usuario:senha"
                                                      //   REGISTRY_USR  = "usuario"
                                                      //   REGISTRY_PSW  = "senha"
}
```

**Em um trecho específico, com `withCredentials`** (o valor só existe dentro do bloco):

```groovy
steps {
    withCredentials([usernamePassword(credentialsId: 'registry-login',
                                      usernameVariable: 'REG_USER',
                                      passwordVariable: 'REG_PASS')]) {
        sh 'echo "$REG_PASS" | docker login ghcr.io -u "$REG_USER" --password-stdin'
    }

    withCredentials([file(credentialsId: 'kubeconfig-producao', variable: 'KUBECONFIG')]) {
        sh 'kubectl get pods -n loja'                  // o kubectl lê o caminho em $KUBECONFIG
    }

    sshagent(credentials: ['ssh-deploy-producao']) {   // plugin SSH Agent
        sh 'ssh deploy@10.0.11.20 "cd /opt/minha-loja && docker compose up -d"'
    }
}
```

O Jenkins substitui os valores das credenciais por `****` no log. Mas essa proteção pode ser anulada sem querer por um detalhe de sintaxe do Groovy.

> **Confusão comum:** "tanto faz usar aspas simples ou duplas no `sh`; o Jenkins mascara a senha no log de qualquer jeito". ✅ **Mais preciso:** a diferença decide **quem** substitui a variável, e isso muda tudo. Com **aspas duplas**, `sh "docker login -p ${REG_PASS}"`, quem substitui é o **Groovy**, antes de o comando existir: o Jenkins monta a string com a senha já dentro e a entrega ao shell. A senha passa a fazer parte do **texto do comando**, que pode aparecer na lista de processos do agente (`ps aux`) e em outros lugares que o mascaramento não cobre. Com **aspas simples**, `sh 'docker login -p "$REG_PASS"'`, o Groovy não toca na string: quem lê `$REG_PASS` é o **shell**, como uma variável de ambiente, no momento da execução. O próprio Jenkins avisa no log quando detecta o caso errado: *"A secret was passed to "sh" using Groovy String interpolation, which is insecure"*. A regra: segredos sempre em aspas **simples**, lidos como `$VARIAVEL` pelo shell. Aspas duplas só para valores não sensíveis que precisam do Groovy, como `"${IMAGEM}"`. E prefira `--password-stdin` a `-p`, para a senha não aparecer na linha de comando de jeito nenhum.

---

## 6. `options`: o comportamento do pipeline

```groovy
options {
    timeout(time: 30, unit: 'MINUTES')                // mata o build se passar de 30 min
    buildDiscarder(logRotator(numToKeepStr: '30'))    // guarda só os últimos 30 builds
    disableConcurrentBuilds()                         // nunca dois builds deste job ao mesmo tempo
    timestamps()                                      // horário em cada linha do log (plugin Timestamper)
    skipDefaultCheckout()                             // não faz checkout automático (você faz quando quiser)
}
```

O `buildDiscarder` merece atenção: sem ele, o Jenkins guarda **todos** os builds, com logs e artefatos, no `JENKINS_HOME`, e o disco do controller enche aos poucos. O `disableConcurrentBuilds` tem o papel do `concurrency` do GitHub Actions, e é essencial em jobs de deploy.

---

## 7. `parameters` e `triggers`

### 7.1 Parâmetros: entradas ao iniciar um build

```groovy
parameters {
    choice(name: 'AMBIENTE', choices: ['homologacao', 'producao'], description: 'Destino do deploy')
    string(name: 'VERSAO', defaultValue: '', description: 'Tag da imagem (vazio = o commit atual)')
    booleanParam(name: 'RODAR_E2E', defaultValue: false, description: 'Executar os testes E2E?')
}

stages {
    stage('E2E') {
        when { expression { params.RODAR_E2E } }
        steps { sh './e2e.sh' }
    }
    stage('Deploy') {
        steps { sh "./deploy.sh ${params.AMBIENTE}" }
    }
}
```

O botão **Build Now** vira **Build with Parameters**, com um formulário. Os parâmetros são lidos com `params.NOME`. Um detalhe que confunde: os parâmetros declarados no Jenkinsfile só aparecem na interface **depois da primeira execução**, porque é ela que lê o arquivo e registra a definição.

### 7.2 Gatilhos

```groovy
triggers {
    cron('H 3 * * *')                 // todo dia, em algum minuto da hora 3
    pollSCM('H/5 * * * *')            // a cada ~5 min, verifica se há commits novos
}
```

O **`H`** (de *hash*) é uma particularidade do Jenkins: em vez de um minuto fixo, cada job recebe um minuto calculado a partir do próprio nome. Com `0 3 * * *` em 200 jobs, todos começariam às 03:00 em ponto e sobrecarregariam os agentes; com `H 3 * * *`, eles se espalham ao longo da hora.

O `pollSCM` (perguntar ao repositório periodicamente se há novidades) é o mecanismo antigo. Em um **Multibranch Pipeline** ligado ao GitHub, os **webhooks** avisam o Jenkins a cada push e PR, e não é preciso declarar `triggers` para isso. É o que o [capítulo prático](03_jenkins_na_pratica.md) configura.

---

## 8. `when`: condições para uma etapa

```groovy
stage('Publicar imagem') {
    when { branch 'main' }                         // só na main
    steps { ... }
}

stage('Comentar no PR') {
    when { changeRequest() }                       // só em builds de pull request
    steps { ... }
}

stage('Deploy em produção') {
    when {
        allOf {                                    // todas as condições
            branch 'main'
            not { changeRequest() }
            expression { currentBuild.currentResult == 'SUCCESS' }
        }
    }
    steps { ... }
}

stage('Release') {
    when { tag pattern: 'v\\d+\\.\\d+\\.\\d+', comparator: 'REGEXP' }   // só em tags v1.2.3
    steps { ... }
}
```

Por padrão, o Jenkins reserva o agente da etapa **antes** de avaliar o `when`. Se a condição for falsa, um executor terá sido ocupado (e um container, criado) à toa. Com `beforeAgent true` dentro do `when`, a condição é avaliada primeiro:

```groovy
when {
    beforeAgent true
    branch 'main'
}
```

---

## 9. Paralelismo e matriz

### 9.1 `parallel`

Etapas independentes podem rodar ao mesmo tempo, cada uma em seu agente, como os jobs paralelos do GitHub Actions:

```groovy
stage('Verificações') {
    parallel {
        stage('Lint') {
            agent { docker { image 'python:3.12-slim' } }
            steps { sh 'pip install ruff && ruff check . && ruff format --check .' }
        }
        stage('Testes unitários') {
            agent { docker { image 'python:3.12-slim' } }
            steps { sh 'pip install -r requirements-dev.txt && pytest -m "not integration"' }
        }
        stage('Busca de segredos') {
            agent { docker { image 'zricethezav/gitleaks:latest' } }
            steps { sh 'gitleaks detect --source . --no-banner' }
        }
    }
}
```

Para que uma falha em um ramo interrompa os outros imediatamente (o `fail-fast` do GitHub Actions), acrescente `failFast true` à etapa que contém o `parallel`.

### 9.2 `matrix`

```groovy
stage('Testes por versão') {
    matrix {
        axes {
            axis {
                name 'PYTHON'
                values '3.12', '3.13', '3.14'
            }
        }
        agent { docker { image "python:${PYTHON}-slim" } }
        stages {
            stage('Testar') {
                steps { sh 'pip install -r requirements-dev.txt && pytest -m "not integration"' }
            }
        }
    }
}
```

---

## 10. `input`: aprovação manual

O `input` pausa o pipeline e espera uma pessoa decidir. É o equivalente aos *required reviewers* dos environments do [pipeline de CD](../cd/03_pipeline_de_cd_na_pratica.md):

```groovy
stage('Aprovação para produção') {
    agent none                                      // ← não ocupa executor enquanto espera
    steps {
        timeout(time: 1, unit: 'DAYS') {            // sem resposta em 1 dia, o build é abortado
            input message: 'Implantar esta versão em produção?',
                  ok: 'Implantar',
                  submitter: 'equipe-loja'          // só quem está neste grupo pode aprovar
        }
    }
}
```

Na página do build aparece a pergunta, com os botões **Implantar** e **Abort**.

> **Confusão comum:** "basta colocar o `input` entre as etapas; o pipeline espera e depois continua". ✅ **Mais preciso:** o pipeline espera, mas **onde** ele espera importa. Se o `input` estiver em uma etapa que roda em um agente (por exemplo, com `agent any` no topo do pipeline), o build fica **segurando um executor** durante toda a espera, que pode durar horas ou dias. Com alguns builds aguardando aprovação, os agentes ficam todos ocupados sem fazer nada, e os builds novos de outras equipes entram na fila. A solução é fazer o `input` em uma etapa com **`agent none`**, que roda no controller sem ocupar executor, e ter um **`timeout`** em volta, para que um pedido esquecido não fique aberto para sempre. Com `agent none` no topo e um agente declarado em cada etapa que realmente executa algo, o problema não aparece.

---

## 11. `post`: o que fazer no fim

O bloco `post` executa passos de acordo com o **resultado**, ao fim de uma etapa ou do pipeline inteiro:

```groovy
post {
    always {                                     // sempre
        junit allowEmptyResults: true, testResults: 'relatorios/*.xml'
        archiveArtifacts artifacts: 'relatorios/**', allowEmptyArchive: true
    }
    success {                                    // terminou verde
        echo "Imagem publicada: ${IMAGEM}"
    }
    unstable {                                   // terminou, mas com testes falhando
        echo 'Há testes falhando: veja a aba Test Results.'
    }
    failure {                                    // falhou
        echo "Build quebrado: ${BUILD_URL}"      // ou uma notificação para o chat da equipe
    }
    fixed {                                      // o anterior falhou e este passou
        echo 'Build consertado.'
    }
    cleanup {                                    // por último, depois de todos os outros
        cleanWs()                                // apaga o workspace (plugin Workspace Cleanup)
    }
}
```

| Condição | Roda quando |
|---|---|
| `always` | Sempre |
| `success` / `failure` / `unstable` / `aborted` | O resultado final foi esse |
| `changed` | O resultado é diferente do build anterior |
| `fixed` | O anterior falhou ou ficou instável, e este passou |
| `regression` | O anterior passou, e este não |
| `cleanup` | Sempre, depois de todas as outras condições |

### Testes que falham: FAILURE ou UNSTABLE?

O passo `junit` lê relatórios no formato JUnit XML (o `--junitxml` do pytest) e mostra, na interface, cada teste, sua duração e a tendência ao longo dos builds. Se encontrar falhas, ele marca o build como **UNSTABLE**. Mas há um detalhe: se o `pytest` falha, o `sh` termina com erro e o build já vira **FAILURE** antes do `junit` rodar. Para separar "não conseguiu testar" (FAILURE) de "testou e há falhas" (UNSTABLE), uma forma comum é:

```groovy
steps {
    sh 'pytest --junitxml=relatorios/unitarios.xml || true'   // não interrompe por falha de teste
}
post {
    always { junit 'relatorios/unitarios.xml' }               // o junit decide: UNSTABLE se houver falhas
}
```

Cuidado: com o `|| true`, um erro que impeça os testes de rodar (um import quebrado, por exemplo) também deixa de falhar o `sh`. O `junit` só detecta isso se o relatório não for gerado (por isso o `allowEmptyResults` do exemplo anterior deve ficar `false` em etapas de teste). Muitas equipes preferem não usar o `|| true` e tratar qualquer falha de teste como FAILURE, por ser mais simples e mais seguro.

---

## 12. `stash` e `unstash`: arquivos entre agentes

Quando etapas rodam em agentes diferentes (seção 3), o workspace de uma não existe na outra. O `stash` guarda arquivos no controller, e o `unstash` os recupera em outro agente:

```groovy
stage('Build do frontend') {
    agent { docker { image 'node:20-alpine' } }
    steps {
        sh 'npm ci && npm run build'
        stash name: 'frontend', includes: 'dist/**'
    }
}
stage('Imagem') {
    agent { label 'docker' }
    steps {
        unstash 'frontend'                       // dist/ aparece neste workspace
        sh 'docker build -t loja-web:${GIT_COMMIT} .'
    }
}
```

É o equivalente ao par `upload-artifact`/`download-artifact` do GitHub Actions. Use para arquivos pequenos e médios: o `stash` passa pelo controller, e arquivos de vários gigabytes o sobrecarregam (para esses, um registry ou um bucket é o lugar certo).

---

## 13. Shared Libraries: reaproveitando entre repositórios

Com dezenas de repositórios Python, todos os Jenkinsfiles acabam iguais. Uma **Shared Library** é um repositório Git com código Groovy reutilizável, carregado pelos Jenkinsfiles:

```
jenkins-shared-library/            (um repositório próprio)
└── vars/
    └── pipelinePython.groovy      ← cada arquivo em vars/ vira um "passo" disponível
```

```groovy
// vars/pipelinePython.groovy
def call(Map config = [:]) {
    def versao = config.get('python', '3.12')
    pipeline {
        agent { docker { image "python:${versao}-slim" } }
        options { timeout(time: 20, unit: 'MINUTES') }
        stages {
            stage('Dependências') { steps { sh 'pip install -r requirements-dev.txt' } }
            stage('Lint')         { steps { sh 'ruff check . && ruff format --check .' } }
            stage('Testes')       { steps { sh 'pytest -m "not integration" --junitxml=relatorio.xml' } }
        }
        post { always { junit 'relatorio.xml' } }
    }
}
```

E o Jenkinsfile de cada repositório encolhe para:

```groovy
@Library('loja-pipelines') _       // o nome com que a biblioteca foi registrada no Jenkins
pipelinePython(python: '3.12')
```

A biblioteca é registrada uma vez em **Manage Jenkins → System → Global Trusted Pipeline Libraries**. Uma melhoria no pipeline padrão (um scan de segurança novo, por exemplo) passa a valer para todos os repositórios com um commit na biblioteca. É a mesma ideia dos workflows reutilizáveis do [pipeline de CD](../cd/03_pipeline_de_cd_na_pratica.md). O custo é o de qualquer dependência compartilhada: uma mudança errada na biblioteca quebra todos os pipelines ao mesmo tempo, então ela merece testes e versões (`@Library('loja-pipelines@v2') _`).

---

## 14. Da linguagem do GitHub Actions para a do Jenkinsfile

| GitHub Actions | Jenkinsfile declarativo |
|---|---|
| Arquivo em `.github/workflows/ci.yml` | `Jenkinsfile` na raiz |
| `on: push / pull_request` | Multibranch Pipeline + webhook (sem código) |
| `on: schedule: - cron: ...` | `triggers { cron('H 3 * * *') }` |
| `on: workflow_dispatch` com `inputs` | `parameters { ... }` + *Build with Parameters* |
| `jobs.<id>` | `stage('...')` |
| `runs-on: ubuntu-latest` | `agent { label '...' }` |
| `container: python:3.12` | `agent { docker { image 'python:3.12-slim' } }` |
| `steps: - run: ...` | `steps { sh '...' }` |
| `uses: alguma/action@v1` | Um passo de plugin (`junit`, `sshagent`...) ou um passo de Shared Library |
| `needs: [a, b]` | A ordem das `stages` (sequencial por padrão) |
| Jobs sem `needs` (paralelos) | `parallel { ... }` |
| `strategy: matrix:` | `matrix { axes { ... } }` |
| `if: github.ref == 'refs/heads/main'` | `when { branch 'main' }` |
| `env:` | `environment { ... }` |
| `${{ secrets.X }}` | `credentials('x')` / `withCredentials(...)` |
| `environment:` com *required reviewers* | `input` com `submitter` |
| `concurrency:` | `options { disableConcurrentBuilds() }` |
| `timeout-minutes:` | `options { timeout(...) }` |
| `if: always()` / `if: failure()` | `post { always { } }` / `post { failure { } }` |
| `actions/upload-artifact` | `archiveArtifacts` / `stash` |
| Workflow reutilizável (`workflow_call`) | Shared Library |

---

## 15. Ferramentas que ajudam a escrever

Na página de qualquer job do tipo Pipeline, o link **Pipeline Syntax** abre três ferramentas:

- **Snippet Generator:** escolha um passo (por exemplo, `withCredentials`), preencha um formulário e copie o código gerado. É a forma mais rápida de acertar a sintaxe de passos de plugins.
- **Declarative Directive Generator:** o mesmo, para blocos do declarativo (`agent`, `when`, `options`, `post`).
- **Global Variables Reference:** a lista de variáveis disponíveis (`env`, `params`, `currentBuild`).

Na página de um build já executado, o **Replay** permite editar o Jenkinsfile **só para aquela reexecução**, sem commit: útil para experimentar uma correção rapidamente. Depois, a correção precisa ir para o repositório, senão se perde.

Para validar a sintaxe de um Jenkinsfile antes do commit, o Jenkins oferece um endpoint:

```bash
curl -s -X POST -u "seu-usuario:SEU_TOKEN_DE_API" \
  -F "jenkinsfile=<Jenkinsfile" \
  http://localhost:8080/pipeline-model-converter/validate
# Jenkinsfile successfully validated.
```

---

## Próximo passo

Este capítulo apresentou a linguagem peça por peça. O [03_jenkins_na_pratica.md](03_jenkins_na_pratica.md) a usa para montar o pipeline completo da loja: o Jenkins configurado como código (plugins e configuração em arquivos versionados), um Multibranch Pipeline ligado ao GitHub por webhook, agentes efêmeros em containers e no Kubernetes, CI com testes de integração e CD com aprovação, e as práticas de segurança e operação de um Jenkins em produção.
