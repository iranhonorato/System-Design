# Integração Contínua: Fundamentos

**Pré-requisito:** Git básico (branch, commit, merge, pull request) e os capítulos de Docker, especialmente [../../02-containers/docker/02_imagens_e_dockerfile.md](../../02-containers/docker/02_imagens_e_dockerfile.md).

> O Capítulo 6 de **Fundamentals of Software Architecture** (Mark Richards & Neal Ford) conta como a busca por automação do Extreme Programming criou a integração contínua, que levou à automação das operações (DevOps) e, depois, à governança automatizada da arquitetura. A definição de CI segue Martin Fowler (*Continuous Integration*, publicado em 2000 e reescrito em 2024). A pirâmide de testes é detalhada em [Arquitetura de Software: testes_de_software.md](https://github.com/iranhonorato/Arquitetura-de-Software/blob/main/system-design-pt1/testes_de_software.md).

---

## Por que este arquivo existe

As partes anteriores da trilha mostraram como **empacotar** uma aplicação (Docker), como **orquestrá-la** (Kubernetes) e **onde** ela roda (AWS). Em todas elas, alguém digitava os comandos: `docker build`, `docker push`, `kubectl apply`, `ssh` na EC2. Esta última parte trata de tirar as pessoas desse caminho, e ela começa antes do deploy, no momento em que o código de várias pessoas se junta.

A pergunta central deste capítulo é: *como garantir que o código de uma equipe inteira continue funcionando junto, todos os dias, sem um "dia da integração" doloroso no fim de cada ciclo?* A resposta tem um nome, **Integração Contínua** (CI, de *Continuous Integration*), e é antes de tudo uma **prática de equipe**. As ferramentas vêm depois.

A parte de entrega contínua tem três blocos:

| Bloco | Pergunta |
|---|---|
| **CI** (este e os dois próximos) | Como saber, a cada mudança, que o código está correto e pronto para ser entregue? |
| **CD** ([../cd/](../cd/01_entrega_e_implantacao_continua.md)) | Como levar esse código validado até produção, de forma segura e repetível? |
| **Jenkins** ([../jenkins/](../jenkins/01_fundamentos_do_jenkins.md)) | Como fazer tudo isso com o servidor de automação mais usado em ambientes corporativos? |

---

## 1. O problema: o inferno da integração

Imagine uma equipe de três pessoas desenvolvendo a loja virtual dos capítulos anteriores, sem nenhuma prática de integração:

```
Semana 1–3:  Ana    trabalha no carrinho     (branch: carrinho)
             Bruno  trabalha no pagamento    (branch: pagamento)
             Carla  refatora o cadastro      (branch: cadastro)
                                │
                                ▼
Semana 4:           "SEMANA DA INTEGRAÇÃO"
             merge de 3 branches com 3 semanas de mudanças cada
                                │
                                ▼
   ✘ 47 conflitos de merge         ✘ testes quebrados que ninguém sabe por quê
   ✘ a refatoração da Carla mudou a função que o Bruno usava
   ✘ "funcionava na minha branch"  ✘ o deploy atrasa uma semana
```

O problema não é que as pessoas trabalharam mal. É que **cada uma trabalhou contra uma versão do código que deixou de existir**. Enquanto Ana escrevia o carrinho, Carla mudava o cadastro que o carrinho usava, e nenhuma das duas sabia. Quanto mais tempo as branches ficam separadas, mais elas divergem, e o custo de juntá-las cresce **mais rápido que o tempo**: duas semanas de divergência não custam o dobro de uma, custam muito mais, porque os conflitos se combinam.

Há um segundo problema, igualmente caro: **o momento em que o erro é descoberto**. Um bug criado na segunda-feira e descoberto na mesma tarde é corrigido em minutos, porque quem o criou ainda lembra do que fez. O mesmo bug descoberto três semanas depois, misturado a centenas de outras mudanças, exige uma investigação. E se for descoberto em produção, ainda há o custo para os usuários.

```
Custo relativo de corrigir um defeito, pela fase em que é encontrado
(a ordem de grandeza é o que importa, não os números exatos)

  Enquanto escreve o código   ▏ 1x
  Na integração/testes        ▇▇▇▇ 10x
  Em homologação              ▇▇▇▇▇▇▇▇▇ 25x
  Em produção                 ▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇ 100x
```

A integração contínua ataca os dois problemas ao mesmo tempo: **integrar cedo** (para que as divergências sejam pequenas) e **verificar automaticamente cada integração** (para que os erros apareçam minutos depois de criados).

---

## 2. O que é integração contínua

Martin Fowler, que popularizou o termo, a define assim (em tradução livre): *uma prática de desenvolvimento em que cada membro da equipe integra seu trabalho frequentemente, no mínimo diariamente, e cada integração é verificada por um build automatizado, com testes, que detecta erros o mais rápido possível.*

A definição tem duas metades, e as duas são obrigatórias:

1. **Integrar com frequência**: o código de cada pessoa entra na linha principal (`main`) pelo menos uma vez por dia.
2. **Verificar cada integração automaticamente**: um sistema compila, testa e analisa o código a cada mudança, e avisa em minutos se algo quebrou.

### As práticas que sustentam a CI

| Prática | O que significa na prática |
|---|---|
| **Um repositório, uma linha principal** | Todo o código (e os testes, e o Dockerfile, e o pipeline) vive em um repositório, com uma branch principal que todos usam como base |
| **Integrar pelo menos uma vez por dia** | Branches de curta duração: horas ou poucos dias, nunca semanas |
| **Build automatizado** | Um único comando (ou o pipeline) constrói tudo do zero, sem passos manuais |
| **Build que se testa** | Os testes automatizados fazem parte do build; se um falha, o build falha |
| **Build rápido** | O resultado chega em minutos (a referência comum é **menos de 10 minutos**) |
| **A `main` está sempre verde** | Um build quebrado na linha principal é a prioridade número um de quem o quebrou |
| **Todos veem o resultado** | O estado do build é visível para a equipe inteira: no pull request, no chat, no painel |
| **O ambiente de testes imita o de produção** | Mesmas versões, mesma imagem de container. O Docker resolve boa parte disso |

> **Confusão comum:** "nós fazemos CI: temos o GitHub Actions rodando os testes em cada pull request". ✅ **Mais preciso:** ter um **servidor de CI** não é o mesmo que **praticar** CI. Se as branches de funcionalidade vivem três semanas e são integradas na `main` uma vez por mês, a equipe tem testes automatizados (o que é ótimo), mas continua sofrendo o inferno da integração da seção 1, só que com um robô assistindo. O "contínuo" da CI está na **frequência de integração**, e a ferramenta apenas torna essa frequência segura. O teste é simples: quantas vezes por dia o código de cada pessoa entra na `main`? Se a resposta for "algumas vezes por mês", ainda não é CI.

---

## 3. Como integrar com frequência: estratégias de branches

A forma como a equipe usa as branches decide se a CI é possível. Há dois modelos opostos.

### 3.1 Branches de longa duração

Modelos como o **Git Flow** usam várias branches permanentes (`main`, `develop`, `release/*`, `hotfix/*`) e branches de funcionalidade que podem durar semanas. Ele foi criado para softwares com versões lançadas de tempos em tempos (um aplicativo instalado, uma biblioteca). Para uma aplicação web atualizada continuamente, ele reintroduz os problemas da seção 1: muitas branches divergindo ao mesmo tempo.

### 3.2 Desenvolvimento baseado em tronco (*trunk-based development*)

Todos trabalham sobre a `main` (o "tronco"). As branches existem, mas são **curtas**: uma pessoa cria uma branch, faz uma mudança pequena, abre um pull request, o pipeline valida, alguém revisa, e a branch entra na `main` no mesmo dia ou no dia seguinte.

```
main  ──●─────●─────●─────●─────●─────●─────●──▶
         \   /       \   /       \   /
          ●─●         ●─●         ●─●         branches de horas ou 1–2 dias,
        (Ana)       (Bruno)     (Carla)       cada uma validada pelo CI
```

A pergunta natural é: *e se a funcionalidade levar três semanas para ficar pronta?* A resposta é dividi-la em pedaços que possam entrar na `main` sem quebrar nada, mesmo incompletos. As técnicas mais comuns:

- **Feature flags:** o código novo entra na `main` desligado por uma configuração, e é ligado quando estiver pronto. O capítulo de [estratégias de deploy](../cd/02_estrategias_de_deploy.md) mostra como funcionam.
- **Branch by abstraction:** para substituir um componente, cria-se primeiro uma interface comum, depois a implementação nova ao lado da antiga, e só no fim a troca.
- **Começar pelo que não aparece:** primeiro a tabela no banco e o endpoint sem uso, depois a lógica, por último a tela.

| | Branches longas (Git Flow) | Trunk-based |
|---|---|---|
| Duração das branches | Semanas | Horas a 2 dias |
| Tamanho dos merges | Grandes, com muitos conflitos | Pequenos, com poucos conflitos |
| Funcionalidade incompleta | Fica isolada na branch | Entra na `main` desligada (feature flag) |
| Combina com | Software com versões lançadas periodicamente | Aplicações web com entrega contínua |

### 3.3 O pull request como ponto de controle

No modelo com branches curtas, o **pull request** (PR) é o momento em que duas verificações acontecem antes de o código entrar na `main`:

1. **A automática:** o pipeline de CI roda e reporta o resultado no próprio PR.
2. **A humana:** alguém da equipe revisa o código.

A combinação é configurada como uma regra do repositório (a *branch protection*, detalhada no [próximo capítulo](02_github_actions_na_pratica.md)): nenhum código entra na `main` sem PR, sem revisão e sem o pipeline verde.

---

## 4. A anatomia de um pipeline de CI

Um **pipeline** é a sequência de etapas automatizadas que o código percorre a cada mudança. Todas as ferramentas organizam essas etapas da mesma forma, mesmo que com nomes diferentes:

```
  git push / pull request
          │
          ▼  ① GATILHO: o repositório avisa o servidor de CI (webhook)
┌─────────────────────────────────────────────────────────────────────┐
│  PIPELINE                                                           │
│                                                                     │
│  ② PREPARAR     checkout do código, instalar dependências           │
│        │                                                            │
│        ▼                                                            │
│  ③ VERIFICAR    lint e formatação ─┐                                │
│     (rápido)    testes unitários  ─┤  em paralelo, se possível      │
│                 checagem de tipos ─┘                                │
│        │                                                            │
│        ▼                                                            │
│  ④ CONSTRUIR    build da imagem Docker                              │
│        │                                                            │
│        ▼                                                            │
│  ⑤ VERIFICAR    testes de integração (com banco real em container)  │
│     (lento)     análises de segurança                               │
│        │                                                            │
│        ▼                                                            │
│  ⑥ PUBLICAR     o ARTEFATO validado (a imagem) vai para um registry │
└─────────────────────────────────────────────────────────────────────┘
          │
          ▼  ⑦ RESULTADO: ✔ verde (pode fazer merge) ou ✘ vermelho (corrija)
```

Três princípios explicam essa ordem:

**Falhar rápido (*fail fast*).** As verificações mais baratas e mais propensas a falhar vêm primeiro. Um erro de sintaxe deve ser apontado em 20 segundos pelo lint, e não depois de 8 minutos de build e testes de integração. Se o lint falha, o resto nem roda.

**Cada etapa confia na anterior.** Os testes de integração só rodam sobre um código que já passou nos unitários; a imagem só é publicada se tudo antes dela passou. Um artefato que chega ao fim do pipeline carrega a garantia de todas as etapas.

**O resultado é um artefato.** O produto final da CI não é um "ok", é algo **entregável**: uma imagem Docker com uma tag única (o hash do commit), guardada em um registry. Essa imagem é exatamente o que vai para homologação e produção na parte de CD, sem ser reconstruída. Essa regra, **construir uma vez e implantar muitas**, é o tema do [primeiro capítulo de CD](../cd/01_entrega_e_implantacao_continua.md).

### Os termos de cada ferramenta

As ferramentas usam nomes diferentes para as mesmas ideias. A tabela ajuda a reconhecer os conceitos no [GitHub Actions](02_github_actions_na_pratica.md) e no [Jenkins](../jenkins/01_fundamentos_do_jenkins.md):

| Conceito | GitHub Actions | GitLab CI | Jenkins |
|---|---|---|---|
| Arquivo do pipeline | `.github/workflows/*.yml` | `.gitlab-ci.yml` | `Jenkinsfile` |
| O pipeline inteiro | *workflow* | *pipeline* | *pipeline* |
| Um grupo de etapas | *job* | *stage* / *job* | *stage* |
| Uma etapa | *step* | *script* (linhas) | *step* |
| Onde roda | *runner* | *runner* | *agent* (*node*) |
| O que dispara | *event* (`on:`) | *rules* / *only* | *trigger* |
| Segredos | *secrets* | *CI/CD variables* | *credentials* |

---

## 5. O que o pipeline verifica

### 5.1 Testes, em camadas

Os testes automatizados são o coração da CI: sem eles, o pipeline só comprova que o código **compila**, não que **funciona**. Eles se organizam na conhecida **pirâmide de testes**:

```
                ▲
               ╱ ╲          E2E: o sistema inteiro, como o usuário usa
              ╱E2E╲         poucos · lentos (minutos) · frágeis
             ╱─────╲
            ╱ Inte- ╲       Integração: a API com um banco de verdade
           ╱ gração  ╲      alguns · médios (segundos)
          ╱───────────╲
         ╱  Unitários  ╲    Unitários: uma função ou classe, isolada
        ╱───────────────╲   muitos · rápidos (milissegundos) · estáveis
```

| Camada | Exemplo na loja | Tempo típico | Onde roda no pipeline |
|---|---|---|---|
| **Unitário** | `calcular_frete(cep, peso)` devolve o valor certo | ms | Em todo push, logo no início |
| **Integração** | `POST /pedidos` grava o pedido no Postgres | s | Em todo PR, depois dos unitários |
| **E2E** | Abrir o site, colocar no carrinho, pagar | min | Antes do deploy, ou periodicamente |

A forma de pirâmide é uma recomendação de **proporção**: muitos testes rápidos na base, poucos lentos no topo. Uma suíte invertida (muitos testes E2E e poucos unitários) deixa o pipeline lento e instável, e desestimula a integração frequente que é o objetivo de tudo.

### 5.2 Análise estática

Ferramentas que examinam o código **sem executá-lo**:

- **Lint:** encontra erros e padrões perigosos (variável não definida, import sem uso, `except:` que engole tudo). Em Python, o **Ruff** reúne em uma ferramenta, e muito mais rápida, o que antes exigia flake8, isort e outras.
- **Formatação:** garante um estilo único (`ruff format --check`), para que as revisões de código discutam lógica e não espaços.
- **Checagem de tipos:** com anotações de tipo, o **mypy** encontra chamadas com argumentos errados antes de o código rodar.

### 5.3 Build e segurança

- **Build da imagem:** prova que o Dockerfile funciona e produz o artefato.
- **Segurança:** dependências com vulnerabilidades conhecidas, segredos esquecidos no código, falhas na imagem. O [terceiro capítulo de CI](03_qualidade_e_seguranca_no_pipeline.md) dedica uma seção a cada uma.

> **Confusão comum:** "o pipeline está verde, então o código está correto". ✅ **Mais preciso:** um pipeline verde significa que **nenhuma das verificações que existem** encontrou problema. Ele não diz nada sobre o que **não** é verificado. Um pipeline que roda só o lint fica verde com uma API que devolve resultados errados. A confiança que um pipeline transmite é exatamente proporcional à qualidade dos testes que ele executa. Por isso, a pergunta útil ao revisar um PR não é só "o CI passou?", mas também "este PR trouxe testes para o que mudou?". O capítulo de [qualidade no pipeline](03_qualidade_e_seguranca_no_pipeline.md) mostra como medir e melhorar essa confiança.

---

## 6. As regras de convivência

Uma ferramenta de CI só funciona com alguns acordos da equipe, que valem em qualquer tecnologia:

**A `main` quebrada é a prioridade.** Se o build da `main` falhou, quem fez o último merge corrige imediatamente, ou **reverte** o commit (`git revert`) e corrige com calma. Reverter não é derrota: é a forma mais rápida de devolver a `main` ao estado verde para o resto da equipe. Enquanto a `main` está vermelha, ninguém sabe se as próprias mudanças funcionam.

**Rode as verificações localmente antes do push.** O pipeline é a rede de segurança, não o primeiro teste. Rodar `ruff check` e `pytest` antes do push (ou automatizar isso com *pre-commit hooks*, no [terceiro capítulo](03_qualidade_e_seguranca_no_pipeline.md)) evita esperar 5 minutos para descobrir um erro de digitação.

**Commits e PRs pequenos.** Um PR de 100 linhas é revisado em minutos e, se algo der errado, é fácil achar a causa e reverter. Um PR de 2.000 linhas é aprovado sem leitura atenta.

**O pipeline é código.** O arquivo do pipeline fica no repositório, ao lado do código. Ele é versionado, revisado em PRs e evolui com a aplicação. Isso é **Infraestrutura como Código** (do [capítulo de virtualização](../../01-infraestrutura/fundamentos/02_virtualizacao_containers_e_iac.md)) aplicada ao processo de entrega.

**Mantenha o pipeline rápido.** Se o feedback leva 40 minutos, as pessoas param de esperar, empilham mudanças e integram com menos frequência, e a CI volta a ser apenas um servidor de testes. A meta de menos de 10 minutos para o feedback principal é um requisito, não um luxo.

> **Confusão comum:** "testes lentos não são problema: o pipeline roda sozinho, eu só olho o resultado depois". ✅ **Mais preciso:** o tempo do pipeline determina o **comportamento** da equipe. Com 5 minutos de espera, a pessoa espera o resultado, corrige o que falhou com o contexto ainda na cabeça e segue. Com 45 minutos, ela troca de tarefa, volta horas depois para um erro que já esqueceu, e passa a juntar várias mudanças em um push só "para não esperar tantas vezes". O PR cresce, a revisão piora e a integração fica menos frequente. Um pipeline lento corrói, aos poucos, exatamente a prática que ele deveria sustentar. Cada minuto ganho no pipeline (com cache, paralelismo e a ordem certa das etapas, no [terceiro capítulo](03_qualidade_e_seguranca_no_pipeline.md)) é um investimento na prática da equipe.

---

## 7. As ferramentas

Todas as ferramentas de CI implementam o mesmo modelo da seção 4. Elas diferem em **onde rodam** e em **quanto você administra**:

| Ferramenta | Onde roda | Quem administra o servidor | Quando costuma ser escolhida |
|---|---|---|---|
| **GitHub Actions** | Nuvem do GitHub (ou runners próprios) | O GitHub | Código no GitHub; a opção mais simples para começar |
| **GitLab CI** | Nuvem do GitLab ou instalação própria | O GitLab ou você | Código no GitLab; plataforma completa (repositório, CI, registry) |
| **Jenkins** | Seus servidores | **Você** | Empresas com requisitos de rede interna, integrações específicas ou pipelines muito customizados |
| **CircleCI, Buildkite** | Nuvem (com opções híbridas) | O fornecedor | Equipes que querem pipelines rápidos independentes do repositório |
| **Azure DevOps Pipelines** | Nuvem da Microsoft | A Microsoft | Ecossistema Microsoft |

Esta trilha usa duas delas. O **GitHub Actions** ([próximo capítulo](02_github_actions_na_pratica.md)) é a forma mais rápida de colocar a CI em prática, sem instalar nada. O **Jenkins** ([bloco próprio](../jenkins/01_fundamentos_do_jenkins.md)) é o servidor de automação mais presente em empresas, e entendê-lo mostra o que as ferramentas na nuvem fazem por você.

---

## Próximo passo

Este capítulo apresentou a prática: integrar com frequência, verificar cada integração, manter a `main` verde e produzir um artefato confiável. Falta implementá-la.

O [02_github_actions_na_pratica.md](02_github_actions_na_pratica.md) constrói, passo a passo, o pipeline de CI de uma API FastAPI: lint, testes unitários, testes de integração com Postgres, cobertura, build da imagem e as regras de proteção da `main`.
