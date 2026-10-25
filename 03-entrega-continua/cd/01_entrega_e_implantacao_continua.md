# Entrega Contínua e Implantação Contínua

**Pré-requisito:** os três capítulos de CI, até [../ci/03_qualidade_e_seguranca_no_pipeline.md](../ci/03_qualidade_e_seguranca_no_pipeline.md)

> O conceito de *deployment pipeline* e a regra de construir o artefato uma única vez vêm de *Continuous Delivery* (Jez Humble & David Farley, 2010). A separação entre configuração e código segue o fator III do *The Twelve-Factor App*. A **implantabilidade** (*deployability*) como característica de arquitetura vem do Capítulo 4 de **Fundamentals of Software Architecture** (Mark Richards & Neal Ford) e do Capítulo 3 (*Architectural Modularity*) de **Software Architecture: The Hard Parts**, que a inclui entre os motivos para dividir um sistema em partes implantáveis separadamente. As métricas seguem o programa de pesquisa DORA (*DevOps Research and Assessment*).

---

## Por que este arquivo existe

A CI termina com um artefato: uma imagem Docker com a tag do commit, que passou por lint, testes, scans e build. Ela está **pronta**, mas não está **em lugar nenhum**. Entre esse ponto e um usuário usando a versão nova existe um caminho inteiro: publicar a imagem, colocá-la em um ambiente de testes, validar, decidir ir para produção, trocar a versão sem derrubar o sistema e, se algo der errado, voltar.

Quando esse caminho é manual, ele tem os mesmos defeitos que o [capítulo de virtualização](../../01-infraestrutura/fundamentos/02_virtualizacao_containers_e_iac.md) apontou na configuração manual de servidores: depende de quem sabe, varia a cada vez e dá medo. O resultado previsível é que os deploys ficam raros, e deploys raros ficam grandes e arriscados.

A pergunta central deste capítulo é: *o que significa entregar software continuamente, e quais princípios tornam um deploy tão rotineiro que deixa de ser um evento?* Os dois capítulos seguintes tratam do **como**: as [estratégias de deploy](02_estrategias_de_deploy.md) e o [pipeline de CD na prática](03_pipeline_de_cd_na_pratica.md).

---

## 1. O problema: o "dia do deploy"

O cenário clássico de uma equipe sem CD:

```
  ┌──────────── 3 meses de desenvolvimento ─────────────┐
  │  200 mudanças acumuladas, nenhuma em produção        │
  └──────────────────────────┬───────────────────────────┘
                             ▼
          SEXTA, 22h: JANELA DE MANUTENÇÃO
   ┌──────────────────────────────────────────────────────┐
   │ 1. avisar os usuários que o sistema vai sair do ar   │
   │ 2. seguir o documento "passo a passo do deploy"      │
   │    (34 passos, última atualização há 8 meses)        │
   │ 3. ssh nos 4 servidores, um por um                   │
   │ 4. rodar as migrações do banco à mão                 │
   │ 5. algo dá errado no passo 19                        │
   │ 6. qual das 200 mudanças causou isso?                │
   │ 7. voltar atrás? como? ninguém testou o rollback     │
   └──────────────────────────────────────────────────────┘
                             ▼
               domingo, 4h: "estabilizado"
```

Cada item dessa lista é um risco diferente:

- **Tamanho:** com 200 mudanças, quando algo quebra, a causa está escondida entre elas.
- **Processo manual:** cada deploy é feito de um jeito ligeiramente diferente, e o documento está desatualizado.
- **Raridade:** como acontece a cada três meses, ninguém pratica. O rollback, então, nunca foi exercitado.
- **Medo:** como é arriscado, faz-se menos vezes; como se faz menos vezes, cada um fica maior e mais arriscado. É um ciclo que se reforça.

A resposta da entrega contínua é inverter esse ciclo: **deploys pequenos, automáticos e frequentes**, tão rotineiros que deixam de ser um evento. Martin Fowler resume a ideia em uma frase que parece contraditória: *se dói, faça com mais frequência.* Com a repetição, a dor obriga a automatizar, e o que é automatizado deixa de doer.

---

## 2. Três termos que se confundem

A sigla **CD** é usada para duas práticas diferentes, e a distinção importa:

```
         CI                    ENTREGA CONTÍNUA                  IMPLANTAÇÃO CONTÍNUA
  (Continuous Integration)   (Continuous Delivery)            (Continuous Deployment)

  commit                      commit                           commit
    │                           │                                │
  build + testes              build + testes                   build + testes
    │                           │                                │
  artefato ✔                  artefato ✔                       artefato ✔
                                │                                │
                              homologação (automático)         homologação (automático)
                                │                                │
                              ┌─▼──────────────────┐             │
                              │ decisão HUMANA:    │             │
                              │ "pode ir"          │             │
                              └─┬──────────────────┘             │
                                │                                │
                              produção                         produção (automático)
```

| Prática | O que garante | Quem decide ir para produção |
|---|---|---|
| **Integração contínua** | Cada mudança é integrada e verificada | Ninguém: ela termina no artefato |
| **Entrega contínua** | Todo artefato que passa no pipeline **pode** ir para produção a qualquer momento, com um clique | Uma pessoa, com base em uma decisão de negócio |
| **Implantação contínua** | Todo artefato que passa no pipeline **vai** para produção, sem intervenção | O próprio pipeline |

A diferença entre as duas últimas é **uma única etapa**: a aprovação humana antes de produção. Todo o resto (automação, testes, ambientes, artefato único) é igual.

> **Confusão comum:** "entrega contínua significa que todo commit vai automaticamente para produção". ✅ **Mais preciso:** isso é **implantação** contínua. Na **entrega** contínua, todo commit que passa no pipeline **fica pronto** para produção, e a ida acontece quando alguém decide. A decisão pode depender de fatores que não são técnicos: uma campanha de marketing, a aprovação de uma área regulada, uma janela combinada com um cliente. O requisito central da entrega contínua não é a ausência de aprovação, e sim que **a aprovação seja a única coisa faltando**. Se, depois do "pode ir", ainda é preciso alguém executar 30 passos manuais, a equipe não pratica entrega contínua, apenas tem um botão de aprovação na frente de um processo manual.

### Qual escolher

| Situação | Mais adequado |
|---|---|
| Testes automatizados confiáveis, monitoramento bom, feature flags, rollback rápido | **Implantação contínua** |
| Setor regulado, exigência de aprovação formal, contratos com janelas de mudança | **Entrega contínua** |
| Ainda sem testes confiáveis | Nenhuma das duas, por enquanto: invista primeiro na CI |

Muitas equipes começam com entrega contínua (aprovação manual para produção) e, conforme ganham confiança nos testes e no monitoramento, removem a aprovação.

---

## 3. O pipeline de implantação

O conceito central de *Continuous Delivery*, de Humble e Farley, é o **pipeline de implantação** (*deployment pipeline*): a sequência automatizada que leva uma mudança do commit até produção, em que cada estágio aumenta a confiança no artefato.

```
 ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
 │ ① COMMIT     │──▶│ ② PUBLICAR   │──▶│ ③ HOMOLOGAÇÃO│──▶│ ④ APROVAÇÃO  │──▶│ ⑤ PRODUÇÃO   │
 │   (a CI)     │   │   artefato   │   │   (staging)  │   │   (opcional) │   │              │
 │ lint, testes │   │ push da      │   │ deploy +     │   │ clique ou    │   │ deploy com   │
 │ build, scans │   │ imagem :sha  │   │ testes E2E   │   │ regra        │   │ estratégia   │
 │              │   │ no registry  │   │ smoke tests  │   │ automática   │   │ segura + ver-│
 │              │   │              │   │              │   │              │   │ ificação     │
 └──────────────┘   └──────────────┘   └──────────────┘   └──────────────┘   └──────────────┘
   minutos            segundos           minutos            minutos a horas    minutos

 ◀───────────────── cada estágio só roda se o anterior passou ─────────────────▶
 ◀────── a confiança no artefato aumenta a cada estágio; o custo de cada um também ──────▶
```

Três propriedades definem um bom pipeline de implantação:

1. **Toda mudança passa por ele.** Inclusive a correção urgente da madrugada. Um "atalho para emergências" que pula o pipeline é exatamente o caminho que nunca foi testado, usado no pior momento.
2. **Ele é o único caminho até produção.** Ninguém faz deploy da própria máquina, nem edita arquivos no servidor via SSH. Se a única forma de mudar produção é o pipeline, produção é sempre o resultado de um commit conhecido.
3. **Ele é rápido o bastante para ser usado.** Se levar horas, a equipe volta a acumular mudanças.

---

## 4. Construa uma vez, implante em todo lugar

A regra mais importante de um pipeline de implantação é: **o artefato é construído uma única vez, no CI, e esse mesmo artefato passa por todos os ambientes.**

```
 ✘ ERRADO: reconstruir em cada ambiente

   commit abc123 ─┬─▶ docker build ─▶ imagem A ─▶ homologação   ✔ testada
                  └─▶ docker build ─▶ imagem B ─▶ produção      ? nunca testada

 ✔ CERTO: construir uma vez, promover

   commit abc123 ──▶ docker build ──▶ minha-api:abc123 ─┬─▶ homologação   ✔ testada
                                                        └─▶ produção      ✔ a MESMA imagem
```

Por que a imagem B seria diferente, se o código é o mesmo? Porque o build depende de coisas que mudam com o tempo: a imagem base `python:3.12-slim` pode ter sido atualizada no Docker Hub, uma dependência sem versão fixa pode ter saído em versão nova, o cache pode ter se comportado diferente. Reconstruir significa colocar em produção algo que **nenhum teste viu**.

Com o Docker, a regra fica natural: a imagem tem uma tag única (o hash do commit, como no [capítulo de GitHub Actions](../ci/02_github_actions_na_pratica.md)), é publicada uma vez no registry, e cada ambiente apenas a **baixa**. "Promover" uma versão de homologação para produção é dizer a produção qual tag usar.

### E as diferenças entre os ambientes?

Se a imagem é a mesma em todo lugar, o que muda entre homologação e produção? **A configuração**: a URL do banco, as chaves de APIs externas, o nível de log, o número de réplicas. Essa é a regra do fator III do *Twelve-Factor App*: **a configuração fica no ambiente, não no código**. Ela já apareceu nesta trilha várias vezes:

| Onde | Como a configuração chega à mesma imagem |
|---|---|
| Docker / Compose | Variáveis de ambiente e arquivo `.env` ([Compose e produção](../../02-containers/docker/04_compose_e_producao.md)) |
| Kubernetes | ConfigMaps e Secrets diferentes em cada namespace ou cluster ([configuração, estado e segurança](../../02-containers/kubernetes/03_configuracao_estado_e_seguranca.md)) |
| Pipeline | *Environments* do GitHub, cada um com seus próprios secrets ([pipeline de CD](03_pipeline_de_cd_na_pratica.md)) |

> **Confusão comum:** "precisamos de uma imagem para cada ambiente, porque a configuração de produção é diferente". ✅ **Mais preciso:** a configuração diferente é justamente o que **não** deve estar na imagem. Uma imagem com a URL do banco de produção embutida tem dois problemas: não pode ser testada em homologação (ela apontaria para o banco de produção), e espalha um segredo para todo lugar onde a imagem for copiada. Se o código precisa saber em que ambiente está, ele lê uma variável (`AMBIENTE=producao`), e não é compilado de forma diferente. A regra prática: se uma mudança de configuração exige um `docker build` novo, há configuração morando no lugar errado.

---

## 5. Ambientes

Um **ambiente** é uma instalação completa do sistema, com seus dados e sua configuração. Os mais comuns:

| Ambiente | Para quê | Quem usa | Dados |
|---|---|---|---|
| **Local** | Desenvolver e testar a mudança | Quem desenvolve | Fictícios, em containers |
| **CI** | Verificar cada commit | O pipeline | Criados e destruídos a cada execução |
| **Homologação** (*staging*) | Validar a versão em condições parecidas com as de produção | Equipe, QA, produto | Fictícios ou anonimizados |
| **Produção** | Atender os usuários | Os usuários | Reais |

### Paridade entre ambientes

Homologação só serve para alguma coisa se **se parecer com produção**. Cada diferença entre os dois é um lugar onde um problema pode passar por homologação e aparecer só em produção:

- **Mesma imagem** (seção 4): garantido pelo pipeline.
- **Mesmas versões de dependências externas:** se produção usa Postgres 16, homologação não pode usar 14.
- **Mesma topologia:** se produção tem 3 réplicas atrás de um load balancer, uma homologação com uma réplica nunca vai revelar um problema de sessão guardada em memória (o problema *stateful* de [Arquitetura de Software: escalabilidade_horizontal_vs_vertical.md](https://github.com/iranhonorato/Arquitetura-de-Software/blob/main/system-design-pt2/escalabilidade_horizontal_vs_vertical.md)).
- **Mesma forma de implantar:** o mesmo pipeline, os mesmos manifestos, a mesma estratégia de deploy. Se o deploy em homologação é diferente do de produção, o deploy de produção nunca foi testado.

A infraestrutura como código ajuda muito aqui: com o Terraform ([VPC](../../01-infraestrutura/aws/02_vpc_redes_na_aws.md)) e os manifestos com Kustomize ([do Compose ao cluster](../../02-containers/kubernetes/04_do_compose_ao_cluster.md)), homologação e produção são o mesmo código com valores diferentes, e não duas instalações feitas à mão que divergem com o tempo.

### Ambientes efêmeros

Uma evolução comum é criar um **ambiente temporário para cada pull request** (*preview environment*): o pipeline sobe a versão do PR em um namespace próprio do Kubernetes, com uma URL própria (`pr-142.homolog.exemplo.com`), e o destrói quando o PR é fechado. Quem revisa pode testar a mudança funcionando, e não só ler o código.

---

## 6. Separar implantar de liberar

Dois eventos costumam ser tratados como um só, e separá-los é uma das ideias mais úteis da entrega contínua:

- **Implantar** (*deploy*): colocar uma nova versão do código **rodando** em produção.
- **Liberar** (*release*): tornar uma funcionalidade **visível** para os usuários.

Com **feature flags** (detalhadas no [próximo capítulo](02_estrategias_de_deploy.md)), o código de uma funcionalidade nova pode ser implantado desligado, ficar dias em produção sem ninguém ver, e ser ligado quando a equipe quiser: para 1% dos usuários, só para os funcionários, ou para todos de uma vez. Se algo der errado, desliga-se a flag, sem novo deploy.

Isso muda a natureza do risco. O deploy deixa de ser o momento de "lançar a funcionalidade", e passa a ser uma operação técnica rotineira, que pode acontecer várias vezes por dia. O lançamento vira uma decisão de produto, tomada com um clique e revertida com outro. Também é isso que permite o *trunk-based development* do [capítulo de fundamentos de CI](../ci/01_fundamentos_de_integracao_continua.md): código incompleto pode entrar na `main` e até em produção, desde que desligado.

---

## 7. Quando algo dá errado: rollback e roll-forward

Por melhor que seja o pipeline, alguns problemas só aparecem em produção. A pergunta não é se isso vai acontecer, e sim **quanto tempo leva para se recuperar**. Há dois caminhos:

| | Rollback | Roll-forward |
|---|---|---|
| **O que é** | Voltar para a versão anterior | Corrigir o problema e implantar uma versão nova |
| **Velocidade** | Rápida, se o artefato anterior ainda existe | Depende de achar e corrigir a causa |
| **Quando usar** | Primeira reação a um problema sério | Quando o rollback é impossível ou mais arriscado (por exemplo, depois de uma migração de banco irreversível) |

Com a regra da seção 4, o rollback é simples: a imagem anterior (`minha-api:9f8e7d6`) continua no registry, e voltar é dizer ao ambiente para usar a tag anterior. No Kubernetes, é o `kubectl rollout undo` do [capítulo de workloads](../../02-containers/kubernetes/02_workloads_rede_e_escala.md).

O que complica o rollback quase sempre é o **banco de dados**: se a versão nova renomeou uma coluna, a versão antiga não funciona mais com o banco atualizado. O [próximo capítulo](02_estrategias_de_deploy.md) mostra o padrão *expand and contract*, que mantém o banco compatível com as duas versões durante a transição.

> **Confusão comum:** "nosso rollback está documentado, então estamos cobertos". ✅ **Mais preciso:** um rollback que nunca foi executado é uma hipótese, não um plano. O caminho de volta tem os mesmos riscos do caminho de ida (permissões, configuração, dados), com um agravante: é usado em um momento de incidente, sob pressão. A forma confiável de garantir que ele funciona é que ele seja **o mesmo mecanismo do deploy**, só que apontando para a versão anterior, e que seja **exercitado** regularmente, por exemplo em homologação. Se o deploy é "o pipeline aplica a tag X", o rollback é "o pipeline aplica a tag anterior", e foi testado centenas de vezes sem ninguém perceber.

---

## 8. Implantabilidade como característica da arquitetura

Richards e Ford listam a **implantabilidade** (*deployability*) entre as características de arquitetura: a facilidade, a frequência e o risco com que o sistema pode ser implantado. *Software Architecture: The Hard Parts* vai além e a coloca entre os motivos para **dividir** um sistema (os *modularity drivers*, ao lado de testabilidade, escalabilidade e disponibilidade): partes que mudam em ritmos muito diferentes, ou que têm riscos de implantação muito diferentes, são candidatas a serem implantadas separadamente.

A consequência prática para o CD é que **a arquitetura limita o pipeline**. Um monólito com um único artefato pode ter um pipeline excelente, mas toda mudança implanta tudo. Serviços que podem ser implantados de forma independente permitem pipelines independentes, mas cada um precisa garantir compatibilidade com as versões dos outros (contratos de API, migrações de banco compatíveis). Não existe um lado certo: o pipeline e a arquitetura precisam ser desenhados juntos.

---

## 9. Medindo a entrega: as métricas DORA

Como saber se o processo de entrega está melhorando? O programa de pesquisa **DORA** (*DevOps Research and Assessment*), que há anos estuda milhares de equipes, chegou a quatro métricas que, juntas, descrevem bem a capacidade de entrega:

| Métrica | Pergunta | Mede | Equipes de alto desempenho |
|---|---|---|---|
| **Frequência de deploy** | Com que frequência implantamos em produção? | Velocidade | Sob demanda, várias vezes ao dia |
| **Lead time de mudanças** | Quanto tempo do commit até produção? | Velocidade | Menos de um dia |
| **Taxa de falha de mudanças** | Que porcentagem dos deploys causa um problema em produção? | Estabilidade | Baixa (na casa de 5% a 15%) |
| **Tempo de recuperação** | Quanto tempo para restaurar o serviço depois de uma falha? | Estabilidade | Menos de uma hora |

A descoberta mais importante da pesquisa é que **velocidade e estabilidade não competem**: as equipes que implantam com mais frequência também são as que falham menos e se recuperam mais rápido. Isso contradiz a intuição de que "ir devagar é mais seguro", e a explicação está nas seções anteriores: deploys frequentes são pequenos (fáceis de entender e reverter), automáticos (sem erros manuais) e praticados (o rollback funciona).

Use as métricas para acompanhar a **tendência** da própria equipe, e não para comparar equipes ou avaliar pessoas. Uma métrica usada como meta individual passa a ser manipulada, como a cobertura de testes do [capítulo de qualidade](../ci/03_qualidade_e_seguranca_no_pipeline.md).

---

## Próximo passo

Este capítulo definiu os princípios: o pipeline de implantação como único caminho até produção, o artefato construído uma vez, a configuração no ambiente, a separação entre implantar e liberar, e a recuperação rápida. Falta responder a uma pergunta prática: no momento de trocar a versão em produção, **como** fazer isso sem derrubar o sistema e sem expor todos os usuários a um possível problema de uma só vez?

O [02_estrategias_de_deploy.md](02_estrategias_de_deploy.md) compara as estratégias (recreate, rolling, blue-green, canary, feature flags) e mostra como lidar com o banco de dados durante a troca.
