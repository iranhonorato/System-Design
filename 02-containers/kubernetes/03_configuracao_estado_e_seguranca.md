# Kubernetes: Configuração, Estado e Segurança

**Pré-requisito:** [02_workloads_rede_e_escala.md](02_workloads_rede_e_escala.md)

> Segue a documentação oficial do Kubernetes (*Configuration*, *Storage*, *Workloads* e *Security*). A discussão sobre bancos de dados replicados retoma [Arquitetura de Software: arquitetura_master_slave.md](https://github.com/iranhonorato/Arquitetura-de-Software/blob/main/system-design-pt2/arquitetura_master_slave.md).

---

## Por que este arquivo existe

O capítulo anterior construiu uma aplicação *stateless* completa, mas deixou três problemas em aberto:

1. **Configuração.** URLs, parâmetros e senhas ainda estariam escritos dentro do manifesto do Deployment. A mesma imagem precisa rodar em vários ambientes, mudando só a configuração, como o capítulo [Docker: Compose e produção](../docker/04_compose_e_producao.md) defendeu.
2. **Estado.** Um banco de dados não é intercambiável como uma réplica da API. Seus dados precisam sobreviver à morte do Pod, e cada réplica do banco tem uma identidade própria.
3. **Permissões.** Até aqui, qualquer Pod pode falar com qualquer outro Pod, e nada impede uma aplicação comprometida de consultar a API do cluster.

Este capítulo trata dos três, nessa ordem, e apresenta os tipos de workload que faltam: DaemonSet, Job e CronJob.

---

## 1. ConfigMap: configuração fora da imagem

Um **ConfigMap** guarda pares chave-valor de configuração **não sensível**, separados da imagem e do Deployment.

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
data:
  LOG_LEVEL: "info"
  REDIS_URL: "redis://redis:6379"
  # Também pode guardar arquivos inteiros:
  config.toml: |
    [cache]
    ttl_segundos = 300
```

O Pod pode consumi-lo de duas formas:

```yaml
    spec:
      containers:
        - name: api
          image: seu-usuario/minha-api:1.1
          # Forma 1: como variáveis de ambiente
          envFrom:
            - configMapRef:
                name: api-config
          # Forma 2: como arquivos, montados em um diretório
          volumeMounts:
            - name: config
              mountPath: /etc/minha-api
      volumes:
        - name: config
          configMap:
            name: api-config
```

Com a forma 2, o container encontra `/etc/minha-api/config.toml` (e um arquivo para cada chave). Em cada ambiente (dev, homologação, produção) existe um ConfigMap `api-config` com valores diferentes, e o Deployment e a imagem são **os mesmos**.

Um detalhe que costuma surpreender: alterar um ConfigMap **não reinicia** os Pods. Variáveis de ambiente são lidas uma vez, quando o container inicia, e só mudam com um Pod novo (`kubectl rollout restart deployment/minha-api`). Arquivos montados são atualizados no disco depois de alguns segundos, mas a aplicação só percebe se reler o arquivo.

---

## 2. Secret: dados sensíveis

Um **Secret** tem a mesma estrutura de um ConfigMap, mas é destinado a senhas, tokens e certificados. A forma mais simples de criar um é pela linha de comando, sem deixar a senha em nenhum arquivo:

```bash
kubectl create secret generic postgres-cred \
  --from-literal=POSTGRES_USER=app \
  --from-literal=POSTGRES_PASSWORD='troque-esta-senha'
```

E o Pod o consome como o ConfigMap:

```yaml
          envFrom:
            - secretRef:
                name: postgres-cred
```

O Kubernetes trata Secrets de forma um pouco diferente de ConfigMaps: o kubelet os entrega aos nodes só quando um Pod daquele node precisa deles; quando montados como arquivos, ficam em memória (`tmpfs`) em vez de no disco do node; e as permissões de acesso (seção 6) costumam ser bem mais restritas.

> **Confusão comum:** "os valores do Secret estão em base64, então estão criptografados". ✅ **Mais preciso:** base64 é uma **codificação**, não uma criptografia. Qualquer pessoa reverte em um comando:
> ```bash
> kubectl get secret postgres-cred -o jsonpath='{.data.POSTGRES_PASSWORD}' | base64 -d
> # troque-esta-senha
> ```
> O base64 existe só para permitir valores binários (como certificados) em um campo de texto. A proteção real de um Secret vem de três outras coisas: **RBAC** restringindo quem pode lê-lo (seção 6), **criptografia em repouso** no etcd (que precisa ser ativada no API server; os clusters gerenciados, como o EKS, oferecem isso com uma chave do KMS), e **não versionar** os manifestos de Secret no Git. Em produção, o padrão é guardar os segredos em um cofre externo (AWS Secrets Manager, HashiCorp Vault) e sincronizá-los com o cluster por uma ferramenta como o *External Secrets Operator*. O manifesto no Git passa a dizer apenas "busque o segredo X no cofre", e nunca contém o valor.

---

## 3. Armazenamento persistente

### 3.1 O problema

O sistema de arquivos de um container some quando ele é recriado, como o capítulo [Docker: volumes e redes](../docker/03_volumes_e_redes.md) mostrou. No Docker, a solução foi um volume **no disco do host**. No Kubernetes, isso não basta: o Pod do banco pode ser recriado em **outro node**, e o disco do node antigo não vai junto.

A solução precisa de um armazenamento que **não pertença a nenhum node** e possa ser conectado a qualquer um. Na nuvem, isso existe: na AWS, é o **EBS** (*Elastic Block Store*), um disco de rede que pode ser desconectado de uma máquina e conectado a outra.

### 3.2 PersistentVolume, PersistentVolumeClaim e StorageClass

O Kubernetes separa o assunto em três objetos, para que quem desenvolve a aplicação não precise saber que tipo de disco existe por trás:

| Objeto | O que é | Quem cria |
|---|---|---|
| **StorageClass** | Um "tipo" de disco disponível: SSD da AWS, disco padrão, NFS | Quem administra o cluster (já vem pronta nos clusters gerenciados) |
| **PersistentVolumeClaim (PVC)** | Um **pedido**: "preciso de 10 GiB, de tal classe" | Você, junto com a aplicação |
| **PersistentVolume (PV)** | O disco **real** que atende o pedido | O próprio cluster, automaticamente (*provisionamento dinâmico*) |

```
 Você escreve           O cluster faz                    A nuvem entrega
┌─────────────┐   ┌──────────────────────────┐   ┌──────────────────────┐
│ PVC         │──▶│ StorageClass "gp3"       │──▶│ Disco EBS de 10 GiB  │
│ 10Gi, gp3   │   │ cria um PV para o pedido │   │ (vol-0abc123...)     │
└──────┬──────┘   └──────────────────────────┘   └──────────┬───────────┘
       │                                                    │
       │ o Pod monta o PVC         o PV é "o disco" ◀──────┘
       ▼
     Pod do Postgres  →  /var/lib/postgresql/data
```

É uma divisão parecida com a de um restaurante: você pede "um prato vegetariano" (o PVC), o cardápio define as opções disponíveis (as StorageClasses) e a cozinha prepara o prato específico (o PV).

```yaml
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dados-postgres
spec:
  accessModes:
    - ReadWriteOnce            # um node por vez pode montar para leitura e escrita
  storageClassName: gp3        # no kind, use "standard"
  resources:
    requests:
      storage: 10Gi
```

```bash
kubectl get pvc
# NAME             STATUS   VOLUME                                     CAPACITY   STORAGECLASS
# dados-postgres   Bound    pvc-3f1c2a8e-...                           10Gi       gp3
```

`Bound` significa que o pedido foi atendido por um PV. Os **modos de acesso** indicam como o disco pode ser compartilhado:

| Modo | Significado | Exemplo de armazenamento |
|---|---|---|
| `ReadWriteOnce` (RWO) | Leitura e escrita, por **um node** de cada vez | Discos de bloco: AWS EBS, GCP Persistent Disk |
| `ReadWriteMany` (RWX) | Leitura e escrita, por **vários nodes** ao mesmo tempo | Sistemas de arquivos de rede: AWS EFS, NFS |
| `ReadOnlyMany` (ROX) | Somente leitura, por vários nodes | Dados de referência compartilhados |

> **Confusão comum:** "com um PVC, o Pod do banco pode ser movido para qualquer node do cluster". ✅ **Mais preciso:** um disco EBS existe em **uma única zona de disponibilidade** da AWS (veja o [../../01-infraestrutura/aws/02_vpc_redes_na_aws.md](../../01-infraestrutura/aws/02_vpc_redes_na_aws.md)). Se o volume foi criado na zona `sa-east-1a`, o Pod que o usa só pode rodar em nodes da `sa-east-1a`, e o scheduler respeita isso automaticamente. Se não houver node livre naquela zona, o Pod fica `Pending`, mesmo com nodes ociosos em outras zonas. E se a zona inteira tiver um problema, o disco fica inacessível até ela voltar. Um PVC dá **persistência**, não **alta disponibilidade**: para sobreviver à perda de uma zona, o banco precisa de **réplicas** com cópias próprias dos dados em zonas diferentes, que é o assunto da próxima seção. A mesma lição do capítulo [Docker: volumes e redes](../docker/03_volumes_e_redes.md) continua valendo: volume não é backup.

---

## 4. StatefulSet: workloads com identidade

### 4.1 Por que o Deployment não serve para bancos

O Deployment trata todos os Pods como **intercambiáveis**: nomes aleatórios (`minha-api-7d9c8b6f5d-4xk2p`), qualquer um pode ser removido primeiro, e todos compartilham o mesmo template. Isso é perfeito para uma API e inadequado para um banco replicado, em que:

- cada réplica tem **seus próprios dados** (um disco por réplica, não um compartilhado);
- as réplicas têm **papéis** (uma é a primária, as outras são réplicas de leitura, como em [Arquitetura de Software: arquitetura_master_slave.md](https://github.com/iranhonorato/Arquitetura-de-Software/blob/main/system-design-pt2/arquitetura_master_slave.md)), e precisam se encontrar por nomes **previsíveis**;
- a ordem importa: a primária precisa subir antes das réplicas.

### 4.2 O que o StatefulSet garante

O **StatefulSet** é o objeto para aplicações com estado. Comparado ao Deployment, ele oferece:

| Garantia | Deployment | StatefulSet |
|---|---|---|
| Nome dos Pods | Aleatório (`api-7d9c-4xk2p`) | **Fixo e ordenado** (`postgres-0`, `postgres-1`) |
| Disco | Compartilhado ou nenhum | **Um PVC por réplica**, que acompanha aquele nome para sempre |
| Ordem de criação | Todos ao mesmo tempo | **Um por vez**, em ordem (0, depois 1, depois 2) |
| Nome na rede | Só o do Service, que balanceia | **Um nome DNS por Pod** (`postgres-0.postgres`) |

Se o `postgres-1` morrer, o substituto se chama **`postgres-1`** de novo e se reconecta ao **mesmo disco** (`dados-postgres-1`). Para o resto do sistema, é como se a mesma réplica tivesse reiniciado.

```yaml
# statefulset.yaml (um Postgres de uma réplica, para estudo)
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  clusterIP: None              # Service "headless": sem IP virtual, um registro DNS por Pod
  selector:
    app: postgres
  ports:
    - port: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres        # o Service headless acima
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
            # Usa uma subpasta: a raiz de um disco recém-formatado contém
            # a pasta "lost+found", e o Postgres se recusa a iniciar em pasta não vazia
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: dados
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:        # um PVC criado AUTOMATICAMENTE para cada réplica
    - metadata:
        name: dados
      spec:
        accessModes: [ReadWriteOnce]
        resources:
          requests:
            storage: 10Gi
```

Os três traços (`---`) separam dois objetos no mesmo arquivo. Com esse manifesto, o cluster cria o Pod `postgres-0` e o PVC `dados-postgres-0`. A API se conecta em `postgres:5432` (ou `postgres-0.postgres` para falar com aquela réplica específica).

> **Confusão comum:** "um StatefulSet com `replicas: 3` me dá um Postgres replicado com failover". ✅ **Mais preciso:** o StatefulSet dá **identidade** (nomes, discos e ordem estáveis), mas não sabe nada sobre Postgres. Com `replicas: 3`, você terá três bancos **independentes e vazios**, cada um com seu disco, sem nenhuma replicação entre eles. Configurar qual é a primária, fazer as réplicas copiarem dela, detectar a queda da primária e promover outra (todo o conteúdo de [Arquitetura de Software: arquitetura_master_slave.md](https://github.com/iranhonorato/Arquitetura-de-Software/blob/main/system-design-pt2/arquitetura_master_slave.md)) continua sendo trabalho seu, ou de um **Operator**: um controller especializado que codifica esse conhecimento operacional (para Postgres, por exemplo, o CloudNativePG). Muitas equipes concluem que o custo não compensa e usam um banco **gerenciado** (AWS RDS, Cloud SQL) fora do cluster, deixando o Kubernetes só para as aplicações *stateless*. É uma escolha legítima e muito comum.

---

## 5. Os demais tipos de workload

Além de Deployment e StatefulSet, há três tipos para necessidades específicas.

### 5.1 DaemonSet: um Pod em cada node

Um **DaemonSet** garante que **cada node** do cluster (ou cada node de um grupo) rode exatamente uma cópia de um Pod. Quando um node novo entra no cluster, o Pod é criado nele automaticamente.

Os usos são sempre tarefas **do node**, e não da aplicação: agentes de coleta de logs (Fluent Bit), agentes de métricas (node-exporter do Prometheus), plugins de rede e armazenamento. O próprio `kube-proxy` do capítulo *Kubernetes: fundamentos e arquitetura* roda como um DaemonSet.

### 5.2 Job: rodar até terminar

Um Deployment mantém Pods rodando **para sempre**: se o processo termina, ele é reiniciado. Um **Job** é o oposto: roda um Pod **até ele terminar com sucesso**, e então para. Se falhar, tenta de novo, até um limite.

```yaml
# job-migracao.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: migracao-banco
spec:
  backoffLimit: 3                    # até 3 novas tentativas em caso de falha
  template:
    spec:
      restartPolicy: Never           # obrigatório em Jobs: Never ou OnFailure
      containers:
        - name: migracao
          image: seu-usuario/minha-api:1.1
          command: ["alembic", "upgrade", "head"]
          envFrom:
            - secretRef:
                name: postgres-cred
```

Usos típicos: migrações de banco, importações de dados, processamentos pontuais. É o equivalente ao `docker compose exec api python manage.py migrate` do capítulo [Docker: Compose e produção](../docker/04_compose_e_producao.md), só que registrado, repetível e com novas tentativas.

### 5.3 CronJob: Jobs agendados

Um **CronJob** cria um Job em horários definidos, com a mesma sintaxe do `cron` do Linux:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-noturno
spec:
  schedule: "0 3 * * *"              # todo dia às 03:00
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: postgres:16-alpine
              command: ["/bin/sh", "-c"]
              args: ["pg_dump -h postgres -U app loja > /backup/loja.sql"]
              # (em um caso real: enviar o arquivo para o S3 e montar o Secret da senha)
```

### 5.4 Qual escolher

| A aplicação... | Use |
|---|---|
| Roda continuamente e qualquer cópia é igual a outra (API, frontend, worker) | **Deployment** |
| Roda continuamente e cada cópia tem dados e identidade próprios (banco, fila, Kafka) | **StatefulSet** |
| Precisa de uma cópia em cada máquina (agente de logs, métricas) | **DaemonSet** |
| Executa uma tarefa e termina (migração, importação) | **Job** |
| Executa uma tarefa em horários definidos (backup, relatório) | **CronJob** |

---

## 6. Segurança dentro do cluster

### 6.1 ServiceAccount: a identidade de um Pod

Pessoas acessam o cluster com credenciais guardadas no kubeconfig (capítulo *Kubernetes: fundamentos e arquitetura*). Pods também podem precisar falar com a API do cluster (um controller, uma ferramenta de CI que cria Jobs), e para isso usam uma **ServiceAccount**: uma identidade **para processos**, não para pessoas.

Todo namespace tem uma ServiceAccount chamada `default`, e todo Pod que não indica outra usa essa. O Kubernetes monta automaticamente um token dela dentro do container, em `/var/run/secrets/kubernetes.io/serviceaccount/`.

A ideia é a mesma de uma **IAM Role** da AWS (veja o [../../01-infraestrutura/aws/03_iam_identidades_e_permissoes.md](../../01-infraestrutura/aws/03_iam_identidades_e_permissoes.md)): uma identidade atribuída a uma carga de trabalho, que recebe permissões sem que ninguém precise guardar uma senha.

### 6.2 RBAC: quem pode fazer o quê

O **RBAC** (*Role-Based Access Control*) define o que cada identidade (pessoa ou ServiceAccount) pode fazer na API do cluster. São dois tipos de objeto:

- uma **Role** lista **permissões**: quais verbos (`get`, `list`, `create`, `delete`...) em quais recursos (`pods`, `secrets`, `deployments`...);
- um **RoleBinding** **associa** uma Role a uma identidade.

```yaml
# rbac.yaml: a ServiceAccount "monitor" pode LER Pods e logs no namespace "loja", e nada mais
apiVersion: v1
kind: ServiceAccount
metadata:
  name: monitor
  namespace: loja
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: leitor-de-pods
  namespace: loja
rules:
  - apiGroups: [""]                      # "" = o grupo principal (Pods, Services, Secrets...)
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: monitor-le-pods
  namespace: loja
subjects:
  - kind: ServiceAccount
    name: monitor
    namespace: loja
roleRef:
  kind: Role
  name: leitor-de-pods
  apiGroup: rbac.authorization.k8s.io
```

Role e RoleBinding valem **dentro de um namespace**. Para permissões no cluster inteiro (por exemplo, listar nodes), existem as versões **ClusterRole** e **ClusterRoleBinding**.

O próprio `kubectl` permite testar uma permissão:

```bash
kubectl auth can-i list pods -n loja --as=system:serviceaccount:loja:monitor
# yes
kubectl auth can-i list secrets -n loja --as=system:serviceaccount:loja:monitor
# no
```

O RBAC segue a lógica do **menor privilégio**, a mesma do IAM: tudo é negado, a não ser que uma regra permita. Não existem regras de negação; para tirar uma permissão, remove-se a regra que a concedia.

> **Confusão comum:** "minha aplicação não fala com a API do Kubernetes, então ServiceAccount não é assunto para mim". ✅ **Mais preciso:** por padrão, **todo** Pod recebe um token montado em `/var/run/secrets/kubernetes.io/serviceaccount/token`, mesmo que nunca o use. Se a aplicação for comprometida (por uma injeção de comando, por exemplo), esse token é a primeira coisa que um atacante procura, e o estrago depende das permissões que ele tem. Duas práticas resolvem: deixar a ServiceAccount `default` **sem permissões** (o padrão, desde que ninguém crie um RoleBinding para ela) e, em Pods que não falam com a API, desligar a montagem com `automountServiceAccountToken: false` no `spec` do Pod.

### 6.3 NetworkPolicy: quem pode falar com quem

O capítulo [Kubernetes: fundamentos e arquitetura](01_fundamentos_e_arquitetura.md) avisou: por padrão, **qualquer Pod alcança qualquer Pod** do cluster, em qualquer namespace. Uma **NetworkPolicy** funciona como um firewall entre Pods, definido por labels:

```yaml
# networkpolicy.yaml: só os Pods da API podem abrir conexões com o Postgres
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: postgres-so-da-api
spec:
  podSelector:
    matchLabels:
      app: postgres              # a política protege os Pods do Postgres
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: minha-api     # e só aceita conexões vindas da API
      ports:
        - port: 5432
```

A partir do momento em que um Pod é selecionado por **alguma** NetworkPolicy, tudo o que ela não permite explicitamente passa a ser bloqueado. É a mesma lógica dos **Security Groups** da AWS (veja o [../../01-infraestrutura/aws/02_vpc_redes_na_aws.md](../../01-infraestrutura/aws/02_vpc_redes_na_aws.md)), aplicada dentro do cluster.

Um detalhe que costuma passar despercebido: quem aplica as NetworkPolicies é o **plugin de rede** do cluster (o CNI). Plugins como Calico e Cilium as aplicam; outros, como o Flannel, **ignoram** a política sem avisar. Antes de confiar nelas, teste: tente se conectar ao Postgres a partir de um Pod sem a label `app: minha-api`.

### 6.4 securityContext: o processo dentro do container

As boas práticas de imagem do capítulo [Docker: Compose e produção](../docker/04_compose_e_producao.md) (não rodar como root, por exemplo) podem ser **exigidas** pelo manifesto, e não apenas esperadas da imagem:

```yaml
    spec:
      securityContext:
        runAsNonRoot: true                  # recusa iniciar se a imagem rodar como root
        runAsUser: 1000
      containers:
        - name: api
          image: seu-usuario/minha-api:1.1
          securityContext:
            allowPrivilegeEscalation: false # o processo não pode ganhar mais privilégios
            readOnlyRootFilesystem: true    # o sistema de arquivos do container é só leitura
            capabilities:
              drop: ["ALL"]                 # remove todos os poderes especiais do kernel
```

Com `readOnlyRootFilesystem: true`, um atacante que comprometa a aplicação não consegue gravar arquivos no container. Se a aplicação precisar de uma pasta temporária, monte um volume `emptyDir` só para ela.

---

## Próximo passo

Os três primeiros capítulos de Kubernetes apresentaram os objetos do Kubernetes um a um. Falta juntar tudo: pegar o sistema do capítulo [Docker: Compose e produção](../docker/04_compose_e_producao.md) (API, Postgres e Redis), descrevê-lo inteiro em manifestos, subir em um cluster, e aprender a investigar quando um Pod não sobe.

O [04_do_compose_ao_cluster.md](04_do_compose_ao_cluster.md) faz essa migração passo a passo, mostra como diagnosticar os erros mais comuns e apresenta as ferramentas e os serviços gerenciados usados em produção.
