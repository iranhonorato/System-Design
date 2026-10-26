# Estratégias de Deploy

**Pré-requisito:** [01_entrega_e_implantacao_continua.md](01_entrega_e_implantacao_continua.md). Os exemplos em Kubernetes assumem [../../02-containers/kubernetes/02_workloads_rede_e_escala.md](../../02-containers/kubernetes/02_workloads_rede_e_escala.md).

> Segue a documentação do Kubernetes (*Deployments*), do Argo Rollouts e do AWS Elastic Load Balancing (*weighted target groups*). O padrão *expand and contract* (também chamado *parallel change*) vem de Danilo Sato e Martin Fowler, e a terminologia de *feature toggles* de Pete Hodgson. Load balancers e health checks são detalhados em [Arquitetura de Software: load_balancer.md](https://github.com/iranhonorato/Arquitetura-de-Software/blob/main/system-design-pt2/load_balancer.md).

---

## Por que este arquivo existe

O capítulo anterior estabeleceu o objetivo: deploys pequenos, frequentes e sem medo. Mas no instante exato da troca de versão sempre existe um risco: a versão nova pode ter um defeito que nenhum teste pegou, pode demorar para iniciar, pode não funcionar com os dados reais. A pergunta central deste capítulo é: *como trocar a versão em produção de forma que os usuários não percebam a troca e, se a versão nova tiver um problema, o menor número possível de pessoas seja afetado, pelo menor tempo possível?*

Não existe uma estratégia melhor em tudo. Cada uma faz uma troca diferente entre **custo**, **velocidade**, **complexidade** e **tamanho do estrago** quando algo dá errado. O capítulo apresenta as estratégias da mais simples à mais sofisticada, com exemplos em Kubernetes, e termina com o problema que todas elas compartilham: o banco de dados.

---

## 1. Duas perguntas para avaliar qualquer estratégia

Toda estratégia de deploy responde a duas perguntas:

1. **Durante a troca, quem atende os usuários?** Se ninguém, há *downtime*. Se as duas versões ao mesmo tempo, o sistema precisa suportar isso.
2. **Se a versão nova tiver um defeito, quantos usuários são afetados, e por quanto tempo?** Esse é o **raio de explosão** (*blast radius*) da mudança.

```
                        raio de explosão (usuários afetados por um defeito)
                        pequeno ◀──────────────────────────────────▶ grande
  custo e        alto │  canary        blue-green
  complexidade        │
                      │        rolling update
                 baixo│                                   recreate
```

Guarde as duas perguntas: elas aparecem na tabela comparativa da seção 9.

---

## 2. Recreate: desligar tudo, ligar tudo

A estratégia mais simples: derrubar todas as instâncias da versão antiga e, depois, subir as da versão nova.

```
 tempo ─────────────────────────────────────────────────────▶
 v1   ████████████████▏
 v2                      ▕████████████████████████████████
                     ▲  ▲
                     │  └── v2 sobe
                     └───── v1 cai        ◀─ DOWNTIME entre os dois ─▶
```

É o que o pipeline do antigo hands-on de CD fazia na EC2: `docker stop`, `docker rm`, `docker run`. Entre o `stop` e o novo container ficar pronto, ninguém atende.

No Kubernetes:

```yaml
spec:
  strategy:
    type: Recreate
```

**Quando usar:** quando as duas versões **não podem** coexistir de jeito nenhum (por exemplo, uma aplicação que exige acesso exclusivo a um recurso), ou em ambientes de desenvolvimento, onde alguns segundos fora do ar não importam.

**Problema:** downtime em todo deploy, e um raio de explosão de 100%: se a versão nova estiver quebrada, todos os usuários são afetados até o rollback.

---

## 3. Rolling update: troca gradual

As instâncias são substituídas **aos poucos**: sobe uma nova, ela fica pronta, cai uma antiga, e assim por diante. É a estratégia padrão do Kubernetes, detalhada no [capítulo de workloads](../../02-containers/kubernetes/02_workloads_rede_e_escala.md):

```
 réplicas  ①  ②  ③  ④
 início    v1 v1 v1 v1
 passo 1   v1 v1 v1 v1 v2*      (* iniciando; ainda não recebe tráfego)
 passo 2   v1 v1 v1 v2          (v2 ficou pronta → uma v1 sai)
 passo 3   v1 v1 v2 v2
 passo 4   v1 v2 v2 v2
 fim       v2 v2 v2 v2
```

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # até 1 réplica a mais durante a troca
      maxUnavailable: 0    # a capacidade nunca cai
```

**O que ela exige:**

- **Readiness probe bem feita.** O rolling update só é seguro se o cluster souber quando uma réplica nova está **de fato** pronta. Sem readiness, uma réplica que ainda está carregando já recebe tráfego, e erra.
- **Compatibilidade entre versões.** Durante a troca, v1 e v2 atendem ao mesmo tempo, atrás do mesmo Service. Um usuário pode ter uma requisição atendida pela v2 e a seguinte pela v1. Se a v2 mudou o formato de uma resposta, o frontend precisa aceitar os dois formatos.

**Rollback:** `kubectl rollout undo`, que faz um rolling update de volta para a versão anterior.

**Limite:** o raio de explosão cresce durante a troca. No passo 4, 75% do tráfego já está na v2. Se o defeito só aparece sob carga real, ele é descoberto quando boa parte dos usuários já foi afetada, e o rollback também leva o tempo de uma troca gradual.

---

## 4. Blue-green: dois ambientes, uma chave

Mantêm-se **dois ambientes completos**: o *blue* (azul), com a versão atual recebendo todo o tráfego, e o *green* (verde), onde a versão nova é implantada e testada **sem receber tráfego de usuários**. Quando o verde está validado, o tráfego é trocado de uma vez.

```
 ANTES DA TROCA                              DEPOIS DA TROCA

         usuários                                    usuários
            │                                           │
     ┌──────▼──────┐                             ┌──────▼──────┐
     │  roteador   │                             │  roteador   │
     └──────┬──────┘                             └──────┬──────┘
            │ 100%                                      │ 100%
            ▼                                           ▼
   ┌────────────────┐  ┌────────────────┐   ┌────────────────┐  ┌────────────────┐
   │ BLUE  (v1)     │  │ GREEN (v2)     │   │ BLUE  (v1)     │  │ GREEN (v2)     │
   │ em produção    │  │ testes internos│   │ parado, pronto │  │ em produção    │
   └────────────────┘  └────────────────┘   │ para o rollback│  └────────────────┘
                                            └────────────────┘
```

### No Kubernetes: trocando o selector do Service

O [capítulo de workloads](../../02-containers/kubernetes/02_workloads_rede_e_escala.md) mostrou que um Service encontra Pods por **labels**. Isso torna o blue-green quase trivial: dois Deployments com uma label `versao` diferente, e um Service cujo selector aponta para um deles.

```yaml
# Dois Deployments convivendo: minha-api-blue (v1) e minha-api-green (v2)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minha-api-green
spec:
  replicas: 3
  selector:
    matchLabels: { app: minha-api, cor: green }
  template:
    metadata:
      labels: { app: minha-api, cor: green }
    spec:
      containers:
        - name: api
          image: seu-usuario/minha-api:a1b2c3d     # a versão nova
          ports:
            - containerPort: 8000
---
# O Service de produção aponta para UMA cor
apiVersion: v1
kind: Service
metadata:
  name: minha-api
spec:
  selector: { app: minha-api, cor: blue }        # ← a "chave"
  ports:
    - port: 80
      targetPort: 8000
---
# Um Service extra para testar o green antes da troca
apiVersion: v1
kind: Service
metadata:
  name: minha-api-preview
spec:
  selector: { app: minha-api, cor: green }
  ports:
    - port: 80
      targetPort: 8000
```

O fluxo do deploy:

```bash
# 1. Implanta o green e espera ficar pronto
kubectl apply -f deployment-green.yaml
kubectl rollout status deployment/minha-api-green

# 2. Testa o green pelo Service de preview (smoke tests, seção 8)
kubectl run teste --rm -it --image=busybox -- wget -qO- http://minha-api-preview/healthz

# 3. A troca: um único comando muda 100% do tráfego
kubectl patch service minha-api -p '{"spec":{"selector":{"app":"minha-api","cor":"green"}}}'

# 4. Rollback, se preciso: o mesmo comando, de volta para blue
kubectl patch service minha-api -p '{"spec":{"selector":{"app":"minha-api","cor":"blue"}}}'
```

Na AWS, sem Kubernetes, a mesma ideia é feita com **dois target groups** atrás de um Application Load Balancer: o listener aponta para o target group *blue*, e a troca é alterar o listener para o *green*.

**Vantagens:** a versão nova é testada **no ambiente de produção real** antes de receber usuários, a troca é instantânea, e o rollback é tão rápido quanto a troca, porque o ambiente antigo continua de pé.

**Custos:** o dobro de infraestrutura durante o deploy, e o raio de explosão continua de 100% no momento da troca: se o defeito só aparece com tráfego real, todos os usuários o veem ao mesmo tempo (por pouco tempo, graças ao rollback rápido).

> **Confusão comum:** "com blue-green, o rollback é instantâneo, então qualquer mudança é segura". ✅ **Mais preciso:** o rollback instantâneo vale para o **código**, porque o ambiente blue continua lá. Mas os dois ambientes quase sempre compartilham **o mesmo banco de dados**, e o banco não tem uma cor. Se a v2 executou uma migração que renomeou uma coluna, voltar o tráfego para o blue coloca a v1 diante de um banco que ela não entende mais, e o "rollback instantâneo" vira um segundo incidente. O mesmo vale para mensagens já publicadas em filas, arquivos gravados e e-mails enviados: nada disso volta com a troca do selector. O blue-green só cumpre a promessa se as mudanças de banco forem compatíveis com as duas versões, que é o assunto da seção 6.

---

## 5. Canary: um pouco de cada vez, medindo

O nome vem dos canários que os mineiros levavam para as minas: se o gás tóxico aparecesse, o pássaro reagia primeiro, e os mineiros tinham tempo de sair. No deploy **canary**, a versão nova recebe primeiro **uma pequena fração do tráfego real**, e a decisão de avançar ou voltar é tomada **com base em métricas**.

```
  tráfego para a v2

  100% │                                           ████████
       │                                           █
   50% │                              ██████████████
       │                              █
   10% │              ████████████████
    5% │  ████████████              ▲             ▲
       └──┬───────────┬─────────────┼─────────────┼─────────▶ tempo
          │           │             │             │
        início   métricas ok?   métricas ok?   métricas ok?
                 → avança        → avança       → 100%

   Em qualquer passo: taxa de erro ou latência acima do limite → volta para 0%
```

### O que torna o canary diferente

O canary não é apenas "um rolling update mais lento". A diferença está em dois pontos:

1. **A fração do tráfego é controlada com precisão**, independentemente do número de réplicas: 5% do tráfego, não "1 de 4 réplicas".
2. **Cada passo tem uma verificação automática**: a taxa de erros e a latência da v2 são comparadas com as da v1 (ou com limites fixos). Se piorarem, o rollback é automático.

O resultado é o menor raio de explosão possível: um defeito que só aparece com tráfego real afeta 5% dos usuários por alguns minutos, e é revertido sem ninguém acordar de madrugada.

### No Kubernetes: Argo Rollouts

O Deployment padrão não controla porcentagens de tráfego. Ferramentas como o **Argo Rollouts** (ou o Flagger) substituem o Deployment por um objeto `Rollout` com passos de canary:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: minha-api
spec:
  replicas: 10
  selector:
    matchLabels: { app: minha-api }
  template:
    metadata:
      labels: { app: minha-api }
    spec:
      containers:
        - name: api
          image: seu-usuario/minha-api:a1b2c3d
          ports:
            - containerPort: 8000
  strategy:
    canary:
      steps:
        - setWeight: 5              # 5% do tráfego para a versão nova
        - pause: { duration: 10m }  # observa por 10 minutos
        - setWeight: 25
        - pause: { duration: 10m }
        - setWeight: 50
        - pause: { duration: 10m }
        # depois do último passo, vai para 100%
      analysis:
        templates:
          - templateName: taxa-de-erro-baixa   # consulta o Prometheus a cada passo
```

O `AnalysisTemplate` referenciado faz uma consulta ao sistema de métricas (por exemplo, "a porcentagem de respostas 5xx da versão nova nos últimos 5 minutos") e reprova o passo se o valor passar do limite. Reprovado um passo, o Rollout volta todo o tráfego para a versão estável. A qualidade de um canary depende inteiramente da qualidade das métricas, e é por isso que a observabilidade é o tema que vem depois de CD nesta trilha. O `AnalysisTemplate` completo, consultando o Prometheus, está em [../../04-observabilidade/pratica/03_observabilidade_no_kubernetes.md](../../04-observabilidade/pratica/03_observabilidade_no_kubernetes.md), seção 8.

Na AWS, sem Kubernetes, o mesmo efeito é obtido com **pesos** entre target groups de um Application Load Balancer (95% para o *estável*, 5% para o *canary*), ajustados pelo pipeline.

> **Confusão comum:** "canary e teste A/B são a mesma coisa: os dois mandam parte dos usuários para uma versão diferente". ✅ **Mais preciso:** o mecanismo é parecido (dividir tráfego), mas a pergunta é outra. O **canary** responde a uma pergunta **técnica**: "a versão nova funciona tão bem quanto a antiga?". Ele dura minutos ou horas, olha para erros e latência e termina com 100% em uma versão. O **teste A/B** responde a uma pergunta de **produto**: "o botão verde vende mais que o azul?". Ele dura dias ou semanas, precisa que **o mesmo usuário** veja sempre a mesma variação (para a comparação ter valor estatístico), mede conversão e comportamento, e as duas variações podem conviver no mesmo código, controladas por feature flags. Misturar os dois produz decisões ruins: um canary que espera semanas por significância estatística, ou um teste A/B interrompido porque a latência subiu 2 ms.

---

## 6. O banco de dados: o problema compartilhado

Rolling update, blue-green e canary têm algo em comum: **durante um tempo, a versão antiga e a nova rodam ao mesmo tempo, sobre o mesmo banco**. E o rollback de qualquer uma delas devolve a versão antiga ao banco que a versão nova já alterou. Uma migração incompatível quebra todas as estratégias.

### O exemplo que quebra

A equipe decide renomear a coluna `nome` para `nome_completo` na tabela `clientes`. A forma "direta":

```sql
-- migração da v2
ALTER TABLE clientes RENAME COLUMN nome TO nome_completo;
```

No momento em que a migração roda, toda réplica da v1 que ainda está no ar passa a falhar com `column "nome" does not exist`. E se a v2 tiver um problema e for revertida, a v1 continua falhando. Não existe ordem segura para fazer essa mudança em um passo só.

### O padrão *expand and contract*

A solução é dividir a mudança em **várias implantações**, cada uma compatível com a anterior e com a seguinte:

```
 FASE 1: EXPANDIR                FASE 2: MIGRAR                  FASE 3: CONTRAIR
 (deploy A)                      (deploy B)                      (deploy C, dias depois)

 ALTER TABLE clientes            o código passa a LER de         ALTER TABLE clientes
   ADD COLUMN nome_completo;     nome_completo                     DROP COLUMN nome;
                                 (e para de usar nome)
 o código ESCREVE nas duas                                       o código já não sabe
 colunas e LÊ de nome            UPDATE para copiar os dados     que nome existiu
                                 antigos (em lotes)
 ┌────────────────────────┐      ┌────────────────────────┐      ┌────────────────────────┐
 │ v1 funciona? ✔         │      │ versão A funciona? ✔   │      │ versão B funciona? ✔   │
 │ (ignora a coluna nova) │      │ (as duas colunas estão │      │ (não usa mais nome)    │
 │ rollback para v1? ✔    │      │  preenchidas)          │      │                        │
 └────────────────────────┘      │ rollback para A? ✔     │      └────────────────────────┘
                                 └────────────────────────┘
```

Em cada fase, a versão anterior continua funcionando com o banco da fase atual, e por isso **qualquer** deploy pode ser revertido. O preço é que uma mudança que parecia de uma linha vira três deploys espaçados no tempo. É um preço que equipes com deploys frequentes pagam com tranquilidade, porque cada deploy é barato, e é mais um motivo pelo qual deploys raros e grandes tornam tudo mais difícil.

As regras gerais que derivam do exemplo:

- **Adicionar é seguro; remover e renomear não.** Colunas e tabelas novas não afetam o código antigo. Remoções vêm só depois que nenhuma versão em produção as usa.
- **Colunas novas obrigatórias começam opcionais** (ou com valor padrão). Um `NOT NULL` sem padrão quebra a versão antiga, que não sabe preencher a coluna.
- **A migração roda antes do código que depende dela**, como uma etapa própria do pipeline (o Job de migração do [capítulo do Compose ao cluster](../../02-containers/kubernetes/04_do_compose_ao_cluster.md)).
- **Migrações grandes rodam em lotes**: um `UPDATE` em 50 milhões de linhas em uma única transação pode travar a tabela por minutos.

---

## 7. Feature flags: separando o deploy da liberação

O [capítulo anterior](01_entrega_e_implantacao_continua.md) defendeu separar **implantar** de **liberar**. A ferramenta para isso é a **feature flag** (ou *feature toggle*): uma condição no código, controlada por configuração, que liga ou desliga um comportamento **sem novo deploy**.

```python
# A versão mais simples: uma variável de ambiente
import os

NOVO_CALCULO_DE_FRETE = os.getenv("FLAG_NOVO_FRETE", "false") == "true"

@app.get("/frete")
def frete(peso_kg: float, distancia_km: float, cep: str):
    if NOVO_CALCULO_DE_FRETE:
        return {"valor": frete_por_regiao(peso_kg, cep)}      # código novo, implantado desligado
    return {"valor": calcular_frete(peso_kg, distancia_km)}   # comportamento atual
```

A variável de ambiente já separa o deploy da liberação, mas ligar a flag ainda exige reiniciar a aplicação. Serviços de feature flags (LaunchDarkly, Unleash, Flagsmith, ou o padrão aberto **OpenFeature**) guardam o estado das flags fora da aplicação e permitem regras mais finas, alteradas em tempo real:

```python
# Com um serviço de flags: a decisão pode depender de quem é o usuário
if flags.ativa("novo-frete", usuario=usuario_atual):
    ...
# Regras configuradas no painel do serviço, sem deploy:
#   - ligado para funcionários
#   - ligado para 10% dos clientes de São Paulo
#   - desligado para todos os outros
```

### Os tipos de flag

Pete Hodgson classifica as flags pelo **propósito** e pela **duração**, e a distinção ajuda a não transformar o código em um labirinto de `if`s:

| Tipo | Para quê | Duração |
|---|---|---|
| **De liberação** (*release toggle*) | Esconder código incompleto ou ainda não lançado | Dias a semanas: remova depois do lançamento |
| **De experimento** | Testes A/B | Semanas: remova ao fim do experimento |
| **Operacional** (*kill switch*) | Desligar uma funcionalidade cara ou problemática em um incidente | Longa: faz parte da operação |
| **De permissão** | Funcionalidades para planos pagos ou grupos específicos | Longa: é regra de negócio |

O maior risco das feature flags é o **acúmulo**: flags de liberação esquecidas depois do lançamento deixam no código caminhos que ninguém mais testa. Uma regra simples ajuda: ao criar uma flag de liberação, crie também a tarefa de removê-la.

---

## 8. Verificando o deploy

Todas as estratégias dependem de responder, durante ou logo depois da troca: **a versão nova está funcionando?** Há três níveis de verificação, do mais simples ao mais completo:

**Health checks e readiness.** O mínimo: a aplicação responde. O cluster ou o load balancer só manda tráfego para instâncias que passam no check (veja as probes no [capítulo de workloads](../../02-containers/kubernetes/02_workloads_rede_e_escala.md)).

**Smoke tests.** Um conjunto pequeno de testes rápidos, executado pelo pipeline **contra o ambiente recém-implantado**, que exercita os caminhos críticos de verdade:

```bash
#!/usr/bin/env bash
# smoke-test.sh: falha (exit 1) se qualquer verificação falhar
set -euo pipefail
BASE_URL="$1"

curl --fail --silent --retry 5 --retry-delay 3 "$BASE_URL/healthz" > /dev/null
curl --fail --silent "$BASE_URL/frete?peso_kg=2&distancia_km=100" | grep -q '"valor"'
echo "smoke tests ok em $BASE_URL"
```

**Análise de métricas.** O nível do canary: comparar taxa de erros, latência e métricas de negócio (pedidos por minuto, pagamentos aprovados) antes e depois da troca. É o único nível que pega problemas que só aparecem com tráfego real e variado.

O pipeline usa o resultado para decidir: verificação aprovada, o deploy segue (ou termina); reprovada, o pipeline dispara o rollback. O [próximo capítulo](03_pipeline_de_cd_na_pratica.md) monta exatamente esse fluxo.

---

## 9. Comparando as estratégias

| | Recreate | Rolling update | Blue-green | Canary |
|---|---|---|---|---|
| **Downtime** | Sim | Não | Não | Não |
| **v1 e v2 ao mesmo tempo?** | Não | Sim, durante a troca | Só durante a validação (sem tráfego) | Sim, durante todo o processo |
| **Raio de explosão de um defeito** | 100% | Cresce durante a troca | 100%, por pouco tempo | Pequeno (a fração do canary) |
| **Velocidade do rollback** | Um deploy inteiro | Uma troca gradual | Instantâneo | Instantâneo |
| **Infraestrutura extra** | Nenhuma | Pouca (`maxSurge`) | O dobro, durante o deploy | Pouca |
| **Complexidade** | Mínima | Baixa (padrão do Kubernetes) | Média | Alta (exige métricas confiáveis) |
| **Bom para** | Desenvolvimento; versões incompatíveis | A maioria das aplicações | Validar em produção antes de expor; rollback rápido | Sistemas com muito tráfego, onde um defeito custa caro |

E duas técnicas que se combinam com qualquer uma delas:

- **Expand and contract** no banco de dados, para que as versões possam coexistir e o rollback seja possível.
- **Feature flags**, para que a liberação de funcionalidades nem dependa do deploy.

Um caminho de evolução comum: começar com **rolling update** e boas probes (o que o Kubernetes já oferece), acrescentar **feature flags** para as funcionalidades de maior risco, e adotar **canary** com análise automática quando o tráfego e as métricas justificarem.

---

## Próximo passo

Este capítulo e o anterior descreveram **o que** um pipeline de CD deve fazer: publicar o artefato uma vez, promovê-lo por ambientes, trocar a versão com uma estratégia segura, verificar e reverter se preciso. Falta escrevê-lo.

O [03_pipeline_de_cd_na_pratica.md](03_pipeline_de_cd_na_pratica.md) constrói o pipeline completo no GitHub Actions: publicação da imagem com a tag do commit, deploy na EC2 (primeiro por SSH, depois sem nenhuma chave guardada, com OIDC e Session Manager), deploy no Kubernetes (com `kubectl` e com GitOps) e promoção de homologação para produção com aprovação.
