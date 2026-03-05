# Continuous Integration e Continuous Delivery/Deployment (CI/CD)

## 1. Contextualização: Por que CI/CD surgiu? 

Antes de falarmos das definições, precisamos entender o problema. 

Em modelos tradicionais de desenvolvimento (como o modelo cascata):

* O desenvolvimento era longo 
* A integração acontecia apenas no final 
* Bugs eram descobertos tardiamente
* O  deploy era manual e arriscado 
* Atualizações eram raras 

**Resultado:**

* Alto risco
* Retrabalho
* Falhas em produção
* Times sobrecarregados

Com o avanço do Agile e posteriormente do DevOps, surgiu a necessidade de:

* Entregar software mais rapidamente
* Reduzir riscos
* Automatizar processos
* Garantir qualidade contínua

É nesse contexto que surgem CI e CD.

## 2. Como CI e CD resolvem esses problemas? 

2.1 Problema: Integração Tardia
---

Antes os desenvolvedores trabalhavam semanas ou meses isoladamente. Quando finalmente integravam o código, surgiam inúmeros conflitos. Isso gerava o famoso “Inferno de Integração”

**E Como a CI resolve isso:**

* Continuous Integration força integração frequente.
* Commits pequenos e frequentes
* Integração várias vezes ao dia
* Pipeline executado automaticamente

**Resultado:**

* Conflitos são pequenos e resolvidos rapidamente
* Problemas aparecem no mesmo dia
* O código principal (main/master) permanece estável

A CI transforma a integração de um evento traumático em uma rotina diária.


2.2 Problema: Descoberta Tardia de Bugs
---

Antes os testes eram manuais. Bugs descobertos apenas na homologação ou, pior, em produção. Quanto mais tarde o bug é encontrado, maior o custo de correção.

**Como a CI resolve isso:**

A CI executa automaticamente:

* Testes unitários
* Testes de integração
* Análise estática
* Lint
* Verificação de segurança
* Cada commit é validado automaticamente.

**Resultado:**

* Feedback imediato
* Redução drástica do custo de correção
* Código mais confiável

CI transforma qualidade em algo contínuo, não uma fase do projeto.

2.3 Problema: Deploy Manual e Arriscado
---

Antes o deploy dependia de:

* Scripts manuais
* Acesso SSH
* Configuração feita "na mão"
* Conhecimento tácito de um membro da equipe
* Altíssimo risco.

**Como CD resolve isso:**

Continuous Delivery/Deployment automatiza:

* Build do artefato
* Empacotamento
* Publicação
* Deploy em ambientes
* Tudo via pipeline.

Exemplo:

``Commit → Testes → Build Docker → Deploy automático``

**Resultado:**

* Processo repetível
* Sem erro humano
* Redução de risco
* Deploy deixa de ser um evento estressante

Deploy vira rotina técnica, não evento dramático.


2.4 Problema: Ambientes Diferentes (Dev ≠ Prod)
---

Antes existia o famoso dilema: "Funciona na minha máquina mas falha em produção."

Causa comum:

* Dependências diferentes
* Versões diferentes
* Configurações manuais

**Como CI/CD resolve isso:**

Com:

* Docker
* Infraestrutura como código (IaC)
* Pipelines automatizados
* O mesmo artefato testado é o que vai para produção.

**Resultado:**

* Padronização de ambientes
* Redução de inconsistências
* Mais previsibilidade

Arquiteturalmente, isso reduz variabilidade — um fator crítico em sistemas complexos.

2.5 Problema: Releases Lentas
---

Antes os releases eram trimestrais ou semestrais. Grandes pacotes de mudanças.

``Grande pacote = grande risco.``

**Como CD resolve isso:**

Com entrega contínua:

* Releases pequenas
* Mudanças incrementais
* Deploy frequente

**Resultado:**

* Redução de risco por mudança
* Feedback rápido do usuário
* Capacidade de adaptação ao negócio

A arquitetura passa a suportar evolução contínua.

2.6 Problema: Falta de Governança Arquitetural
---

Em arquiteturas tradicionais, decisões arquiteturais eram violadas ao longo do tempo.

**Como CI/CD ajuda:**

Com:

* Testes automatizados
* Fitness functions
* Verificações automatizadas de conformidade

Podemos:

* Garantir que regras arquiteturais sejam testadas
* Automatizar compliance
* Detectar degradação estrutural

Isso conecta CI/CD diretamente com governança arquitetural.


2.7 Problema: Alto Risco em Produção
---

Grandes mudanças → grandes falhas.

**Como CD reduz risco:**

Com estratégias como:

* Blue-Green Deployment
* Canary Releases
* Rolling Updates
* Feature Toggles

**Permite:**

* Testar com poucos usuários
* Fazer rollback rápido
* Reduzir impacto de falhas

CI/CD reduz o MTTR (Mean Time to Recovery), uma das métricas DORA.

## 3. Continuous Integration (CI)

**Definição:** Continuous Integration (Integração Contínua) é a prática de integrar alterações de código frequentemente (várias vezes ao dia) em um repositório compartilhado, executando testes automáticos a cada integração.

**Objetivos da CI:**

* Detectar erros rapidamente
* Evitar conflitos grandes de merge
* Garantir que o código sempre esteja em estado funcional
* Automatizar validações

**Como funciona a CI na prática (Fluxo típico):**

```Plain
Desenvolvedor (cria feature)
        │
        │ git commit / git push
        ▼
Repositório Remoto (GitHub / GitLab / Bitbucket)
        │
        │ Webhook automático
        ▼
Servidor de CI (GitHub Actions / GitLab CI / Jenkins)
        │
        │ Pipeline iniciado
        ▼
Build do Projeto
        │
        ▼
Testes Automatizados
        │
        ▼
Análise Estática de Código
        │
        ▼
Resultado do Pipeline
        │
        ├── ✔ Sucesso → Código aceito / Merge permitido
        │
        └── ✖ Falha → Desenvolvedor corrige e envia novo commit
```

**Elementos da CI**

* Repositório de Código (Git): Centraliza o versionamento.

* Pipeline: Sequência automatizada de etapas (Exemplo: ``Build → Test → Lint → Package``)

* Servidor de CI: Conta com ferramentas como Jenkins, GitHub Actions, GitLab CI, Azure DevOps, CircleCI


## 4. Continuous Delivery (CD)


**Definição:** Continuous Delivery é a prática de manter o software sempre pronto para ser publicado em produção, com deploy automatizado até ambientes de homologação ou staging. A liberação final ainda pode depender de aprovação manual.


**Objetivos:**

* Reduzir risco de deploy
* Tornar releases previsíveis
* Automatizar entrega
* Permitir deploy a qualquer momento


**Como funciona a CD na prática (Fluxo típico):**

```Plain
Desenvolvedor (nova feature ou correção)
        │
        │ git commit / git push
        ▼
Repositório Remoto (GitHub / GitLab)
        │
        │ Webhook automático
        ▼
Servidor de CI/CD
        │
        ▼
Testes Automatizados
        │
        ▼
Build da Aplicação
        │
        ▼
Geração da Imagem Docker
        │
        ▼
Publicação da Imagem no Registry (Docker Hub / ECR / GCR)
        │
        ▼
Deploy Automático em Ambiente de Staging
        │
        ▼
Time de Negócio Valida Funcionalidades
        │
        ├── ✔ Aprovado
        │        ▼
        │   Deploy em Produção
        │
        └── ✖ Reprovado
                 ▼
        Correções → Novo Commit → Reinicia Pipeline
```


## 5. Continuous Deployment (também CD)

**Definição:** Continuous Deployment vai além da Delivery: Toda mudança aprovada na CI é automaticamente implantada em produção, sem intervenção humana.

| Conceito                   | Vai automaticamente para produção?      |
|----------------------------|-----------------------------------------|
| Continuous Delivery        | Não (precisa aprovação)                 |
| Continuous Deployment      | Sim                                     |

## 6. Pipeline CI/CD Completo

```Plain 
Developer → Git Push →

   → CI:
      - Build
      - Testes unitários
      - Análise estática


   → CD:
      - Build de artefato
      - Testes de integração
      - Testes E2E
      - Deploy staging
      - Deploy produção
```

## 7. Problemas Comuns em CI/CD

* Pipeline lento
* Testes instáveis (flaky tests)
* Dependência de ambiente
* Falta de rollback
* Cultura resistente à automação


## 8. Boas Práticas

* Commits pequenos e frequentes
* Testes automatizados (unitários, integração, E2E)
* Pipeline rápido (ideal < 10 minutos)
* Infraestrutura como código (IaC)
* Versionamento semântico
* Feature toggles
* Deploy incremental (Blue-Green, Canary)