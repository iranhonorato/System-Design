# Hands-On: Continuous Delivery/Deployment com GitHub Actions

## Objetivo

Configurar um pipeline de CD completo que, a cada `git push` na branch `main`, execute automaticamente:

1. Testes e validação do código
2. Build da imagem Docker
3. Push da imagem para o Docker Hub
4. Conexão SSH à EC2
5. Atualização do container em produção

```
git push → main
        ↓
GitHub Actions
        ↓
✅ Testes passam
        ↓
🐳 Build da imagem Docker
        ↓
📤 Push para Docker Hub
        ↓
🔑 SSH automático na EC2
        ↓
⬇️  Pull da nova imagem
        ↓
♻️  Reinício do container
        ↓
🚀 Aplicação atualizada em produção
```

---

## Por que esse fluxo é necessário?

O GitHub Actions executa cada pipeline em uma **máquina virtual temporária** chamada GitHub Runner. Essa máquina:
- É criada do zero a cada execução
- Não tem acesso à sua EC2 por padrão
- Não conhece suas senhas ou tokens
- É destruída ao final do pipeline

Por isso, precisamos fornecer todas as credenciais necessárias de forma segura — e é exatamente isso que os próximos passos configuram.

---

## PASSO 1 — Criar usuário de deploy na EC2

### Por que criar um usuário separado?

Usar o usuário `ubuntu` para automações é uma má prática de segurança. O princípio do **mínimo privilégio** diz que cada agente (humano ou automatizado) deve ter apenas as permissões estritamente necessárias.

| Aspecto | Sem separação | Com separação |
|---|---|---|
| **Risco** | GitHub Actions comprometido = acesso total ao servidor | GitHub Actions comprometido = acesso limitado ao deploy |
| **Rastreabilidade** | Impossível distinguir ação humana de automação nos logs | Logs mostram claramente quem fez o quê |
| **Controle** | Revogar acesso derruba humanos e automações | Revoga automação sem afetar acesso humano |

### Criando o usuário

Conecte-se à EC2 com seu usuário `ubuntu` e execute:

```bash
# Criar o usuário de deploy
sudo adduser deploy-python
```

O sistema pedirá uma senha e algumas informações (pode pressionar Enter para deixar em branco exceto a senha). Após criar, adicione ao grupo docker:

```bash
# Permitir que o usuário execute docker sem sudo
sudo usermod -aG docker deploy-python
```

**Por que o grupo docker?** Por padrão, apenas o root pode executar comandos Docker. Adicionar um usuário ao grupo `docker` concede essa permissão sem precisar de `sudo`.

### Verificando

```bash
# Mudar para o novo usuário
su - deploy-python

# Testar acesso ao Docker
docker ps
```

Se listar containers sem erro, está configurado corretamente. Se aparecer "permission denied", faça logout e login novamente para que o grupo seja aplicado.

---

## PASSO 2 — Gerar o par de chaves SSH para o GitHub Actions

### Como funciona a autenticação automática

O GitHub Actions precisa conectar à EC2 **sem interação humana** — não há ninguém para digitar uma senha. A solução é autenticação por chave SSH:

1. Você gera um par de chaves (pública + privada)
2. A **chave pública** vai para a EC2 (autoriza conexões)
3. A **chave privada** vai para os Secrets do GitHub (permite que o Actions se autentique)

**Execute no seu computador** (não na EC2):

```bash
# Gerar par de chaves RSA de 4096 bits
ssh-keygen -t rsa -b 4096 -C "github-actions-deploy"
```

Quando perguntar o nome do arquivo, escolha algo descritivo:
```
Enter file in which to save the key: github-actions-python-ec2
```

Deixe a passphrase em branco (pressione Enter duas vezes) — o GitHub Actions não consegue digitar senhas interativamente.

Isso cria dois arquivos:
- `github-actions-python-ec2` → **chave privada** (vai para o GitHub Secrets)
- `github-actions-python-ec2.pub` → **chave pública** (vai para a EC2)

> **Regra de ouro:** a chave privada nunca deve sair do seu computador, exceto para ir para os Secrets criptografados do GitHub. Nunca a comite, nunca a compartilhe por email ou chat.

---

## PASSO 3 — Instalar a chave pública na EC2

O SSH usa um mecanismo simples: o servidor só aceita conexões de quem tem a chave privada correspondente a uma chave pública listada em `~/.ssh/authorized_keys`.

### Obtendo a chave pública

No seu computador:
```bash
cat github-actions-python-ec2.pub
```

Copie **todo** o conteúdo (começa com `ssh-rsa` e termina com `github-actions-deploy`).

### Instalando na EC2

Conecte-se à EC2 com seu usuário `ubuntu` e:

```bash
# Criar a estrutura de diretórios SSH para o usuário deploy-python
sudo mkdir -p /home/deploy-python/.ssh

# Abrir (ou criar) o arquivo de chaves autorizadas
sudo nano /home/deploy-python/.ssh/authorized_keys
```

Cole a chave pública, salve (`CTRL+X`, `Y`, `Enter`).

### Ajustando permissões (crítico)

O SSH é extremamente rigoroso com permissões. Se as permissões estiverem erradas, **a autenticação por chave é silenciosamente ignorada**:

```bash
# Definir o dono correto dos arquivos
sudo chown -R deploy-python:deploy-python /home/deploy-python/.ssh

# A pasta .ssh deve ser acessível apenas pelo dono (700 = rwx------)
sudo chmod 700 /home/deploy-python/.ssh

# O arquivo de chaves deve ser legível apenas pelo dono (600 = rw-------)
sudo chmod 600 /home/deploy-python/.ssh/authorized_keys
```

### Testando a conexão

No seu computador:
```bash
ssh -i github-actions-python-ec2 deploy-python@18.231.250.104
```

Se conectar sem pedir senha, está funcionando. Esse é exatamente o que o GitHub Actions fará.

---

## PASSO 4 — Configurar Secrets no GitHub

### O que são GitHub Secrets?

Secrets são variáveis criptografadas armazenadas no GitHub que ficam disponíveis para os workflows, mas que:
- Não aparecem nos logs (são mascaradas automaticamente)
- Não são visíveis no código do repositório
- Só podem ser acessadas pelos workflows do próprio repositório

**Por que não colocar direto no arquivo YAML?**

```yaml
# ❌ NUNCA FAÇA ISSO:
- name: Deploy
  with:
    host: 18.231.250.104
    password: minha-senha-super-secreta   # visível para qualquer um!
```

Qualquer pessoa com acesso ao repositório (ou que clonar) teria suas credenciais. Em repositórios públicos, seria ainda pior — o mundo inteiro veria.

### Como adicionar Secrets

No seu repositório GitHub:
```
Settings → Secrets and variables → Actions → New repository secret
```

Crie cada um dos seguintes secrets:

| Name | Valor | Como obter |
|---|---|---|
| `EC2_HOST` | `18.231.250.104` | IP da sua instância EC2 |
| `EC2_USER` | `deploy-python` | Usuário criado no Passo 1 |
| `EC2_SSH_KEY` | Conteúdo da chave privada | Ver abaixo |
| `DOCKER_USERNAME` | Seu usuário do Docker Hub | Sua conta no hub.docker.com |
| `DOCKER_PASSWORD` | Personal Access Token | Ver abaixo |

### Obtendo o valor de `EC2_SSH_KEY`

A chave privada é um arquivo de texto. No Windows:
```powershell
notepad github-actions-python-ec2
```

Copie **todo o conteúdo**, incluindo as linhas de cabeçalho e rodapé:
```
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXk...
(várias linhas de texto codificado)
...
-----END OPENSSH PRIVATE KEY-----
```

### Gerando o Personal Access Token do Docker Hub

1. Acesse [hub.docker.com](https://hub.docker.com)
2. Vá em **Account Settings → Security → Personal Access Tokens**
3. Clique em **Generate New Token**
4. Nome: `github-actions-cd`
5. Permissão: **Read & Write** (precisa para fazer push de imagens)
6. Clique em **Generate**

> O token aparece **apenas uma vez**. Copie e salve imediatamente no Secret `DOCKER_PASSWORD`.

---

## PASSO 5 — Criar o Workflow do GitHub Actions

### Estrutura de arquivos

Crie a seguinte estrutura no seu projeto:

```
seu-projeto/
├── .github/
│   └── workflows/
│       └── cd.yml      ← arquivo do pipeline
├── app.py
├── requirements.txt
└── Dockerfile
```

O GitHub Actions detecta automaticamente arquivos `.yml` dentro de `.github/workflows/` e os executa conforme as configurações.

### O arquivo do pipeline

```yaml
name: CI/CD - Python App

# Dispara o pipeline quando há push na branch main
on:
  push:
    branches: [ "main" ]

jobs:
  build-and-deploy:
    # O runner é uma VM Ubuntu temporária criada pelo GitHub
    runs-on: ubuntu-latest

    steps:
      # 1. Baixa o código do repositório para o runner
      - name: Checkout código
        uses: actions/checkout@v4

      # 2. Instala Python no runner
      - name: Set up Python
        uses: actions/setup-python@v3
        with:
          python-version: "3.10"

      # 3. Instala as dependências do projeto
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install flake8 pytest
          if [ -f requirements.txt ]; then pip install -r requirements.txt; fi

      # 4. Verifica qualidade de código com flake8
      # E9: erros de sintaxe, F63/F7/F82: undefined names
      - name: Lint
        run: |
          flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics

      # 5. Autentica no Docker Hub com as credenciais dos Secrets
      - name: Login Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      # 6. Constrói a imagem Docker e envia para o Docker Hub
      - name: Build and push imagem
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ secrets.DOCKER_USERNAME }}/app_users:latest

      # 7. Conecta na EC2 via SSH e atualiza o container
      - name: Deploy na EC2
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            # Baixa a nova versão da imagem
            docker pull ${{ secrets.DOCKER_USERNAME }}/app_users:latest
            # Para o container atual (|| true evita erro se não existir)
            docker stop app_users || true
            # Remove o container antigo
            docker rm app_users || true
            # Sobe o novo container
            docker run -d -p 80:5000 --name app_users \
              ${{ secrets.DOCKER_USERNAME }}/app_users:latest
```

### Entendendo cada step

**`actions/checkout@v4`:** faz o download do código do repositório para a máquina do runner. Sem isso, os steps seguintes não teriam o código para trabalhar.

**`actions/setup-python@v3`:** instala o Python na versão especificada. O runner tem Python disponível, mas essa action garante a versão correta.

**`docker/login-action@v3`:** autentica no Docker Hub. Os steps seguintes podem fazer push de imagens porque já estão autenticados.

**`docker/build-push-action@v5`:** executa `docker build` e `docker push` em um único step, com otimizações de cache.

**`appleboy/ssh-action@master`:** estabelece uma conexão SSH com a EC2 usando a chave privada dos Secrets e executa os comandos do `script`.

---

## Verificando a Execução

Após fazer push para a branch `main`, acesse a aba **Actions** no GitHub. Você verá o workflow aparecendo com um ícone de carregamento.

Clique no workflow para ver os logs de cada step em tempo real:

```
✅ Checkout código       (2s)
✅ Set up Python         (5s)
✅ Install dependencies  (30s)
✅ Lint                  (3s)
✅ Login Docker Hub      (4s)
✅ Build and push imagem (45s)
✅ Deploy na EC2         (15s)
```

Se algum step falhar (❌), clique nele para ver o log completo com a mensagem de erro.

---

## Testando o Deploy

Com o pipeline funcionando, teste o fluxo completo:

```bash
# 1. Faça uma mudança no código
echo "# teste de CD" >> app.py

# 2. Commit e push
git add .
git commit -m "test: verificar pipeline de CD"
git push origin main

# 3. Acesse a aba Actions e veja o pipeline executar

# 4. Após o pipeline verde, acesse a aplicação
curl http://18.231.250.104
```

---

## Troubleshooting Comum

**Pipeline falha no step de SSH:**
- Verifique se o Secret `EC2_SSH_KEY` contém a chave completa (incluindo BEGIN e END)
- Confirme que a porta 22 está aberta no Security Group para qualquer IP (`0.0.0.0/0`) ou pelo menos para os IPs da GitHub (`140.82.112.0/20`)

**Permission denied no Docker:**
- Verifique se o usuário `deploy-python` está no grupo `docker`: `groups deploy-python`
- Se acabou de adicionar ao grupo, faça logout e login na sessão SSH

**Imagem não encontrada na EC2:**
- Verifique se o Docker Hub username no Secret é idêntico ao do YAML
- Confirme que o `docker login` foi bem-sucedido no step anterior

**Container não sobe:**
- Conecte-se à EC2 e verifique os logs: `docker logs app_users`
