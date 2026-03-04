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

Diferente das VMs, o Docker não instala um sistema operacional inteiro dentro de cada container. Ele usa recursos do próprio "núcleo" (Kernel) do sistema hospedeiro para isolar os processos (Imagine que as VMs são casas independentes e o Docker são apartamentos em um prédio):

* **Namespaces**: Garantem que cada container tenha sua própria visão do sistema (rede, usuários, processos), como se estivesse sozinho no computador.

* **Control Groups (cgroups)**: Limitam quanto de CPU e Memória cada container pode usar, evitando que um app "fominha" derrube o computador inteiro.

### 2.3 O Sistema de Arquivos em Camadas (Union File Systems)

Este é o segredo da leveza do Docker. As imagens são compostas por camadas sobrepostas:

*  Se você tem uma imagem de Python, ela tem uma camada com o Linux básico e outra com o Python instalado.
* Quando você cria uma aplicação sua baseada nela, o Docker apenas adiciona uma camada fina de escrita no topo.
* As camadas de baixo são somente leitura e podem ser compartilhadas entre vários containers, economizando muito espaço em disco.

---

## 3. Primeiros passos 

### 3.1 Instalação (O Motor)

Antes de tudo, você precisa do Docker rodando no seu computador.

* Acesse o site oficial **`Docker Desktop`**. 
* Baixe a versão para o seu sistema (Windows, Mac ou Linux).
* Dica para Windows: Ele vai pedir para instalar o "WSL 2" (Windows Subsystem for Linux). Aceite e instale, pois é isso que permite ao Docker rodar com performance nativa no Windows.
* Após instalar e reiniciar, abra o Docker Desktop e espere o ícone da baleia ficar parado (verde).

**Observação:**
---

WSL significa Windows Subsystem for Linux (Subsistema do Windows para Linux). É uma funcionalidade do Windows que permite rodar Linux diretamente dentro do Windows, sem precisar instalar máquina virtual ou fazer dual boot.

**O que isso significa na prática?** 

Com o WSL você pode:

* Usar o terminal do Linux (bash)
* Instalar distribuições como Ubuntu, Debian, Kali
* Rodar comandos como ls, grep, apt, ssh
* Desenvolver com ferramentas Linux (Node, Python, Docker, etc.)
* Trabalhar com desenvolvimento backend, DevOps e programação em geral

**Existe mais de uma versão?**

* WSL = tecnologia do Windows para rodar Linux dentro do Windows
* WSL 1 = primeira versão (tradução de chamadas do Linux para o Windows)
* WSL 2 = segunda versão, usa um kernel Linux real (muito mais rápido e compatível)

---

### 3.2 O Primeiro Teste (Hello World)

Abra o seu terminal (PowerShell no Windows, ou Terminal no Mac/Linux) e digite: **`docker run hello-world`**

O que vai acontecer?

* O Client pergunta ao Daemon se ele tem a imagem hello-world.
* Como você acabou de instalar, ele não terá. Ele vai dizer: Unable to find image... locally.
* Ele vai baixar (Pull) a imagem do Docker Hub.
* Ele vai criar o Container e rodar. Você verá uma mensagem de boas-vindas na tela.

### 3.3 Rodando um Servidor de Simulado:

Vamos subir um servidor de sites (Nginx) sem instalar nada no seu OS. O objetivo é simular o ambiente de um servidor real sem sair da sua máquina.

**`docker run -d -p 8080:80 --name meu-site nginx`**

Explicando os termos:

* **`-d`** (Detached): Roda o container em segundo plano (você pode continuar usando o terminal).
* **`-p 7777:80`**: Redireciona a porta 8080 do seu PC para a porta 80 dentro do container.
* **`--name meu-site`**: Dá um nome amigável ao seu container.
* Teste agora: Abra o navegador e digite **`localhost:7777`**. Você verá a página "Welcome to nginx!".

O que fizemos aqui: 

* 1) Baixou um software pronto (Nginx) sem precisar configurar instaladores .exe ou .msi.
* 2) Reservou uma fatia da sua memória para esse software rodar sem interferir em outros programas.
* 3) Criou uma ponte de comunicação (-p) para que seu navegador pudesse falar com um software "preso" dentro de um container.

### 3.4 Rodando um Servidor de Verdade: 

Para realizar essa atividade precisamos de um **`Dockerfile`**, mas afinal o que é isso? 

**`Dockerfile`** é um arquivo de texto simples, sem extensão (o nome é apenas Dockerfile), que contém uma lista de instruções e encontra-se localizado no mesmo diretório do nosso projeto. O Docker lê esse arquivo e executa cada linha para "buildar" (construir) uma imagem personalizada para o seu projeto.

**Por que precisamos dele no diretório do projeto?**

Ter o Dockerfile junto com o seu código (no mesmo diretório) é o que garante a **portabilidade**.

* **Padronização:** Qualquer pessoa que baixar seu projeto (um novo colega de equipe, por exemplo) só precisa digitar docker build. O Dockerfile vai garantir que o ambiente dele seja idêntico ao seu.
* **Automação:** Quando você envia seu código para a nuvem (como AWS ou Google Cloud), o servidor lá lê o seu Dockerfile e sabe exatamente como "montar" sua aplicação para colocá-la no ar.
* **Versionamento:** O Dockerfile fica salvo no seu Git. Se você mudar a versão do seu banco de dados, essa mudança fica registrada no histórico do projeto.

**Como é a estrutura de um dockerfile?**

Um Dockerfile funciona em camadas. Cada comando cria uma nova camada na imagem. A estrutura básica segue este fluxo:

* 1) **FROM:** De onde vamos começar? (A imagem base).
* 2) **WORKDIR:** Onde vamos trabalhar dentro do container? (A pasta do projeto).
* 3) **COPY/ADD:** Quais arquivos do meu PC eu quero levar para dentro do container?
* 4) **RUN:** Quais comandos de instalação preciso rodar? (Ex: npm install ou pip install).
* 5) **EXPOSE:** Qual porta o container vai "abrir"?
* 6) **CMD:** Qual o comando final para ligar a aplicação?

**Exemplo prático:**

Imagine que você tem um site simples em Node.js. O seu Dockerfile seria assim:

```dockerfile 
# 1. Define a imagem base (já vem com Node.js instalado)
FROM node:18

# 2. Cria uma pasta dentro do container para o seu código
WORKDIR /app

# 3. Copia os arquivos de dependências primeiro (otimiza o cache)
COPY package*.json ./

# 4. Roda o comando para instalar as bibliotecas
RUN npm install

# 5. Copia o restante dos arquivos do seu projeto
COPY . .

# 6. Informa que o app usa a porta 3000
EXPOSE 3000

# 7. O comando que "liga" o site de fato
CMD ["node", "server.js"]
```

**Como usar esse arquivo?**

Depois de criar esse arquivo na raiz do seu projeto, você usa dois comandos no terminal:

* Construir a imagem:

```bash
docker build -t meu-projeto .
```

O ponto . diz que o Dockerfile está na pasta atual.

* Rodar o container:

```bash
docker run -p 3000:3000 meu-projeto
```

**Por que essa ordem importa?**

Repare que no exemplo eu copiei o package.json antes do resto do código.
O motivo: O Docker é inteligente. Se você mudar uma linha de texto no seu site, mas não instalar nenhuma biblioteca nova, o Docker percebe que a camada do RUN npm install não mudou e pula ela, tornando o processo de "rebuild" absurdamente rápido.


### 3.5 Lista de Comandos: 

## **Comandos de Criação e Construção**

Cria uma imagem a partir do Dockerfile que está na pasta atual.
---

```bash
docker build -t nome-da-imagem .
```

 O **`-t`** serve para dar uma "tag" (nome) à imagem. **Esse comando precisa ser dado no mesmo diretorio do projeto.**


Lista todas as imagens que você já baixou ou construiu no seu computador.
---
```bash
docker images
```

## **Comandos de Execução (Ciclo de Vida)**


Baixa a imagem, cria e inicia novo um container (se não tiver). 
--- 
```bash
docker run -d --name meu-container -p 7777:99 nome-da-imagem
```

* **`-d`**: Roda em segundo plano (background).

* **`-p`**: Mapeia a porta (PC:Container).

* **`7777:99`**: 7777 (Lado do Host/Seu PC) é a porta que você vai digitar no seu navegador (localhost:8080). Você pode escolher quase qualquer número aqui (ex: 7777, 9000, 5000), desde que não esteja sendo usado por outro programa. 99 (Lado do Container) é a porta onde o software dentro do container está configurado para "ouvir"

Desliga o container graciosamente.
---
```bash
docker stop nome-do-container
```


Liga um container que estava parado (mantendo as alterações feitas nele).
---
```bash
docker start nome-do-container
```


Desliga e liga novamente.
---
```bash
docker restart nome-do-container
```


Lista os containers que estão rodando agora.
---
```bash
docker ps
```


Lista todos os containers (rodando e parados).
---
```bash
docker ps -a
```


## **Comandos de Limpeza (Descarte)**


Apaga um container (ele precisa estar parado).
---
```bash
docker rm nome-do-container
```


Apaga uma imagem do seu disco.
---
```bash
docker rmi nome-da-imagem
```


O "limpa tudo": apaga todos os containers parados e imagens sem uso de uma vez.
---
```bash
docker system prune
```


# **Comandos de Rede e Comunicação**
---

Criar uma rede 
---
```bash
docker network create minha-rede
```


Rodar o banco de dados na rede
---
```bash
docker run -d --name banco --network minha-rede mongo
```


Rodar o Site na mesma rede
---
```bash
docker run -d --name site --network minha-rede meu-site-imagem
```


Listar todas as redes criadas
---
```bash
docker network ls
```


Mostra detalhes da rede, incluindo quais containers estão nela e quais os IPs internos (caso você precise muito saber o IP).
---
```bash
docker network inspect nome-da-rede
```


# **Comandos de Inspeção (O que está acontecendo lá dentro?)**
---

Mostra em tempo real o que o seu app está "printando" no console (ajuda muito a achar erros).
---
```bash
docker logs -f nome-do-container
```

Este comando "entra" no container. É como se você abrisse um terminal dentro daquela máquina isolada para navegar nas pastas.
---
```bash
docker exec -it nome-do-container sh
```
