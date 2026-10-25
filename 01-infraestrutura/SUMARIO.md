# Sumário: Infraestrutura

Esta pasta trata do que sustenta qualquer software antes de uma linha de código rodar: as camadas física, lógica e de nuvem, a evolução que levou aos containers e, na prática, a infraestrutura da AWS em que servidores e clusters se apoiam. Ela tem duas partes:

- **`fundamentos/`**: os conceitos, sem depender de nenhum provedor. É o ponto de partida do repositório inteiro.
- **`aws/`**: a aplicação desses conceitos na AWS: servidores (EC2), rede (VPC) e permissões (IAM).

---

## Trilha de Leitura

```
FUNDAMENTOS                         (o começo do repositório)
  ① fundamentos/01_o_que_e_infraestrutura.md
  ② fundamentos/02_virtualizacao_containers_e_iac.md
         │
         ▼
   ... parte de containers: ../02-containers/ (Docker e Kubernetes) ...
         │
         ▼
AWS
  ③ aws/01_ec2_servidores_na_nuvem.md
  ④ aws/02_vpc_redes_na_aws.md
  ⑤ aws/03_iam_identidades_e_permissoes.md
         │
         ▼
   ... entrega contínua: ../03-entrega-continua/ ...
```

Na trilha completa do repositório (veja o [README](../README.md)), a parte de AWS vem **depois** de [../02-containers/](../02-containers/SUMARIO.md): a EC2 roda o Compose dos capítulos de Docker, e as últimas seções de VPC e IAM tratam do EKS, o Kubernetes gerenciado da AWS. Quem não for usar Kubernetes pode ler a parte de AWS logo depois de Docker, pulando apenas essas seções finais.

---

## ① [fundamentos/01_o_que_e_infraestrutura.md](fundamentos/01_o_que_e_infraestrutura.md)

**Pré-requisito:** nenhum.

Segue uma requisição desde o DNS até o servidor para mostrar o papel de cada camada da infraestrutura: física (servidores, RAID, rede, facilities), lógica (sistema operacional, hypervisors, DNS, firewall, load balancer) e cloud. Compara IaaS, PaaS e SaaS pelo que cada um deixa sob sua responsabilidade e explica por que as características operacionais de um sistema (disponibilidade, escalabilidade, elasticidade) dependem tanto da infraestrutura quanto do código.

> **Problema que deixa em aberto:** como organizar aplicações dentro de um servidor?

---

## ② [fundamentos/02_virtualizacao_containers_e_iac.md](fundamentos/02_virtualizacao_containers_e_iac.md)

**Pré-requisito:** ①

Conta as três eras (bare metal → máquinas virtuais → containers), cada uma como resposta ao principal problema da anterior: dependency hell e desperdício, depois o peso de um sistema operacional por VM. Apresenta namespaces e cgroups como a base dos containers e fecha com **Infraestrutura como Código**, comparando Dockerfile, Terraform e Ansible e a ideia de estado desejado e idempotência.

> **Problema que deixa em aberto:** como exatamente o isolamento de um container funciona? (Continua em [../02-containers/docker/01_arquitetura_e_isolamento.md](../02-containers/docker/01_arquitetura_e_isolamento.md).)

---

## ③ [aws/01_ec2_servidores_na_nuvem.md](aws/01_ec2_servidores_na_nuvem.md)

**Pré-requisito:** ①, e os capítulos de Docker até o de Compose.

As cinco decisões ao criar uma instância (região, AMI, tipo, par de chaves, Security Group), os tipos de instância e o modelo *burstable*, o ciclo de vida (stop, start, terminate) e o que cada estado cobra. Acesso por **SSH** passo a passo, IP público, privado e Elastic IP, preparação do servidor com Docker, o sistema do capítulo de Compose rodando na nuvem, *user data*, discos **EBS** e snapshots, modelos de preço e como evitar surpresas na fatura.

> **Problema que deixa em aberto:** a instância está solta em uma rede pública que ainda não foi explicada.

---

## ④ [aws/02_vpc_redes_na_aws.md](aws/02_vpc_redes_na_aws.md)

**Pré-requisito:** ③ e [../02-containers/docker/03_volumes_e_redes.md](../02-containers/docker/03_volumes_e_redes.md). A seção final assume a parte de Kubernetes.

Fundamentos de IP e **CIDR**, o que é uma **VPC**, regiões e **zonas de disponibilidade**, **sub-redes** públicas e privadas (e o que de fato as diferencia), **tabelas de rotas**, **Internet Gateway**, **NAT Gateway**, **Security Groups** vs **Network ACLs**. Monta uma VPC de três camadas em duas zonas e segue os pacotes por ela. Cobre acesso sem porta 22 (bastion vs Session Manager), VPC Endpoints, peering, Flow Logs, a VPC em Terraform e o planejamento de IPs para o EKS.

> **Problema que deixa em aberto:** quem pode chamar a API da AWS?

---

## ⑤ [aws/03_iam_identidades_e_permissoes.md](aws/03_iam_identidades_e_permissoes.md)

**Pré-requisito:** ④. A seção final assume a parte de Kubernetes.

Autenticação e autorização na AWS, o usuário **root**, **users, groups e roles** (e o Identity Center para pessoas), a anatomia de uma **política** JSON e dos **ARNs**, políticas de identidade e de recurso, permissions boundaries e SCPs, a **lógica de avaliação** (por que um `Deny` sempre vence), credenciais temporárias com **STS**, **roles para EC2** e IMDSv2, **GitHub Actions com OIDC** sem chaves guardadas e as ferramentas de menor privilégio. Fecha juntando **EKS + VPC + IAM**: access entries, roles de cluster e de node, e **Pod Identity**.

> **Continua em:** [../03-entrega-continua/SUMARIO.md](../03-entrega-continua/SUMARIO.md), cujos pipelines usam OIDC, ECR e Session Manager.

---

## Referência Rápida por Assunto

| Quero entender... | Leia |
|---|---|
| O que é infraestrutura e quais são suas camadas | ① |
| A diferença entre IaaS, PaaS e SaaS | ① |
| Por que containers substituíram VMs (e quando não substituem) | ② |
| Terraform, Ansible e Infraestrutura como Código | ② |
| Criar e acessar um servidor na AWS | ③ |
| Tipos de instância e créditos de CPU | ③ |
| Por que o IP público da instância mudou | ③ |
| Discos EBS, snapshots e custos da EC2 | ③ |
| CIDR e endereços IP privados | ④ |
| VPC, sub-redes públicas e privadas, rotas | ④ |
| Internet Gateway e NAT Gateway | ④ |
| Security Group vs Network ACL | ④ |
| Acessar instâncias privadas sem SSH aberto | ④, ⑤ |
| Users, groups, roles e políticas do IAM | ⑤ |
| Por que meu acesso foi negado (lógica de avaliação) | ⑤ |
| Credenciais temporárias, STS e roles para EC2 | ⑤ |
| GitHub Actions na AWS sem access keys (OIDC) | ⑤ |
| Permissões de Pods na AWS (Pod Identity) | ⑤ |
