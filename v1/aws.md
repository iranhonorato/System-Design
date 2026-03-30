# Guia Completo: Como Utilizar uma Máquina Virtual na AWS (EC2)

Este guia descreve passo a passo como acessar e utilizar uma máquina virtual na AWS, explicando os conceitos envolvidos e os comandos utilizados.

## 1. O que é uma Máquina Virtual na AWS?

Na AWS, uma máquina virtual é chamada de Instância EC2 (Elastic Compute Cloud).

Ela é:

* Um computador rodando em um data center da Amazon
* Acessível remotamente pela internet
* Totalmente configurável (CPU, memória, disco, sistema operacional)
* Usada para hospedar aplicações, bancos de dados, APIs, sites, etc.

**Observação:** Informações da minha máquina AWS
---
```txt
Instância: 
aluno_13

DNS IPV4:
http://ec2-18-231-250-104.sa-east-1.compute.amazonaws.com/

IP: 
18.231.250.104 
```

## 2. Conceitos Fundamentais Antes da Conexão

Antes de acessar a máquina, é importante entender alguns conceitos:

### 2.1 Chave SSH (.pem)

* Arquivo fornecido na criação da instância
* Funciona como uma chave privada digital
* Substitui o uso de senha
* Deve ser protegida

Exemplo: 

```bash 
chave-linux.pem
```

### 2.2 IP Público e DNS

* IP público: 18.231.250.104
* DNS público: ec2-18-231-250-104.sa-east-1.compute.amazonaws.com

Ambos apontam para sua instância na AWS.


### 2.3 Usuário padrão

Depende do sistema operacional escolhido:

| Sistema Operacional | Usuário Padrão |
|----------------------|----------------|
| Ubuntu               | ubuntu         |
| Amazon Linux         | ec2-user       |
| Debian               | admin          |

No nosso caso: 

```bash 
ubuntu
```

## 3. Acessando a Máquina via SSH (Windows PowerShell)

### 3.1 Navegando até a pasta da chave

Você precisa estar na pasta onde está o arquivo ``.pem``.

```PowerShell 
PS C:\Users\irani> cd desktop
PS C:\Users\irani\desktop> cd "Insper - CComp"
PS C:\Users\irani\desktop\Insper - CComp> cd "Projeto de Software e Gestão Ágil"
```

### 3.2 Conectando via SSH

```PowerShell 
PS C:\Users\irani\desktop\Insper - CComp\Projeto de Software e Gestão Ágil> ssh -i "chave-linux.pem" ubuntu@18.231.250.104
```

Explicando o comando acima: 

| Parte               | Significado                             |
|---------------------|------------------------------------------|
| ssh                 | Protocolo de conexão remota segura       |
| -i                  | Indica o arquivo de chave                |
| chave-linux.pem     | Sua chave privada                       |
| ubuntu              | Usuário remoto                          |
| 18.231.250.104      | IP da instância                         |


### 3.3 Primeira Conexão (Authenticity)

Na primeira vez, o sistema perguntará:

```Plain text 
Are you sure you want to continue connecting (yes/no)?
```

Digite:


```Plain text 
yes
```

Isso:

* Salva a impressão digital do servidor
* Evita ataques de "man-in-the-middle"
* Adiciona o host ao arquivo known_hosts

## 4. O Que Mudou Após a Conexão?

Antes:

```PowerShell
PS C:\Users\irani>
```

Depois:

```PoowerShell
ubuntu@ip-172-31-9-94:~$
```

Isso significa que:

* Você saiu do seu computador
* Agora está executando comandos na máquina da AWS
* Tudo digitado será executado na nuvem

## 5. Entendendo as Informações Iniciais do Sistema

Ao conectar, o sistema mostra informações como:

```Plain text
System load: 0.0
Usage of /: 28%
50 updates can be applied
```

O que isso significa?

| Informação          | Significado                             |
|---------------------|-----------------------------------------|
| System load 0.0     | Máquina ociosa                          |
| Usage of / 28%      | 28% do disco usado                      |
| 50 updates          | Atualizações pendentes                  |


## 6. Comandos Básicos no Linux (Após Conectar)

**Ver arquivos na pasta atual**

```bash
ls
```

**Ver diretório atual**

```bash
pwd
```

**Entrar em uma pasta**

```bash
cd nome_da_pasta
```

**Atualizar o sistema (recomendado)**

```bash
sudo apt update
sudo apt upgrade -y
```

**Ver IP da máquina**

```bash
curl ifconfig.me
```

## 7. Fluxo Geral de Uso de uma VM na AWS

* Criar instância EC2
* Baixar chave .pem
* Configurar Security Group (porta 22 aberta para SSH)
* Conectar via SSH
* Instalar dependências (Docker, Node, Java, etc.)
* Deploy da aplicação
* Acessar via navegador usando IP público

## **OBSERVAÇÃO: SEGURANÇA**

**Sempre:**
---
* Proteja o arquivo .pem
* Restrinja acesso SSH no Security Group
* Use firewall
* Atualize o sistema

**Nunca:**
---

* Compartilhe sua chave privada
* Deixe porta 22 aberta para 0.0.0.0/0 em produção


## 9. Estrutura Conceitual da Arquitetura

```Plain text
Seu Notebook (Windows)
        │
        │ SSH (chave privada)
        ▼
Internet (canal criptografado)
        |
        ▼
AWS Data Center
        |
        ▼
Instância EC2 (Ubuntu Linux)
        |
        ▼
Aplicações / APIs / Banco de Dados
```