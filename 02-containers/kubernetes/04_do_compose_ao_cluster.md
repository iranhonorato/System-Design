# Kubernetes na Prática: do Compose ao Cluster

**Pré-requisito:** [03_configuracao_estado_e_seguranca.md](03_configuracao_estado_e_seguranca.md)

> Segue a documentação oficial do Kubernetes (*Tasks: Debug Applications*, *Managing Workloads*) e do kind. A decisão sobre adotar ou não o Kubernetes retoma o princípio de evitar complexidade acidental de [Arquitetura de Software: escalabilidade_horizontal_vs_vertical.md](https://github.com/iranhonorato/Arquitetura-de-Software/blob/main/system-design-pt2/escalabilidade_horizontal_vs_vertical.md).

---

## Por que este arquivo existe

Os três capítulos anteriores apresentaram os objetos do Kubernetes um a um, cada um isolado em seu próprio exemplo. Na prática, o trabalho é outro: você tem um sistema que já funciona com Docker Compose e precisa levá-lo para um cluster. E, quando algo não sobe, precisa descobrir o porquê olhando para um `STATUS` como `CrashLoopBackOff` ou `ImagePullBackOff`.

Este capítulo faz as duas coisas. Primeiro, migra o sistema do capítulo [Docker: Compose e produção](../docker/04_compose_e_producao.md) (API, Postgres e Redis) para manifestos Kubernetes, arquivo por arquivo. Depois, monta um roteiro de diagnóstico para os erros mais comuns. Por fim, apresenta as ferramentas usadas em produção e os clusters gerenciados, que levam aos capítulos de VPC e IAM.

---

## 1. Traduzindo o Compose para o Kubernetes

Cada parte do `docker-compose.yml` tem um equivalente, normalmente mais explícito:

| No Compose | No Kubernetes | Onde ver |
|---|---|---|
| Um serviço (`api:`) | Um **Deployment** (ou StatefulSet) + um **Service** | Kubernetes 02 e 03 |
| `image:` | `image:` no template do Pod | Kubernetes 02 |
| `build: .` | **Não existe.** A imagem é construída antes (pelo CI) e enviada a um registry | Docker 02 |
| `ports: "8000:8000"` | **Service** (interno) e **Ingress** (externo) | Kubernetes 02 |
| `environment:` | **ConfigMap** (valores comuns) e **Secret** (sensíveis) | Kubernetes 03 |
| arquivo `.env` | Secret criado por linha de comando ou sincronizado de um cofre | Kubernetes 03 |
| `volumes:` nomeados | **PersistentVolumeClaim** (via `volumeClaimTemplates` no StatefulSet) | Kubernetes 03 |
| `depends_on` + `healthcheck` | **Probes** + **initContainers** + nova tentativa de conexão na aplicação | Kubernetes 02 e seção 2.4 |
| `restart: unless-stopped` | **Automático**: é o próprio loop de reconciliação | Kubernetes 01 |
| A rede automática do projeto | O DNS de **Services** do cluster | Kubernetes 02 |
| `docker compose exec api migrate` | Um **Job** | Kubernetes 03 |

A linha do `build:` é a mudança mais importante de fluxo. Os nodes do cluster não têm o seu código-fonte: eles só sabem **baixar imagens** de um registry. O caminho vira:

```
código ──▶ docker build ──▶ docker push (registry) ──▶ manifesto aponta a tag ──▶ cluster baixa
          └──────────── feito pelo CI (03-entrega-continua/ci/) ───────────────────────────┘
```

---

## 2. Os manifestos, um por um

Uma organização comum é uma pasta `k8s/` no repositório, com um arquivo por componente:

```
minha-loja/
├── app/ ...
├── Dockerfile
├── docker-compose.yml        ← continua existindo, para desenvolvimento local
└── k8s/
    ├── 00-namespace.yaml
    ├── 10-config.yaml
    ├── 20-postgres.yaml
    ├── 30-redis.yaml
    ├── 40-api.yaml
    └── 50-migracao-job.yaml
```

Os números no início dos nomes não têm significado para o Kubernetes. Eles só deixam a ordem de leitura evidente para quem abre a pasta.

### 2.1 Namespace e configuração

```yaml
# k8s/00-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: loja
```

```yaml
# k8s/10-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
  namespace: loja
data:
  REDIS_URL: "redis://redis:6379"
  LOG_LEVEL: "info"
```

O Secret **não** fica em arquivo. Ele é criado por comando (ou, em produção, sincronizado de um cofre, como no capítulo *Kubernetes: configuração, estado e segurança*):

```bash
kubectl create secret generic postgres-cred -n loja \
  --from-literal=POSTGRES_USER=app \
  --from-literal=POSTGRES_PASSWORD='troque-esta-senha' \
  --from-literal=POSTGRES_DB=loja
```

### 2.2 Postgres: StatefulSet

É o StatefulSet do capítulo [Kubernetes: configuração, estado e segurança](03_configuracao_estado_e_seguranca.md), agora com uma readiness probe que faz o papel do `healthcheck` do Compose:

```yaml
# k8s/20-postgres.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: loja
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - port: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: loja
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          envFrom:
            - secretRef:
                name: postgres-cred
          env:
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          ports:
            - containerPort: 5432
          readinessProbe:                       # o equivalente ao healthcheck do Compose
            exec:
              command: ["sh", "-c", "pg_isready -U \"$POSTGRES_USER\" -d \"$POSTGRES_DB\""]
            periodSeconds: 5
          resources:
            requests: { cpu: "250m", memory: "256Mi" }
            limits:   { memory: "512Mi" }
          volumeMounts:
            - name: dados
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: dados
      spec:
        accessModes: [ReadWriteOnce]
        resources:
          requests:
            storage: 5Gi
```

### 2.3 Redis: Deployment

Aqui o Redis é usado como **cache**, e por isso pode perder os dados em uma reinicialização. Ele vira um Deployment simples, sem disco persistente:

```yaml
# k8s/30-redis.yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: loja
spec:
  selector:
    app: redis
  ports:
    - port: 6379
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: loja
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:7.2-alpine
          args: ["--maxmemory", "200mb", "--maxmemory-policy", "allkeys-lru"]
          ports:
            - containerPort: 6379
          resources:
            requests: { cpu: "100m", memory: "128Mi" }
            limits:   { memory: "256Mi" }
```

O `--maxmemory 200mb` fica abaixo do limit de 256Mi de propósito. Se o Redis pudesse usar toda a memória do limit, o kernel o mataria (`OOMKilled`) antes de ele começar a descartar chaves.

### 2.4 API: Deployment, com espera pelo banco

```yaml
# k8s/40-api.yaml
apiVersion: v1
kind: Service
metadata:
  name: minha-api
  namespace: loja
spec:
  selector:
    app: minha-api
  ports:
    - port: 80
      targetPort: 8000
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minha-api
  namespace: loja
spec:
  replicas: 3
  selector:
    matchLabels:
      app: minha-api
  strategy:
    rollingUpdate: { maxSurge: 1, maxUnavailable: 0 }
  template:
    metadata:
      labels:
        app: minha-api
    spec:
      automountServiceAccountToken: false     # a API não fala com o cluster (capítulo *Kubernetes: configuração, estado e segurança*)
      initContainers:                         # roda ATÉ TERMINAR, antes do container principal
        - name: espera-postgres
          image: postgres:16-alpine
          command: ["sh", "-c", "until pg_isready -h postgres -p 5432; do echo aguardando; sleep 2; done"]
      containers:
        - name: api
          image: seu-usuario/minha-api:1.0
          ports:
            - containerPort: 8000
          envFrom:
            - configMapRef:
                name: api-config
          env:
            - name: POSTGRES_USER
              valueFrom: { secretKeyRef: { name: postgres-cred, key: POSTGRES_USER } }
            - name: POSTGRES_PASSWORD
              valueFrom: { secretKeyRef: { name: postgres-cred, key: POSTGRES_PASSWORD } }
            - name: DATABASE_URL                # $(VAR) usa as variáveis definidas acima
              value: "postgresql://$(POSTGRES_USER):$(POSTGRES_PASSWORD)@postgres:5432/loja"
          readinessProbe:
            httpGet: { path: /ready, port: 8000 }
            periodSeconds: 5
          livenessProbe:
            httpGet: { path: /healthz, port: 8000 }
            periodSeconds: 10
          resources:
            requests: { cpu: "250m", memory: "256Mi" }
            limits:   { memory: "512Mi" }
          securityContext:
            runAsNonRoot: true
            allowPrivilegeEscalation: false
```

O **initContainer** é a peça nova. Ele roda antes do container principal e precisa **terminar com sucesso** para que a API seja iniciada. Aqui, ele repete `pg_isready` até o banco responder, o que reproduz o `depends_on: condition: service_healthy` do Compose. Mas só **na inicialização**: se o banco cair depois, é a readinessProbe da API (e a lógica de nova tentativa dentro da aplicação) que trata o problema.

O `runAsNonRoot: true` exige que a imagem tenha um usuário sem privilégios, como a imagem multi-stage do capítulo [Docker: Compose e produção](../docker/04_compose_e_producao.md).

### 2.5 Migração: Job

```yaml
# k8s/50-migracao-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: migracao-1-0
  namespace: loja
spec:
  backoffLimit: 3
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migracao
          image: seu-usuario/minha-api:1.0
          command: ["alembic", "upgrade", "head"]
          env:
            - name: POSTGRES_USER
              valueFrom: { secretKeyRef: { name: postgres-cred, key: POSTGRES_USER } }
            - name: POSTGRES_PASSWORD
              valueFrom: { secretKeyRef: { name: postgres-cred, key: POSTGRES_PASSWORD } }
            - name: DATABASE_URL
              value: "postgresql://$(POSTGRES_USER):$(POSTGRES_PASSWORD)@postgres:5432/loja"
```

O nome inclui a versão (`migracao-1-0`) porque Jobs são imutáveis: para rodar a migração da próxima versão, cria-se um Job novo.

---

## 3. Subindo no cluster local

Com o cluster kind do capítulo [Kubernetes: fundamentos e arquitetura](01_fundamentos_e_arquitetura.md):

```bash
# 1. Constrói a imagem e a carrega DIRETO nos nodes do kind (sem precisar de registry)
docker build -t seu-usuario/minha-api:1.0 .
kind load docker-image seu-usuario/minha-api:1.0 --name estudo

# 2. Cria o namespace, depois o Secret, depois aplica a pasta inteira
kubectl apply -f k8s/00-namespace.yaml
kubectl create secret generic postgres-cred -n loja \
  --from-literal=POSTGRES_USER=app \
  --from-literal=POSTGRES_PASSWORD='troque-esta-senha' \
  --from-literal=POSTGRES_DB=loja
kubectl apply -f k8s/

# 3. Acompanha
kubectl get pods -n loja --watch
```

```
NAME                         READY   STATUS     RESTARTS   AGE
postgres-0                   0/1     Running    0          5s
redis-6c9f7d8b4-xk2lp        1/1     Running    0          5s
minha-api-5f8d9c7b6-2mxkq    0/1     Init:0/1   0          5s    ← esperando o banco
minha-api-5f8d9c7b6-8wq4n    0/1     Init:0/1   0          5s
minha-api-5f8d9c7b6-tz9vb    0/1     Init:0/1   0          5s
migracao-1-0-hb7kd           0/1     Error      0          5s    ← banco ainda não estava pronto
postgres-0                   1/1     Running    0          9s
migracao-1-0-q2x8m           0/1     Completed  0          14s   ← nova tentativa: sucesso
minha-api-5f8d9c7b6-2mxkq    0/1     PodInitializing  0    15s
minha-api-5f8d9c7b6-2mxkq    1/1     Running    0          18s
...
```

Repare em duas coisas nessa saída. A API passa por `Init:0/1` (o initContainer rodando) até o banco responder. E a primeira tentativa do Job de migração falha, porque não tem initContainer; o `backoffLimit` faz o Job tentar de novo, e a segunda tentativa funciona. Em um pipeline real, a migração costuma rodar como uma etapa separada, depois que o banco já está de pé.

Para testar a API sem configurar um Ingress:

```bash
kubectl port-forward -n loja service/minha-api 8080:80
curl http://localhost:8080/ready
```

E para ver tudo o que foi criado:

```bash
kubectl get all,pvc,configmap,secret -n loja
```

Para atualizar a versão: construir `minha-api:1.1`, carregá-la no kind, mudar a tag em `40-api.yaml` e rodar `kubectl apply -f k8s/40-api.yaml`. O rolling update do capítulo [Kubernetes: workloads, rede e escala](02_workloads_rede_e_escala.md) faz o resto.

---

## 4. Quando algo não funciona: roteiro de diagnóstico

O primeiro passo é sempre o mesmo: **olhar a coluna `STATUS`** do `kubectl get pods`. Cada status aponta para uma etapa diferente do fluxo do capítulo [Kubernetes: fundamentos e arquitetura](01_fundamentos_e_arquitetura.md) (seção 5), e por isso cada um tem um lugar diferente para investigar:

```
kubectl get pods
        │
        ├── Pending ─────────────────▶ o scheduler não achou lugar para o Pod
        │                               → kubectl describe pod  (seção "Events")
        │
        ├── ErrImagePull /
        │   ImagePullBackOff ────────▶ o kubelet não conseguiu baixar a imagem
        │                               → kubectl describe pod
        │
        ├── CrashLoopBackOff ────────▶ o container inicia e morre, repetidamente
        │                               → kubectl logs --previous
        │
        ├── Running, mas READY 0/1 ──▶ a readinessProbe está falhando
        │                               → kubectl describe pod  +  kubectl logs
        │
        └── Running e READY 1/1,
            mas não responde ────────▶ o problema está no Service ou na rede
                                        → kubectl get endpointslices  +  port-forward
```

### Pending

O Pod existe, mas não foi colocado em nenhum node.

```bash
kubectl describe pod minha-api-5f8d9c7b6-2mxkq -n loja
# Events:
#   Warning  FailedScheduling  0/3 nodes are available: 3 Insufficient memory.
```

| Mensagem no evento | Causa | O que fazer |
|---|---|---|
| `Insufficient cpu` / `Insufficient memory` | Os *requests* não cabem em nenhum node | Reduzir os requests ou acrescentar nodes |
| `pod has unbound immediate PersistentVolumeClaims` | O PVC não conseguiu um disco | `kubectl describe pvc`: StorageClass errada ou inexistente |
| `node(s) had volume node affinity conflict` | O disco está em uma zona sem node livre | Ver a confusão sobre zonas no capítulo [Kubernetes: configuração, estado e segurança](03_configuracao_estado_e_seguranca.md) |

### ErrImagePull e ImagePullBackOff

O kubelet tentou baixar a imagem e falhou. O sufixo *BackOff* indica que ele está esperando cada vez mais entre as tentativas. As causas, quase sempre: **nome ou tag errados** (um erro de digitação), **registry privado sem credencial** (é preciso um Secret do tipo `docker-registry` e o campo `imagePullSecrets`), ou, no kind, a imagem **não foi carregada** com `kind load`.

### CrashLoopBackOff

O container inicia, o processo termina com erro, o kubelet reinicia, e ele termina de novo. A resposta está nos logs, mas da execução **anterior**, porque a atual pode ainda nem ter começado:

```bash
kubectl logs minha-api-5f8d9c7b6-2mxkq -n loja --previous
# sqlalchemy.exc.OperationalError: could not translate host name "postgress" to address
```

(Aqui, um erro de digitação no nome do Service dentro do `DATABASE_URL`.) Se os logs estiverem vazios, olhe o `describe`:

```bash
kubectl describe pod minha-api-5f8d9c7b6-2mxkq -n loja
#     Last State:  Terminated
#       Reason:    OOMKilled        ← limit de memória pequeno demais
#       Exit Code: 137
```

| Exit code | Significado comum |
|---|---|
| `1` | Erro da aplicação: ver os logs |
| `137` | Morto com `SIGKILL`: geralmente `OOMKilled`, ou liveness falhando repetidamente |
| `127` | Comando não encontrado: `command` ou `CMD` errado |
| `126` | Sem permissão para executar: `runAsNonRoot` com arquivos que só o root executa |

### Running, mas não Ready

O processo está de pé, mas a readinessProbe falha, e por isso o Pod não recebe tráfego. Os eventos dizem qual probe e por quê:

```bash
kubectl describe pod minha-api-5f8d9c7b6-2mxkq -n loja
#   Warning  Unhealthy  Readiness probe failed: HTTP probe failed with statuscode: 503
```

Em seguida, os logs da aplicação mostram por que o `/ready` responde 503, que normalmente é uma dependência inacessível.

### Running e Ready, mas sem resposta

Se os Pods estão saudáveis e mesmo assim as requisições não chegam, o problema está **entre** o cliente e os Pods:

```bash
# 1. O Service está encontrando Pods? Lista vazia = selector não bate com as labels
kubectl get endpointslices -n loja -l kubernetes.io/service-name=minha-api

# 2. Comparar o selector do Service com as labels dos Pods
kubectl get service minha-api -n loja -o jsonpath='{.spec.selector}'
kubectl get pods -n loja --show-labels

# 3. Testar o Pod diretamente, sem passar pelo Service
kubectl port-forward -n loja pod/minha-api-5f8d9c7b6-2mxkq 8080:8000
```

Os erros mais comuns aqui são um **selector que não bate** (um `app: minha-api` contra um `app: api`) e um **`targetPort` errado** (o Service manda para a 80, mas o container ouve na 8000).

### Ferramentas que valem para todos os casos

```bash
# Eventos recentes do namespace, em ordem cronológica
kubectl get events -n loja --sort-by=.lastTimestamp

# Um terminal dentro do container (se a imagem tiver shell)
kubectl exec -it minha-api-5f8d9c7b6-2mxkq -n loja -- sh

# Um container temporário de diagnóstico ANEXADO ao Pod (para imagens sem shell)
kubectl debug -it minha-api-5f8d9c7b6-2mxkq -n loja --image=busybox --target=api

# Um Pod descartável para testar a rede de dentro do cluster
kubectl run teste --rm -it -n loja --image=busybox -- wget -qO- http://minha-api/ready
```

> **Confusão comum:** "o Pod está em `CrashLoopBackOff`, então o problema é do Kubernetes". ✅ **Mais preciso:** `CrashLoopBackOff` é o Kubernetes **funcionando**: ele cumpre o estado desejado (reiniciar o container que morreu) e aumenta a espera entre tentativas (10s, 20s, 40s, até 5 minutos) para não sobrecarregar o node. A causa está quase sempre **na aplicação ou na configuração dela**: uma variável de ambiente faltando, uma dependência inacessível, um comando errado, um limit de memória pequeno demais ou uma liveness agressiva demais. A pergunta útil não é "por que o Kubernetes reinicia?", e sim "por que o processo termina?", e quem responde é o `kubectl logs --previous`. Uma forma rápida de separar os dois mundos: rode a mesma imagem com as mesmas variáveis no Docker (`docker run -e ...`). Se ela morrer lá também, o problema não tem nada a ver com o cluster.

---

## 5. As ferramentas do ecossistema

Manter arquivos YAML à mão funciona para um sistema e um ambiente. Com vários ambientes e muitos serviços, surgem ferramentas para evitar repetição e automatizar a aplicação dos manifestos.

### Kustomize: variações sem copiar arquivos

Embutido no próprio `kubectl` (`kubectl apply -k`). Você mantém uma **base** com os manifestos comuns e **sobreposições** (*overlays*) que alteram só o que muda por ambiente:

```
k8s/
├── base/                     ← os manifestos da seção 2
│   └── kustomization.yaml
└── overlays/
    ├── dev/                  ← replicas: 1, recursos menores
    │   └── kustomization.yaml
    └── producao/             ← replicas: 5, tag da imagem fixada
        └── kustomization.yaml
```

```bash
kubectl apply -k k8s/overlays/producao
```

### Helm: pacotes de manifestos

O **Helm** é um gerenciador de pacotes para Kubernetes. Um pacote, chamado **chart**, é um conjunto de manifestos com variáveis, e quem instala ajusta só os valores. É a forma mais comum de instalar software de terceiros no cluster:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoramento prometheus-community/kube-prometheus-stack -n monitoramento --create-namespace
```

Um único comando instala Prometheus, Grafana e dezenas de objetos configurados para monitorar o cluster.

### GitOps: o Git como fonte da verdade

Em vez de o pipeline rodar `kubectl apply`, uma ferramenta **dentro do cluster** (Argo CD ou Flux) observa um repositório Git e aplica continuamente o que estiver lá. Para fazer um deploy, você faz um commit mudando a tag da imagem. É o loop de reconciliação do capítulo [Kubernetes: fundamentos e arquitetura](01_fundamentos_e_arquitetura.md) estendido até o Git: o estado desejado passa a ser **o que está no repositório**, e qualquer mudança manual no cluster é desfeita automaticamente.

---

## 6. Kubernetes gerenciado

Operar o **control plane** (API server, etcd com backups e alta disponibilidade, atualizações de versão a cada poucos meses) é um trabalho grande e especializado. Os provedores de nuvem oferecem clusters em que essa parte fica com eles:

| Provedor | Serviço |
|---|---|
| AWS | **EKS** (Elastic Kubernetes Service) |
| Google Cloud | **GKE** (Google Kubernetes Engine) |
| Azure | **AKS** (Azure Kubernetes Service) |

A divisão de responsabilidades lembra a tabela de IaaS e PaaS do capítulo [Fundamentos de infraestrutura](../../01-infraestrutura/fundamentos/01_o_que_e_infraestrutura.md):

```
                          Cluster próprio    EKS (nodes gerenciados)    EKS + Fargate / Auto Mode
Control plane             ■ você             □ AWS                      □ AWS
Nodes (VMs, atualizações) ■ você             ■ você (com ajuda da AWS)  □ AWS
Manifestos e aplicações   ■ você             ■ você                     ■ você
```

No EKS, o control plane tem um custo fixo por cluster (cerca de US$ 0,10 por hora, ou uns US$ 73 por mês), somado ao das instâncias EC2 que servem de nodes.

Um cluster EKS também não existe sozinho. Ele precisa de duas coisas da AWS, que são os assuntos dos próximos capítulos:

- **uma rede**: os nodes e os Pods recebem IPs de uma **VPC**, divididos em sub-redes públicas e privadas espalhadas por zonas de disponibilidade ([../../01-infraestrutura/aws/02_vpc_redes_na_aws.md](../../01-infraestrutura/aws/02_vpc_redes_na_aws.md));
- **identidades e permissões**: o cluster precisa de permissão para criar load balancers e discos, os nodes precisam de permissão para baixar imagens, e os Pods podem precisar acessar o S3, tudo controlado pelo **IAM** ([../../01-infraestrutura/aws/03_iam_identidades_e_permissoes.md](../../01-infraestrutura/aws/03_iam_identidades_e_permissoes.md)).

---

## 7. Quando usar Kubernetes (e quando não)

| Situação | Opção mais adequada |
|---|---|
| Um sistema pequeno, um servidor dá conta | **Docker Compose** em uma EC2 ([../../03-entrega-continua/cd/03_pipeline_de_cd_na_pratica.md](../../03-entrega-continua/cd/03_pipeline_de_cd_na_pratica.md)) |
| Poucos serviços, sem equipe para operar um cluster | Containers gerenciados sem cluster: **AWS ECS/Fargate**, **Google Cloud Run** |
| Muitos serviços, várias equipes, necessidade de padronizar deploy, escala e observabilidade | **Kubernetes gerenciado** (EKS, GKE, AKS) |
| Exigência de rodar fora da nuvem ou em várias nuvens com a mesma ferramenta | **Kubernetes** (gerenciado ou próprio) |

> **Confusão comum:** "toda empresa séria roda em Kubernetes, então meu projeto também deveria". ✅ **Mais preciso:** o Kubernetes resolve problemas de **coordenação em escala**: muitos serviços, muitas máquinas, muitas equipes publicando ao mesmo tempo. Em troca, cobra um custo **fixo e permanente**: alguém precisa entender os quatro capítulos anteriores em profundidade, manter o cluster atualizado, cuidar da rede, das permissões e da observabilidade. Para um sistema com três serviços e uma equipe pequena, esse custo costuma ser maior do que o problema que ele resolve, e é a mesma armadilha de adotar escala horizontal cedo demais, descrita em [Arquitetura de Software: escalabilidade_horizontal_vs_vertical.md](https://github.com/iranhonorato/Arquitetura-de-Software/blob/main/system-design-pt2/escalabilidade_horizontal_vs_vertical.md). A boa notícia é que tudo o que foi feito antes não se perde: uma aplicação bem empacotada (imagem enxuta, configuração por variáveis, *stateless*, health checks) migra do Compose para o ECS ou para o Kubernetes com pouco esforço quando a necessidade aparecer.

---

## Próximo passo

A parte de Kubernetes termina aqui, com um sistema completo rodando em um cluster local. Para levá-lo à nuvem, falta entender a infraestrutura da AWS em que um cluster (ou uma simples EC2) se apoia: os servidores virtuais, a rede privada em que eles vivem e o sistema de permissões que controla quem pode fazer o quê.

O [../../01-infraestrutura/aws/01_ec2_servidores_na_nuvem.md](../../01-infraestrutura/aws/01_ec2_servidores_na_nuvem.md) começa pelo mais concreto: criar um servidor na AWS, acessá-lo por SSH e rodar nele o mesmo Compose dos capítulos de Docker. Em seguida vêm a [VPC](../../01-infraestrutura/aws/02_vpc_redes_na_aws.md) e o [IAM](../../01-infraestrutura/aws/03_iam_identidades_e_permissoes.md).
