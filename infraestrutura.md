# Infraestrutura 

**Definição:** Infraestrutura é o conjunto de recursos (processamento, memória, armazenamento e rede) necessários para sustentar o ciclo de vida de uma aplicação. Imagine que infraestrutura (ou "infra") é o alicerce físico e lógico que permite que um software exista e seja acessado. Se o seu código é o "carro", a infraestrutura é a estrada, o posto de combustível, a sinalização e a garagem.

Podemos dividir a infraestrutura em três camadas principais:

**1) Infraestrutura Física (Hardware)**
---

É o "ferro" propriamente dito. Tudo o que você pode tocar em um Data Center:

* **Servidores:** Computadores potentes (sem monitor ou teclado) que ficam ligados 24h.
* **Storage (Armazenamento):** Racks cheios de SSDs e discos rígidos para guardar os dados.
* **Rede:** Roteadores de alta performance, switches e quilômetros de cabos de fibra óptica que conectam os servidores entre si e com a internet.
* **Facilities:** Ar-condicionado industrial, geradores de energia e baterias (UPS) para garantir que nada desligue.

**2) Infraestrutura Lógica (Software de Base)**
---

É a camada que gerencia o hardware e permite que as aplicações rodem:

**Sistemas Operacionais:** Windows Server, distribuições Linux (Ubuntu, CentOS, Debian).
**Virtualizadores:** O software que cria as VMs (VMware, Hyper-V, KVM).
**Orquestradores e Roteamento:** Softwares que gerenciam endereços de IP, Firewalls, DNS (que transforma o nome do site em IP) e Load Balancers (que distribuem o tráfego).

**3) Infraestrutura como Serviço (IaaS) e Nuvem**
---

Antigamente, "infra" significava você ter uma sala gelada na sua empresa. Hoje, a infraestrutura é frequentemente alugada (AWS, Google Cloud, Azure).

* Quando você usa o Docker, você ainda precisa de infraestrutura: o container precisa de CPU, Memória RAM e Rede do servidor hospedeiro para funcionar.
* O Docker simplifica a infraestrutura porque ele abstrai a "sujeira" do Sistema Operacional, entregando apenas os recursos brutos que o app precisa.

**Por que o termo "Infraestrutura" mudou com o Docker?**
---
Antes, a infraestrutura era algo rígido. Você levava semanas para configurar um servidor. Com o Docker e o conceito de Infraestrutura como Código (IaC):

* A infraestrutura se torna um arquivo de texto (como o Dockerfile ou o Docker Compose).
* O desenvolvedor define no arquivo: "Eu preciso de 512MB de RAM e da porta 8080".
* O Docker lê o arquivo e "monta" essa infraestrutura lógica instantaneamente.

## Contexto Histórico: 

### 3) Era Bare Metal (O Hardware "Nu")
---

Até o final dos anos 90, essa era a única forma de rodar software. "Bare Metal" significa instalar o Sistema Operacional diretamente no hardware físico.

**Como era feito:**

Uma empresa comprava um servidor físico (Dell, HP, IBM). Instalava-se um Sistema Operacional (ex: Windows Server ou Red Hat) e, dentro dele, instalavam-se as aplicações (Banco de Dados, Servidor Web).

**Problemas observados:**

* A "Guerra" de Bibliotecas: Se o App A precisava da biblioteca Java 8 e o App B precisava da Java 11, você não podia rodar ambos no mesmo servidor sem conflitos críticos no sistema.

* Escalabilidade Lenta: Se o site crescesse e o servidor ficasse lento, você tinha que comprar outro servidor físico, esperar chegar, montar no rack e configurar do zero. Levava-se semanas.

* Desperdício Financeiro: Por segurança, as empresas rodavam apenas uma aplicação por servidor para evitar que um erro no App A derrubasse o App B. Como resultado, servidores potentes operavam com 10% de uso.

### 2) Era da Virtualização (As Máquinas Virtuais)
---

Em 1999/2000, a VMware popularizou a virtualização, mudando o jogo. A ideia era criar uma camada de software que "fingia" ser hardware.

**Como a tecnologia melhorou:**

Introduziu-se o Hypervisor (uma camada entre o hardware e o SO). Ele permite criar várias fatias independentes de hardware virtual (VMs).

**Como era feito com VMs:**

Em um único servidor físico, você criava 5 VMs. Cada VM tinha seu próprio disco virtual, sua própria memória RAM reservada e, crucialmente, seu próprio Sistema Operacional completo (Guest OS).

**Problemas observados (O "Peso"):**

* **Overhead de Recursos:** Para rodar um simples site de 50MB, você era obrigado a rodar um Windows ou Linux de 2GB por baixo. Você multiplicava o consumo de RAM e CPU apenas para manter os Sistemas Operacionais "vivos".

* **Lentidão no Boot:** Ligar uma VM é como ligar um computador; leva minutos para carregar o Kernel, os drivers e os serviços do sistema.

* **Portabilidade limitada:** As imagens de VM são gigantescas (gigabytes), dificultando o envio para a nuvem ou para outros desenvolvedores.

### 3) Era da Conteinerização (O Docker)
---
O Docker (lançado em 2013) percebeu que não precisávamos virtualizar o hardware, mas sim isolar o processo.

**Como a tecnologia melhorou:**

Em vez de cada aplicação carregar seu próprio Sistema Operacional (Guest OS), o Docker utiliza o Kernel (o motor) do Sistema Operacional que já está rodando no servidor físico (Host OS).

**Como é feito com Docker:**

Você cria um pacote chamado Container. Ele contém apenas o seu código e as bibliotecas que ele precisa para rodar. Ele compartilha o Kernel com o vizinho, mas não consegue "enxergar" o vizinho. Imagine um sistema operacional como o Linux é dividido em duas partes principais:

- **O Kernel (O Motor):** É a parte que fala com o hardware (processador, memória, disco).
- **O User Space (A Lataria/Interior):** É onde ficam os arquivos, bibliotecas (.dll, .so), gerenciadores de pacotes (apt, yum) e o seu código.

**A sacada técnica do Docker:**

O segredo é que o Docker compartilha o motor (Kernel), mas isola a lataria (User Space).

**Como ele isola as dependências sem o Hardware?**

O Docker usa uma tecnologia do Kernel chamada Namespaces. Imagine que o Kernel é um recepcionista de um prédio. No *Bare Metal*, todo mundo está no saguão. Se alguém grita "Porta 80 é minha!", todos ouvem. No Docker, o Kernel cria "bolhas de realidade" (Namespaces). Por exemplo: O Container "A" acha que o disco dele só tem a pasta /app. Ele não consegue ver a pasta /app do vizinho porque o Kernel mente para ele. Além disso, o Container "A" acha que o IP dele é 172.17.0.2, o vizinho tem outro etc. Outro faotr é: seu app dentro do container acha que é o processo nº 1 (o dono do sistema). No seu Windows/Linux real, ele é apenas o processo nº 5432.

**O Salto de Eficiência:**

* **Leveza:** Enquanto uma VM pesa GBs, um container pesa MBs.

* **Velocidade:** Um container inicia em milissegundos, pois o Kernel já está ligado. Ele é apenas um processo isolado.

* **Imutabilidade (Infraestrutura como Código):** O ambiente é definido no Dockerfile. Se rodar no seu PC, rodará igual em qualquer lugar, pois o "ambiente" está lacrado no container.

**OBSERVAÇÃO**
---
**O que é uma VM "tradicional"?**

Uma Máquina Virtual (VM) é uma tentativa de simular um computador físico completo dentro de outro. Para isso, ela usa um software chamado Hypervisor (como VirtualBox ou VMware).

**Como uma VM é montada:**

* **Hardware Real:** O seu computador físico.
* **Hypervisor:** O software que divide os recursos do hardware.
* **Sistema Operacional Convidado (Guest OS):** Aqui está o "peso". Cada VM precisa de uma cópia inteira de um sistema operacional (Windows ou Linux). Se você subir 10 VMs, terá 10 sistemas operacionais rodando simultaneamente.
* **Binários/Bibliotecas:** O que o seu app precisa.
* **App:** O seu código.

**O problema:** As VMs são pesadas. Elas demoram minutos para ligar (boot) e consomem muita RAM e disco só para manter o sistema operacional "convidado" funcionando, antes mesmo de rodar o seu app.

**Como o Docker se diferencia?**

O Docker não simula o hardware. **Ele compartilha o Kernel** (o núcleo) do sistema operacional que já está rodando no seu computador (o Host).

**Como o Docker é montado:**

* **Hardware Real:** O seu computador.
* **Sistema Operacional Host:** O seu Windows, Mac ou Linux.
* **Docker Engine:** Em vez de um Hypervisor pesado, temos um motor leve que gerencia o isolamento.
* **Binários/Bibliotecas:** Apenas o estritamente necessário para o app.
* **App:** O seu código dentro do container.
---