# DevOps: da Infraestrutura à Entrega Contínua

Este repositório é um guia de estudo sobre **como uma aplicação sai do código e chega aos usuários, e continua funcionando depois**. Ele percorre, em uma trilha única e progressiva, tudo o que existe entre um `git commit` e um sistema em produção: a infraestrutura que sustenta o software, os containers que o empacotam, o Kubernetes que o orquestra, a nuvem da AWS em que ele roda e os pipelines de CI/CD que automatizam todo o caminho.

A proposta é explicar não só **o que** cada ferramenta faz, mas **por que ela existe**: qual problema concreto motivou sua criação e qual problema novo ela deixa para a próxima resolver. Cada capítulo parte de uma situação real, resolve-a com exemplos que podem ser executados na sua máquina e termina apontando a pergunta que o capítulo seguinte responde.

O conteúdo de **arquitetura de software** (estilos arquiteturais, escalabilidade, load balancer, Nginx, API Gateway, replicação de bancos, Redis, segurança, Auth0) fica no repositório irmão, [Arquitetura de Software](https://github.com/iranhonorato/Arquitetura-de-Software). Os capítulos daqui apontam para ele sempre que um conceito de lá é usado.

---

## Natureza do projeto

Este projeto não é uma aplicação de software em execução, e sim uma **base de conhecimento** em arquivos Markdown, organizada em pastas temáticas. Ela reúne:

- **infraestrutura**: as camadas física, lógica e de nuvem; a evolução do bare metal às máquinas virtuais e aos containers; Infraestrutura como Código;
- **Docker**: namespaces e cgroups, imagens e camadas, Dockerfile e cache de build, volumes, redes, Docker Compose, multi-stage builds e boas práticas de imagens de produção;
- **Kubernetes**: cluster, control plane, nodes, Pods, o loop de reconciliação, Deployments, Services, Ingress, probes, auto-scaling, ConfigMaps, Secrets, volumes persistentes, StatefulSets, RBAC, NetworkPolicy, a migração de um sistema do Compose para um cluster e o diagnóstico de erros;
- **AWS**: servidores EC2 e acesso por SSH, redes com VPC (CIDR, sub-redes, rotas, Internet e NAT Gateway, Security Groups e NACLs) e permissões com IAM (políticas, roles, STS, OIDC, Pod Identity no EKS);
- **integração contínua**: a prática, a anatomia de um pipeline, GitHub Actions na prática, testes de integração com banco real, portões de qualidade e segurança (SAST, dependências, segredos, scan de imagem) e fitness functions de arquitetura;
- **entrega e implantação contínuas**: o pipeline de implantação, artefato único promovido entre ambientes, estratégias de deploy (rolling, blue-green, canary), migrações de banco compatíveis, feature flags, deploy na EC2 e no Kubernetes, GitOps e métricas DORA;
- **Jenkins**: arquitetura de controller e agentes, Jenkinsfile, Jenkins configurado como código, agentes efêmeros no Kubernetes, segurança e operação.

A ideia central é construir uma visão integral: entender como cada camada se apoia na anterior, e como, juntas, elas permitem que uma equipe entregue software com frequência, segurança e pouco esforço manual.

---

## Objetivo

Fornecer uma trilha de aprendizagem clara para quem deseja entender:

- o que sustenta um software antes de qualquer linha de código rodar, e por que a indústria chegou aos containers;
- como empacotar uma aplicação de forma que ela rode igual em qualquer lugar, e como operar muitas delas em muitas máquinas;
- como a rede e as permissões da nuvem protegem um sistema em produção;
- como automatizar o caminho do código até a produção, com verificações em cada etapa e recuperação rápida quando algo dá errado.

Não é preciso conhecimento prévio de infraestrutura ou DevOps: o primeiro capítulo parte do zero, e cada leitura seguinte assume apenas o vocabulário das anteriores. Ajuda ter familiaridade com a linha de comando, Git e alguma linguagem de programação (os exemplos usam Python/FastAPI).

---

## Estrutura do conteúdo

O repositório está dividido em pastas numeradas. Dentro de cada uma, os capítulos também são numerados, e cada pasta tem um `SUMARIO.md` com a ordem de leitura, o resumo de cada capítulo e uma tabela de referência rápida por assunto.

```
.
├── README.md
│
├── 01-infraestrutura/                         ── o que sustenta o software
│   ├── SUMARIO.md
│   ├── fundamentos/
│   │   ├── 01_o_que_e_infraestrutura.md          camadas física, lógica e cloud; IaaS, PaaS e SaaS
│   │   └── 02_virtualizacao_containers_e_iac.md  bare metal → VM → container; Terraform e Ansible
│   └── aws/
│       ├── 01_ec2_servidores_na_nuvem.md         criar, acessar e preparar um servidor; custos
│       ├── 02_vpc_redes_na_aws.md                CIDR, sub-redes, rotas, NAT, Security Groups
│       └── 03_iam_identidades_e_permissoes.md    políticas, roles, STS, OIDC, EKS + IAM
│
├── 02-containers/                             ── empacotar e orquestrar
│   ├── SUMARIO.md
│   ├── docker/
│   │   ├── 01_arquitetura_e_isolamento.md        daemon, namespaces, cgroups, portas
│   │   ├── 02_imagens_e_dockerfile.md            camadas, Dockerfile, cache, tags
│   │   ├── 03_volumes_e_redes.md                 persistência e comunicação entre containers
│   │   └── 04_compose_e_producao.md              Compose, multi-stage, boas práticas, comandos
│   └── kubernetes/
│       ├── 01_fundamentos_e_arquitetura.md       cluster, control plane, nodes, Pods
│       ├── 02_workloads_rede_e_escala.md         Deployment, Service, Ingress, probes, HPA
│       ├── 03_configuracao_estado_e_seguranca.md ConfigMap, Secret, PVC, StatefulSet, RBAC
│       └── 04_do_compose_ao_cluster.md           migração completa, diagnóstico, EKS
│
├── 03-entrega-continua/                       ── automatizar o caminho até a produção
│   ├── SUMARIO.md
│   ├── ci/
│   │   ├── 01_fundamentos_de_integracao_continua.md  a prática, o pipeline, a pirâmide de testes
│   │   ├── 02_github_actions_na_pratica.md           o pipeline de CI completo, passo a passo
│   │   └── 03_qualidade_e_seguranca_no_pipeline.md   cobertura, flaky tests, DevSecOps
│   ├── cd/
│   │   ├── 01_entrega_e_implantacao_continua.md      princípios, ambientes, DORA
│   │   ├── 02_estrategias_de_deploy.md               rolling, blue-green, canary, feature flags
│   │   └── 03_pipeline_de_cd_na_pratica.md           EC2 (SSH e SSM), Kubernetes, GitOps
│   └── jenkins/
│       ├── 01_fundamentos_do_jenkins.md              controller, agentes, jobs, plugins
│       ├── 02_jenkinsfile_e_pipelines.md             a linguagem do Jenkinsfile
│       └── 03_jenkins_na_pratica.md                  JCasC, Multibranch, o CI/CD completo
│
└── 04-observabilidade/                        ── (em preparação)
```

---

## Como utilizar

### A trilha completa

A numeração das pastas indica os grandes blocos, mas a **ordem de leitura recomendada** alterna entre eles em um ponto: a parte de AWS (dentro de `01-infraestrutura/`) é lida **depois** dos containers, porque a EC2 roda o Compose dos capítulos de Docker, e os capítulos de VPC e IAM terminam tratando do EKS, o Kubernetes da AWS.

```
 ① Fundamentos de infraestrutura    01-infraestrutura/fundamentos/   (2 capítulos)
          │
          ▼
 ② Docker                           02-containers/docker/            (4 capítulos)
          │
          ▼
 ③ Kubernetes                       02-containers/kubernetes/        (4 capítulos)
          │
          ▼
 ④ AWS: EC2, VPC, IAM               01-infraestrutura/aws/           (3 capítulos)
          │
          ▼
 ⑤ Integração contínua              03-entrega-continua/ci/          (3 capítulos)
          │
          ▼
 ⑥ Entrega contínua                 03-entrega-continua/cd/          (3 capítulos)
          │
          ▼
 ⑦ Jenkins                          03-entrega-continua/jenkins/     (3 capítulos)
```

Comece por [01-infraestrutura/fundamentos/01_o_que_e_infraestrutura.md](01-infraestrutura/fundamentos/01_o_que_e_infraestrutura.md). Cada capítulo termina com uma seção **Próximo passo**, que aponta exatamente o capítulo seguinte dessa trilha, de modo que é possível percorrê-la inteira sem voltar a este README.

### Caminhos mais curtos

Nem todo objetivo exige a trilha completa:

| Objetivo | Caminho |
|---|---|
| Entender containers do zero | ① → ② |
| Colocar um sistema com Docker Compose em produção, sem Kubernetes | ① → ② → [EC2](01-infraestrutura/aws/01_ec2_servidores_na_nuvem.md) → ⑤ → [pipeline de CD](03-entrega-continua/cd/03_pipeline_de_cd_na_pratica.md) (partes 1 a 3) |
| Aprender Kubernetes, já sabendo Docker | ③ |
| Entender rede e permissões na AWS | [EC2](01-infraestrutura/aws/01_ec2_servidores_na_nuvem.md) → [VPC](01-infraestrutura/aws/02_vpc_redes_na_aws.md) → [IAM](01-infraestrutura/aws/03_iam_identidades_e_permissoes.md) (as seções sobre EKS podem ficar para depois) |
| Montar CI/CD com GitHub Actions | ⑤ → ⑥ |
| Trabalhar com Jenkins | [CI: fundamentos](03-entrega-continua/ci/01_fundamentos_de_integracao_continua.md) → ⑥ → ⑦ |

### Consultando por assunto

Cada `SUMARIO.md` termina com uma tabela **"Quero entender... → Leia"**:

- [01-infraestrutura/SUMARIO.md](01-infraestrutura/SUMARIO.md)
- [02-containers/SUMARIO.md](02-containers/SUMARIO.md)
- [03-entrega-continua/SUMARIO.md](03-entrega-continua/SUMARIO.md)

---

## Como os capítulos são escritos

Todos os capítulos seguem o mesmo formato, para que a leitura seja previsível:

- **Cabeçalho** com os pré-requisitos e as fontes usadas.
- **"Por que este arquivo existe"**: o problema concreto que o capítulo resolve, normalmente deixado em aberto pelo anterior.
- **Seções numeradas**, do concreto ao abstrato, com **diagramas**, **tabelas comparativas** e **exemplos executáveis**: comandos com a saída esperada, Dockerfiles, manifestos, workflows e Jenkinsfiles completos.
- **"Confusão comum → Mais preciso"**: blocos que destacam os erros de entendimento mais frequentes sobre cada tema, aqueles que normalmente só aparecem quando algo quebra em produção. Por exemplo: *"o pipeline está verde, então o código está correto"*, *"um container é uma VM mais leve"*, *"base64 é criptografia"*.
- **"Próximo passo"**: a pergunta que o capítulo deixa em aberto e o capítulo que a responde.

Um mesmo sistema atravessa a trilha inteira: a **API de uma loja virtual** em FastAPI, com Postgres e Redis. Ele começa como um container, vira um Compose, migra para o Kubernetes, roda em uma EC2 e em um cluster EKS, e termina sendo testado e implantado automaticamente por pipelines no GitHub Actions e no Jenkins.

---

## Referências

Onde o assunto é tratado por elas, o material se apoia em três referências centrais de arquitetura de software:

- *Fundamentals of Software Architecture*, de Mark Richards e Neal Ford: características operacionais, implantabilidade e *fitness functions*;
- *Software Architecture: The Hard Parts*, de Neal Ford, Mark Richards, Pramod Sadalage e Zhamak Dehghani: modularidade, implantação independente e o padrão sidecar;
- *System Design Interview*, de Alex Xu: o caminho de uma requisição, a arquitetura em camadas e a replicação entre data centers.

Para os temas operacionais que esses livros não cobrem em detalhe (Docker, Kubernetes, AWS, GitHub Actions, Jenkins), o conteúdo segue as **documentações oficiais** de cada ferramenta e referências consolidadas da área, como *Continuous Delivery* (Jez Humble e David Farley), os textos de Martin Fowler sobre integração contínua e o programa de pesquisa DORA. Cada capítulo cita, no cabeçalho, as fontes que usou.

As versões de ferramentas, imagens e actions citadas nos exemplos refletem o estado delas em outubro de 2026. Elas mudam com frequência: antes de copiar um exemplo para um projeto real, confira a versão atual.

---

## Conclusão

Este repositório tenta transformar um conjunto de ferramentas que costuma ser aprendido solto e fora de ordem (Docker, Kubernetes, AWS, CI/CD, Jenkins) em uma trilha coesa, em que cada capítulo explica o problema que motivou o próximo. Ao final, a mesma linha de código percorre todo o caminho: entra na `main` por um pull request verificado em minutos, vira uma imagem com uma tag única, é implantada em homologação e em produção com uma estratégia segura e rollback automático, e roda em um cluster dentro de uma rede privada, acessando a nuvem com permissões mínimas, tudo descrito em arquivos versionados.
