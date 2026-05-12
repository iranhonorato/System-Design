# Infraestrutura de Software

## O que é Infraestrutura?

Infraestrutura é o conjunto de recursos computacionais — processamento, memória, armazenamento e rede — que sustenta o ciclo de vida de uma aplicação. É o alicerce que permite que um software exista, seja acessado e funcione de forma confiável.

Uma boa analogia: se o seu código é um restaurante, a infraestrutura é o prédio, a eletricidade, o encanamento, a cozinha industrial e as estradas que permitem que os clientes cheguem. Sem esses fundamentos, o melhor chef do mundo não consegue servir ninguém.

## As três camadas da infraestrutura

### Camada 1: Infraestrutura Física (Hardware)

É o "ferro" em si — tudo que existe materialmente em um Data Center:

**Servidores:** Computadores de alta performance sem monitor ou teclado, projetados para rodar 24 horas por dia, 365 dias por ano. Um servidor moderno pode ter 64+ núcleos de CPU, centenas de GB de RAM e SSDs NVMe extremamente rápidos.

**Storage (Armazenamento):** Sistemas de disco especializados que oferecem alta redundância. É comum usar RAID — uma técnica que distribui dados entre vários discos de forma que a falha de um disco não cause perda de dados.

**Rede:** Switches e roteadores de alta capacidade conectam os servidores entre si com velocidades de 10, 25 ou até 100 Gigabits por segundo. Cabos de fibra óptica substituem o cobre em distâncias maiores por serem mais rápidos e menos suscetíveis a interferências.

**Facilities:** Itens frequentemente ignorados, mas críticos:
- Ar-condicionado industrial (os servidores geram muito calor)
- Geradores de energia (UPS) para continuar operando durante quedas de luz
- Sistemas de supressão de incêndio (sem sprinklers — a água destruiria os equipamentos)
- Segurança física com câmeras, catracas e controle de acesso biométrico

### Camada 2: Infraestrutura Lógica (Software de Base)

É a camada de software que gerencia o hardware e cria a fundação para as aplicações:

**Sistemas Operacionais:** Windows Server e distribuições Linux (Ubuntu Server, Debian, Rocky Linux, AlmaLinux) são os mais comuns. Em ambientes de produção, Linux domina por ser gratuito, estável e extremamente configurável. O CentOS, amplamente usado no passado, chegou ao fim de vida (EOL) em dezembro de 2021 — suas substituições diretas são **Rocky Linux** e **AlmaLinux**, ambas compatíveis com RHEL.

**Virtualizadores (Hypervisors):** Software que cria máquinas virtuais — VMware ESXi, Microsoft Hyper-V, KVM (Linux). Veremos mais sobre isso na seção histórica.

**Gerenciamento de Rede:** DNS (resolve nomes como `www.google.com` para IPs), Firewalls (controlam o tráfego de entrada e saída), Load Balancers (distribuem requisições entre múltiplos servidores).

### Camada 3: Infraestrutura como Serviço (Cloud / IaaS)

A forma moderna de provisionar infraestrutura. Em vez de comprar servidores, você aluga capacidade computacional de provedores de nuvem:

- **AWS (Amazon Web Services):** EC2 (servidores virtuais), S3 (armazenamento), RDS (bancos gerenciados)
- **Google Cloud Platform (GCP):** Compute Engine, Cloud Storage, Cloud SQL
- **Microsoft Azure:** Virtual Machines, Blob Storage, Azure SQL

**Vantagem fundamental:** em vez de esperar semanas para comprar e instalar um servidor, você provisiona capacidade computacional em minutos — e paga apenas pelo que usa.

---

## Evolução histórica: de Bare Metal ao Docker

Entender a evolução tecnológica é essencial para compreender *por que* o Docker existe e que problema ele resolve de fato.

### Era 1: Bare Metal (Hardware Direto)

**Período:** Décadas de 1980 a meados de 1990.

"Bare Metal" significa rodar o sistema operacional diretamente no hardware físico, sem nenhuma camada de abstração entre os dois.

```
┌─────────────────────────────────────┐
│         Aplicação                   │
├─────────────────────────────────────┤
│         Sistema Operacional         │
├─────────────────────────────────────┤
│         Hardware Físico             │
└─────────────────────────────────────┘
```

**Como funcionava:** uma empresa comprava um servidor Dell ou HP, instalava Windows Server ou Red Hat, e dentro dele colocava suas aplicações.

**Problemas críticos:**

**1. Conflito de dependências ("Dependency Hell"):**
O App A precisava do Java 8 e o App B precisava do Java 11. As duas versões não coexistiam facilmente no mesmo SO. Solução forçada: um servidor por aplicação.

**2. Desperdício absurdo de recursos:**
Por segurança, rodava-se uma aplicação por servidor. Servidores potentes de 32 núcleos de CPU operavam com 5-10% de utilização. O restante ficava ocioso.

**3. Escalabilidade lenta e cara:**
Se o site crescia e precisava de mais capacidade, você comprava um novo servidor físico, esperava chegar (semanas), montava no rack, configurava e só então tinha capacidade adicional.

**4. Ambientes inconsistentes:**
Configurar dois servidores idênticos manualmente resultava inevitavelmente em diferenças sutis. O clássico "funciona na minha máquina" tem raízes aqui.

---

### Era 2: Máquinas Virtuais (Virtualização)

**Período:** Final dos anos 1990 até meados de 2010.

A VMware popularizou a virtualização: uma camada de software (o **Hypervisor**) que simula hardware, permitindo rodar múltiplos sistemas operacionais no mesmo servidor físico.

```
┌──────────┬──────────┬──────────┐
│  App A   │  App B   │  App C   │
├──────────┼──────────┼──────────┤
│  SO      │  SO      │  SO      │  ← cada VM tem seu SO
│ (Linux)  │(Windows) │ (Linux)  │
├──────────┴──────────┴──────────┤
│         Hypervisor              │  ← gerencia as VMs
├─────────────────────────────────┤
│         Hardware Físico         │
└─────────────────────────────────┘
```

**Como melhorou:**
- Um servidor físico rodando 10 VMs em vez de 10 servidores físicos
- Isolamento real: um crash em uma VM não afeta as outras
- Portabilidade: VMs podiam ser copiadas e movidas entre servidores

**Tipos de Hypervisor:**
- **Tipo 1 (bare-metal):** roda diretamente no hardware. Ex: VMware ESXi, Microsoft Hyper-V. Mais eficiente.
- **Tipo 2 (hosted):** roda como aplicação em cima de um SO. Ex: VirtualBox, VMware Workstation. Mais fácil de usar em desktops.

**Problemas persistentes:**

**1. Overhead do Guest OS:**
Cada VM carrega um sistema operacional completo — kernel, drivers, serviços de sistema. Para rodar uma API de 50MB, você precisava de um Ubuntu de 2GB por baixo. Multiplicado por 10 VMs = 20GB de RAM só para sistemas operacionais.

**2. Lentidão no boot:**
Iniciar uma VM é como ligar um computador. Pode levar 1-5 minutos para o kernel carregar, inicializar serviços e chegar ao estado operacional.

**3. Portabilidade limitada:**
Uma imagem de VM típica pesa vários gigabytes. Transferir isso entre ambientes é lento e custoso.

---

### Era 3: Containers (Docker e a Conteinerização)

**Período:** 2013 até hoje.

O Docker (lançado em 2013 pela dotCloud) trouxe uma percepção fundamental: **não precisamos virtualizar o hardware — precisamos isolar os processos**.

```
┌──────────┬──────────┬──────────┐
│  App A   │  App B   │  App C   │
├──────────┼──────────┼──────────┤
│ Libs A   │ Libs B   │ Libs C   │  ← cada container tem suas próprias libs
├──────────┴──────────┴──────────┤
│       Docker Engine             │  ← gerencia o isolamento
├─────────────────────────────────┤
│    Sistema Operacional Host     │  ← ÚNICO kernel, compartilhado
├─────────────────────────────────┤
│         Hardware Físico         │
└─────────────────────────────────┘
```

**A sacada técnica:** containers compartilham o **kernel** do sistema operacional host, mas têm seu próprio espaço de usuário (User Space) isolado. É como apartamentos em um prédio: compartilham a estrutura (kernel), mas cada um tem seus próprios cômodos (libs, arquivos, processos).

**Por que isso é revolucionário:**

| Característica | VM | Container |
|---|---|---|
| **Tamanho típico** | 1-10 GB | 10-500 MB |
| **Tempo de boot** | 1-5 minutos | Milissegundos |
| **Isolamento** | Completo (hardware virtual) | Processo (namespace) |
| **Overhead de recursos** | Alto (SO por VM) | Mínimo |
| **Portabilidade** | Limitada (arquivos grandes) | Alta (imagens leves) |

**A tecnologia por baixo:**

**Namespaces:** o kernel do Linux permite criar "visões isoladas" do sistema para cada container:
- **PID namespace:** o container acha que o seu processo é o PID 1 (o primeiro processo do sistema)
- **Net namespace:** cada container tem sua própria interface de rede e tabela de roteamento
- **Mount namespace:** cada container enxerga seu próprio sistema de arquivos
- **User namespace:** usuários dentro do container são mapeados para usuários diferentes no host

**Control Groups (cgroups):** enquanto namespaces isolam visibilidade, cgroups limitam consumo de recursos. Você pode dizer "este container pode usar no máximo 512MB de RAM e 0.5 vCPU". Se ele tentar usar mais, o kernel corta.

---

## Infraestrutura como Código (IaC)

Um dos conceitos mais importantes da infraestrutura moderna: **descrever a infraestrutura em arquivos de texto versionáveis**, em vez de configurar servidores manualmente.

**Antes do IaC:**
- Administrador acessava o servidor via SSH
- Instalava dependências manualmente
- Fazia configurações "na mão"
- Ninguém sabia exatamente o que estava instalado
- Reproduzir o ambiente era quase impossível

**Com IaC:**
```
# Exemplo conceitual de Dockerfile (IaC para containers)
FROM python:3.11
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "app.py"]
```

Esse arquivo é:
- **Versionável no Git:** você sabe exatamente como o ambiente era em cada ponto do tempo
- **Reproduzível:** qualquer pessoa com esse arquivo cria exatamente o mesmo ambiente
- **Auditável:** mudanças no ambiente ficam visíveis no histórico de commits
- **Automatizável:** pipelines de CI/CD executam esse arquivo automaticamente

**Outras ferramentas de IaC populares:**
- **Terraform:** provisiona infraestrutura de nuvem (instâncias EC2, bancos RDS, redes VPC)
- **Ansible:** configura servidores existentes (instala pacotes, configura serviços)
- **Docker Compose:** orquestra múltiplos containers localmente
- **Kubernetes manifests:** descreve como aplicações devem rodar em clusters

---

## Orquestração de Containers: Kubernetes

Quando o número de containers cresce (dezenas ou centenas de instâncias), o Docker sozinho não é suficiente. O **Kubernetes (K8s)** é o padrão da indústria para orquestrar containers em produção.

```
┌────────────────────────────────────────────────────────┐
│                  CLUSTER KUBERNETES                     │
│                                                        │
│  Control Plane                                         │
│  ┌──────────────────────────────────────────────────┐  │
│  │  API Server  │  Scheduler  │  etcd (estado)      │  │
│  └──────────────────────────────────────────────────┘  │
│                                                        │
│  Worker Nodes                                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐   │
│  │   Node 1    │  │   Node 2    │  │   Node 3    │   │
│  │  [Pod][Pod] │  │  [Pod][Pod] │  │  [Pod][Pod] │   │
│  └─────────────┘  └─────────────┘  └─────────────┘   │
└────────────────────────────────────────────────────────┘
```

O Kubernetes cuida automaticamente de:
- **Self-healing:** reinicia containers que falham
- **Auto-scaling:** aumenta/diminui réplicas conforme a carga
- **Rolling updates:** atualiza containers sem downtime
- **Service discovery:** containers se encontram pelo nome, não pelo IP
- **Load balancing:** distribui tráfego entre réplicas

Em nuvem, os provedores oferecem Kubernetes gerenciado: **AWS EKS**, **Google GKE**, **Azure AKS** — você não gerencia o control plane.

---

## Por que tudo isso importa para desenvolvedores?

A fronteira entre desenvolvimento e infraestrutura se dissolveu com o movimento **DevOps**. Hoje, é esperado que desenvolvedores entendam:

- Como suas aplicações são empacotadas (Docker)
- Como são deployadas (CI/CD pipelines)
- Como escalam (auto-scaling, load balancers)
- Como falham e se recuperam (health checks, restart policies)
- Como são monitoradas (logs, métricas, alertas)

Entender infraestrutura não é papel só do time de operações — é conhecimento fundamental para construir software que funciona bem em produção.
