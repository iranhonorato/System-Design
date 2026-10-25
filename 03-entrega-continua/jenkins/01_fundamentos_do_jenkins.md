# Jenkins: Fundamentos e Arquitetura

**Pré-requisito:** [../ci/01_fundamentos_de_integracao_continua.md](../ci/01_fundamentos_de_integracao_continua.md) e o bloco de CD. Docker, até [../../02-containers/docker/04_compose_e_producao.md](../../02-containers/docker/04_compose_e_producao.md).

> Segue a documentação oficial do Jenkins (*User Handbook*: *Installing Jenkins*, *Managing Jenkins*, *Using credentials* e *Architecting for scale*).

---

## Por que este arquivo existe

Os blocos de CI e CD usaram o GitHub Actions, em que **outra empresa** opera tudo: os servidores, as máquinas que executam os jobs, a interface, as atualizações, a segurança da plataforma. Você só escreve o YAML. Essa é a grande vantagem das plataformas hospedadas, e também o seu limite. E quando o pipeline precisa acessar um banco de dados que só existe na rede interna da empresa? Quando uma regra de compliance proíbe que o código saia da rede da empresa? Quando o build precisa de uma máquina com hardware específico, ou de uma integração com um sistema de 20 anos atrás?

Nesses casos, a automação roda em um servidor **seu**, e esse servidor quase sempre é o **Jenkins**: o servidor de automação mais usado no mundo, presente na maioria das grandes empresas. A pergunta central deste capítulo é: *o que é o Jenkins, como ele funciona por dentro e como colocá-lo de pé?* É o equivalente, para a automação, do que o [capítulo de arquitetura do Docker](../../02-containers/docker/01_arquitetura_e_isolamento.md) fez para os containers e o de [fundamentos do Kubernetes](../../02-containers/kubernetes/01_fundamentos_e_arquitetura.md) fez para os clusters.

O bloco tem três capítulos:

| Capítulo | Pergunta |
|---|---|
| **01 (este)** | O que é o Jenkins, como é sua arquitetura e como instalá-lo? |
| [02](02_jenkinsfile_e_pipelines.md) | Como escrever pipelines como código, no `Jenkinsfile`? |
| [03](03_jenkins_na_pratica.md) | Como montar o CI/CD completo da loja no Jenkins, configurado como código e operado com segurança? |

---

## 1. O que é o Jenkins

O **Jenkins** é um **servidor de automação** de código aberto, escrito em Java. Ele executa tarefas automaticamente, em resposta a eventos (um push, um horário, um clique), e mostra os resultados. Na prática, ele é usado principalmente para CI/CD, mas pode automatizar qualquer coisa que se faça pela linha de comando: backups, relatórios, tarefas de manutenção.

### Um pouco de história

O projeto nasceu em 2004, na Sun Microsystems, com o nome **Hudson**, criado por Kohsuke Kawaguchi, que estava cansado de quebrar o build da equipe e só descobrir depois. Quando a Oracle comprou a Sun e surgiu uma disputa sobre o nome do projeto, a comunidade o renomeou para **Jenkins**, em 2011. Hoje ele é mantido pela comunidade sob a *CD Foundation*, da Linux Foundation.

Essa idade explica tanto a força quanto as dificuldades do Jenkins. A força: **duas décadas de integrações**, com quase 2.000 plugins para praticamente qualquer ferramenta, linguagem ou sistema que exista. As dificuldades: uma interface com camadas de épocas diferentes, configuração espalhada, e muitos tutoriais antigos ensinando práticas que o próprio projeto já desaconselha (a seção 5 trata da principal).

### O que muda em relação ao GitHub Actions

| | GitHub Actions (hospedado) | Jenkins (auto-hospedado) |
|---|---|---|
| **Quem opera o servidor** | O GitHub | **Você**: instalação, atualização, backup, segurança |
| **Onde os jobs rodam** | VMs do GitHub (ou runners seus) | **Suas** máquinas (agentes) |
| **Acesso à rede interna** | Só com runners próprios | Natural: o Jenkins já está dentro da rede |
| **Integrações** | Actions do marketplace | Plugins |
| **Pipeline como código** | YAML em `.github/workflows/` | `Jenkinsfile` (Groovy) |
| **Custo** | Minutos de runner (com cota gratuita) | Os servidores e o tempo de quem os opera |
| **Repositório** | Só GitHub | Qualquer um: GitHub, GitLab, Bitbucket, servidores Git internos |

Os conceitos são os mesmos do [capítulo de fundamentos de CI](../ci/01_fundamentos_de_integracao_continua.md): gatilhos, etapas, máquinas que executam, segredos. O que o Jenkins acrescenta é a responsabilidade de **operar a plataforma**.

---

## 2. A arquitetura: controller e agentes

### 2.1 As peças

Um ambiente Jenkins tem dois tipos de máquina:

```
                      ┌─────────────────────────────────────────────┐
  pessoas ──web──▶    │              CONTROLLER                     │
  (navegador)  :8080  │                                             │
                      │  ┌──────────┐ ┌──────────┐ ┌─────────────┐  │
  GitHub ──webhook──▶ │  │interface │ │  fila de │ │ agendador   │  │
                      │  │   web    │ │  builds  │ │ (triggers)  │  │
                      │  └──────────┘ └──────────┘ └─────────────┘  │
                      │  ┌───────────────────────────────────────┐  │
                      │  │ JENKINS_HOME: configuração, jobs,     │  │
                      │  │ histórico, plugins, credenciais       │  │
                      │  └───────────────────────────────────────┘  │
                      └───────────┬─────────────────┬───────────────┘
                                  │ conexão de      │
                                  │ agente (:50000  │
                                  │ ou SSH)         │
               ┌──────────────────▼──────┐   ┌──────▼────────────────────┐
               │ AGENTE "linux-docker"   │   │ AGENTE "windows"          │
               │ ┌─────────┐ ┌─────────┐ │   │ ┌─────────┐               │
               │ │executor │ │executor │ │   │ │executor │               │
               │ │  build  │ │  (livre)│ │   │ │  build  │               │
               │ │  #42    │ │         │ │   │ │  #17    │               │
               │ └─────────┘ └─────────┘ │   │ └─────────┘               │
               │ workspace/, docker, git │   │ workspace\, .NET, git     │
               └─────────────────────────┘   └───────────────────────────┘
```

**Controller** (antes chamado de *master*). O cérebro: serve a interface web, recebe os webhooks, mantém a **fila** de builds, decide em qual agente cada build roda, guarda a configuração e o histórico, e coordena a execução. Tudo o que ele sabe fica em um diretório, o **`JENKINS_HOME`**.

**Agentes** (também chamados de *nodes*). Os músculos: máquinas que efetivamente **executam** os builds. Cada agente se conecta ao controller e tem um número de **executores**: quantos builds pode rodar ao mesmo tempo. Agentes diferentes podem ter sistemas operacionais e ferramentas diferentes, e são identificados por **labels** (`linux`, `docker`, `windows`, `gpu`).

**Executor.** Uma "vaga" de execução em um agente. Um agente com 2 executores roda até 2 builds ao mesmo tempo.

**Workspace.** O diretório, no agente, onde o build trabalha: é ali que o código é baixado e os comandos rodam. É o equivalente ao diretório da VM do runner no GitHub Actions, com uma diferença importante: **o workspace persiste entre builds**, a não ser que seja limpo explicitamente.

A divisão lembra a do Kubernetes do [capítulo de fundamentos](../../02-containers/kubernetes/01_fundamentos_e_arquitetura.md): um plano de controle que decide e guarda o estado, e nós de trabalho que executam. A diferença é que o Jenkins não tem reconciliação contínua: ele executa ordens (builds) e registra o resultado.

### 2.2 O caminho de um build

```
 ① um push no GitHub dispara um webhook para o controller
 ② o controller identifica o job e coloca um build na FILA
 ③ o build pede um agente com a label exigida (ex.: "docker")
 ④ há um executor livre em um agente com essa label?
       não → o build espera na fila ("Waiting for next available executor")
       sim → o build é atribuído àquele executor
 ⑤ o agente baixa o código para o workspace e executa os passos
 ⑥ a saída (o "console output") é enviada ao controller em tempo real
 ⑦ o resultado (SUCCESS, FAILURE, UNSTABLE, ABORTED) e os artefatos ficam no controller
```

### 2.3 Por que separar controller e agentes

> **Confusão comum:** "uma instalação simples do Jenkins roda os builds no próprio servidor, então é assim que se usa". ✅ **Mais preciso:** por padrão, o controller tem um nó embutido (o *built-in node*) com executores, e é por isso que os tutoriais funcionam sem configurar agente nenhum. Mas a documentação do Jenkins recomenda deixar **zero executores** no controller em qualquer uso sério, por três motivos. **Segurança:** um build executa código do repositório, e no controller esse código tem acesso ao `JENKINS_HOME`, onde ficam **todas** as credenciais de todos os projetos; um PR malicioso pode lê-las. **Estabilidade:** um build que consome toda a memória derruba o controller, e com ele a interface e todos os outros builds. **Escala:** a capacidade fica limitada a uma máquina. Builds rodam em agentes; o controller só coordena. Os agentes podem ser máquinas fixas, mas o mais comum hoje é que sejam **efêmeros**: um container ou um Pod do Kubernetes criado para o build e destruído no fim, como os runners do GitHub Actions ([capítulo prático](03_jenkins_na_pratica.md)).

### 2.4 O `JENKINS_HOME`

Tudo o que o controller sabe fica em arquivos, a maioria em XML:

```
/var/jenkins_home/
├── config.xml               ← configuração global
├── credentials.xml          ← credenciais (cifradas com as chaves de secrets/)
├── secrets/                 ← chaves de criptografia: PROTEJA E FAÇA BACKUP
├── jobs/
│   └── minha-api/
│       ├── config.xml       ← configuração do job
│       └── builds/
│           ├── 41/          ← log, resultado e artefatos de cada build
│           └── 42/
├── plugins/                 ← os plugins instalados
├── nodes/                   ← configuração dos agentes
└── users/
```

Duas consequências práticas: **fazer backup do `JENKINS_HOME` é fazer backup do Jenkins inteiro** (em container, é o volume do capítulo de [volumes e redes](../../02-containers/docker/03_volumes_e_redes.md)); e o diretório `secrets/` é tão sensível quanto as credenciais, porque sem ele elas não podem ser decifradas, e com ele qualquer um que tenha uma cópia de `credentials.xml` pode decifrá-las.

---

## 3. Instalando com Docker

O jeito mais rápido de ter um Jenkins para estudo é a imagem oficial:

```bash
docker volume create jenkins_home

docker run -d --name jenkins \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  --restart unless-stopped \
  jenkins/jenkins:lts-jdk21
```

| Parte | Por quê |
|---|---|
| `-p 8080:8080` | A interface web |
| `-p 50000:50000` | A porta em que agentes externos se conectam ao controller |
| `-v jenkins_home:/var/jenkins_home` | O `JENKINS_HOME` em um volume: sobrevive à recriação do container (e às atualizações da imagem) |
| `jenkins/jenkins:lts-jdk21` | A linha **LTS** (*Long-Term Support*), com versões estáveis a cada 12 semanas, sobre Java 21. A linha semanal (`latest`) recebe novidades primeiro, com menos estabilidade |

O mesmo, em Compose, como ficará no [capítulo prático](03_jenkins_na_pratica.md):

```yaml
# docker-compose.yml
services:
  jenkins:
    image: jenkins/jenkins:lts-jdk21
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - jenkins_home:/var/jenkins_home
    restart: unless-stopped

volumes:
  jenkins_home:
```

### O assistente de configuração

1. Acesse `http://localhost:8080`. O Jenkins pede uma **senha inicial**, gerada na primeira inicialização:
   ```bash
   docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
   # 3f9a1c2b7d4e4a8f9c1b2d3e4f5a6b7c
   ```
2. **Customize Jenkins:** escolha *Install suggested plugins*. Isso instala o conjunto básico: Git, Pipeline, credenciais, a visualização de etapas, entre outros.
3. **Create First Admin User:** crie seu usuário administrador (não continue com o usuário `admin` e a senha inicial).
4. **Instance Configuration:** a URL do Jenkins (`http://localhost:8080/` para estudo). Ela é usada nos links dos e-mails e nos webhooks.

Ao fim, aparece o **painel** (*Dashboard*), vazio.

O passo a passo pela interface é ótimo para conhecer o Jenkins, mas tem o defeito de toda configuração manual: não é versionado nem reproduzível. O [capítulo prático](03_jenkins_na_pratica.md) mostra como descrever **a configuração do próprio Jenkins** em código, com o plugin *Configuration as Code*.

---

## 4. Os conceitos da interface

| Termo | O que é |
|---|---|
| **Job** (ou *item*, ou *projeto*) | Uma automação configurada: "construir e testar a minha-api" |
| **Build** | Uma **execução** de um job, numerada em sequência (#1, #2, #3...) |
| **Console Output** | O log completo de um build, com a saída de cada comando |
| **Workspace** | Os arquivos de trabalho do último build, no agente |
| **Artefatos** | Arquivos guardados ao fim do build (relatórios, pacotes) |
| **Status do build** | **SUCCESS** (verde), **FAILURE** (vermelho, algo falhou), **UNSTABLE** (amarelo: o build terminou, mas testes falharam ou um limite de qualidade não foi atingido), **ABORTED** (cancelado) |
| **Folder** | Uma pasta para organizar jobs (e para dar permissões por grupo) |
| **View** | Uma lista filtrada de jobs no painel |

O status **UNSTABLE** não existe no GitHub Actions e merece atenção: ele permite separar "o build quebrou" (não compilou, um comando falhou) de "o build terminou, mas com qualidade abaixo do esperado" (testes falhando, cobertura baixa).

---

## 5. Os tipos de job

Ao clicar em **New Item**, o Jenkins oferece vários tipos. Os quatro que importam:

### 5.1 Freestyle: configurado na interface

O tipo mais antigo. Tudo é configurado por formulários: de onde vem o código, o que dispara o build, quais comandos rodar, o que fazer depois.

Para ver como funciona: **New Item → `ola-freestyle` → Freestyle project**. Em *Build Steps*, *Add build step → Execute shell*:

```bash
echo "Olá do Jenkins!"
echo "Este é o build número $BUILD_NUMBER"
uname -a
```

Salve e clique em **Build Now**. No histórico, clique no build `#1` e em **Console Output**:

```
Started by user Ana
Running as SYSTEM
Building in workspace /var/jenkins_home/workspace/ola-freestyle
[ola-freestyle] $ /bin/sh -xe /tmp/jenkins123.sh
+ echo Olá do Jenkins!
Olá do Jenkins!
+ echo Este é o build número 1
Este é o build número 1
+ uname -a
Linux 3f9a1c2b7d4e 6.8.0 ... x86_64 GNU/Linux
Finished: SUCCESS
```

Repare em `Building in workspace /var/jenkins_home/...`: o build rodou **no controller**, o que a seção 2.3 desaconselha. Para um teste, tudo bem.

> **Confusão comum:** "jobs freestyle são mais fáceis, então são a melhor forma de começar um projeto". ✅ **Mais preciso:** eles são mais fáceis de **criar**, e muito mais difíceis de **manter**. Toda a configuração fica em um `config.xml` dentro do `JENKINS_HOME`, editado por formulários. Não há histórico de quem mudou o quê, nem revisão em pull request, nem como saber qual configuração foi usada no build de três meses atrás, nem como recriar o job em outro Jenkins sem clicar tudo de novo. Com dezenas de jobs, isso vira exatamente o "servidor floco de neve" do [capítulo de virtualização](../../01-infraestrutura/fundamentos/02_virtualizacao_containers_e_iac.md), só que para pipelines. É o problema que o tipo **Pipeline** resolve, ao levar a definição para um arquivo no repositório. Use Freestyle para tarefas administrativas pequenas, se tanto; para CI/CD, use Pipeline.

### 5.2 Pipeline: definido em código

O pipeline é descrito em um script, o **Jenkinsfile**, em uma linguagem própria baseada em **Groovy**. O script pode ser colado no próprio job (para testar) ou, como deve ser, ficar **no repositório**.

**New Item → `ola-pipeline` → Pipeline**. Em *Pipeline → Definition: Pipeline script*:

```groovy
pipeline {
    agent any
    stages {
        stage('Preparar') {
            steps {
                echo 'Baixando dependências...'
            }
        }
        stage('Testar') {
            steps {
                sh 'echo "rodando os testes"'
            }
        }
        stage('Empacotar') {
            steps {
                sh 'echo "construindo a imagem"'
            }
        }
    }
}
```

**Build Now**, e a página do job passa a mostrar as **etapas** (*stages*) do pipeline, cada uma com sua duração e seu status:

```
            Preparar     Testar     Empacotar
 #1         ✔ 120ms     ✔ 340ms     ✔ 290ms
```

O [próximo capítulo](02_jenkinsfile_e_pipelines.md) é inteiro sobre a sintaxe do Jenkinsfile.

### 5.3 Multibranch Pipeline: um pipeline por branch

Um **Multibranch Pipeline** aponta para um **repositório**, e não para uma branch. O Jenkins procura um `Jenkinsfile` em **cada** branch e em **cada** pull request, e cria automaticamente um sub-job para cada um. Quando uma branch é apagada, o sub-job some. É o equivalente ao `on: pull_request` do GitHub Actions, e é o tipo usado no [capítulo prático](03_jenkins_na_pratica.md).

```
minha-api  (Multibranch Pipeline)
├── main          ← sub-job, criado a partir do Jenkinsfile da main
├── carrinho      ← sub-job da branch de Ana
└── PR-142        ← sub-job do pull request #142
```

### 5.4 Organization Folder: todos os repositórios de uma organização

Um nível acima: aponta para uma **organização** inteira do GitHub (ou um grupo do GitLab) e cria um Multibranch Pipeline para cada repositório que tenha um `Jenkinsfile`. Com ele, um repositório novo ganha CI sem ninguém configurar nada no Jenkins.

| Tipo | Definição do pipeline | Quando usar |
|---|---|---|
| **Freestyle** | Formulários na interface | Tarefas administrativas simples (evite para CI/CD) |
| **Pipeline** | Jenkinsfile (colado no job ou no repositório) | Uma automação única, sem branches (um backup agendado, por exemplo) |
| **Multibranch Pipeline** | Jenkinsfile em cada branch e PR | **CI/CD de um repositório**: o padrão |
| **Organization Folder** | Jenkinsfile em cada repositório | Muitos repositórios com o mesmo padrão |

---

## 6. Plugins

Quase tudo no Jenkins vem de **plugins**: o suporte a Git, o próprio Pipeline, a integração com Docker e Kubernetes, os tipos de credencial, as notificações. Eles são gerenciados em **Manage Jenkins → Plugins**, com as abas *Updates*, *Available*, *Installed* e *Advanced*.

Os plugins que aparecem nesta trilha:

| Plugin | Para quê |
|---|---|
| **Pipeline** (conjunto *workflow-aggregator*) | O tipo Pipeline e o Jenkinsfile |
| **Git** e **GitHub Branch Source** | Baixar código e criar Multibranch Pipelines a partir do GitHub, com status nos PRs |
| **Credentials** e **Credentials Binding** | Guardar segredos e usá-los nos pipelines |
| **Docker Pipeline** | Rodar etapas dentro de containers (`agent { docker ... }`) |
| **Kubernetes** | Criar agentes como Pods em um cluster |
| **Configuration as Code** (JCasC) | Configurar o Jenkins por um arquivo YAML |
| **Pipeline Graph View** | Visualização gráfica das etapas (substitui o antigo Blue Ocean, que não recebe mais novas funcionalidades) |
| **JUnit** | Ler relatórios de testes e mostrar tendências |
| **Role-based Authorization Strategy** | Permissões por papel |

> **Confusão comum:** "cada plugin a mais é uma funcionalidade a mais, então vale instalar o que parecer útil". ✅ **Mais preciso:** cada plugin é **código de terceiros rodando dentro do controller**, com acesso a tudo o que o Jenkins acessa, inclusive as credenciais. Plugins têm vulnerabilidades (o Jenkins publica avisos de segurança de plugins quase todo mês), podem ser abandonados pelos mantenedores, e dependem de versões específicas de outros plugins e do próprio Jenkins. Uma instalação com 150 plugins acaba presa: atualizar um quebra outro, e não atualizar deixa falhas conhecidas abertas. É o mesmo raciocínio das actions de terceiros do [capítulo de GitHub Actions](../ci/02_github_actions_na_pratica.md), agravado porque um plugin roda no servidor central. A prática recomendada: instalar só o necessário, listar os plugins em um arquivo versionado (o `plugins.txt` do [capítulo prático](03_jenkins_na_pratica.md)), atualizar com frequência, testando antes, e remover o que ninguém usa.

---

## 7. Credenciais

Senhas, tokens e chaves usados pelos pipelines ficam no **armazenamento de credenciais** do Jenkins (**Manage Jenkins → Credentials**), cifrados no `JENKINS_HOME`. Cada credencial tem:

- um **ID**, pelo qual os pipelines a referenciam (`registry-token`, `ssh-deploy-producao`);
- um **tipo**;
- um **escopo**: *Global* (disponível para os jobs) ou *System* (só para o próprio Jenkins, como a conexão com um agente).

| Tipo | Exemplo de uso |
|---|---|
| **Username with password** | Login em um registry Docker |
| **Secret text** | Um token de API (GitHub, Slack, Sonar) |
| **SSH Username with private key** | Deploy por SSH, conexão com agentes |
| **Secret file** | Um kubeconfig, um arquivo `.env` |
| **Certificate** | Um certificado de cliente |

Os pipelines nunca contêm o valor: eles pedem a credencial pelo ID, e o Jenkins a entrega só durante a etapa que precisa dela, mascarando o valor nos logs. O [próximo capítulo](02_jenkinsfile_e_pipelines.md) mostra como, e mostra também um erro de sintaxe que faz o valor vazar mesmo assim.

As credenciais também podem ser **por pasta**: uma credencial criada dentro da pasta `time-pagamentos` só é visível para os jobs daquela pasta. É a forma de evitar que o pipeline de um time use a chave de produção de outro.

---

## 8. Agentes

A seção 2 explicou **por que** os builds devem rodar fora do controller. As formas de ter agentes, da mais antiga à mais comum hoje:

| Tipo | Como funciona | Quando usar |
|---|---|---|
| **Permanente via SSH** | O controller entra por SSH em uma máquina fixa e inicia o agente | Máquinas com ferramentas específicas (um Mac para builds iOS, uma máquina com GPU) |
| **Permanente de entrada** (*inbound*) | O agente inicia a conexão com o controller (porta 50000 ou WebSocket) | Agentes atrás de firewall, que o controller não consegue alcançar |
| **Docker** | Cada build (ou etapa) roda dentro de um container | Ambientes de build reproduzíveis, sem instalar ferramentas no agente |
| **Kubernetes** | Cada build ganha um **Pod** novo no cluster, destruído no fim | Escala sob demanda, isolamento total entre builds |

Cada agente tem **labels**, e o pipeline pede o tipo de agente de que precisa por elas, sem saber qual máquina vai executá-lo:

```groovy
pipeline {
    agent { label 'linux && docker' }    // qualquer agente com as duas labels
    // ...
}
```

Os agentes **efêmeros** (Docker e Kubernetes) aproximam o Jenkins do modelo dos runners do GitHub Actions: cada build começa em um ambiente limpo, idêntico ao anterior, e não deixa rastros para o próximo. O [capítulo prático](03_jenkins_na_pratica.md) configura os dois.

---

## 9. Quando usar o Jenkins

| Situação | Mais adequado |
|---|---|
| Código no GitHub, pipelines comuns, equipe sem tempo para operar servidores | **GitHub Actions** |
| Código no GitLab, querendo uma plataforma única | **GitLab CI** |
| Pipelines que precisam alcançar sistemas da rede interna | **Jenkins** (ou runners auto-hospedados de uma plataforma) |
| Exigência de que código e artefatos não saiam da infraestrutura da empresa | **Jenkins** (ou GitLab auto-hospedado) |
| Vários sistemas de controle de versão, ou integrações com ferramentas antigas | **Jenkins** |
| A empresa já tem centenas de pipelines no Jenkins | **Jenkins**, com as boas práticas deste bloco |

O Jenkins não é "pior" nem "ultrapassado". Ele troca **conveniência** (alguém opera para você) por **controle** (você decide onde roda, o que acessa e como se integra). Essa troca é a mesma entre IaaS e PaaS do [capítulo de fundamentos de infraestrutura](../../01-infraestrutura/fundamentos/01_o_que_e_infraestrutura.md), aplicada à automação.

---

## Próximo passo

Este capítulo montou o mapa: controller e agentes, executores e workspaces, o `JENKINS_HOME`, os tipos de job, plugins e credenciais. O pipeline de exemplo da seção 5.2 tinha três etapas que só faziam `echo`.

O [02_jenkinsfile_e_pipelines.md](02_jenkinsfile_e_pipelines.md) apresenta a linguagem do Jenkinsfile por completo: agentes, variáveis de ambiente, credenciais, condições, paralelismo, parâmetros, aprovações manuais e as ações de pós-execução, com a correspondência de cada conceito no GitHub Actions.
