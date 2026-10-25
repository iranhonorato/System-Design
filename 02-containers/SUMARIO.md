# Sumário: Containers

Esta pasta trata de como empacotar uma aplicação com tudo de que ela precisa, e de como rodar muitas dessas aplicações em muitas máquinas. Ela tem duas partes, uma construída sobre a outra:

- **`docker/`**: um container em uma máquina. Como o isolamento funciona, como construir imagens, como dar persistência e rede aos containers e como descrever um sistema inteiro com o Docker Compose.
- **`kubernetes/`**: muitos containers em muitas máquinas. O que é um cluster, como o Kubernetes mantém as aplicações no estado declarado e como migrar o sistema do Compose para ele.

---

## Trilha de Leitura

```
pré-requisito: ../01-infraestrutura/fundamentos/ (①, ②)
         │
         ▼
DOCKER: UM CONTAINER, UMA MÁQUINA
  ① docker/01_arquitetura_e_isolamento.md
  ② docker/02_imagens_e_dockerfile.md
  ③ docker/03_volumes_e_redes.md
  ④ docker/04_compose_e_producao.md
         │
         ▼
KUBERNETES: MUITOS CONTAINERS, MUITAS MÁQUINAS
  ⑤ kubernetes/01_fundamentos_e_arquitetura.md
  ⑥ kubernetes/02_workloads_rede_e_escala.md
  ⑦ kubernetes/03_configuracao_estado_e_seguranca.md
  ⑧ kubernetes/04_do_compose_ao_cluster.md
         │
         ▼
continua em: ../01-infraestrutura/aws/ (EC2, VPC, IAM)
```

---

## ① [docker/01_arquitetura_e_isolamento.md](docker/01_arquitetura_e_isolamento.md)

**Pré-requisito:** [../01-infraestrutura/fundamentos/02_virtualizacao_containers_e_iac.md](../01-infraestrutura/fundamentos/02_virtualizacao_containers_e_iac.md)

Instalação, a arquitetura cliente → daemon → registry e o caminho de um `docker run`. Mostra cada **namespace** com um comando real (`ps`, `hostname`, `ip addr` dentro de um container) e os **cgroups** limitando memória até o *OOM kill* (código de saída 137). Termina com o primeiro servidor (Nginx) e o mapeamento de portas.

> **Problema que deixa em aberto:** como empacotar a minha própria aplicação?

---

## ② [docker/02_imagens_e_dockerfile.md](docker/02_imagens_e_dockerfile.md)

**Pré-requisito:** ①

Imagem vs container, camadas, *union file system* e camada de escrita. Constrói uma API FastAPI com um **Dockerfile** comentado instrução por instrução, demonstra o **cache de build** com a ordem certa e a errada, e cobre `.dockerignore`, `CMD` vs `ENTRYPOINT` (e por que a forma *exec* importa para o `docker stop`), tags e publicação no Docker Hub.

> **Problema que deixa em aberto:** onde ficam os dados, e como um container fala com outro?

---

## ③ [docker/03_volumes_e_redes.md](docker/03_volumes_e_redes.md)

**Pré-requisito:** ②

Demonstra que a camada de escrita some com o container e resolve isso com **volumes** (um Postgres que sobrevive à remoção) e **bind mounts** (código editável em desenvolvimento). Explica por que `localhost` não funciona entre containers e monta uma **rede** em que a API encontra o banco pelo nome.

> **Problema que deixa em aberto:** subir um sistema com vários containers exige muitos comandos manuais.

---

## ④ [docker/04_compose_e_producao.md](docker/04_compose_e_producao.md)

**Pré-requisito:** ③

Traduz o sistema do capítulo anterior para **Docker Compose** e evolui para API + Postgres + Redis com `.env`, healthchecks e políticas de reinício. Cobre **multi-stage builds** em Python e Node, boas práticas de imagens de produção, quando partir para o Kubernetes e o padrão sidecar/service mesh. Termina com um **apêndice de referência de comandos**.

> **Problema que deixa em aberto:** o Compose orquestra uma máquina só. E quando o sistema precisa de várias?

---

## ⑤ [kubernetes/01_fundamentos_e_arquitetura.md](kubernetes/01_fundamentos_e_arquitetura.md)

**Pré-requisito:** ④

O que o Kubernetes é, para que serve e para que não serve. Apresenta a ideia central de **estado desejado e loop de reconciliação**, a anatomia de um **cluster** (control plane com API server, etcd, scheduler e controller-manager; **nodes** com kubelet, kube-proxy e containerd), o **Pod** como menor unidade e o caminho completo de um `kubectl apply`. Cobre manifestos, labels, selectors e namespaces, e monta um cluster local de três nodes com o kind.

> **Problema que deixa em aberto:** um Pod sozinho não se recria, não escala e muda de IP.

---

## ⑥ [kubernetes/02_workloads_rede_e_escala.md](kubernetes/02_workloads_rede_e_escala.md)

**Pré-requisito:** ⑤

**ReplicaSet** e **Deployment** (réplicas, rolling update, rollback), **probes** (liveness, readiness, startup), **requests e limits** (e o `OOMKilled`), **Service** (ClusterIP, NodePort, LoadBalancer, DNS interno e por que o IP é virtual), **Ingress** e Gateway API, e **HorizontalPodAutoscaler**, com a diferença entre escalar Pods e escalar nodes.

> **Problema que deixa em aberto:** configuração, segredos e bancos de dados.

---

## ⑦ [kubernetes/03_configuracao_estado_e_seguranca.md](kubernetes/03_configuracao_estado_e_seguranca.md)

**Pré-requisito:** ⑥

**ConfigMap** e **Secret** (e por que base64 não é criptografia), armazenamento persistente com **PV, PVC e StorageClass**, **StatefulSet** para bancos (e por que ele não replica nada sozinho), **DaemonSet**, **Job** e **CronJob**. Na segurança: **ServiceAccount**, **RBAC**, **NetworkPolicy** e `securityContext`.

> **Problema que deixa em aberto:** juntar tudo em um sistema real e saber o que fazer quando ele não sobe.

---

## ⑧ [kubernetes/04_do_compose_ao_cluster.md](kubernetes/04_do_compose_ao_cluster.md)

**Pré-requisito:** ⑦

Migra o sistema do Compose (API, Postgres, Redis) para manifestos, arquivo por arquivo, e sobe no kind. Traz um **roteiro de diagnóstico** por status (`Pending`, `ImagePullBackOff`, `CrashLoopBackOff`, *não Ready*, *sem resposta*), o ecossistema (Kustomize, Helm, GitOps), os clusters gerenciados (**EKS**, GKE, AKS) e quando **não** usar Kubernetes.

> **Continua em:** [../01-infraestrutura/aws/01_ec2_servidores_na_nuvem.md](../01-infraestrutura/aws/01_ec2_servidores_na_nuvem.md): a infraestrutura da AWS em que servidores e clusters se apoiam.

---

## Referência Rápida por Assunto

| Quero entender... | Leia |
|---|---|
| Namespaces, cgroups e o que é um container por dentro | ① |
| Por que meu container foi morto (código 137) | ① |
| Mapeamento de portas e por que `EXPOSE` não publica nada | ① |
| Como escrever um Dockerfile e aproveitar o cache | ② |
| `CMD` vs `ENTRYPOINT`, tags e Docker Hub | ② |
| Persistir dados de um banco em container | ③ |
| Fazer um container falar com outro | ③ |
| Docker Compose, healthchecks e `.env` | ④ |
| Imagens de produção pequenas e seguras (multi-stage) | ④ |
| Sidecar e service mesh | ④ |
| Lista de comandos do Docker | ④ (apêndice) |
| O que é Kubernetes e quando vale a pena | ⑤, ⑧ |
| Cluster, control plane, node, kubelet, etcd | ⑤ |
| Pods e o caminho de um `kubectl apply` | ⑤ |
| Deployment, rolling update e rollback | ⑥ |
| Liveness, readiness e startup probes | ⑥ |
| Requests, limits e `OOMKilled` | ⑥ |
| Service, Ingress e DNS do cluster | ⑥ |
| Auto-scaling de Pods e de nodes | ⑥ |
| ConfigMap e Secret | ⑦ |
| Volumes persistentes (PV, PVC) e StatefulSet | ⑦ |
| DaemonSet, Job e CronJob | ⑦ |
| RBAC, ServiceAccount e NetworkPolicy | ⑦ |
| Migrar do Compose para o Kubernetes | ⑧ |
| Por que meu Pod está em `CrashLoopBackOff` / `Pending` | ⑧ |
| Helm, Kustomize, GitOps, EKS | ⑧ |
