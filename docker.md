# Docker 

## 1. O que é Docker: 

**Definição:** O Docker é uma plataforma que permite empacotar uma aplicação e todas as suas dependências (bibliotecas, configurações, banco de dados) dentro de uma unidade chamada Container. 

A ideia central é: **"Se funciona na minha máquina, vai funcionar em qualquer lugar"**.

---
### 1.1 Conceitos Chave:

**Imagem:** 

A imagem é um arquivo estático e imutável. Ela contém tudo o que sua aplicação precisa: o sistema operacional simplificado, o código, as bibliotecas (como Python ou Node.js) e as configurações. Imagine a imagem como sendo uma "receita do bolo". Além disso, uma imagem é feita de fatias.

- Camada 1: Um Linux básico.
- Camada 2: O Python instalado.
- Camada 3: O seu código de inteligência artificial.

Por que a imagem é dita um "aquivo estático"? Para garantir que, se você enviar essa imagem para um colega, ele terá exatamente o mesmo ambiente que você.

**Container:** 

O container é a instância viva da imagem. Quando você diz ao Docker para "rodar" uma imagem, ele cria um container, ou seja, o container é basicamente uma imagem em execução. Imagine que se a imagem é a "receita do bolo", o container é "o bolo pronto em cima da mesa".


**Docker Hub:** 

O Docker Hub é o lugar onde a comunidade e as empresas (como Microsoft, Oracle, Google) deixam suas imagens prontas para você usar. Uma espécie de "App Store" de imagens prontas (onde você baixa imagens do Python, MySQL, Nginx, etc.).

No Docker Hub você pode subir suas próprias imagens para o Docker Hub (públicas ou privadas) para baixar no servidor da sua empresa depois. Além disso, lá existem imagens verificadas. Imagine que você precisa de Python, você baixa a imagem oficial do Python, que já foi testada e é segura.

Para entender como o Docker funciona "por baixo do capô", precisamos olhar para a diferença entre ele e uma Máquina Virtual (VM) tradicional. Enquanto uma VM cria um computador inteiro virtualizado, o Docker é muito mais cirúrgico.

---

### 1.2 Mas afinal, pra que tudo isso? 

O Docker não é apenas uma "modinha" entre desenvolvedores; ele resolve dores reais:

* **Isolamento:** Você pode rodar duas versões diferentes da mesma linguagem no mesmo computador sem que uma quebre a outra.

* **Portabilidade:** O container que você criou no Windows vai rodar identicamente no Linux de um servidor na nuvem (AWS, Azure, Google Cloud).

* **Agilidade:** Subir um banco de dados inteiro com Docker leva segundos, enquanto instalar manualmente pode levar horas.

* **Padronização:** Garante que todo o time de desenvolvimento esteja usando exatamente o mesmo ambiente.


## 2. Como funciona o Docker: 

### 2.1 A Arquitetura: Cliente e Servidor

O Docker utiliza uma arquitetura cliente-servidor. Quando você digita um comando no terminal, você está falando com o **Docker Client** (É onde você está. Quando você abre o seu terminal - CMD, PowerShell ou Bash - e digita docker run, você está usando o **Docker Client**). 

O Docker Clint envia a ordem para o **Docker Daemon** (Chamado de **dockerd** - o verdadeiro motor. Ele fica rodando em segundo plano no sistema operacional. Ele ouve as solicitações do Cliente. Quando você pede para rodar um container, é o Daemon que vai verificar se a imagem existe, baixar se necessário e reservar a memória RAM para ela. Ele gerencia o isolamento, garantindo que um container não invada o espaço do outro).

O **Registry**, por sua vez, é onde ficam guardadas as imagens. O mais famoso é o Docker Hub. Quando o Daemon percebe que você quer rodar algo que não tem no computador, ele corre até o Registry, baixa a imagem e a armazena localmente.

---


### 2.2 A Mágica do Isolamento (Kernel do Linux)

**Antes de etendermos esse ponto, precisamos entender o que é uma VM:**

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

Diferente das VMs, o Docker não instala um sistema operacional inteiro dentro de cada container. Ele usa recursos do próprio "núcleo" (Kernel) do sistema hospedeiro para isolar os processos (Imagine que as VMs são casas independentes e o Docker são apartamentos em um prédio):



* **Namespaces**: Garantem que cada container tenha sua própria visão do sistema (rede, usuários, processos), como se estivesse sozinho no computador.

* **Control Groups (cgroups)**: Limitam quanto de CPU e Memória cada container pode usar, evitando que um app "fominha" derrube o computador inteiro.

### 2.3 O Sistema de Arquivos em Camadas (Union File Systems)

Este é o segredo da leveza do Docker. As imagens são compostas por camadas sobrepostas:

*  Se você tem uma imagem de Python, ela tem uma camada com o Linux básico e outra com o Python instalado.
* Quando você cria uma aplicação sua baseada nela, o Docker apenas adiciona uma camada fina de escrita no topo.
* As camadas de baixo são somente leitura e podem ser compartilhadas entre vários containers, economizando muito espaço em disco.

---
