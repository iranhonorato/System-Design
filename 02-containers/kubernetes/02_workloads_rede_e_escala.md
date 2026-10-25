# Kubernetes: Workloads, Rede e Escala

**Pré-requisito:** [01_fundamentos_e_arquitetura.md](01_fundamentos_e_arquitetura.md)

> Segue a documentação oficial do Kubernetes (*Workloads*, *Services, Load Balancing, and Networking* e *Horizontal Pod Autoscaling*). As estratégias de deploy retomam [../../03-entrega-continua/cd/02_estrategias_de_deploy.md](../../03-entrega-continua/cd/02_estrategias_de_deploy.md), e os health checks retomam a distinção *liveness*/*readiness* de [Arquitetura de Software: load_balancer.md](https://github.com/iranhonorato/Arquitetura-de-Software/blob/main/system-design-pt2/load_balancer.md).

---

## Por que este arquivo existe

O capítulo anterior terminou com um Pod rodando e três fragilidades: se ele morrer, ninguém o recria; não há como ter várias cópias dele; e seu IP muda a cada recriação. Uma aplicação de verdade precisa de quatro coisas que um Pod sozinho não oferece:

| Necessidade | Objeto que resolve | Seção |
|---|---|---|
| Manter N cópias rodando, atualizar e voltar atrás | **Deployment** (e ReplicaSet) | 1–2 |
| Saber se cada cópia está viva e pronta | **Probes** | 3 |
| Reservar e limitar CPU e memória | **requests e limits** | 4 |
| Um endereço estável para cópias que mudam | **Service** | 5 |
| Uma entrada HTTP vinda de fora do cluster | **Ingress** | 6 |
| Ajustar o número de cópias à carga | **HorizontalPodAutoscaler** | 7 |

Ao final, você terá uma API com várias réplicas, atualizável sem downtime, acessível por um nome e escalando sozinha.

---

## 1. ReplicaSet: manter N cópias

Um **ReplicaSet** tem uma única tarefa: garantir que exista, o tempo todo, um número fixo de Pods iguais. É o loop de reconciliação do capítulo [Kubernetes: fundamentos e arquitetura](01_fundamentos_e_arquitetura.md) aplicado à contagem de Pods.

Ele encontra "seus" Pods por um **selector** de labels e, se faltar algum, cria a partir de um **template**, um modelo de Pod embutido no próprio ReplicaSet.

Na prática, você quase nunca cria um ReplicaSet diretamente: o **Deployment** faz isso por você. Mas vale ver o ReplicaSet agindo, porque é ele quem produz o self-healing:

```bash
kubectl get pods -l app=minha-api
# NAME                         READY   STATUS    RESTARTS   AGE
# minha-api-7d9c8b6f5d-4xk2p   1/1     Running   0          5m
# minha-api-7d9c8b6f5d-9mzq8   1/1     Running   0          5m
# minha-api-7d9c8b6f5d-tr7wn   1/1     Running   0          5m

kubectl delete pod minha-api-7d9c8b6f5d-4xk2p

kubectl get pods -l app=minha-api
# NAME                         READY   STATUS              RESTARTS   AGE
# minha-api-7d9c8b6f5d-9mzq8   1/1     Running             0          5m
# minha-api-7d9c8b6f5d-tr7wn   1/1     Running             0          5m
# minha-api-7d9c8b6f5d-bq5lc   0/1     ContainerCreating   0          2s   ← substituto
```

O Pod apagado foi substituído em segundos, com outro nome. O nome tem três partes: o Deployment (`minha-api`), um hash que identifica o ReplicaSet (`7d9c8b6f5d`) e um sufixo aleatório do Pod.

---

## 2. Deployment: réplicas, atualização e rollback

O **Deployment** é o objeto que você vai usar para praticamente toda aplicação *stateless*: APIs, sites, workers. Ele gerencia ReplicaSets, e é isso que permite **trocar de versão de forma controlada**.

### 2.1 O manifesto

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minha-api
  labels:
    app: minha-api
spec:
  replicas: 3                     # quantas cópias manter
  selector:
    matchLabels:
      app: minha-api              # "meus Pods são os que têm esta label"...
  template:                       # ...e é assim que eles são criados:
    metadata:
      labels:
        app: minha-api            # ← precisa bater com o selector acima
    spec:
      containers:
        - name: api
          image: seu-usuario/minha-api:1.0
          ports:
            - containerPort: 8000
```

O bloco `template` é um Pod completo (o mesmo `spec` do capítulo *Kubernetes: fundamentos e arquitetura*), usado como molde. A ligação entre o Deployment e seus Pods acontece **pela label**: o `selector.matchLabels` precisa casar com as `labels` do template, ou o Kubernetes recusa o manifesto.

```bash
kubectl apply -f deployment.yaml
kubectl get deployments
# NAME        READY   UP-TO-DATE   AVAILABLE   AGE
# minha-api   3/3     3            3           20s
```

A hierarquia que se forma:

```
Deployment  minha-api               ← você cria e edita este
    │
    └── ReplicaSet  minha-api-7d9c8b6f5d   (versão 1.0)   ← o Deployment cria
            ├── Pod  ...-4xk2p                            ← o ReplicaSet cria
            ├── Pod  ...-9mzq8
            └── Pod  ...-tr7wn
```

### 2.2 Escalando

Mudar o número de réplicas é mudar o estado desejado. O jeito declarativo é editar `replicas: 5` no arquivo e rodar `kubectl apply` de novo. Para testes, existe o atalho:

```bash
kubectl scale deployment minha-api --replicas=5
```

### 2.3 Rolling update: trocando de versão sem downtime

Quando você muda o **template** do Deployment (normalmente, a tag da imagem), ele cria um **ReplicaSet novo** e transfere as réplicas do antigo para o novo **aos poucos**. É a estratégia **Rolling Update** descrita em [../../03-entrega-continua/cd/02_estrategias_de_deploy.md](../../03-entrega-continua/cd/02_estrategias_de_deploy.md), executada pelo próprio cluster:

```bash
kubectl set image deployment/minha-api api=seu-usuario/minha-api:1.1
kubectl rollout status deployment/minha-api
# Waiting for deployment "minha-api" rollout to finish: 1 of 3 updated replicas are available...
# Waiting for deployment "minha-api" rollout to finish: 2 of 3 updated replicas are available...
# deployment "minha-api" successfully rolled out
```

```
         ReplicaSet v1.0          ReplicaSet v1.1
início   ● ● ●                    (vazio)
passo 1  ● ● ●                    ○          ← sobe 1 nova (ainda não pronta)
passo 2  ● ●                      ●          ← a nova ficou pronta: derruba 1 antiga
passo 3  ● ●                      ● ○
passo 4  ●                        ● ●
passo 5  ●                        ● ● ○
fim      (0 réplicas, mas fica)   ● ● ●

● pronta e recebendo tráfego    ○ iniciando
```

O ritmo é controlado por dois parâmetros:

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # até 1 Pod ACIMA do número desejado durante a troca
      maxUnavailable: 0    # nenhum Pod abaixo: a capacidade nunca cai
```

Com `maxUnavailable: 0`, o cluster só derruba uma réplica antiga **depois** que uma nova estiver pronta. Mas "pronta" depende de como o cluster sabe que um Pod está pronto, e isso é assunto da seção 3. Sem uma *readiness probe*, o Kubernetes considera o Pod pronto assim que o container inicia, mesmo que a aplicação ainda esteja carregando.

### 2.4 Rollback

O ReplicaSet antigo **não é apagado**: ele fica com 0 réplicas, guardando o template da versão anterior. Voltar atrás é reativá-lo:

```bash
kubectl rollout history deployment/minha-api
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         <none>

kubectl rollout undo deployment/minha-api                  # volta para a revisão anterior
kubectl rollout undo deployment/minha-api --to-revision=1  # ou para uma específica
```

Como os Pods antigos são recriados a partir de uma imagem que já está em cache nos nodes, o rollback costuma levar segundos.

> **Confusão comum:** "publiquei uma nova `minha-api:latest`, rodei `kubectl apply` e nada aconteceu". ✅ **Mais preciso:** o Deployment só inicia um rollout quando o **template muda**. Se o manifesto diz `image: minha-api:latest` antes e depois, para o Kubernetes nada mudou, mesmo que a imagem por trás da tag seja outra. E, mesmo que os Pods fossem recriados, cada node pode ter a versão antiga da `latest` em cache. É o problema da tag mutável do capítulo [Docker: imagens e Dockerfile](../docker/02_imagens_e_dockerfile.md), agravado: além de não saber o que está rodando, você também perde o histórico de rollout e o `rollout undo`, porque todas as revisões apontam para a mesma tag. Em Kubernetes, use **sempre tags únicas** por versão (`1.4.2` ou o hash do commit). O pipeline de CD deve gerar a tag e atualizá-la no manifesto.

---

## 3. Probes: como o cluster sabe se o Pod está bem

O kubelet de cada node (capítulo *Kubernetes: fundamentos e arquitetura*) pode testar periodicamente cada container. Esses testes se chamam **probes**, e existem três, cada um com uma consequência diferente quando falha:

| Probe | Pergunta | Se falhar... |
|---|---|---|
| **`livenessProbe`** | "O processo está vivo, ou travou?" | O kubelet **reinicia** o container |
| **`readinessProbe`** | "Está pronto para receber tráfego agora?" | O Pod é **retirado do Service** (não recebe requisições), mas **não** é reiniciado |
| **`startupProbe`** | "Já terminou de iniciar?" | As outras duas só começam depois que esta passar. Se ela nunca passar, o container é reiniciado |

É exatamente a separação entre *liveness* e *readiness* defendida em [Arquitetura de Software: load_balancer.md](https://github.com/iranhonorato/Arquitetura-de-Software/blob/main/system-design-pt2/load_balancer.md). O Kubernetes a transformou em dois campos do Pod.

```yaml
    spec:
      containers:
        - name: api
          image: seu-usuario/minha-api:1.1
          ports:
            - containerPort: 8000
          startupProbe:              # dá até 30 x 2s = 60s para a aplicação subir
            httpGet:
              path: /healthz
              port: 8000
            failureThreshold: 30
            periodSeconds: 2
          livenessProbe:             # depois disso, verifica a cada 10s se o processo responde
            httpGet:
              path: /healthz
              port: 8000
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:            # e a cada 5s se pode receber tráfego
            httpGet:
              path: /ready
              port: 8000
            periodSeconds: 5
            failureThreshold: 2
```

Do lado da aplicação, os dois endpoints têm papéis diferentes:

```python
@app.get("/healthz")
def liveness():
    # Raso: só prova que o processo está de pé e respondendo.
    return {"status": "vivo"}

@app.get("/ready")
def readiness():
    # Verifica o que esta instância precisa para atender bem.
    if not pool_do_banco.tem_conexao_livre():
        raise HTTPException(status_code=503)
    return {"status": "pronto"}
```

As probes também funcionam com comandos (`exec`) e com conexões TCP (`tcpSocket`), para aplicações que não falam HTTP.

> **Confusão comum:** "a `livenessProbe` deve verificar o banco de dados, para reiniciar a API quando o banco cair". ✅ **Mais preciso:** reiniciar a API **não conserta o banco**. Se a liveness consulta o banco e ele fica lento por 30 segundos, todas as réplicas falham a probe ao mesmo tempo e o kubelet reinicia **todas**. Elas voltam, ainda sem banco, falham de novo e são reiniciadas de novo: o status vira `CrashLoopBackOff`, com o cluster esperando cada vez mais entre as tentativas. O resultado é que, quando o banco se recupera, a API está fora do ar por causa dos reinícios. A liveness deve ser **rasa** e verificar só o próprio processo. Dependências pertencem à readiness, que tira o Pod do tráfego **sem matá-lo** e o devolve sozinha quando a dependência volta.

---

## 4. Requests e limits: reservando e limitando recursos

Cada container pode declarar quanto de CPU e memória precisa:

```yaml
          resources:
            requests:            # o que o container RESERVA
              cpu: "250m"        # 250 milicores = 1/4 de um núcleo
              memory: "256Mi"
            limits:              # o máximo que ele PODE USAR
              cpu: "1"
              memory: "512Mi"
```

Os dois campos têm funções bem diferentes:

- **`requests` é usado pelo scheduler.** Ao escolher um node, o scheduler soma os *requests* dos Pods que já estão lá e só coloca um Pod novo se couber. O request é uma **reserva**: o node "guarda" aquela capacidade para o Pod, mesmo que ele não use.
- **`limits` é aplicado pelo kernel**, através dos cgroups que você viu no capítulo [Docker: arquitetura e isolamento](../docker/01_arquitetura_e_isolamento.md). E o que acontece ao atingir o limite depende do recurso:

| Recurso | Ao atingir o limit |
|---|---|
| **CPU** | O container é **desacelerado** (*throttling*). Fica lento, mas continua rodando. |
| **Memória** | O container é **morto** pelo kernel (`OOMKilled`, código de saída 137) e reiniciado. |

```bash
kubectl describe pod minha-api-7d9c8b6f5d-9mzq8
# ...
#     Last State:     Terminated
#       Reason:       OOMKilled
#       Exit Code:    137
#     Restart Count:  4
```

Se um Pod fica no status `Pending` para sempre, a primeira suspeita é a falta de capacidade: nenhum node tem CPU ou memória livre suficiente para os *requests* dele. O `kubectl describe pod` mostra o motivo nos eventos: `0/3 nodes are available: 3 Insufficient memory`.

> **Confusão comum:** "não preciso declarar requests: assim o container usa o que precisar". ✅ **Mais preciso:** sem requests, o scheduler acha que o Pod **não ocupa nada** e pode empilhar dezenas deles no mesmo node. Quando todos começam a trabalhar, o node fica sem memória, e o kernel passa a matar processos. Os Pods sem requests nem limits são justamente os primeiros escolhidos para serem despejados quando o node aperta (o Kubernetes os chama de classe *BestEffort*). Requests bem calibrados são o que permite ao scheduler distribuir bem a carga, e também são a base do auto-scaling da seção 7. O ponto de partida razoável é medir o consumo real (`kubectl top pods`) e declarar o request perto do uso típico e o limit de memória com uma folga acima do pico.

---

## 5. Service: um endereço estável para Pods que mudam

### 5.1 O problema

Uma API com 3 réplicas tem 3 IPs, e eles mudam a cada atualização, a cada falha e a cada escala. Quem quiser chamar essa API (o frontend, outro serviço, um Ingress) precisaria acompanhar a lista o tempo todo. É o mesmo problema que o DNS do Docker resolveu no capítulo [Docker: volumes e redes](../docker/03_volumes_e_redes.md), só que agora entre muitas máquinas.

### 5.2 A solução

Um **Service** dá a um grupo de Pods **um nome e um IP fixos**, e distribui as conexões entre os Pods que estiverem prontos naquele momento. Ele encontra os Pods pelo **selector**, como o Deployment:

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: minha-api
spec:
  selector:
    app: minha-api        # manda tráfego para os Pods com esta label...
  ports:
    - port: 80            # ...recebendo na porta 80 do Service...
      targetPort: 8000    # ...e entregando na porta 8000 do container
```

```bash
kubectl apply -f service.yaml
kubectl get service minha-api
# NAME        TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
# minha-api   ClusterIP   10.96.142.31   <none>        80/TCP    5s
```

O `CLUSTER-IP` (`10.96.142.31`) não muda enquanto o Service existir, não importa quantas vezes os Pods sejam trocados. E qualquer Pod do cluster pode chamá-lo **pelo nome**, graças ao DNS interno do cluster (o CoreDNS, que roda no `kube-system`):

```bash
# De dentro de qualquer Pod do mesmo namespace:
curl http://minha-api

# De outro namespace, com o nome completo:
curl http://minha-api.loja.svc.cluster.local
#          └──┬────┘ └┬─┘ └─────┬───────┘
#          Service  namespace  sufixo do cluster
```

Para ver quais Pods estão por trás do Service neste instante:

```bash
kubectl get endpointslices -l kubernetes.io/service-name=minha-api
# NAME              ADDRESSTYPE   PORTS   ENDPOINTS
# minha-api-x8k2m   IPv4          8000    10.244.1.17,10.244.2.9,10.244.2.14
```

Essa lista é atualizada continuamente: Pods que falham na **readinessProbe** saem dela, e voltam quando passam de novo. É assim que a seção 3 se liga a esta.

### 5.3 Os tipos de Service

O campo `type` define **de onde** o Service pode ser alcançado. Cada tipo acrescenta uma camada ao anterior:

```
type: ClusterIP  (padrão)
  Só dentro do cluster.                   Pod ──▶ minha-api:80 ──▶ Pods

type: NodePort
  ClusterIP + uma porta (30000–32767)     Internet ──▶ IP-de-qualquer-node:30080
  aberta em TODOS os nodes.                                   ──▶ Service ──▶ Pods

type: LoadBalancer
  NodePort + um load balancer da nuvem    Internet ──▶ LB da AWS (IP/DNS público)
  criado automaticamente.                               ──▶ nodes ──▶ Service ──▶ Pods
```

| Tipo | Acessível de onde | Uso típico |
|---|---|---|
| **ClusterIP** | Só de dentro do cluster | Comunicação entre serviços internos (API → banco, API → Redis) |
| **NodePort** | De fora, pelo IP de um node + porta alta | Testes e clusters locais; raramente em produção |
| **LoadBalancer** | De fora, por um IP ou DNS público | Expor um serviço diretamente na nuvem (o `cloud-controller-manager` do capítulo *Kubernetes: fundamentos e arquitetura* cria o LB) |

Durante o desenvolvimento, para acessar um Service ClusterIP da sua máquina sem expô-lo, existe um atalho:

```bash
kubectl port-forward service/minha-api 8080:80
# Agora http://localhost:8080 chega ao Service, de dentro do cluster
```

> **Confusão comum:** "o Service é um container (ou um proxy) que recebe o tráfego e repassa para os Pods". ✅ **Mais preciso:** um Service do tipo ClusterIP **não é um processo**. Não há nenhum programa ouvindo no `10.96.142.31`. Esse IP é **virtual**: o `kube-proxy` de cada node (capítulo *Kubernetes: fundamentos e arquitetura*) escreve regras no `iptables` (ou IPVS) do kernel dizendo "pacotes para `10.96.142.31:80` devem ser reescritos para o IP de um destes Pods". O balanceamento acontece no kernel do node de **origem**, antes de o pacote sair da máquina. Duas consequências práticas: `ping` no IP de um Service normalmente não responde (as regras tratam as portas TCP/UDP declaradas, não ICMP), e o balanceamento é por **conexão**, não por requisição. Com conexões persistentes (HTTP/2, gRPC), um cliente pode ficar preso a um único Pod, o que exige balanceamento na camada de aplicação (camada 7, veja [Arquitetura de Software: load_balancer.md](https://github.com/iranhonorato/Arquitetura-de-Software/blob/main/system-design-pt2/load_balancer.md)).

---

## 6. Ingress: a entrada HTTP do cluster

### 6.1 O problema

Com um Service `LoadBalancer` para cada aplicação, um sistema com 10 serviços HTTP teria 10 load balancers na nuvem, cada um com seu custo e seu IP. E nenhum deles faria roteamento por caminho ou domínio (`/api` para um serviço, `/admin` para outro), nem centralizaria os certificados TLS.

### 6.2 A solução

Um **Ingress** descreve regras de roteamento **HTTP** (camada 7) para dentro do cluster: qual domínio e qual caminho vão para qual Service.

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: loja
spec:
  ingressClassName: nginx            # qual controller vai implementar estas regras
  tls:
    - hosts: [loja.exemplo.com]
      secretName: loja-tls           # certificado guardado em um Secret (capítulo *Kubernetes: configuração, estado e segurança*)
  rules:
    - host: loja.exemplo.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: minha-api
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
```

```
                         Internet
                            │  https://loja.exemplo.com
                            ▼
                ┌───────────────────────┐
                │  Ingress Controller   │  ← UM load balancer da nuvem na frente dele
                │  (TLS, regras L7)     │
                └─────┬───────────┬─────┘
             /api/*   │           │   /*
                      ▼           ▼
             Service minha-api   Service frontend   (ClusterIP)
                      │           │
                   Pods API     Pods frontend
```

O Ingress Controller é, essencialmente, o **proxy reverso** descrito em [Arquitetura de Software: nginx.md](https://github.com/iranhonorato/Arquitetura-de-Software/blob/main/system-design-pt2/nginx.md), configurado automaticamente a partir dos objetos Ingress do cluster. Em um único ponto de entrada, ele concentra TLS, roteamento por host e caminho, e muitas das funções de [Arquitetura de Software: api_gateway.md](https://github.com/iranhonorato/Arquitetura-de-Software/blob/main/system-design-pt2/api_gateway.md).

> **Confusão comum:** "criei o Ingress, mas nada responde: o Kubernetes está com defeito". ✅ **Mais preciso:** o objeto Ingress é **só uma declaração de regras**, e o Kubernetes não traz nenhum programa que as execute. É preciso instalar um **Ingress Controller** no cluster: um conjunto de Pods (com Nginx, Traefik, HAProxy, Envoy ou o AWS Load Balancer Controller) que observa os objetos Ingress e se configura a partir deles. Sem controller, o Ingress é criado sem erro e é ignorado. O campo `ingressClassName` diz qual controller deve tratá-lo. Vale saber também que a comunidade Kubernetes desenvolveu a **Gateway API** como sucessora do Ingress, com mais recursos (divisão de tráfego por peso, cabeçalhos, protocolos além de HTTP) e papéis separados para quem opera a infraestrutura e quem configura as rotas. O popular controller *ingress-nginx* foi descontinuado pelo projeto, e para clusters novos a Gateway API é a direção recomendada. Os conceitos deste capítulo valem para os dois.

---

## 7. Auto-scaling: ajustando réplicas à carga

### 7.1 HorizontalPodAutoscaler

O **HorizontalPodAutoscaler** (HPA) é um loop de reconciliação que **altera o `replicas` de um Deployment** conforme uma métrica. O caso clássico é a CPU:

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: minha-api
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: minha-api
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70     # mira em 70% do *request* de CPU, na média dos Pods
```

O cálculo é proporcional. Com 3 réplicas usando, em média, 140% do request de CPU, a meta de 70% exige o dobro de réplicas:

```
réplicas desejadas = ⌈ réplicas atuais × (uso atual ÷ meta) ⌉
                   = ⌈ 3 × (140 ÷ 70) ⌉ = 6
```

Duas condições precisam estar presentes para isso funcionar:

1. **O `metrics-server` instalado no cluster**, que coleta o uso de CPU e memória dos Pods. Ele vem pronto nos clusters gerenciados e é instalado à parte no kind.
2. **`requests` de CPU declarados** (seção 4). A porcentagem é calculada **sobre o request**. Sem ele, o HPA não tem base de cálculo e não age.

```bash
kubectl get hpa minha-api --watch
# NAME        REFERENCE              TARGETS   MINPODS   MAXPODS   REPLICAS
# minha-api   Deployment/minha-api   35%/70%   3         20        3
# minha-api   Deployment/minha-api   140%/70%  3         20        3
# minha-api   Deployment/minha-api   140%/70%  3         20        6
# minha-api   Deployment/minha-api   72%/70%   3         20        6
```

Para reduzir réplicas, o HPA é deliberadamente mais lento: por padrão, espera alguns minutos de carga baixa antes de remover Pods, para não ficar oscilando a cada pico curto.

### 7.2 E quando os nodes enchem?

> **Confusão comum:** "com o HPA, o cluster escala sozinho até onde for preciso". ✅ **Mais preciso:** o HPA escala **Pods**, não **máquinas**. Se os nodes do cluster já estão cheios, os Pods novos que ele pede ficam em `Pending`, esperando um espaço que não existe. Para que o número de **nodes** também acompanhe a demanda, é preciso um segundo componente, que observa Pods `Pending` por falta de recursos e cria máquinas novas na nuvem: o **Cluster Autoscaler** ou, na AWS, o **Karpenter**. Os dois níveis se complementam: o HPA ajusta as réplicas em segundos, e o autoscaler de nodes ajusta a capacidade em minutos (o tempo de ligar uma VM). É a **elasticidade** de [Arquitetura de Software: escalabilidade_horizontal_vs_vertical.md](https://github.com/iranhonorato/Arquitetura-de-Software/blob/main/system-design-pt2/escalabilidade_horizontal_vs_vertical.md), automatizada em duas camadas.

---

## Juntando tudo

Os objetos deste capítulo formam a estrutura padrão de uma aplicação *stateless* no Kubernetes:

```
                 Internet
                    │
            ┌───────▼────────┐
            │    Ingress     │  regras HTTP: host, caminho, TLS
            └───────┬────────┘
            ┌───────▼────────┐
            │    Service     │  nome e IP estáveis; só Pods "ready"
            └───────┬────────┘
                    │ selector: app=minha-api
            ┌───────▼────────┐        ┌──────────────┐
            │   Deployment   │◀───────│     HPA      │  ajusta replicas
            └───────┬────────┘        └──────────────┘
            ┌───────▼────────┐
            │   ReplicaSet   │  mantém N Pods da versão atual
            └───────┬────────┘
             ┌──────┼──────┐
            Pod    Pod    Pod   probes + requests/limits
```

Cada objeto tem uma única responsabilidade, e todos se ligam por **labels e selectors**. Nenhum guarda referência direta a outro.

---

## Próximo passo

Tudo neste capítulo foi *stateless*: Pods intercambiáveis, que podem ser destruídos e recriados em qualquer node. Mas a aplicação ainda tem senhas e configurações escritas no manifesto, e o banco de dados do capítulo [Docker: Compose e produção](../docker/04_compose_e_producao.md) ainda não tem onde morar, porque um banco **não** é intercambiável: cada réplica tem seus próprios dados, e eles não podem sumir com o Pod.

O [03_configuracao_estado_e_seguranca.md](03_configuracao_estado_e_seguranca.md) trata de ConfigMaps, Secrets, armazenamento persistente, StatefulSets, os demais tipos de workload e das permissões dentro do cluster.
