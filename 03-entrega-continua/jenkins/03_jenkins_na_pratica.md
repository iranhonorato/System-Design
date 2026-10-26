# Jenkins na Prática: o CI/CD Completo da Loja

**Pré-requisito:** [02_jenkinsfile_e_pipelines.md](02_jenkinsfile_e_pipelines.md), [../cd/03_pipeline_de_cd_na_pratica.md](../cd/03_pipeline_de_cd_na_pratica.md). A seção 6 assume a parte de Kubernetes.

> Segue a documentação oficial do Jenkins (*Configuration as Code*, *Managing Plugins*, *Using Docker with Pipeline*, *Securing Jenkins*) e dos plugins *GitHub Branch Source*, *Job DSL* e *Kubernetes*. Os nomes e as chaves de configuração dos plugins mudam entre versões: quando um exemplo não funcionar exatamente como está, a documentação embutida do seu Jenkins (citada em cada seção) mostra a sintaxe da versão instalada.

---

## Por que este arquivo existe

Os dois capítulos anteriores apresentaram as peças: controller e agentes, tipos de job, plugins, credenciais e a linguagem do Jenkinsfile. Este capítulo monta, com elas, o mesmo resultado que os blocos de CI e CD construíram no GitHub Actions: lint, testes unitários e de integração, build e publicação da imagem, deploy em homologação, aprovação e deploy em produção.

A pergunta central é: *como montar um Jenkins que possa ser recriado do zero a partir do Git, que execute os builds em agentes isolados, que seja disparado automaticamente pelo GitHub e que seja seguro o bastante para guardar as credenciais de produção?* O capítulo segue esta ordem:

| Seção | O que constrói |
|---|---|
| 1 | O Jenkins **como código**: imagem, plugins e configuração em arquivos |
| 2 | Um **agente** com Docker, separado do controller |
| 3 | O job **Multibranch** ligado ao GitHub, com webhook e status nos PRs |
| 4 | O **Jenkinsfile** completo da loja |
| 5 | Executando e acompanhando o pipeline |
| 6 | **Agentes no Kubernetes**: um Pod novo por build |
| 7 | **Segurança e operação** de um Jenkins em produção |
| 8 | Diagnóstico e comparação final entre as ferramentas |

---

## 1. O Jenkins como código

O [capítulo de fundamentos](01_fundamentos_do_jenkins.md) instalou o Jenkins pela interface: assistente, plugins sugeridos, usuário criado à mão. Funciona, mas ninguém sabe exatamente o que foi configurado, e recriar o servidor significa repetir tudo de memória. É o mesmo problema que a Infraestrutura como Código resolveu para os servidores, no [capítulo de virtualização](../../01-infraestrutura/fundamentos/02_virtualizacao_containers_e_iac.md).

O Jenkins pode ser descrito por três arquivos versionados:

```
jenkins-loja/                      (um repositório só para a infraestrutura do Jenkins)
├── Dockerfile                     ← a imagem do controller, com os plugins
├── plugins.txt                    ← a lista de plugins
├── casc.yaml                      ← a configuração do Jenkins (Configuration as Code)
├── agente/
│   └── Dockerfile                 ← a imagem do agente (seção 2)
├── docker-compose.yml             ← controller + agente
└── .env                           ← senhas e tokens (NÃO vai para o Git)
```

### 1.1 Os plugins: `plugins.txt`

```text
# plugins.txt: um plugin por linha (nome-do-plugin ou nome-do-plugin:versão)
configuration-as-code
job-dsl
workflow-aggregator
git
github-branch-source
credentials-binding
docker-workflow
ssh-agent
junit
timestamper
ws-cleanup
lockable-resources
pipeline-graph-view
matrix-auth
kubernetes
```

Para reprodutibilidade total, fixe as versões (`git:5.6.0`). O comando `jenkins-plugin-cli --list` dentro de um Jenkins existente mostra as instaladas.

### 1.2 A imagem do controller

```dockerfile
# Dockerfile
FROM jenkins/jenkins:lts-jdk21

# Sem o assistente de configuração: o casc.yaml configura tudo
ENV JAVA_OPTS="-Djenkins.install.runSetupWizard=false"

# Onde o plugin Configuration as Code procura a configuração
ENV CASC_JENKINS_CONFIG=/var/jenkins_conf/casc.yaml

# Os plugins são baixados no BUILD da imagem, não na inicialização
COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN jenkins-plugin-cli --plugin-file /usr/share/jenkins/ref/plugins.txt

COPY casc.yaml /var/jenkins_conf/casc.yaml
```

Instalar os plugins no build da imagem segue a lógica do [capítulo de imagens](../../02-containers/docker/02_imagens_e_dockerfile.md): a imagem é um artefato imutável e testável. Atualizar um plugin vira uma mudança no `plugins.txt`, revisada em PR, que gera uma imagem nova, que pode ser testada antes de substituir a de produção.

### 1.3 A configuração: `casc.yaml`

O plugin **Configuration as Code** (JCasC) lê um YAML e aplica a configuração do Jenkins na inicialização: segurança, credenciais, agentes, URLs, e até os jobs (com o plugin Job DSL).

```yaml
# casc.yaml
jenkins:
  systemMessage: "Jenkins da loja: configurado como código. Mudanças pela interface serão perdidas."
  numExecutors: 0                        # nenhum build no controller (fundamentos do Jenkins, seção 2.3)
  slaveAgentPort: 50000                  # a porta dos agentes de entrada
  securityRealm:
    local:
      allowsSignup: false
      users:
        - id: "admin"
          password: "${JENKINS_ADMIN_PASSWORD}"   # lida de uma variável de ambiente
  authorizationStrategy:
    globalMatrix:
      entries:
        - user:
            name: "admin"
            permissions:
              - "Overall/Administer"
        - group:
            name: "authenticated"
            permissions:
              - "Overall/Read"
              - "Job/Read"
              - "Job/Build"
  nodes:
    - permanent:
        name: "agente-docker"
        labelString: "docker linux"
        numExecutors: 2
        remoteFS: "/home/jenkins/agent"
        launcher:
          inbound: {}                    # o agente se conecta ao controller (em versões antigas: "jnlp")

credentials:
  system:
    domainCredentials:
      - credentials:
          - usernamePassword:
              scope: GLOBAL
              id: "registry-login"
              username: "${REGISTRY_USER}"
              password: "${REGISTRY_TOKEN}"
          - usernamePassword:
              scope: GLOBAL
              id: "github-app-ou-token"
              username: "${GITHUB_USER}"
              password: "${GITHUB_TOKEN}"
          - file:                         # "Secret file": um kubeconfig por ambiente (seção 4)
              scope: GLOBAL
              id: "kubeconfig-homologacao"
              fileName: "kubeconfig"
              secretBytes: "${readFileBase64:/run/secrets/kubeconfig-homologacao}"
          - file:
              scope: GLOBAL
              id: "kubeconfig-producao"
              fileName: "kubeconfig"
              secretBytes: "${readFileBase64:/run/secrets/kubeconfig-producao}"

unclassified:
  location:
    url: "http://localhost:8080/"

jobs:
  - script: >
      multibranchPipelineJob('minha-api') {
        branchSources {
          branchSource {
            source {
              github {
                id('minha-api')
                repoOwner('seu-usuario')
                repository('minha-api')
                repositoryUrl('https://github.com/seu-usuario/minha-api')
                configuredByUrl(true)
                credentialsId('github-app-ou-token')
              }
            }
          }
        }
        orphanedItemStrategy {
          discardOldItems { numToKeep(10) }
        }
      }
```

Três observações:

- **`${JENKINS_ADMIN_PASSWORD}`** e as demais variáveis são lidas do **ambiente do container**, que as recebe do `.env`. Para arquivos inteiros, como os kubeconfigs, o `${readFileBase64:...}` lê um arquivo montado no container (por exemplo, como *secrets* do Compose em `/run/secrets/`). O `casc.yaml` vai para o Git; os valores, não. É a mesma separação entre configuração e segredos do [primeiro capítulo de CD](../cd/01_entrega_e_implantacao_continua.md).
- **A mensagem do sistema** avisa que mudanças pela interface serão perdidas, e é verdade: a cada reinício, o JCasC reaplica o arquivo. A interface vira um lugar para **ver**, e o Git, o lugar para **mudar**.
- **O bloco `jobs`** cria o job Multibranch da seção 3 com o plugin **Job DSL**. A sintaxe dos blocos de cada plugin está documentada no próprio Jenkins, em `http://localhost:8080/plugin/job-dsl/api-viewer/index.html`. Para a sintaxe do JCasC, a página **Manage Jenkins → Configuration as Code → View Configuration** exporta a configuração atual em YAML, e é a melhor forma de descobrir a chave certa de qualquer opção: configure uma vez pela interface, exporte e copie o trecho.

---

## 2. Um agente com Docker

### 2.1 A imagem do agente

O agente parte da imagem oficial `jenkins/inbound-agent` e acrescenta as ferramentas que os pipelines da loja usam: o cliente Docker e o `kubectl`.

```dockerfile
# agente/Dockerfile
FROM jenkins/inbound-agent:latest-jdk21

USER root
RUN apt-get update \
 && apt-get install -y --no-install-recommends ca-certificates curl gnupg \
 && install -m 0755 -d /etc/apt/keyrings \
 && curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc \
 && echo "deb [signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
      > /etc/apt/sources.list.d/docker.list \
 && apt-get update \
 && apt-get install -y --no-install-recommends docker-ce-cli docker-buildx-plugin \
 && curl -fsSLo /usr/local/bin/kubectl \
      "https://dl.k8s.io/release/$(curl -fsSL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" \
 && chmod +x /usr/local/bin/kubectl \
 && rm -rf /var/lib/apt/lists/*
USER jenkins
```

Só o **cliente** Docker é instalado. O **daemon** que vai executar os containers é o da máquina hospedeira, alcançado pelo socket montado na seção 2.2.

### 2.2 Controller e agente no Compose

```yaml
# docker-compose.yml
services:
  jenkins:
    build: .
    ports:
      - "8080:8080"
    environment:
      JENKINS_ADMIN_PASSWORD: ${JENKINS_ADMIN_PASSWORD}
      REGISTRY_USER: ${REGISTRY_USER}
      REGISTRY_TOKEN: ${REGISTRY_TOKEN}
      GITHUB_USER: ${GITHUB_USER}
      GITHUB_TOKEN: ${GITHUB_TOKEN}
    volumes:
      - jenkins_home:/var/jenkins_home
    restart: unless-stopped

  agente:
    build: ./agente
    environment:
      JENKINS_URL: http://jenkins:8080          # o nome do serviço, pela rede do Compose
      JENKINS_AGENT_NAME: agente-docker         # o nome definido no casc.yaml
      JENKINS_SECRET: ${JENKINS_AGENT_SECRET}   # gerado pelo controller (veja abaixo)
      JENKINS_AGENT_WORKDIR: /home/jenkins/agent
    volumes:
      - agente_work:/home/jenkins/agent
      - /var/run/docker.sock:/var/run/docker.sock
    group_add:
      - "${DOCKER_GID}"                         # o grupo dono do docker.sock no host
    depends_on:
      - jenkins
    restart: unless-stopped

volumes:
  jenkins_home:
  agente_work:
```

Repare que a porta 50000 **não** está publicada: o agente está na mesma rede do Compose e alcança o controller pelo nome `jenkins`, como a API alcançava o `banco` no [capítulo de volumes e redes](../../02-containers/docker/03_volumes_e_redes.md). Só a interface web (8080) precisa ser acessível de fora.

### 2.3 Subindo

```bash
# 1. Preencha o .env (fora do Git)
cat > .env <<'EOF'
JENKINS_ADMIN_PASSWORD=troque-esta-senha
REGISTRY_USER=seu-usuario
REGISTRY_TOKEN=ghp_token-com-write-packages
GITHUB_USER=seu-usuario
GITHUB_TOKEN=ghp_token-com-leitura-do-repositorio
JENKINS_AGENT_SECRET=
EOF
echo "DOCKER_GID=$(stat -c %g /var/run/docker.sock)" >> .env

# 2. Sobe só o controller
docker compose up -d --build jenkins
```

O controller precisa existir antes do agente, porque é ele que gera o **segredo** com que o agente se identifica. Em `http://localhost:8080/computer/agente-docker/`, a página do agente mostra um comando de conexão com um valor longo depois de `-secret`. Copie esse valor para `JENKINS_AGENT_SECRET` no `.env` e suba o agente:

```bash
docker compose up -d --build agente
docker compose logs -f agente
# INFO: Connected
```

Na página **Manage Jenkins → Nodes**, o `agente-docker` aparece conectado, com 2 executores. O nó embutido (*Built-In Node*) aparece com 0.

> **Confusão comum:** "montar o `/var/run/docker.sock` no agente é a forma normal de ter Docker no Jenkins". ✅ **Mais preciso:** é a forma **mais simples**, e tem um custo conhecido. Quem tem acesso ao socket do Docker controla o daemon do host, e o [capítulo de arquitetura do Docker](../../02-containers/docker/01_arquitetura_e_isolamento.md) mostrou que isso equivale a ter **root na máquina**. Na prática, qualquer pipeline que rode nesse agente, inclusive o de um pull request, pode criar um container que monte o disco inteiro do host. Para um laboratório na sua máquina, é aceitável. Em produção, as alternativas são: agentes **efêmeros no Kubernetes** (seção 6), em que cada build roda em um Pod descartável; builds de imagem **sem daemon privilegiado** (BuildKit em modo *rootless*, Buildah); ou, no mínimo, agentes dedicados por nível de confiança, de forma que os builds de PRs nunca rodem na mesma máquina que guarda acesso a produção.

---

## 3. O job Multibranch ligado ao GitHub

### 3.1 O que o job faz

O job `minha-api`, criado pelo bloco `jobs` do `casc.yaml`, é um **Multibranch Pipeline** (o tipo descrito no [capítulo de fundamentos](01_fundamentos_do_jenkins.md), seção 5.3). Ele:

1. lista as branches e os pull requests do repositório no GitHub;
2. cria um sub-job para cada um que tenha um `Jenkinsfile`;
3. dispara um build a cada push e a cada atualização de PR;
4. publica o resultado de volta no GitHub, como um **check** no commit e no PR.

### 3.2 A credencial do GitHub

O Jenkins precisa ler o repositório e escrever o status dos checks. Há duas opções:

| Opção | Como | Observação |
|---|---|---|
| **Token pessoal** (*fine-grained*) | Um token com acesso só ao repositório, permissão de leitura do conteúdo e de escrita em *commit statuses* | Simples para estudo; fica ligado à conta de uma pessoa |
| **GitHub App** | Um aplicativo instalado na organização, com permissões próprias | Recomendado para equipes: não depende de ninguém, tem limites de requisição maiores e tokens de curta duração |

O exemplo usa o token, na credencial `github-app-ou-token` do `casc.yaml`.

### 3.3 O webhook

Para que o GitHub **avise** o Jenkins a cada push, configure um webhook no repositório, em **Settings → Webhooks → Add webhook**:

```
Payload URL:   https://jenkins.exemplo.com/github-webhook/
Content type:  application/json
Events:        "Let me select individual events" → Pushes, Pull requests
```

O endereço precisa ser alcançável **pelo GitHub**, ou seja, pela internet. Um Jenkins em `localhost` não é. Para estudo, há duas saídas: um túnel temporário (como `smee.io` ou `ngrok`, que repassam os webhooks para a sua máquina) ou, sem webhook nenhum, configurar o job para **verificar o repositório periodicamente** (*Scan Multibranch Pipeline Triggers → Periodically if not otherwise run*, a cada poucos minutos). Em produção, o Jenkins fica atrás de um load balancer ou proxy reverso com HTTPS, e o webhook aponta para ele.

### 3.4 O status no pull request

Com o plugin *GitHub Branch Source*, cada build reporta seu resultado ao GitHub. No PR aparece um check como:

```
✔ continuous-integration/jenkins/pr-merge — This commit looks good
```

Esse check pode ser marcado como **obrigatório** na regra de proteção da `main` (a mesma do [capítulo de GitHub Actions](../ci/02_github_actions_na_pratica.md), seção 4). A partir daí, o Jenkins passa a ser o portão do merge, exatamente como o GitHub Actions era. O sufixo `pr-merge` indica que o Jenkins testou o **resultado do merge** do PR com a `main`, e não só a branch isolada, que é a combinação que de fato vai entrar.

---

## 4. O Jenkinsfile da loja

O projeto é o mesmo do [capítulo de GitHub Actions](../ci/02_github_actions_na_pratica.md): a API FastAPI com testes unitários e de integração, o Dockerfile multi-stage e os manifestos Kubernetes do [capítulo do Compose ao cluster](../../02-containers/kubernetes/04_do_compose_ao_cluster.md). O `Jenkinsfile` fica na raiz:

```groovy
// Jenkinsfile
pipeline {
    agent none                                   // cada etapa escolhe o seu (e a aprovação não ocupa executor)

    options {
        timeout(time: 60, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '30'))
        timestamps()
    }

    environment {
        REGISTRY = 'ghcr.io/seu-usuario'
    }

    stages {

        // ───────────────────────────── CI: em todo PR e na main ─────────────────────────────
        stage('Verificações') {
            failFast true
            parallel {
                stage('Lint') {
                    agent {
                        docker {
                            image 'python:3.12-slim'
                            label 'docker'
                        }
                    }
                    steps {
                        sh 'pip install --quiet ruff==0.17.2'
                        sh 'ruff check . && ruff format --check .'
                    }
                }
                stage('Testes unitários') {
                    agent {
                        docker {
                            image 'python:3.12-slim'
                            label 'docker'
                        }
                    }
                    steps {
                        sh 'pip install --quiet -r requirements-dev.txt'
                        sh 'mkdir -p relatorios && pytest -m "not integration" --junitxml=relatorios/unitarios.xml --cov=app --cov-report=xml:relatorios/coverage.xml'
                    }
                    post {
                        always { junit 'relatorios/unitarios.xml' }
                    }
                }
                stage('Busca de segredos') {
                    agent {
                        docker {
                            image 'zricethezav/gitleaks:v8.21.2'
                            label 'docker'
                            args '--entrypoint='
                        }
                    }
                    steps {
                        sh 'gitleaks detect --source . --no-banner --redact'
                    }
                }
            }
        }

        stage('Testes de integração') {
            agent { label 'docker' }
            steps {
                script {
                    // Um Postgres descartável, como o "services:" do GitHub Actions
                    docker.image('postgres:16-alpine').withRun(
                        '-e POSTGRES_USER=app -e POSTGRES_PASSWORD=app -e POSTGRES_DB=loja'
                    ) { pg ->
                        // Espera o banco aceitar conexões
                        docker.image('postgres:16-alpine').inside("--link ${pg.id}:db") {
                            sh 'until pg_isready -h db -U app; do sleep 1; done'
                        }
                        // Roda os testes em um container Python ligado ao banco
                        docker.image('python:3.12-slim').inside("--link ${pg.id}:db") {
                            sh '''
                                pip install --quiet -r requirements-dev.txt
                                mkdir -p relatorios
                                DATABASE_URL=postgresql://app:app@db:5432/loja \
                                  pytest -m integration --junitxml=relatorios/integracao.xml
                            '''
                        }
                    }
                }
            }
            post {
                always { junit 'relatorios/integracao.xml' }
            }
        }

        stage('Build da imagem') {
            agent { label 'docker' }
            environment {
                IMAGEM = "${REGISTRY}/minha-api:${GIT_COMMIT}"
            }
            steps {
                sh 'docker build -t "$IMAGEM" .'
            }
        }

        // ──────────────────────────── CD: só na main ────────────────────────────
        stage('Publicar imagem') {
            when {
                beforeAgent true
                branch 'main'
            }
            agent { label 'docker' }
            environment {
                IMAGEM = "${REGISTRY}/minha-api:${GIT_COMMIT}"
                REG    = credentials('registry-login')       // cria REG_USR e REG_PSW
            }
            steps {
                sh '''
                    docker build -t "$IMAGEM" .
                    echo "$REG_PSW" | docker login ghcr.io -u "$REG_USR" --password-stdin
                    docker push "$IMAGEM"
                '''
            }
            post {
                always { sh 'docker logout ghcr.io || true' }
            }
        }

        stage('Deploy em homologação') {
            when {
                beforeAgent true
                branch 'main'
            }
            agent { label 'docker' }
            environment {
                IMAGEM = "${REGISTRY}/minha-api:${GIT_COMMIT}"
            }
            steps {
                withCredentials([file(credentialsId: 'kubeconfig-homologacao', variable: 'KUBECONFIG')]) {
                    sh '''
                        kubectl set image deployment/minha-api api="$IMAGEM" -n loja
                        kubectl rollout status deployment/minha-api -n loja --timeout=5m
                    '''
                }
                sh './scripts/smoke-test.sh https://homolog.loja.exemplo.com'
            }
            post {
                failure {
                    withCredentials([file(credentialsId: 'kubeconfig-homologacao', variable: 'KUBECONFIG')]) {
                        sh 'kubectl rollout undo deployment/minha-api -n loja'
                    }
                }
            }
        }

        stage('Aprovação') {
            when {
                beforeAgent true
                branch 'main'
            }
            steps {
                timeout(time: 1, unit: 'DAYS') {
                    input message: "Implantar ${env.GIT_COMMIT?.take(7)} em produção?",
                          ok: 'Implantar',
                          submitter: 'admin'
                }
            }
        }

        stage('Deploy em produção') {
            when {
                beforeAgent true
                branch 'main'
            }
            agent { label 'docker' }
            options { lock('deploy-producao') }        // plugin Lockable Resources: um deploy por vez
            environment {
                IMAGEM = "${REGISTRY}/minha-api:${GIT_COMMIT}"
            }
            steps {
                withCredentials([file(credentialsId: 'kubeconfig-producao', variable: 'KUBECONFIG')]) {
                    sh '''
                        kubectl set image deployment/minha-api api="$IMAGEM" -n loja
                        kubectl annotate deployment/minha-api -n loja --overwrite \
                          kubernetes.io/change-cause="jenkins build $BUILD_NUMBER, commit $GIT_COMMIT"
                        kubectl rollout status deployment/minha-api -n loja --timeout=5m
                    '''
                }
                sh './scripts/smoke-test.sh https://loja.exemplo.com'
            }
            post {
                failure {
                    withCredentials([file(credentialsId: 'kubeconfig-producao', variable: 'KUBECONFIG')]) {
                        sh 'kubectl rollout undo deployment/minha-api -n loja'
                    }
                }
            }
        }
    }

    post {
        failure {
            echo "Pipeline falhou: ${env.BUILD_URL}"   // aqui entraria uma notificação para o chat da equipe
        }
        fixed {
            echo 'Pipeline voltou a passar.'
        }
    }
}
```

### Lendo o pipeline etapa por etapa

**`agent none` no topo.** Cada etapa pede seu próprio agente. Isso permite que a etapa de **Aprovação**, que não declara agente, espere sem ocupar executor (a confusão sobre `input` do [capítulo anterior](02_jenkinsfile_e_pipelines.md), seção 10).

**Verificações, em paralelo.** As três verificações rápidas rodam ao mesmo tempo, cada uma em um container próprio criado a partir de uma imagem pública: o agente só precisa de Docker. O `failFast true` interrompe as outras se uma falhar. No container do gitleaks, `args '--entrypoint='` desliga o *entrypoint* da imagem, para que o Jenkins consiga manter o container aberto e executar o `sh` dentro dele.

**Testes de integração.** O bloco `script` usa a API do plugin *Docker Pipeline* diretamente: `withRun` inicia um container do Postgres e o remove no fim, e `inside` executa passos dentro de outro container, ligado ao primeiro por `--link`, com o nome `db`. É a versão Jenkins dos *service containers* do GitHub Actions, e é por isso que o `DATABASE_URL` usa `db:5432`, e não `localhost`: aqui os testes rodam **em um container**, o caso descrito na confusão sobre `localhost` do [capítulo de GitHub Actions](../ci/02_github_actions_na_pratica.md), seção 6. (O `--link` é um recurso antigo do Docker, ainda funcional e usado nos exemplos oficiais do Jenkins. Uma rede Docker criada para o build faz o mesmo de forma mais moderna.)

**Build da imagem, em todo build.** Como no CI do GitHub Actions, construir a imagem prova que o Dockerfile funciona, inclusive nos PRs.

**Publicar, só na `main`.** O `when { branch 'main' }` separa o CI (em todo PR) do CD (só depois do merge). O `beforeAgent true` evita reservar um agente para uma etapa que não vai rodar. A credencial `registry-login` vira `REG_USR` e `REG_PSW`, lidas pelo **shell** em aspas simples, como manda a confusão sobre interpolação do [capítulo anterior](02_jenkinsfile_e_pipelines.md). A imagem é construída de novo nesta etapa porque ela pode cair em outro agente; com o cache de camadas do Docker no agente, essa segunda construção reaproveita o trabalho da etapa anterior.

**Homologação e produção.** O mesmo mecanismo do [pipeline de CD do GitHub Actions](../cd/03_pipeline_de_cd_na_pratica.md), parte 4.1: `kubectl set image`, espera do rollout, smoke test e `rollout undo` se algo falhar. A diferença está na credencial: em vez de OIDC, o Jenkins usa um **kubeconfig** guardado como credencial do tipo *Secret file*, entregue ao `kubectl` pela variável `KUBECONFIG` só dentro do `withCredentials`. Cada ambiente tem o seu, com permissões limitadas ao namespace `loja` pelo RBAC do [capítulo de configuração e segurança do Kubernetes](../../02-containers/kubernetes/03_configuracao_estado_e_seguranca.md).

**Um deploy de produção por vez.** O `lock('deploy-producao')` (plugin *Lockable Resources*, já incluído no `plugins.txt`) garante que dois builds da `main` nunca atualizem produção ao mesmo tempo, o papel do `concurrency` do GitHub Actions.

> **Confusão comum:** "o Jenkinsfile vem do repositório, então um pull request pode alterá-lo para imprimir as credenciais de produção". ✅ **Mais preciso:** seria verdade se o Jenkins executasse cegamente o Jenkinsfile de qualquer PR. O *GitHub Branch Source* tem uma regra de **confiança** para isso: por padrão, só PRs de pessoas com permissão de escrita no repositório têm o **próprio** Jenkinsfile executado. Para PRs de forks de pessoas de fora, o Jenkins usa o Jenkinsfile da **branch de destino**, ignorando o que o PR mudou nele. É a mesma preocupação do `pull_request_target` do [capítulo de GitHub Actions](../ci/02_github_actions_na_pratica.md). Duas camadas a mais completam a proteção: as etapas que usam credenciais de produção só rodam na `main` (`when { branch 'main' }`), e essas credenciais podem ser criadas **dentro da pasta do job** e com escopo restrito, de forma que outros jobs nem as enxerguem. E vale lembrar o que a seção 2 mostrou: um PR que roda em um agente com o socket do Docker pode atacar o próprio agente. Builds de fontes não confiáveis devem rodar em agentes que não guardem nada de valor.

---

## 5. Executando e acompanhando

Com o Jenkinsfile na `main` e o job `minha-api` existente, o primeiro **scan** do repositório cria os sub-jobs:

```
minha-api
├── main        #1  ✔ (executou até a aprovação)
└── PR-7        #1  ✔ (só as etapas de CI)
```

No build da `main`, a visualização das etapas (plugin *Pipeline Graph View*) mostra:

```
 Verificações ──────────────┐
 ├─ Lint            ✔ 18s   │
 ├─ Testes unitários ✔ 41s  ├──▶ Testes de integração ✔ 52s ──▶ Build da imagem ✔ 1m10s
 └─ Busca de segredos ✔ 6s  │
                            ┘
     ──▶ Publicar imagem ✔ 25s ──▶ Deploy em homologação ✔ 1m40s ──▶ Aprovação ⏸ ──▶ Deploy em produção
```

A etapa de aprovação mostra a pergunta e os botões **Implantar** e **Abort**. A aba **Tests** reúne os resultados dos dois relatórios JUnit, com a tendência ao longo dos builds. O **Console Output** de cada etapa traz o log, com o horário em cada linha (`timestamps()`).

O build do **PR-7** para no fim do *Build da imagem*: todas as etapas de CD foram puladas pelo `when`, e aparecem na visualização como *skipped*. O resultado vai para o PR no GitHub como um check verde.

---

## 6. Agentes no Kubernetes: um Pod por build

O agente da seção 2 é **permanente**: está sempre ligado, acumula arquivos nos workspaces e tem o socket do Docker. Com o plugin **Kubernetes**, o Jenkins passa a criar, para cada build, um **Pod** em um cluster, executa o build nele e o apaga no fim. É o modelo dos runners efêmeros do GitHub Actions, construído sobre o que a parte de [Kubernetes](../../02-containers/kubernetes/01_fundamentos_e_arquitetura.md) ensinou.

### 6.1 Ligando o Jenkins ao cluster

No `casc.yaml`, uma *cloud* do tipo Kubernetes:

```yaml
jenkins:
  clouds:
    - kubernetes:
        name: "loja-k8s"
        namespace: "jenkins-agentes"            # os Pods de build ficam isolados neste namespace
        jenkinsUrl: "http://jenkins.jenkins.svc.cluster.local:8080"
        jenkinsTunnel: "jenkins-agent.jenkins.svc.cluster.local:50000"
        containerCapStr: "20"                   # no máximo 20 Pods de build ao mesmo tempo
```

O caso mais simples é o próprio controller rodando **dentro** do cluster (há um chart Helm oficial do Jenkins, o caminho de instalação mais comum em Kubernetes). Nesse caso, o plugin usa a **ServiceAccount** do Pod do controller para falar com a API do cluster, e essa ServiceAccount precisa de um **RBAC** que permita criar e apagar Pods **só** no namespace `jenkins-agentes` (a Role e o RoleBinding do [capítulo de configuração e segurança](../../02-containers/kubernetes/03_configuracao_estado_e_seguranca.md), seção 6.2).

### 6.2 O Pod do build, declarado no Jenkinsfile

```groovy
pipeline {
    agent {
        kubernetes {
            yaml '''
                apiVersion: v1
                kind: Pod
                spec:
                  containers:
                    - name: python
                      image: python:3.12-slim
                      command: ["sleep"]
                      args: ["infinity"]
                      resources:
                        requests: { cpu: "500m", memory: "512Mi" }
                        limits:   { memory: "1Gi" }
                    - name: postgres
                      image: postgres:16-alpine
                      env:
                        - { name: POSTGRES_USER,     value: app }
                        - { name: POSTGRES_PASSWORD, value: app }
                        - { name: POSTGRES_DB,       value: loja }
            '''
            defaultContainer 'python'
        }
    }
    stages {
        stage('Testes') {
            steps {
                sh 'pip install --quiet -r requirements-dev.txt'
                sh 'until python -c "import psycopg; psycopg.connect(\'postgresql://app:app@localhost:5432/loja\')"; do sleep 1; done'
                sh '''
                    pytest -m "not integration"
                    DATABASE_URL=postgresql://app:app@localhost:5432/loja pytest -m integration
                '''
            }
        }
    }
}
```

O detalhe mais interessante está no `DATABASE_URL`, que voltou a usar **`localhost`**. O Postgres não é outro container em uma rede Docker: ele é **outro container do mesmo Pod**. E o [capítulo de fundamentos do Kubernetes](../../02-containers/kubernetes/01_fundamentos_e_arquitetura.md) (seção 4) mostrou que os containers de um Pod compartilham o mesmo IP e se enxergam por `localhost`. É o padrão sidecar, usado aqui como banco de testes descartável. Além dos dois containers declarados, o plugin acrescenta automaticamente ao Pod o container do agente Jenkins (`jnlp`), que conversa com o controller.

Quando o build termina, o Pod é apagado: nada sobra para o próximo build, e nenhum build enxerga os arquivos de outro. Os `requests` e `limits` fazem o scheduler do Kubernetes distribuir os builds pelos nodes, e, com o autoscaler de nodes do [capítulo de workloads](../../02-containers/kubernetes/02_workloads_rede_e_escala.md), o cluster ganha máquinas quando a fila de builds cresce.

Construir **imagens** dentro de um Pod é o ponto delicado: não há um daemon Docker disponível, e montar o socket do node traria de volta o problema da seção 2. As opções são ferramentas que constroem imagens sem daemon privilegiado, como o **BuildKit** em modo *rootless* ou o **Buildah**, rodando como mais um container do Pod do build.

---

## 7. Segurança e operação

Um Jenkins guarda as credenciais de todos os ambientes, inclusive de produção, e executa código de todos os repositórios. Isso o torna um dos alvos mais valiosos da infraestrutura. As práticas a seguir vêm da documentação de segurança do próprio projeto.

### 7.1 Acesso

- **Nunca** deixe o Jenkins sem autenticação, nem com leitura anônima, se ele estiver acessível pela rede. Logs de build costumam revelar nomes de servidores, estrutura interna e, com descuido, segredos.
- Use **autorização por matriz** (`globalMatrix`, como no `casc.yaml`) ou por papéis (*Role-based Authorization Strategy*), e dê permissões por **pasta**: a equipe de pagamentos administra os próprios jobs, e só enxerga os próprios.
- Integre o login com o provedor de identidade da empresa (LDAP, SAML, OIDC), em vez de usuários locais.
- A proteção **CSRF** vem ligada por padrão: mantenha-a. Scripts e integrações usam **tokens de API** de usuário, e não a senha.

### 7.2 Execução de código

- **Zero executores no controller** (`numExecutors: 0`), sempre.
- Os Jenkinsfiles rodam em um **sandbox** do Groovy: métodos considerados perigosos são bloqueados, e o administrador precisa aprová-los em **Manage Jenkins → In-process Script Approval**. Aprove com critério: aprovar um método como `java.lang.Runtime.exec` abre o controller para qualquer Jenkinsfile.
- **Shared Libraries "confiáveis"** (*trusted*), configuradas globalmente, rodam **fora** do sandbox. O repositório delas deve ter a mesma proteção de qualquer repositório de produção: PR obrigatório, revisão e poucas pessoas com permissão de escrita.
- Separe os agentes por **nível de confiança**: os que executam PRs não devem ter acesso às credenciais nem à rede de produção.

### 7.3 Credenciais

- Prefira credenciais de **curta duração**. Na AWS, em vez de um kubeconfig com token permanente, o agente pode assumir uma **IAM role** (no EKS, pelo Pod Identity da ServiceAccount do Pod de build, como no [capítulo de IAM](../../01-infraestrutura/aws/03_iam_identidades_e_permissoes.md), seção 11), e o `aws eks update-kubeconfig` gera tokens temporários.
- Crie credenciais no **menor escopo possível**: dentro da pasta do job que as usa, e não em *Global*.
- Para ambientes maiores, o Jenkins pode buscar segredos de um cofre externo (plugins para HashiCorp Vault e AWS Secrets Manager), em vez de guardá-los no `JENKINS_HOME`.

### 7.4 Operação

| Tarefa | Como |
|---|---|
| **Atualizar o Jenkins e os plugins** | Nova versão no `Dockerfile`/`plugins.txt`, testada com uma **cópia** do `JENKINS_HOME` antes de ir para produção. Acompanhe os avisos de segurança em `jenkins.io/security/advisories` |
| **Backup** | O volume do `JENKINS_HOME` (snapshot do disco ou cópia agendada). Com o JCasC, a configuração já está no Git; o backup protege o histórico de builds, as credenciais e o diretório `secrets/` |
| **Disco** | `buildDiscarder` em todos os jobs, e `cleanWs()` nos agentes permanentes |
| **Memória do controller** | O Jenkins é uma aplicação Java: ajuste o heap (`JAVA_OPTS="-Xmx2g ..."`) e monitore o uso |
| **Disponibilidade** | O controller é um ponto único de falha: se ele cai, nenhum build roda. Com o JCasC e backups, recriá-lo leva minutos; mantenha esse procedimento testado |
| **Monitoramento** | A fila de builds (*builds esperando executor* é sinal de falta de agentes), o tempo dos builds e a taxa de falhas, exportados para o sistema de métricas (plugin *Prometheus metrics*) |

---

## 8. Diagnóstico e comparação final

### 8.1 Erros comuns

| Sintoma | Causa provável | O que fazer |
|---|---|---|
| Build parado em *Waiting for next available executor* | Nenhum agente com a label pedida está livre (ou conectado) | **Manage Jenkins → Nodes**: o agente está online? A label bate? |
| `Jenkins doesn't have label 'docker'` | Nenhum agente tem essa label | Conferir o `labelString` no `casc.yaml` |
| `docker: not found` | O agente não tem o cliente Docker | Instalar na imagem do agente (seção 2.1) |
| `permission denied ... /var/run/docker.sock` | O usuário `jenkins` do agente não está no grupo dono do socket | `DOCKER_GID` no `.env` e `group_add` no Compose |
| `No such DSL method 'xyz'` | O plugin que fornece o passo não está instalado | Acrescentar ao `plugins.txt` (o *Snippet Generator* mostra os passos disponíveis) |
| `Scripts not permitted to use method ...` | O sandbox bloqueou um método | Reescrever sem o método, ou aprovar conscientemente em *In-process Script Approval* |
| `A secret was passed to "sh" using Groovy String interpolation` | Credencial em aspas duplas | Aspas simples e `$VARIAVEL` (capítulo anterior, seção 5.3) |
| O webhook não dispara builds | O GitHub não alcança o Jenkins | Em **Settings → Webhooks → Recent Deliveries**, o GitHub mostra a resposta de cada entrega |
| Mudanças feitas na interface somem após reiniciar | O JCasC reaplicou o `casc.yaml` | Fazer a mudança no arquivo, não na interface |
| O sub-job do PR não aparece | O PR não tem `Jenkinsfile`, ou a regra de confiança o ignorou | **Scan Repository Log** do job Multibranch |

### 8.2 Jenkins, GitHub Actions e GitLab CI

| | GitHub Actions | GitLab CI | Jenkins |
|---|---|---|---|
| **Quem opera** | O GitHub | O GitLab, ou você (instalação própria) | Você |
| **Configuração do pipeline** | YAML | YAML | Groovy (declarativo ou scripted) |
| **Configuração da plataforma** | Interface e API do GitHub | Interface ou `gitlab.rb` | JCasC (YAML) + plugins |
| **Onde os jobs rodam** | VMs do GitHub, ou runners próprios | Runners do GitLab, ou próprios | Agentes próprios (permanentes, Docker, Kubernetes) |
| **Extensões** | Actions do marketplace | Templates e componentes | ~2.000 plugins |
| **Integração com o repositório** | Nativa (só GitHub) | Nativa (só GitLab) | Qualquer um, por plugins |
| **Aprovação manual** | Environments com revisores | `when: manual` + ambientes protegidos | `input` |
| **Credenciais na nuvem sem segredo guardado** | OIDC nativo | OIDC nativo | Via identidade do agente (por exemplo, IAM role do Pod) |
| **Ponto forte** | Simplicidade, ecossistema | Plataforma completa em um produto | Flexibilidade, controle e acesso à rede interna |
| **Custo principal** | Minutos de runner | Licença e/ou minutos | Pessoas e servidores para operar |

A escolha raramente é técnica no sentido estrito. Ela depende de onde o código já está, de onde os pipelines precisam chegar, das regras da empresa e de quem vai cuidar da plataforma. Os conceitos (CI com feedback rápido, artefato único, promoção entre ambientes, estratégias de deploy, rollback automático, menor privilégio) são os mesmos nas três, e são eles o que este bloco procurou ensinar.

---

## Fim da trilha de entrega contínua

Com este capítulo, o caminho que começou no [capítulo de fundamentos de infraestrutura](../../01-infraestrutura/fundamentos/01_o_que_e_infraestrutura.md) está completo. Uma linha de código escrita por alguém da equipe:

1. entra na `main` por um pull request pequeno, verificado por lint, testes e scans em poucos minutos (**CI**);
2. vira uma imagem Docker enxuta, com uma tag única, publicada uma vez (**containers**);
3. é implantada em homologação e, depois da aprovação, em produção, com rolling update, smoke test e rollback automático (**CD**);
4. roda em Pods de um cluster Kubernetes (**orquestração**), em nodes de sub-redes privadas de uma VPC, acessando a AWS por identidades com permissões mínimas (**infraestrutura**);
5. tudo isso descrito em arquivos versionados: Dockerfile, manifestos, Terraform, workflows, Jenkinsfile, `casc.yaml`.

Falta uma última pergunta, que vários capítulos deixaram em aberto: **como saber se o sistema está funcionando bem em produção?** O canary precisava de métricas confiáveis, o diagnóstico do Kubernetes dependia de logs, as métricas DORA precisam ser medidas. Esse é o tema da pasta de **observabilidade**, a próxima parte do repositório, que começa em [../../04-observabilidade/fundamentos/01_monitoramento_e_observabilidade.md](../../04-observabilidade/fundamentos/01_monitoramento_e_observabilidade.md).
