# Infraestrutura de Software

**Pré-requisito:** nenhum. Este é o ponto de partida da trilha.

> Baseado no Capítulo 1 (*Scale from Zero to Millions of Users*) de **System Design Interview — An Insider's Guide** (Alex Xu), que parte justamente do caminho de uma requisição em um servidor único, e no Capítulo 4 (*Architecture Characteristics Defined*) de **Fundamentals of Software Architecture** (Mark Richards & Neal Ford), que define as características operacionais como o ponto de encontro entre arquitetura e operações.

---

## Por que este arquivo existe

Quando se aprende a programar, o código parece ser o sistema inteiro: você escreve, roda, funciona. Mas o código só "roda" porque existe uma pilha enorme de coisas por baixo dele: uma máquina física, um sistema operacional, uma rede, um nome de domínio, energia, refrigeração. Enquanto tudo isso funciona, ninguém repara. Quando algo falha, a aplicação cai junto, por melhor que seja o código.

Este arquivo apresenta essa pilha. A pergunta central é: *o que precisa existir, e funcionar, para que uma linha de código chegue até um usuário?* Os próximos capítulos (virtualização, containers, Docker) só fazem sentido depois que essa pilha está clara, porque cada um deles é uma forma diferente de organizar as mesmas camadas.

---

## 1. O que é infraestrutura

Infraestrutura é o conjunto de recursos computacionais (processamento, memória, armazenamento e rede) que sustenta o ciclo de vida de uma aplicação. É o alicerce que permite que um software exista, seja acessado e funcione de forma confiável.

Uma analogia ajuda: se o seu código é um restaurante, a infraestrutura é o prédio, a eletricidade, o encanamento, a cozinha industrial e as estradas que levam os clientes até a porta. O melhor chef do mundo não serve ninguém se faltar gás ou se a rua estiver interditada.

O ponto importante da analogia é que **o cliente nunca vê a infraestrutura**: ele vê o prato. Mas a qualidade da experiência (o prato chegou rápido? o restaurante estava aberto?) depende tanto da cozinha quanto do chef. Em software é igual: tempo de resposta e disponibilidade são, em grande parte, propriedades da infraestrutura.

---

## 2. Um exemplo concreto: o caminho de uma requisição

Antes de classificar as camadas, vale seguir uma requisição real do início ao fim. Alex Xu começa seu livro exatamente por aqui, com o sistema mais simples possível: **tudo rodando em um único servidor**.

```
                  ① "qual o IP de api.minhaloja.com?"
  ┌──────────┐  ──────────────────────────────────────▶  ┌─────────┐
  │          │                                            │   DNS   │
  │ Usuário  │  ◀──────────────────────────────────────   └─────────┘
  │(navegador│        ② "é 15.125.23.214"
  │ ou app)  │
  │          │  ③ GET /produtos  (HTTP para 15.125.23.214)
  │          │  ──────────────────────────────────────▶  ┌──────────────────┐
  │          │                                            │  Servidor web    │
  │          │  ◀──────────────────────────────────────   │  (app + banco)   │
  └──────────┘        ④ resposta HTML ou JSON             └──────────────────┘
```

Passo a passo:

1. O usuário digita `api.minhaloja.com`. O navegador não sabe o que é isso: computadores se encontram por **endereço IP**, não por nome.
2. Ele pergunta a um servidor **DNS** (*Domain Name System*), que funciona como uma lista telefônica da internet e devolve o IP correspondente.
3. Com o IP em mãos, o navegador abre uma conexão e envia uma requisição HTTP.
4. O servidor executa o código da aplicação, talvez consulte um banco de dados, e devolve a resposta.

Esse fluxo de quatro passos parece trivial, mas cada passo depende de uma camada diferente da infraestrutura:

| Passo | O que precisa existir |
|---|---|
| Resolver o nome | Um serviço de DNS configurado (infraestrutura **lógica**) |
| Chegar ao servidor | Cabos, roteadores, link de internet (infraestrutura **física**) e regras de firewall (**lógica**) |
| Executar o código | Uma máquina com CPU, memória e disco (**física**) e um sistema operacional (**lógica**) |
| Fazer tudo isso sem comprar hardware | Um provedor de nuvem (**cloud**) |

As próximas seções detalham cada uma dessas camadas.

---

## 3. As três camadas da infraestrutura

### 3.1 Camada física: o hardware

É o "ferro" em si, tudo o que existe materialmente em um **data center**:

- **Servidores:** computadores de alta performance sem monitor nem teclado, projetados para rodar 24 horas por dia, 365 dias por ano. Um servidor moderno pode ter mais de 64 núcleos de CPU, centenas de GB de RAM e SSDs NVMe muito rápidos.
- **Armazenamento (storage):** sistemas de disco especializados em redundância. É comum usar **RAID**, uma técnica que distribui os dados entre vários discos de forma que a falha de um deles não cause perda de dados.
- **Rede:** switches e roteadores conectam os servidores entre si a 10, 25 ou até 100 Gbit/s. Em distâncias maiores, a fibra óptica substitui o cobre por ser mais rápida e menos suscetível a interferência.
- **Facilities:** itens que costumam ser esquecidos, mas são críticos:
  - ar-condicionado industrial, porque servidores geram muito calor;
  - nobreaks (UPS) e geradores, para continuar operando durante quedas de energia;
  - supressão de incêndio a gás, sem sprinklers, porque a água destruiria os equipamentos;
  - segurança física com câmeras, catracas e controle biométrico.

### 3.2 Camada lógica: o software de base

É o software que gerencia o hardware e cria a fundação sobre a qual as aplicações rodam:

- **Sistemas operacionais:** Windows Server e distribuições Linux (Ubuntu Server, Debian, Rocky Linux, AlmaLinux). Em produção, o Linux domina por ser gratuito, estável e muito configurável. O CentOS, muito usado no passado, chegou ao fim de vida (EOL) em dezembro de 2021; seus substitutos diretos são o **Rocky Linux** e o **AlmaLinux**, ambos compatíveis com o RHEL.
- **Virtualizadores (hypervisors):** software que cria máquinas virtuais, como VMware ESXi, Microsoft Hyper-V e KVM. O [próximo capítulo](02_virtualizacao_containers_e_iac.md) conta por que eles surgiram.
- **Serviços de rede:**
  - **DNS**, que traduz nomes como `www.google.com` em IPs (o passo ① do exemplo acima);
  - **firewalls**, que controlam o tráfego de entrada e saída;
  - **load balancers**, que distribuem requisições entre vários servidores (detalhados em [Arquitetura de Software: load_balancer.md](https://github.com/iranhonorato/Arquitetura-de-Software/blob/main/system-design-pt2/load_balancer.md)).

### 3.3 Camada cloud: infraestrutura como serviço

É a forma moderna de obter as duas camadas anteriores. Em vez de comprar servidores, você **aluga** capacidade computacional de um provedor:

| Provedor | Servidores virtuais | Armazenamento de arquivos | Banco gerenciado |
|---|---|---|---|
| **AWS** (Amazon Web Services) | EC2 | S3 | RDS |
| **Google Cloud Platform** | Compute Engine | Cloud Storage | Cloud SQL |
| **Microsoft Azure** | Virtual Machines | Blob Storage | Azure SQL |

A vantagem fundamental é o **tempo**: em vez de esperar semanas para comprar, receber e instalar um servidor, você cria um em minutos, e paga só pelo tempo em que ele fica ligado.

> **Confusão comum:** "na nuvem não existe servidor físico" (ou "serverless significa sem servidor"). ✅ **Mais preciso:** a nuvem é o data center **de outra empresa**. Cada instância EC2 roda em um servidor físico real, com RAID, refrigeração e geradores, dentro de um prédio da AWS. O que muda é **quem cuida** da camada física: você deixa de comprar e manter o hardware e passa a alugá-lo. O mesmo vale para *serverless*: os servidores continuam lá, você só não os enxerga nem os administra. Por isso falhas de hardware continuam acontecendo na nuvem, e é por isso que os provedores dividem suas regiões em **zonas de disponibilidade**, ou seja, data centers separados para você distribuir sua aplicação.

---

## 4. Quanto você gerencia: IaaS, PaaS e SaaS

"Nuvem" não é uma coisa só. Os serviços se diferenciam por **quantas camadas o provedor assume por você**. Quanto mais o provedor gerencia, menos controle você tem, mas também menos trabalho.

```
                 On-premise     IaaS          PaaS          SaaS
                 (seu data      (ex.: EC2)    (ex.: Heroku, (ex.: Gmail,
                  center)                      Elastic       Google Docs)
                                               Beanstalk)
 Aplicação       ■ você         ■ você        ■ você        □ provedor
 Dados           ■ você         ■ você        ■ você        □ provedor
 Runtime         ■ você         ■ você        □ provedor    □ provedor
 Sist. Operac.   ■ você         ■ você        □ provedor    □ provedor
 Virtualização   ■ você         □ provedor    □ provedor    □ provedor
 Servidores      ■ você         □ provedor    □ provedor    □ provedor
 Rede/Storage    ■ você         □ provedor    □ provedor    □ provedor
```

- **IaaS (Infrastructure as a Service):** você recebe uma máquina virtual "crua" e cuida de tudo do sistema operacional para cima: atualizações de segurança, instalação do Python, configuração do servidor web. É o modelo da EC2, usado em [../aws/01_ec2_servidores_na_nuvem.md](../aws/01_ec2_servidores_na_nuvem.md).
- **PaaS (Platform as a Service):** você entrega o código e o provedor cuida do sistema operacional e do runtime. Menos trabalho, menos flexibilidade.
- **SaaS (Software as a Service):** você usa o software pronto. Não há infraestrutura para gerenciar.

> **Confusão comum:** "se a empresa migrou para a nuvem, não precisa mais de ninguém cuidando de infraestrutura". ✅ **Mais preciso:** a nuvem transfere **algumas** responsabilidades, e a tabela acima mostra exatamente quais. Em IaaS, aplicar patches no sistema operacional, configurar firewall (os *Security Groups* da AWS), fazer backup e monitorar continuam sendo trabalho seu. Os provedores chamam isso de **modelo de responsabilidade compartilhada**: eles garantem a segurança *da* nuvem (prédio, hardware, hypervisor), e você é responsável pela segurança *na* nuvem (o que roda nas suas máquinas, quem tem acesso, como os dados são guardados). A maioria dos vazamentos em nuvem vem de configuração errada do lado do cliente, como um bucket S3 público, e não de falha do provedor.

---

## 5. Por que isso importa para quem desenvolve

Durante muito tempo existiu uma divisão rígida: o time de desenvolvimento escrevia o código e "jogava por cima do muro" para o time de operações colocar em produção. Quando algo quebrava, cada lado culpava o outro.

O movimento **DevOps** dissolveu esse muro. A ideia é que quem constrói o software também participe de como ele é empacotado, entregue e operado. Na prática, hoje se espera que um desenvolvedor entenda:

- **como a aplicação é empacotada** (containers e Docker, em [../../02-containers/](../../02-containers/SUMARIO.md));
- **como é entregue** (pipelines de CI/CD, em [../../03-entrega-continua/](../../03-entrega-continua/SUMARIO.md));
- **como escala** (auto scaling, load balancers);
- **como falha e se recupera** (health checks, políticas de reinício);
- **como é observada**: logs, métricas e alertas. Alex Xu observa que, para um site pequeno em poucos servidores, essas ferramentas são boa prática, mas para um sistema que atende um negócio grande, investir nelas é essencial.

Há também uma razão mais profunda, de arquitetura. Richards e Ford dividem as características que um sistema precisa ter em grupos, e um deles é o das **características operacionais**: disponibilidade, confiabilidade, desempenho, escalabilidade, elasticidade e capacidade de recuperação. Os autores destacam que essas características se sobrepõem diretamente às preocupações de operações e DevOps, e que elas **não dependem só do código**. Resumindo o argumento dos autores:

> Por mais que o código seja desenhado para ser rápido ou elástico, se ele depende de um componente (um banco de dados, um servidor, uma rede) que não tem essas propriedades, o sistema como um todo não terá.

Em outras palavras: disponibilidade e escalabilidade são decididas, em boa parte, na infraestrutura. Quem só entende o código só consegue influenciar metade dessas decisões.

---

## Próximo passo

Este capítulo descreveu as camadas que sustentam um software, mas tratou o "servidor" como uma caixa fechada: uma máquina com um sistema operacional e uma aplicação dentro. A forma de organizar essa caixa mudou três vezes nas últimas décadas: um servidor físico por aplicação, depois várias máquinas virtuais por servidor, e por fim containers. Cada mudança resolveu um problema concreto da anterior.

O [02_virtualizacao_containers_e_iac.md](02_virtualizacao_containers_e_iac.md) conta essa evolução e explica, a partir dela, por que o Docker existe.
