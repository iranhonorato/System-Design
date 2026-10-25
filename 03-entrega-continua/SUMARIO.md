# Sumário: Entrega Contínua

Esta pasta trata de como tirar as pessoas do caminho entre o código e a produção. As partes anteriores do repositório ensinaram a empacotar (Docker), orquestrar (Kubernetes) e hospedar (AWS) uma aplicação, sempre com alguém digitando os comandos. Aqui, cada mudança passa a ser verificada, empacotada e implantada automaticamente. Ela tem três partes:

- **`ci/`**: **integração contínua**. Como garantir, a cada mudança, que o código continua correto, com um pipeline no GitHub Actions.
- **`cd/`**: **entrega e implantação contínuas**. Como levar o artefato validado até produção com segurança: princípios, estratégias de deploy e o pipeline completo.
- **`jenkins/`**: o mesmo CI/CD no **Jenkins**, o servidor de automação auto-hospedado mais usado em empresas.

---

## Trilha de Leitura

```
pré-requisitos: ../02-containers/ e ../01-infraestrutura/aws/
         │
         ▼
CI: VERIFICAR CADA MUDANÇA
  ① ci/01_fundamentos_de_integracao_continua.md
  ② ci/02_github_actions_na_pratica.md
  ③ ci/03_qualidade_e_seguranca_no_pipeline.md
         │
         ▼
CD: LEVAR ATÉ PRODUÇÃO
  ④ cd/01_entrega_e_implantacao_continua.md
  ⑤ cd/02_estrategias_de_deploy.md
  ⑥ cd/03_pipeline_de_cd_na_pratica.md
         │
         ▼
JENKINS: O MESMO, EM UM SERVIDOR PRÓPRIO
  ⑦ jenkins/01_fundamentos_do_jenkins.md
  ⑧ jenkins/02_jenkinsfile_e_pipelines.md
  ⑨ jenkins/03_jenkins_na_pratica.md
```

Os três blocos usam o **mesmo projeto**: a API FastAPI da loja, com testes unitários e de integração, a imagem multi-stage de [../02-containers/docker/04_compose_e_producao.md](../02-containers/docker/04_compose_e_producao.md) e os manifestos de [../02-containers/kubernetes/04_do_compose_ao_cluster.md](../02-containers/kubernetes/04_do_compose_ao_cluster.md). Assim, o bloco de Jenkins pode ser lido como uma tradução direta do que foi feito no GitHub Actions.

O bloco de **CI** depende só de Docker e Git; ele pode ser lido antes da parte de Kubernetes. O de **CD** usa a EC2, o IAM (OIDC) e o Kubernetes. O de **Jenkins** assume os dois anteriores.

---

## ① [ci/01_fundamentos_de_integracao_continua.md](ci/01_fundamentos_de_integracao_continua.md)

**Pré-requisito:** Git básico e [../02-containers/docker/02_imagens_e_dockerfile.md](../02-containers/docker/02_imagens_e_dockerfile.md)

O **inferno da integração** e o custo de descobrir erros tarde. A definição de CI como **prática de equipe**, e não como ferramenta, e as práticas que a sustentam. Estratégias de branches (*trunk-based* vs branches longas) e o pull request como ponto de controle. A **anatomia de um pipeline** (*fail fast*, artefato), a pirâmide de testes, análise estática, as regras de convivência (a `main` sempre verde, pipelines rápidos) e as ferramentas do mercado, com a tabela de termos equivalentes entre GitHub Actions, GitLab CI e Jenkins.

> **Problema que deixa em aberto:** como implementar tudo isso?

---

## ② [ci/02_github_actions_na_pratica.md](ci/02_github_actions_na_pratica.md)

**Pré-requisito:** ①

Constrói, camada por camada, o pipeline de CI de uma API FastAPI: workflows, eventos, jobs, steps e runners; o primeiro workflow e uma falha proposital; **proteção da `main`**; jobs paralelos, `needs`, matriz e cache; **testes de integração com Postgres** em *service containers*; cobertura e artifacts; **build da imagem** com cache; e segurança do workflow (secrets, permissões mínimas, PRs de forks, actions fixadas por hash). Fecha com o `ci.yml` completo.

> **Problema que deixa em aberto:** os testes merecem confiança? O código e a imagem são seguros? O pipeline vai continuar rápido?

---

## ③ [ci/03_qualidade_e_seguranca_no_pipeline.md](ci/03_qualidade_e_seguranca_no_pipeline.md)

**Pré-requisito:** ②

O que a **cobertura** mede (e o que não mede), cobertura de ramos, das linhas alteradas e teste de mutação. **Testes instáveis**: causas e tratamento. Checagem de tipos e **pre-commit** (*shift left*). **DevSecOps**: SAST, análise de dependências, busca de segredos e scan de imagem, com SBOM. **Fitness functions** de arquitetura no pipeline. Técnicas para manter o pipeline rápido e a tabela de **portões de qualidade**.

> **Problema que deixa em aberto:** a imagem validada ainda não foi para lugar nenhum.

---

## ④ [cd/01_entrega_e_implantacao_continua.md](cd/01_entrega_e_implantacao_continua.md)

**Pré-requisito:** ③

O "dia do deploy" e por que deploys raros são arriscados. A diferença entre **entrega contínua** e **implantação contínua**. O **pipeline de implantação**, a regra **construa uma vez, implante em todo lugar** e a configuração no ambiente. Ambientes e paridade, ambientes efêmeros, a separação entre **implantar e liberar**, rollback vs roll-forward, implantabilidade como característica da arquitetura e as **métricas DORA**.

> **Problema que deixa em aberto:** como trocar a versão em produção sem derrubar o sistema?

---

## ⑤ [cd/02_estrategias_de_deploy.md](cd/02_estrategias_de_deploy.md)

**Pré-requisito:** ④. Os exemplos em Kubernetes assumem [../02-containers/kubernetes/02_workloads_rede_e_escala.md](../02-containers/kubernetes/02_workloads_rede_e_escala.md).

O raio de explosão de uma mudança. **Recreate**, **rolling update**, **blue-green** (com a troca de selector de um Service) e **canary** (com Argo Rollouts e análise de métricas), canary vs teste A/B. O banco de dados como problema compartilhado e o padrão **expand and contract**. **Feature flags** e seus tipos, verificação do deploy (health checks, smoke tests, métricas) e a tabela comparativa.

> **Problema que deixa em aberto:** escrever o pipeline que faz tudo isso.

---

## ⑥ [cd/03_pipeline_de_cd_na_pratica.md](cd/03_pipeline_de_cd_na_pratica.md)

**Pré-requisito:** ⑤, ②, [../01-infraestrutura/aws/01_ec2_servidores_na_nuvem.md](../01-infraestrutura/aws/01_ec2_servidores_na_nuvem.md). As partes 3 e 4 assumem o IAM e o Kubernetes.

O pipeline de CD no GitHub Actions, em quatro versões: **publicação** da imagem com a tag do commit (GHCR e ECR); deploy na **EC2 por SSH**, com usuário dedicado, environments, smoke test e rollback (e o problema da porta 22); deploy na **EC2 sem chaves**, com OIDC e Session Manager; e deploy no **Kubernetes**, por push (`kubectl`) e por pull (**GitOps** com Argo CD). Fecha com o pipeline completo de **homologação → aprovação → produção**, um roteiro de diagnóstico e um checklist.

> **Problema que deixa em aberto:** e quando a automação precisa rodar em um servidor próprio, dentro da rede da empresa?

---

## ⑦ [jenkins/01_fundamentos_do_jenkins.md](jenkins/01_fundamentos_do_jenkins.md)

**Pré-requisito:** ① e o bloco de CD.

O que é o Jenkins, sua história e o que muda em relação a uma plataforma hospedada. A arquitetura **controller e agentes**, executores, workspace, o caminho de um build e por que não executar builds no controller. O `JENKINS_HOME`, a instalação com Docker, os conceitos da interface, os **tipos de job** (Freestyle, Pipeline, Multibranch, Organization Folder), **plugins** e seus riscos, **credenciais** e **agentes**. Fecha com quando usar o Jenkins.

> **Problema que deixa em aberto:** como escrever os pipelines como código?

---

## ⑧ [jenkins/02_jenkinsfile_e_pipelines.md](jenkins/02_jenkinsfile_e_pipelines.md)

**Pré-requisito:** ⑦, ②

A linguagem do **Jenkinsfile**: declarativo vs scripted, a estrutura de um pipeline, `agent` (inclusive em containers), `steps`, variáveis e **credenciais** (e a armadilha da interpolação do Groovy), `options`, `parameters`, `triggers`, `when`, `parallel` e `matrix`, **`input`** para aprovações (sem prender executores), `post` (FAILURE vs UNSTABLE), `stash`, **Shared Libraries** e a tabela de equivalências com o GitHub Actions.

> **Problema que deixa em aberto:** montar o CI/CD completo da loja no Jenkins.

---

## ⑨ [jenkins/03_jenkins_na_pratica.md](jenkins/03_jenkins_na_pratica.md)

**Pré-requisito:** ⑧, ⑥. A seção 6 assume a parte de Kubernetes.

O **Jenkins como código** (imagem com plugins e **Configuration as Code**), um agente com Docker separado do controller (e o custo de montar o socket), o **Multibranch Pipeline** ligado ao GitHub por webhook e com status nos PRs, o **Jenkinsfile completo** da loja (verificações em paralelo, integração com Postgres, publicação, homologação, aprovação e produção com rollback), **agentes efêmeros no Kubernetes**, **segurança e operação**, diagnóstico e a comparação final entre Jenkins, GitHub Actions e GitLab CI.

> **Continua em:** observabilidade, a próxima parte do repositório.

---

## Referência Rápida por Assunto

| Quero entender... | Leia |
|---|---|
| O que é integração contínua (de verdade) | ① |
| Trunk-based development e feature branches | ① |
| Pirâmide de testes e o que o pipeline verifica | ①, ③ |
| Escrever um workflow do GitHub Actions | ② |
| Testes de integração com banco no pipeline | ② |
| Proteger a `main` com checks obrigatórios | ② |
| Segurança de workflows (secrets, forks, actions) | ② |
| O que a cobertura de testes mede | ③ |
| Testes instáveis (*flaky*) | ③ |
| SAST, dependências vulneráveis, segredos e scan de imagem | ③ |
| Fitness functions de arquitetura | ③ |
| Deixar o pipeline mais rápido | ③ |
| Entrega contínua vs implantação contínua | ④ |
| Construir uma vez e promover entre ambientes | ④ |
| Métricas DORA | ④ |
| Blue-green, canary e rolling update | ⑤ |
| Migrações de banco sem quebrar o rollback | ⑤ |
| Feature flags | ⑤ |
| Deploy automático na EC2 (SSH ou Session Manager) | ⑥ |
| Deploy no Kubernetes com `kubectl` ou GitOps (Argo CD) | ⑥ |
| Homologação, aprovação e produção no mesmo pipeline | ⑥ |
| Arquitetura do Jenkins (controller, agentes, executores) | ⑦ |
| Tipos de job e plugins do Jenkins | ⑦ |
| Sintaxe do Jenkinsfile | ⑧ |
| Credenciais no Jenkins sem vazar no log | ⑧ |
| Aprovação manual no Jenkins | ⑧ |
| Jenkins configurado como código (JCasC) | ⑨ |
| Agentes Jenkins no Kubernetes | ⑨ |
| Segurança e operação de um Jenkins | ⑨ |
| Jenkins vs GitHub Actions vs GitLab CI | ⑦, ⑨ |
