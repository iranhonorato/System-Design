# Continuous Integration e Continuous Delivery/Deployment (CI/CD)

## 1. O Problema que o CI/CD Resolve

Para entender CI/CD, precisamos primeiro entender o caos que existia antes.

### O modelo de desenvolvimento tradicional

Em times que não praticam CI/CD, o fluxo típico era:

```
Dev A trabalha sozinho por 3 semanas →
Dev B trabalha sozinho por 3 semanas →
Dev C trabalha sozinho por 3 semanas →
                    ↓
            SEMANA DE INTEGRAÇÃO
         (também conhecida como inferno)
                    ↓
   Conflitos de merge, código quebrado,
   bugs impossíveis de rastrear, stress total
```

Cada desenvolvedor acumulava mudanças em sua própria branch por semanas. Na hora de juntar tudo, os conflitos eram tantos e tão complexos que o processo levava dias, resultava em bugs novos e consumia a energia de todo o time.

Além disso, o deploy era um evento manual, raro (mensal ou trimestral) e extremamente arriscado. Uma frase resumia o sentimento: **"dia de deploy é dia de terror"**.

### Como CI/CD transforma esse cenário

CI/CD substitui esse ciclo de releases raras e arriscadas por um fluxo de **integração e entrega contínuos**. O código vai do desenvolvedor para produção de forma automatizada, frequente e confiável.

---

## 2. Os Três Conceitos

### 2.1 Continuous Integration (Integração Contínua)

**O que é:** a prática de integrar código no repositório compartilhado **várias vezes ao dia**, com cada integração sendo verificada automaticamente por um conjunto de testes e validações.

**O princípio central:** erros são muito mais fáceis de corrigir quando são descobertos **no mesmo dia em que foram criados** — não semanas depois.

**Fluxo da CI:**

```
Desenvolvedor escreve código
        │
        │  git push
        ▼
Repositório (GitHub, GitLab, Bitbucket)
        │
        │  Webhook dispara automaticamente
        ▼
Servidor de CI (GitHub Actions, Jenkins, GitLab CI)
        │
        ├─→ Build do projeto
        ├─→ Testes unitários
        ├─→ Testes de integração
        ├─→ Análise de qualidade de código (lint)
        └─→ Verificação de segurança (SAST)
        │
        ├── ✅ Sucesso → merge liberado, equipe notificada
        └── ❌ Falha  → desenvolvedor corrige antes do merge
```

**Regra de ouro da CI:** o branch principal (`main`/`master`) deve estar **sempre em estado funcional e deployável**. Se você quebrou o build, corrigir isso é prioridade máxima.

### 2.2 Continuous Delivery (Entrega Contínua)

**O que é:** extensão da CI onde, além de validar o código, o pipeline também **prepara automaticamente um artefato pronto para produção** e o deploya em um ambiente de staging/homologação. A decisão de ir para produção ainda é humana.

A diferença crucial: em Continuous Delivery, **você poderia fazer o deploy para produção a qualquer momento** — o software está sempre pronto. Só não vai automaticamente porque existe uma decisão de negócio envolvida.

**Cenário típico:** o time de produto valida a funcionalidade em staging e, quando estiver satisfeito, clica em "Deploy para Produção" com confiança — sabendo que o mesmo artefato já foi testado.

### 2.3 Continuous Deployment (Deploy Contínuo)

**O que é:** vai além da Delivery — **todo commit que passa nos testes vai automaticamente para produção**, sem intervenção humana.

| | Vai automaticamente para produção? | Quando usar |
|---|---|---|
| **Continuous Delivery** | Não (aprovação manual) | Produto com processos de compliance, aprovação de negócio |
| **Continuous Deployment** | Sim | Plataformas digitais com testes robustos e cultura de feature flags |

> **Continuous Deployment não significa descuido** — significa tanto confiança nos testes automatizados que você remove a necessidade de aprovação manual. Empresas como Netflix, Amazon e GitHub fazem dezenas de deploys por dia usando esse modelo.

---

## 3. Por Que CI/CD Funciona: Análise dos Problemas

### 3.1 Integração Tardia → "Inferno de Integração"

**Problema:** desenvolvedores trabalhando em paralelo por semanas acumulam mudanças que conflitam umas com as outras. Na hora de juntar, o custo de resolução é enorme.

**Solução CI:** commits pequenos e frequentes. Se você integra todo dia, os conflitos são pequenos e resolvíveis em minutos, não dias.

### 3.2 Bugs Descobertos Tarde

**Problema:** em modelos tradicionais, bugs eram descobertos em QA (semanas após serem criados) ou, pior, em produção. O custo de corrigir um bug cresce exponencialmente com o tempo.

```
Custo de correção por fase:
  Desenvolvimento:  1x
  Teste:           10x
  Staging:         25x
  Produção:       100x
```

**Solução CI:** testes automatizados executam a cada commit. O desenvolvedor que criou o bug é notificado em minutos e ainda tem o contexto fresco na cabeça.

### 3.3 Deploy Manual e Arriscado

**Problema:** deploys manuais dependem de scripts frágeis, conhecimento tácito de uma pessoa específica, acesso SSH direto a servidores de produção e etapas que variam a cada execução.

**Solução CD:** o deploy é uma série de etapas automatizadas, documentadas no pipeline. Qualquer pessoa do time pode acionar um deploy com confiança — e o processo é idêntico toda vez.

### 3.4 "Funciona na Minha Máquina"

**Problema:** ambiente de desenvolvimento diferente de staging, diferente de produção. Diferentes versões de Python, Node, bibliotecas de sistema, configurações de banco.

**Solução:** Docker + CI/CD. O mesmo container que passou nos testes vai para produção. Não há diferença de ambiente — o ambiente *é* o container.

### 3.5 Releases Grandes = Alto Risco

**Problema:** quanto mais mudanças em uma release, maior o risco. Se algo quebra, é difícil identificar qual mudança causou o problema.

**Solução CD:** releases pequenas e frequentes. Se um bug aparecer, você sabe que foi introduzido pelo último commit — não por uma das 200 mudanças da release trimestral.

---

## 4. Anatomia de um Pipeline CI/CD

Um pipeline é uma sequência de **stages** (etapas) que o código percorre. Cada stage é um conjunto de **jobs** (tarefas). Se qualquer stage falha, o pipeline para e o time é notificado.

```
┌──────────────────────────────────────────────────────────────┐
│                    PIPELINE CI/CD                            │
│                                                              │
│  Stage 1: Build          Stage 2: Test        Stage 3: CD   │
│  ┌─────────────┐         ┌─────────────┐      ┌──────────┐  │
│  │ Checkout    │         │ Unit Tests  │      │ Build    │  │
│  │ Compile     │   →     │ Integration │  →   │ Docker   │  │
│  │ Install deps│         │ Lint        │      │ Deploy   │  │
│  └─────────────┘         │ Security    │      │ Staging  │  │
│                          └─────────────┘      └──────────┘  │
└──────────────────────────────────────────────────────────────┘
```

### Tipos de testes no pipeline

**Testes Unitários:** testam uma função ou classe de forma isolada, sem dependências externas. São rápidos (milissegundos) e devem ser a maioria dos seus testes.

**Testes de Integração:** testam a interação entre componentes — ex: a API + banco de dados reais. São mais lentos mas validam o comportamento do sistema como um todo.

**Testes End-to-End (E2E):** simulam o comportamento do usuário real no sistema. Ex: "abra o browser, faça login, adicione produto ao carrinho, finalize compra". São os mais lentos e devem ser usados com moderação.

**Análise Estática (Lint):** ferramentas que analisam o código sem executá-lo. Detectam problemas de formatação, padrões perigosos, código morto. Ex: `flake8` (Python), `eslint` (JavaScript).

---

## 5. Ferramentas de CI/CD

### Servidores de CI/CD

| Ferramenta | Onde roda | Quando usar |
|---|---|---|
| **GitHub Actions** | Nuvem (GitHub) | Projetos no GitHub — mais simples de configurar |
| **GitLab CI** | Nuvem ou self-hosted | Projetos no GitLab — muito poderoso |
| **Jenkins** | Self-hosted | Ambientes corporativos com necessidades complexas |
| **CircleCI** | Nuvem | Pipelines rápidos e configuração simples |
| **Azure DevOps** | Nuvem (Microsoft) | Ecossistema Microsoft |

### Ferramentas complementares

| Tipo | Exemplos |
|---|---|
| **Registry de imagens Docker** | Docker Hub, AWS ECR, GitHub Container Registry |
| **Qualidade de código** | SonarQube, CodeClimate |
| **Segurança** | Snyk, Trivy (scan de vulnerabilidades em containers) |
| **Notificações** | Slack, Email, PagerDuty |

---

## 6. Estratégias de Deploy

O CD não significa apenas "jogar a nova versão para produção e torcer". Existem estratégias que reduzem o risco:

### Blue-Green Deployment

Mantém dois ambientes idênticos. O "azul" está em produção. O "verde" recebe a nova versão e é testado. Quando tudo está ok, o tráfego é redirecionado do azul para o verde instantaneamente.

```
Estado atual:
  → [Blue: v1.0] ← recebe 100% do tráfego
    [Green: v1.1] ← nova versão sendo preparada

Após validação:
    [Blue: v1.0] ← fica em standby (rollback rápido se necessário)
  → [Green: v1.1] ← recebe 100% do tráfego
```

**Vantagem:** rollback instantâneo — basta redirecionar de volta para o azul.

### Canary Release

A nova versão é liberada para uma pequena porcentagem dos usuários (ex: 5%). Se não aparecerem erros, o percentual aumenta gradualmente até 100%.

```
Semana 1: 5% dos usuários → nova versão
Semana 2: 20% dos usuários → nova versão
Semana 3: 100% dos usuários → nova versão
```

**Vantagem:** falhas afetam poucos usuários; rollback é simples.

### Rolling Update

Containers/instâncias são atualizados gradualmente, um de cada vez. Enquanto alguns já rodam a versão nova, outros ainda rodam a antiga.

---

## 7. Métricas DORA: Medindo a Eficiência do CI/CD

As **métricas DORA** (DevOps Research and Assessment) são os indicadores padrão da indústria para medir a saúde de um processo de entrega de software:

| Métrica | O que mede | Elite performers |
|---|---|---|
| **Deployment Frequency** | Quantas vezes por período você deploya | Várias vezes ao dia |
| **Lead Time for Changes** | Tempo do commit até produção | Menos de 1 hora |
| **Change Failure Rate** | % de deploys que causam incidentes | 0-15% |
| **MTTR** (Mean Time to Recovery) | Tempo para recuperar de uma falha | Menos de 1 hora |

---

## 8. Boas Práticas

**Commits pequenos e frequentes:** commits de 50-100 linhas são muito mais fáceis de revisar, testar e reverter do que commits de 2000 linhas.

**Pipeline rápido:** um pipeline lento desincentiva o uso. O ideal é menos de 10 minutos do push ao resultado.

**Trate o pipeline como código:** os arquivos de configuração do CI/CD (`.yml`, `Jenkinsfile`) ficam no repositório, são versionados e revisados como qualquer outro código.

**Nunca force pushes para main:** proteja o branch principal com regras: código só entra via Pull Request, e só depois que o pipeline verde.

**Feature toggles:** em vez de criar branches de feature de longa duração, use flags de feature. O código vai para produção mas desativado — você ativa quando estiver pronto. Elimina problemas de integração.

**Rollback deve ser tão simples quanto deploy:** se fazer rollback é complexo, você vai evitar fazê-lo mesmo quando deveria. O processo de reverter deve ser tão automatizado quanto o de avançar.
