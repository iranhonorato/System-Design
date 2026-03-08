# Hands-On: Continuous Delivery/Deployment

## Antes de começarmos, precisamos alinhas alguns pontos: 

**1. Qual projeto vamos usar?**

Seu projeto é:

* Backend (Node, Java, Python, etc.)?
* Frontend (React, Vue, etc.)?
* Fullstack?
* API REST?
* Projeto estático?

**2. Onde você quer fazer deploy?**

* AWS EC2 **(Esse tutorial cobre o caso AWS EC2)**
* AWS Elastic Beanstalk
* Docker + EC2
* Vercel / Render / Railway
* Outro?

**3. Arquitetura que vamos montar**

Queremos que sempre que você fizer push na branch main, o GitHub:

* Construa a imagem Docker
* Envie para o Docker Hub
* Conecte na sua EC2
* Faça pull da nova imagem
* Reinicie o container automaticamente

Sem você precisar acessar manualmente a EC2. Logo, a visão geral do fluxo vai ficar mais ou menos assim

```Plain
git push main
     ↓
GitHub Actions roda pipeline
     ↓
Build do projeto
     ↓
SSH automático na EC2
     ↓
Atualiza código
     ↓
Reinicia aplicação
```

Isso é Continuous Delivery real. Agora vamos construir isso passo a passo.

## PASSO 1 — Criar usuário deploy na EC2 (boa prática)

Você pode usar ubuntu mas criar um usuário dedicado é uma boa prática de segurança. Eis os motivos principais: 

* Separação de responsabilidades: Usuário humano (`ubuntu`) e Usuário de automação (`deploy-python`)
* Cada usuário deve ter apenas as permissões necessárias. Se o GitHub Actions for comprometido, o atacante só terá acesso ao usuário de deploy, não ao usuário principal.
* Organização profissional

Chega de papo, vamos lá.Na EC2, vamos criar o usuário: 

```bash 
ubuntu@ip-172-31-9-94:~$ sudo adduser deploy-python  # deploy-python é o nome que eu escolhi pro user
info: Adding user `deploy-python' ...
info: Selecting UID/GID from range 1000 to 59999 ...
info: Adding new group `deploy-python' (1001) ...
info: Adding new user `deploy-python' (1001) with group `deploy-python (1001)' ...
info: Creating home directory `/home/deploy-python' ...
info: Copying files from `/etc/skel' ...
New password:
Retype new password:
passwd: password updated successfully
Changing the user information for deploy-python
Enter the new value, or press ENTER for the default
        Full Name []:
        Room Number []:
        Work Phone []:
        Home Phone []:
        Other []:
Is the information correct? [Y/n] y
info: Adding new user `deploy-python' to supplemental / extra groups `users' ...
info: Adding user `deploy-python' to group `users' ...
ubuntu@ip-172-31-9-94:~$
```

Depois que o usuário for criado, execute:

```bash 
ubuntu@ip-172-31-9-94:~$ sudo usermod -aG docker deploy-python
ubuntu@ip-172-31-9-94:~$
```

O que esse comando faz: 

| Comando             | Significado                       |
|---------------------|-----------------------------------|
| usermod             | modifica usuário                  |
| -aG docker          | adiciona ao grupo docker          |
| deploy-python       | usuário                           |


Ou seja, permite que o usuário deploy-python execute docker sem sudo. Agora, após feito tudo isso, vamos testar se funcionou: 

```bash
ubuntu@ip-172-31-9-94:~$ su - deploy-python
Password:
deploy-python@ip-172-31-9-94:~$ docker ps
```

Se listar os containers sem erro de permissão, está correto. Mas se aparecer erro de permissão, saia e faça logout/login novamente ou reinicie a sessão SSH.

## PASSO 2 — Criar chave SSH no seu computador (NÃO na EC2)

No seu computador local (Windows PowerShell ou Git Bash):

```bash 
PS C:\Users\irani> ssh-keygen -t rsa -b 4096 -C "github-actions-deploy"
Generating public/private rsa key pair. 
Enter file in which to save the key (C:\Users\irani/.ssh/id_rsa): github-actions-python-ec2 # (nome que você quiser)
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in github-actions-python-ec2
Your public key has been saved in github-actions-python-ec2.pub
The key fingerprint is:
SHA256:q4IYjFn+EicUUQyPIbSy1CWV6gAA/91NEIlQf+5OH1s github-actions-deploy
The key's randomart image is:
+---[RSA 4096]----+
|B.+*+=o.oo       |
|.o+++.....       |
|ooooo   . o      |
|oooo . . =       |
|+=o . . S o      |
|+.+..    o       |
| o *    . o . E  |
|. o o  . o . +   |
|   . ..   . o    |
+----[SHA256]-----+
PS C:\Users\irani>
```

Com o comando acima nós criamos um par de chaves criptográficas SSH:

* github-actions-ec2
* github-actions-ec2.pub

**Mas por que precisamos disso?**

Porque o GitHub Actions precisa acessar sua EC2 automaticamente sem você digitar senha e de forma segura. Mas o GitHub é um servidor remoto, ou seja, ele não pode pedir senha interativamente, guardar senha em texto puro e usar login manual. Então, por conta dissom usamos autenticação por chave SSH.


## PASSO 3 — Adicionar chave pública na EC2

Neste passo, vamos adicionar a chave pública na EC2. O objetivo é permitir que:

**“Permita que quem possuir a chave privada correspondente possa entrar como `deploy-python`.”**

Então, pra começar precisamos pegar a chave pública. No seu computador, execute: 

```powershell 
PS C:\Users\irani> cat github-actions-python-ec2.pub
```

Após isso, a chave vai aparecer no terminal e é importante que você copie **TODO** o conteúdo que aparecer (vai começar com *ssh-rsa*). Após isso, guarde o conteúdo pois precisaremos dele daqui a pouco.

Agora precisamos criar estrutura SSH no servidor. Precisamos criar a pasta obrigatória `/home/deploy-python/.ssh` (Sem essa pasta, o SSH simplesmente ignora autenticação por chave.) e abrir (ou criar) o arquivo `authorized_keys` nessa pasta (esse arquivo é necessário pois é onde ficam as chaves de permissão). O serviço SSH procura automaticamente por chaves autorizadas no caminho ~/.ssh/authorized_keys. Caso essa estrutura não exista, a autenticação por chave é ignorada. 

```bash 
# criar a pasta '.ssh'
ubuntu@ip-172-31-9-94:~$ sudo mkdir -p /home/deploy-python/.ssh

# abrir (ou criar) arquivo 'authorized_keys'
ubuntu@ip-172-31-9-94:~$ sudo nano /home/deploy-python/.ssh/authorized_keys
```

Cole a chave pública e depois faça: 

* `CTRL + X`
* `y` 
* `ENTER`

Agora devemos ajustar as permissões 

```bash 
# Define dono correto. Evita acesso indevido.
ubuntu@ip-172-31-9-94:~$ sudo chown -R deploy-python:deploy-python /home/deploy-python/.ssh

# Protege pasta.Só dono acessa.
ubuntu@ip-172-31-9-94:~$ sudo chmod 700 /home/deploy-python/.ssh

# Protege chave. Impede modificação por terceiros.
ubuntu@ip-172-31-9-94:~$ sudo chmod 600 /home/deploy-python/.ssh/authorized_keys
```


## PASSO 4 - Configurar Secrets no GitHub 

Neste passo, vamos armazenar credenciais sensíveis de forma segura no GitHub.

Os Secrets permitem que o GitHub Actions utilize informações confidenciais (como chaves SSH e tokens do Docker) sem expô-las no código do repositório.

Quando o GitHub Actions executa um workflow, ele roda em uma máquina temporária chamada: `GitHub Runner`. Essa máquina:

* Não conhece sua EC2
* Não conhece sua senha
* Não conhece seu Docker Hub
* Não conhece suas chaves

Então precisamos fornecer essas credenciais de forma segura. O GitHub fornece um mecanismo chamado: `Repository Secrets`

Eles são:

* Criptografados
* Não aparecem nos logs
* Não ficam no código
* Só podem ser acessados pelo workflow

**Por que não colocar isso direto no YAML?**

Porque isso seria:

* Extremamente inseguro
* Ficaria visível no repositório
* Poderia ser usado por qualquer pessoa
* Violaria boas práticas DevOps

Vá no seu repositório: 

```Plain 
Settings → Secrets and variables → Actions → New repository secret
```
Crie os seguintes secrets:


| Name                | Secret                                       |
|---------------------|----------------------------------------------|
| EC2_HOST            | 18.231.250.104                               |
| EC2_USER            | deploy-python                                |
| EC2_SSH_KEY         | `key`                                        |
| DOCKER_USERNAME     | Usuário do Docker Hub                        |
| DOCKER_PASSWORD     | `Personal Access Token do Docker Hub`        |


**key** 
---

No PowerShell, digite: 

```PowerShell
PS C:\Users\irani> notepad github-actions-python-ec2
```

Isso vai abrir o arquivo no Bloco de Notas. É importante que você copie TUDO, inclusive as linhas BEGIN e END, depois cole em Secret de EC2_SSH_KEY


**Personal Access Token do Docker Hub** 
---

Vá ao Docker Hub:

```Plain
Busque por Settings 
     ↓
Vá a Personal access tokens
     ↓
Clique em Generate new token
     ↓
Preencha Token description: github-actions-cd
     ↓
Em Access permissions, escolha: Read & Write
     ↓
Clique em: Generate
     ↓
O Docker vai mostrar o token uma única vez. Vai começar com algo assim: dckr_pat_
```

## PASSO FINAL - Criar o arquivo do GitHub Actions

No seu projeto local, crie a seguinte estrutura:

```Plain
.github/
  workflows/
    cd.yml
``` 

Cole este conteúdo e salve-o no arquivo: 

```YAML
name: CI/CD - Python App

on:
  push:
    branches: [ "main" ]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout código
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v3
        with:
          python-version: "3.10"

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install flake8 pytest
          if [ -f requirements.txt ]; then pip install -r requirements.txt; fi

      - name: Lint
        run: |
          flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics

      - name: Login Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build and push imagem
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ secrets.DOCKER_USERNAME }}/app_users:latest

      - name: Deploy na EC2
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            docker pull ${{ secrets.DOCKER_USERNAME }}/app_users:latest
            docker stop app_users || true
            docker rm app_users || true
            docker run -d -p 80:5000 --name app_users \
              ${{ secrets.DOCKER_USERNAME }}/app_users:latest
```

Depois vá no GitHub aba "Actions". Você verá o workflow rodando.