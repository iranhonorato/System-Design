# Kubernetes: Fundamentos e Arquitetura

**Pré-requisito:** [../docker/04_compose_e_producao.md](../docker/04_compose_e_producao.md)

> Segue a documentação oficial do Kubernetes (*Concepts*: *Overview*, *Cluster Architecture* e *Workloads/Pods*). A ligação entre orquestração e características operacionais (disponibilidade, elasticidade) retoma o Capítulo 4 de **Fundamentals of Software Architecture** (Mark Richards & Neal Ford), já usado no [../../01-infraestrutura/fundamentos/01_o_que_e_infraestrutura.md](../../01-infraestrutura/fundamentos/01_o_que_e_infraestrutura.md).

---

## Por que este arquivo existe

O capítulo [Docker: Compose e produção](../docker/04_compose_e_producao.md) apresentou o Kubernetes em uma página: um diagrama, uma tabela de recursos e o conselho de não começar por ele. Este capítulo e os três seguintes abrem esse diagrama por completo.

A pergunta central deste capítulo é: *o que é um cluster Kubernetes, quais peças o compõem e o que acontece, peça por peça, quando você pede para ele rodar uma aplicação?* É o equivalente, para o Kubernetes, do que o [../docker/01_arquitetura_e_isolamento.md](../docker/01_arquitetura_e_isolamento.md) fez para o Docker. Sem esse mapa, os comandos e os arquivos YAML dos próximos capítulos parecem mágica, e os erros parecem aleatórios.

A trilha do Kubernetes tem quatro partes:

| Aula | Pergunta que responde |
|---|---|
| **01 (este)** | O que é um cluster e como ele funciona por dentro? |
| [02](02_workloads_rede_e_escala.md) | Como rodar, atualizar, escalar e expor uma aplicação? |
| [03](03_configuracao_estado_e_seguranca.md) | Como lidar com configuração, segredos, dados persistentes e permissões? |
| [04](04_do_compose_ao_cluster.md) | Como levar o sistema do Compose para um cluster, e como investigar quando algo dá errado? |

---

## 1. O problema que o Kubernetes resolve

O Compose funciona muito bem enquanto o sistema cabe em **uma máquina**. Considere o que acontece quando ele deixa de caber.

Uma loja virtual roda em um servidor com Docker Compose: 3 réplicas da API, um worker e um Redis. A Black Friday chega e a carga triplica. Você aluga mais dois servidores. Agora existem perguntas que nenhum arquivo `docker-compose.yml` responde:

- **Onde colocar cada container?** O servidor 1 está com 90% de memória e o servidor 3 com 20%. Alguém precisa decidir, e redecidir sempre que a carga mudar.
- **O que fazer quando uma máquina cai?** Às 3 da manhã, o servidor 2 tem uma falha de hardware. Os containers que rodavam nele precisam subir em outro lugar, e a API precisa parar de mandar tráfego para eles.
- **Como atualizar sem derrubar?** A versão nova precisa substituir 9 réplicas espalhadas por 3 máquinas, poucas de cada vez, verificando se cada uma ficou saudável antes de continuar, e voltando atrás se algo der errado.
- **Como os containers se encontram?** No Compose, o DNS interno resolve o nome do serviço dentro de uma máquina. Agora as réplicas estão em máquinas diferentes, e seus IPs mudam a cada recriação.

Dá para resolver cada item com scripts e uma pessoa de plantão. É assim que se fazia. O **Kubernetes** é a ferramenta que automatiza **todas** essas decisões, de forma contínua, para qualquer número de máquinas.

> O nome vem do grego *kubernḗtēs*, "timoneiro" (quem pilota o navio), e daí o logotipo em forma de leme. A abreviação **K8s** substitui as 8 letras entre o "K" e o "s". O projeto foi criado pelo Google em 2014, inspirado em seu sistema interno Borg, e doado à recém-criada CNCF (*Cloud Native Computing Foundation*) em 2015.

### Para que serve, em uma frase

**Kubernetes é um sistema que pega um conjunto de máquinas e o transforma em uma única plataforma para rodar containers, mantendo continuamente as aplicações no estado que você declarou.**

Os recursos listados no capítulo *Docker: Compose e produção* (self-healing, auto-scaling, rolling updates, service discovery, balanceamento) são consequências dessa frase. Eles traduzem diretamente as **características operacionais** que Richards e Ford descrevem: disponibilidade (self-healing), elasticidade (auto-scaling) e implantação frequente sem interrupção (rolling updates). O Kubernetes é, em boa parte, uma máquina de entregar essas características sem que cada equipe precise construí-las do zero.

### Para que **não** serve

- **Não é um substituto do Docker no build.** O Kubernetes não constrói imagens. Você continua usando o Dockerfile e o `docker build` do capítulo [Docker: imagens e Dockerfile](../docker/02_imagens_e_dockerfile.md), e o cluster baixa a imagem de um registry.
- **Não torna uma aplicação escalável por mágica.** Se a aplicação guarda sessão em memória ou grava arquivos no disco local, ela não escala horizontalmente, com ou sem Kubernetes. O pré-requisito *stateless* de [Arquitetura de Software: escalabilidade_horizontal_vs_vertical.md](https://github.com/iranhonorato/Arquitetura-de-Software/blob/main/system-design-pt2/escalabilidade_horizontal_vs_vertical.md) continua valendo.
- **Não é obrigatório.** Um servidor com Compose, ou um serviço gerenciado como o AWS ECS ou o Google Cloud Run, resolve a maioria dos sistemas pequenos e médios com muito menos complexidade.

---

## 2. A ideia central: estado desejado e reconciliação

Antes de qualquer componente, vale entender a ideia de que todos eles derivam. No Docker, você dá **ordens**:

```bash
docker run -d minha-api:1.0    # "rode um container agora"
```

Se esse container morrer cinco minutos depois, o Docker não faz nada (a não ser que você tenha configurado uma política de reinício, e mesmo assim só naquela máquina). A ordem foi cumprida; o que acontece depois não é problema dele.

No Kubernetes, você **declara um estado**:

```yaml
# "quero que existam, sempre, 3 réplicas da minha-api:1.0"
replicas: 3
image: minha-api:1.0
```

E o Kubernetes assume a responsabilidade de **manter** esse estado, para sempre. Para isso, cada parte do sistema roda um mesmo ciclo, chamado **loop de reconciliação** (*reconciliation loop*, ou *control loop*):

```
        ┌──────────────────────────────────────────┐
        │                                          │
        ▼                                          │
  ① OBSERVAR                                       │
  "quantas réplicas existem agora?"  → 2           │
        │                                          │
        ▼                                          │
  ② COMPARAR com o estado desejado                 │
  "eu queria 3. Falta 1."                          │
        │                                          │
        ▼                                          │
  ③ AGIR para aproximar os dois                    │
  "criar 1 réplica"  ──────────────────────────────┘
```

O loop nunca termina. Ele roda o tempo todo, e é por isso que o Kubernetes "se cura sozinho": se uma réplica morre, a próxima volta do loop percebe a diferença e cria outra. Ninguém programou um "se morrer, recrie". A recuperação é efeito colateral de comparar continuamente o que existe com o que deveria existir.

Essa é a mesma ideia de **estado desejado e idempotência** que o capítulo [Virtualização, containers e IaC](../../01-infraestrutura/fundamentos/02_virtualizacao_containers_e_iac.md) mostrou no Terraform e no Ansible. A diferença é que o Terraform compara e age **quando você roda `terraform apply`**. O Kubernetes compara e age **sem parar**.

> **Confusão comum:** "o Kubernetes executa os comandos que eu mando". ✅ **Mais preciso:** o Kubernetes não executa comandos, ele **persegue estados**. Quando você aplica um arquivo dizendo `replicas: 3`, não está pedindo "crie 3 containers", está registrando "o mundo deve ter 3". Isso explica comportamentos que surpreendem quem vem do Docker. Se você apagar um Pod à mão com `kubectl delete pod`, ele "volta", porque o estado desejado continua dizendo 3. Para que ele não volte, você muda o **estado desejado** (`replicas: 2`), e não o estado atual. Quase todo "o Kubernetes está desfazendo o que eu fiz" é isso: alguém mexeu no estado atual em vez de mexer na declaração.

---

## 3. A anatomia de um cluster

Um **cluster** Kubernetes é um conjunto de máquinas (físicas ou virtuais) que trabalham juntas. Cada máquina do cluster se chama **node** (nó). Os nodes se dividem em dois papéis:

```
┌───────────────────────────────── CLUSTER ─────────────────────────────────┐
│                                                                           │
│  CONTROL PLANE  (decide)                                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  ┌────────────────┐  ┌────────┐  ┌───────────┐  ┌──────────────────┐ │  │
│  │  │ kube-apiserver │  │  etcd  │  │ scheduler │  │controller-manager│ │  │
│  │  └───────▲────────┘  └────────┘  └───────────┘  └──────────────────┘ │  │
│  └──────────┼──────────────────────────────────────────────────────────┘  │
│             │  todos os componentes conversam SÓ com o API server         │
│  ┌──────────┼──────────────┐  ┌─────────────────────────┐                 │
│  │ WORKER NODE 1  (executa)│  │ WORKER NODE 2           │   ...           │
│  │  ┌─────────┐ ┌────────┐ │  │  ┌─────────┐ ┌────────┐ │                 │
│  │  │ kubelet │ │kube-   │ │  │  │ kubelet │ │kube-   │ │                 │
│  │  └─────────┘ │proxy   │ │  │  └─────────┘ │proxy   │ │                 │
│  │              └────────┘ │  │              └────────┘ │                 │
│  │  ┌────────────────────┐ │  │  ┌────────────────────┐ │                 │
│  │  │ container runtime  │ │  │  │ container runtime  │ │                 │
│  │  │ (containerd)       │ │  │  │ (containerd)       │ │                 │
│  │  └────────────────────┘ │  │  └────────────────────┘ │                 │
│  │   [Pod]  [Pod]  [Pod]   │  │   [Pod]  [Pod]          │                 │
│  └─────────────────────────┘  └─────────────────────────┘                 │
└───────────────────────────────────────────────────────────────────────────┘
```

- O **control plane** (plano de controle) é o cérebro: guarda o estado desejado, decide onde cada coisa roda e roda os loops de reconciliação. Ele **não roda suas aplicações**.
- Os **worker nodes** (nós de trabalho) são os músculos: recebem as decisões e efetivamente executam os containers.

Uma analogia útil é a de um restaurante grande. O control plane é a gerência: recebe os pedidos, anota tudo em um caderno central e distribui as tarefas. Os worker nodes são as cozinhas: cada uma tem um chefe de praça (o kubelet) que recebe as ordens da gerência e garante que os pratos saiam.

### 3.1 Os componentes do control plane

**`kube-apiserver`: a porta de entrada.** Toda interação com o cluster passa por ele: a sua, pelo `kubectl`, e a de todos os outros componentes, inclusive os internos. Ele expõe uma API REST, valida cada requisição (o YAML está bem formado? este usuário tem permissão?) e grava o resultado no etcd. É o **único** componente que fala com o etcd. Todos os outros leem e escrevem o estado através dele.

**`etcd`: a memória do cluster.** Um banco de dados chave-valor distribuído que guarda **todo** o estado do cluster: cada Deployment, cada Pod, cada Secret, cada configuração. Se o etcd for perdido sem backup, o cluster esquece tudo o que deveria estar rodando. Por isso, em produção, ele roda com 3 ou 5 réplicas, usando um algoritmo de consenso (Raft) para que as réplicas concordem sobre o estado mesmo se uma cair.

**`kube-scheduler`: quem decide onde.** Ele observa os Pods que ainda não foram atribuídos a nenhum node e escolhe um node para cada um. A escolha considera os recursos pedidos pelo Pod (CPU, memória) e os disponíveis em cada node, além de regras como "não coloque duas réplicas no mesmo node". O scheduler **só decide**: ele anota "este Pod vai para o node 2" e não inicia nada.

**`kube-controller-manager`: os loops de reconciliação.** Um único processo que roda dezenas de **controllers**, cada um responsável por um tipo de objeto e cada um executando o loop da seção 2. Exemplos:
- o controller de **ReplicaSet** garante que o número de Pods bata com o declarado;
- o controller de **Node** percebe quando um node para de responder e marca seus Pods para serem recriados em outro lugar;
- o controller de **Job** acompanha tarefas que precisam rodar até terminar.

**`cloud-controller-manager`: a ponte com a nuvem.** Existe só em clusters rodando em provedores de nuvem. Ele traduz pedidos do Kubernetes em recursos do provedor: quando você pede um Service do tipo `LoadBalancer` (capítulo *Kubernetes: workloads, rede e escala*), é ele quem cria um load balancer de verdade na AWS.

### 3.2 Os componentes de cada worker node

**`kubelet`: o agente do node.** Um processo que roda em cada node e conversa com o API server. Ele pergunta continuamente "quais Pods foram atribuídos a mim?" e garante que eles estejam rodando, pedindo ao container runtime que crie os containers necessários. Também executa os **health checks** (as *probes* do capítulo *Kubernetes: workloads, rede e escala*) e reporta o estado dos Pods de volta ao API server.

**Container runtime: quem cria os containers.** O software que efetivamente baixa imagens e roda containers. Hoje o mais comum é o **containerd**, a mesma peça que fica por baixo do Docker (veja a confusão "Docker = container" no capítulo [Docker: arquitetura e isolamento](../docker/01_arquitetura_e_isolamento.md)). O kubelet conversa com ele por uma interface padronizada, a CRI (*Container Runtime Interface*).

**`kube-proxy`: a rede dos Services.** Configura regras de rede em cada node (com `iptables` ou IPVS) para que o tráfego enviado a um *Service* chegue a um dos Pods certos. O capítulo *Kubernetes: workloads, rede e escala* mostra isso em detalhe.

> **Confusão comum:** "o Kubernetes deixou de suportar o Docker, então minhas imagens Docker não funcionam mais". ✅ **Mais preciso:** em 2022 (versão 1.24), o Kubernetes deixou de usar o **Docker Engine como runtime** dentro dos nodes, removendo um adaptador chamado *dockershim*. Ele passou a falar diretamente com o containerd, que já era o que o Docker usava por baixo. As **imagens** não mudaram nada: seguem o padrão OCI, e uma imagem construída com `docker build` roda em qualquer cluster. O que mudou foi só uma peça interna do node, invisível para quem desenvolve.

---

## 4. Pod: a menor unidade do Kubernetes

O Kubernetes não gerencia containers diretamente. A menor coisa que ele cria, agenda e monitora é o **Pod**.

Um **Pod** é um envelope em volta de **um ou mais containers** que:

- rodam **sempre no mesmo node**;
- compartilham o **mesmo endereço IP** e as mesmas portas (o mesmo namespace de rede, no sentido do capítulo [Docker: arquitetura e isolamento](../docker/01_arquitetura_e_isolamento.md)), e por isso se enxergam via `localhost`;
- podem compartilhar **volumes**;
- nascem e morrem **juntos**.

```
┌──────────────── Pod (IP 10.244.1.17) ─────────────────┐
│                                                       │
│   ┌──────────────────┐        ┌───────────────────┐   │
│   │  container: api  │◀──────▶│ container: logger │   │
│   │  porta 8000      │localhost│ (sidecar)        │   │
│   └────────┬─────────┘        └─────────┬─────────┘   │
│            └──────────┬─────────────────┘             │
│                 ┌─────▼──────┐                        │
│                 │ volume     │  ← compartilhado       │
│                 │ /var/log   │                        │
│                 └────────────┘                        │
└───────────────────────────────────────────────────────┘
```

Na grande maioria dos casos, um Pod tem **um único container**. Vários containers no mesmo Pod só fazem sentido quando eles são **inseparáveis**: precisam rodar juntos, na mesma máquina, compartilhando rede ou arquivos. É exatamente o padrão **sidecar** descrito no capítulo [Docker: Compose e produção](../docker/04_compose_e_producao.md): um container auxiliar que cuida de logs, de proxy ou de criptografia para o container principal.

### O primeiro Pod

Todo objeto do Kubernetes é descrito em um **manifesto**, um arquivo YAML:

```yaml
# pod.yaml
apiVersion: v1          # versão da API que define este tipo de objeto
kind: Pod               # o tipo de objeto
metadata:
  name: meu-nginx       # nome único (dentro do namespace)
  labels:
    app: web            # etiquetas para encontrar o objeto depois (seção 5)
spec:                   # o ESTADO DESEJADO
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
```

```bash
kubectl apply -f pod.yaml
# pod/meu-nginx created

kubectl get pods -o wide
# NAME        READY   STATUS    RESTARTS   AGE   IP            NODE
# meu-nginx   1/1     Running   0          8s    10.244.1.17   worker-1
```

A coluna `READY 1/1` diz que 1 de 1 container do Pod está pronto, e `NODE` mostra onde o scheduler o colocou.

### Pods são descartáveis

Um Pod **não é consertado**: quando ele morre, é substituído por **outro Pod**, com outro nome e outro IP. Essa é a mesma filosofia de containers descartáveis do capítulo [Docker: volumes e redes](../docker/03_volumes_e_redes.md), levada ao extremo, e tem duas consequências diretas:

1. **Nunca dependa do IP de um Pod.** Ele muda a cada recriação. Para encontrar Pods, use um **Service** (capítulo *Kubernetes: workloads, rede e escala*), que oferece um endereço estável.
2. **Quase nunca crie Pods diretamente.** Um Pod criado à mão, como o acima, não tem ninguém cuidando dele: se o node cair, ele simplesmente some. Na prática, você cria um **Deployment** (capítulo *Kubernetes: workloads, rede e escala*), e ele cria e mantém os Pods por você.

> **Confusão comum:** "Pod é o nome que o Kubernetes dá para um container". ✅ **Mais preciso:** um Pod é uma **unidade de agendamento** que contém containers, e a diferença tem efeitos práticos. Os containers de um Pod compartilham um único IP, então dois containers no mesmo Pod **não podem** usar a mesma porta. Eles se falam por `localhost`, e não pelo nome. E o Pod inteiro é agendado, escalado e removido como um bloco: não dá para escalar só um dos containers de um Pod. Por isso a regra é colocar no mesmo Pod apenas o que é inseparável. A API e o banco, por exemplo, vão em Pods **diferentes**, para poderem escalar e falhar de forma independente.

---

## 5. O que acontece em um `kubectl apply`

Com as peças apresentadas, dá para seguir um pedido do início ao fim. Você aplica um Deployment que pede 2 réplicas de `minha-api:1.0`:

```bash
kubectl apply -f deployment.yaml
```

```
 você
  │ ① kubectl envia o YAML (HTTP) ──────────────▶ kube-apiserver
  │                                                  │ ② valida, autoriza, grava no etcd
  │                                                  ▼
  │                                               ┌──────┐
  │                                               │ etcd │ "Deployment minha-api: 2 réplicas"
  │                                               └──────┘
  │   ③ controller-manager percebe o Deployment novo e cria
  │      um ReplicaSet, que cria 2 Pods "sem node"   ──▶ apiserver ──▶ etcd
  │
  │   ④ scheduler percebe 2 Pods sem node, escolhe
  │      worker-1 e worker-2 e grava a decisão       ──▶ apiserver ──▶ etcd
  │
  │   ⑤ kubelet do worker-1 percebe "tenho um Pod novo"
  │      ──▶ pede ao containerd: baixe minha-api:1.0 e rode
  │      (o mesmo acontece no worker-2)
  │
  │   ⑥ cada kubelet reporta "Pod Running"          ──▶ apiserver ──▶ etcd
  ▼
kubectl get pods  →  2 Pods Running
```

Três observações sobre esse fluxo:

1. **Nenhum componente dá ordens diretamente a outro.** O scheduler não chama o kubelet. Ele grava uma decisão no API server, e o kubelet, que observa o API server, percebe. Todos se coordenam através do estado gravado. Isso torna o sistema robusto: se o scheduler estiver fora do ar por um minuto, nada se perde, e ele retoma de onde o estado indica.
2. **O `kubectl apply` retorna logo no passo ②.** A resposta `deployment created` significa "o estado desejado foi registrado", e não "a aplicação está rodando". Os passos ③ a ⑥ acontecem em segundo plano, e é por isso que se usa `kubectl get pods` (ou `kubectl rollout status`) para acompanhar.
3. **Cada passo é um loop de reconciliação.** Se o worker-1 cair depois do passo ⑥, o controller de Node percebe, o ReplicaSet vê que falta um Pod, cria outro, o scheduler o coloca no worker-2, e o kubelet de lá o inicia. Esses são os mesmos passos de antes, disparados por uma diferença nova entre o desejado e o atual.

> **Confusão comum:** "o `kubectl` se conecta aos servidores e roda os containers". ✅ **Mais preciso:** o `kubectl` é apenas um cliente HTTP do **API server**, como o cliente `docker` é do daemon (capítulo *Docker: arquitetura e isolamento*). Ele nunca fala com os nodes, com o etcd ou com o containerd. Por isso o acesso a um cluster se resume a duas coisas: o endereço do API server e uma credencial, ambos guardados no arquivo `~/.kube/config` (o *kubeconfig*). Quem tem um kubeconfig válido de administrador controla o cluster inteiro, e esse arquivo merece o mesmo cuidado que a chave `.pem` do [../../01-infraestrutura/aws/01_ec2_servidores_na_nuvem.md](../../01-infraestrutura/aws/01_ec2_servidores_na_nuvem.md).

---

## 6. Organizando objetos: manifestos, labels e namespaces

### 6.1 A anatomia de um manifesto

Todo manifesto tem os mesmos quatro campos de topo:

| Campo | Para que serve | Exemplo |
|---|---|---|
| `apiVersion` | Qual versão da API define o tipo | `v1`, `apps/v1`, `networking.k8s.io/v1` |
| `kind` | Qual é o tipo de objeto | `Pod`, `Deployment`, `Service` |
| `metadata` | Identidade: nome, namespace, labels | `name: minha-api` |
| `spec` | O **estado desejado**, escrito por você | `replicas: 3` |

Há um quinto campo, que você **não escreve**: o `status`, preenchido pelo Kubernetes com o **estado atual**. A reconciliação da seção 2 é, literalmente, o esforço contínuo de fazer o `status` alcançar o `spec`:

```bash
kubectl get deployment minha-api -o yaml
```
```yaml
spec:
  replicas: 3            # o que você pediu
status:
  replicas: 3
  readyReplicas: 2       # o que existe e está pronto agora
  unavailableReplicas: 1
```

Quando não souber quais campos um tipo aceita, o próprio cluster documenta:

```bash
kubectl explain deployment.spec.strategy
```

### 6.2 Labels e selectors: como os objetos se encontram

**Labels** são pares chave-valor que você anexa a qualquer objeto. Elas não têm significado para o Kubernetes por si só: você escolhe as chaves e os valores.

```yaml
metadata:
  labels:
    app: minha-api
    ambiente: producao
    versao: "1.4.2"
```

**Selectors** são filtros sobre labels. E aqui está o ponto importante: **é assim que os objetos se ligam uns aos outros no Kubernetes**. Um Deployment sabe quais Pods são seus porque procura Pods com certas labels. Um Service sabe para quais Pods mandar tráfego da mesma forma. Não existe uma lista fixa de "Pods deste Service", e sim uma consulta que é refeita continuamente.

```
  Service "minha-api"                    Pods
  selector: app=minha-api   ───────▶  [app=minha-api]   ✔ recebe tráfego
                            ───────▶  [app=minha-api]   ✔ recebe tráfego
                                      [app=worker]      ✘ ignorado
```

O mesmo filtro funciona na linha de comando:

```bash
kubectl get pods -l app=minha-api
kubectl get pods -l 'ambiente in (producao, homologacao)'
```

### 6.3 Namespaces: divisões lógicas do cluster

Um **namespace** divide um cluster em espaços isolados de nomes. Dois objetos podem ter o mesmo nome se estiverem em namespaces diferentes. Usos típicos: separar ambientes (`dev`, `homologacao`), equipes (`time-pagamentos`) ou aplicações.

```bash
kubectl get namespaces
# NAME              STATUS   AGE
# default           Active   2d     ← onde vão os objetos se você não indicar outro
# kube-system       Active   2d     ← componentes internos do próprio cluster
# kube-public       Active   2d
# kube-node-lease   Active   2d

kubectl create namespace loja
kubectl get pods -n loja                  # lista os Pods do namespace "loja"
kubectl get pods -A                       # lista os Pods de todos os namespaces
```

Rode `kubectl get pods -n kube-system` em um cluster e você verá parte da arquitetura da seção 3 rodando como Pods: o `kube-proxy` de cada node, o DNS interno (`coredns`) e, em clusters locais, até o API server e o etcd.

> **Confusão comum:** "colocar aplicações em namespaces diferentes as isola uma da outra". ✅ **Mais preciso:** namespaces isolam **nomes** e servem de fronteira para **permissões** (capítulo *Kubernetes: configuração, estado e segurança*) e **cotas** de recursos. Eles **não** isolam a rede: por padrão, qualquer Pod de qualquer namespace consegue se conectar a qualquer outro Pod do cluster. Para bloquear isso, é preciso criar **NetworkPolicies** (capítulo *Kubernetes: configuração, estado e segurança*). Também não são uma fronteira de segurança forte entre clientes diferentes: os Pods continuam compartilhando os mesmos nodes e o mesmo kernel (veja a confusão sobre isolamento de containers no capítulo [Virtualização, containers e IaC](../../01-infraestrutura/fundamentos/02_virtualizacao_containers_e_iac.md)).

---

## 7. Montando um cluster para estudar

Não é preciso pagar por um cluster na nuvem para aprender. Três opções rodam na sua máquina:

| Ferramenta | Como funciona | Quando usar |
|---|---|---|
| **kind** (*Kubernetes in Docker*) | Cada node do cluster é um container Docker | Rápido de criar e destruir; permite simular vários nodes |
| **minikube** | Um cluster de um node, em VM ou container | Tem complementos prontos (dashboard, ingress) |
| **Docker Desktop** | Um cluster de um node embutido, ativado nas configurações | Se você já usa o Docker Desktop |

Os exemplos desta trilha usam o **kind**, porque ele simula um cluster com vários nodes, o que permite ver o scheduler distribuindo Pods. Com o Docker rodando:

```bash
# Instala o kubectl e o kind (Linux; no Mac, use "brew install kubectl kind")
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install kubectl /usr/local/bin/
# Para o kind, use a versão mais recente listada em https://kind.sigs.k8s.io
KIND_VERSION=v0.XX.0
curl -Lo ./kind "https://kind.sigs.k8s.io/dl/${KIND_VERSION}/kind-linux-amd64"
sudo install kind /usr/local/bin/
```

```yaml
# kind-cluster.yaml: um control plane e dois workers
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

```bash
kind create cluster --name estudo --config kind-cluster.yaml

kubectl get nodes
# NAME                   STATUS   ROLES           AGE   VERSION
# estudo-control-plane   Ready    control-plane   60s   v1.3x.x
# estudo-worker          Ready    <none>          40s   v1.3x.x
# estudo-worker2         Ready    <none>          40s   v1.3x.x

docker ps --format "{{.Names}}"     # os "servidores" do cluster são containers
# estudo-control-plane
# estudo-worker
# estudo-worker2
```

Para apagar tudo depois: `kind delete cluster --name estudo`.

### Os comandos `kubectl` do dia a dia

O `kubectl` segue sempre a forma `kubectl <verbo> <tipo> [nome] [opções]`:

```bash
kubectl get pods                      # lista (get pods, get deployments, get all...)
kubectl get pods -o wide              # mais colunas (IP, node)
kubectl get pod meu-nginx -o yaml     # o objeto completo, com spec e status
kubectl describe pod meu-nginx        # detalhes + EVENTOS recentes (o 1º lugar para olhar em erros)
kubectl logs meu-nginx                # logs do container (-f acompanha, --previous mostra o anterior)
kubectl exec -it meu-nginx -- sh      # terminal dentro do container
kubectl apply -f arquivo.yaml         # cria ou atualiza a partir de um manifesto
kubectl delete -f arquivo.yaml        # remove o que o manifesto descreve
kubectl explain pod.spec              # documentação de qualquer campo
```

Repare na semelhança com o Docker (`logs`, `exec -it`, `describe` no lugar de `inspect`). A diferença está no `apply`: em vez de um comando por ação, você mantém arquivos que descrevem o estado e os aplica, quantas vezes quiser, com o mesmo resultado.

---

## Próximo passo

Este capítulo montou o mapa: o cluster, os componentes do control plane e dos nodes, o loop de reconciliação e o Pod. Mas um Pod sozinho é frágil. Ele não se recria, não escala e tem um IP que muda. Faltam os objetos que transformam Pods descartáveis em uma aplicação confiável e acessível.

O [02_workloads_rede_e_escala.md](02_workloads_rede_e_escala.md) apresenta o Deployment (réplicas, atualização e rollback), o Service (um endereço estável para Pods que mudam), o Ingress (a entrada HTTP do cluster), os health checks e o auto-scaling.
