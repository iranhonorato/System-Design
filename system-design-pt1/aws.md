# Máquinas Virtuais na AWS (EC2)

## 1. O que é o EC2?

O **EC2 (Elastic Compute Cloud)** é o serviço da AWS que fornece servidores virtuais na nuvem. Em vez de comprar e manter hardware físico, você aluga capacidade computacional por hora ou segundo, configura o tipo de máquina que precisa e tem acesso a um servidor Linux ou Windows rodando em um data center da Amazon.

**"Elastic"** no nome não é por acaso — você pode aumentar ou diminuir a capacidade conforme a demanda, sem precisar comprar hardware.

### O que você pode fazer com uma instância EC2:
- Hospedar aplicações web e APIs
- Rodar bancos de dados
- Executar pipelines de processamento de dados
- Servir de base para containers Docker
- Fazer deploy de aplicações em produção

### Informações da instância usada neste guia:
```
Instância: aluno_13
DNS IPv4:  ec2-18-231-250-104.sa-east-1.compute.amazonaws.com
IP público: 18.231.250.104
Região:     sa-east-1 (São Paulo)
```

---

## 2. Conceitos Fundamentais

### 2.1 SSH e a Chave `.pem`

Para acessar um servidor Linux remotamente com segurança, usamos o protocolo **SSH (Secure Shell)** — uma conexão criptografada que garante que ninguém intercepte seus comandos.

Em vez de usar senha (que poderia ser roubada ou adivinhada), o EC2 usa **autenticação por chave criptográfica**:

```
┌─────────────────────────────────────────────────────┐
│              PAR DE CHAVES SSH                       │
│                                                      │
│  Chave Privada (.pem)    Chave Pública               │
│  ┌──────────────────┐    ┌──────────────────┐        │
│  │  Fica com você   │    │  Fica no servidor │        │
│  │  NUNCA compartilhe│   │  (authorized_keys)│        │
│  └──────────────────┘    └──────────────────┘        │
│                                                      │
│  Como uma fechadura: a chave pública é a fechadura   │
│  (qualquer um pode ver), a privada é a chave         │
│  (só você tem).                                      │
└─────────────────────────────────────────────────────┘
```

A AWS gera o par de chaves no momento da criação da instância e fornece o arquivo `.pem` (chave privada) para download — **apenas uma vez**. Se você perder esse arquivo, perderá o acesso à instância.

### 2.2 IP Público vs. DNS Público

Sua instância EC2 tem dois endereços que apontam para a mesma máquina:

- **IP Público:** `18.231.250.104` — endereço numérico direto
- **DNS Público:** `ec2-18-231-250-104.sa-east-1.compute.amazonaws.com` — nome legível

Ambos funcionam para conexão SSH e para acessar serviços web. O DNS é mais descritivo e inclui a região (`sa-east-1` = São Paulo).

> **Atenção:** em instâncias EC2 sem Elastic IP, o IP público **muda toda vez que a instância é reiniciada**. Para produção, associe um Elastic IP (um IP fixo) à sua instância.

### 2.3 Usuário Padrão por Sistema Operacional

Cada imagem de SO no EC2 tem um usuário padrão pré-configurado:

| Sistema Operacional | Usuário Padrão |
|---|---|
| Ubuntu | `ubuntu` |
| Amazon Linux 2 | `ec2-user` |
| Debian | `admin` |
| Rocky Linux / AlmaLinux | `rocky` / `ec2-user` |
| Red Hat (RHEL) | `ec2-user` |

Neste guia usamos **Ubuntu** → usuário `ubuntu`.

### 2.4 Security Groups (Firewall da AWS)

Um **Security Group** é um firewall virtual que controla o tráfego de entrada e saída da sua instância. Por padrão, todo tráfego de entrada é bloqueado — você precisa abrir explicitamente as portas necessárias.

Portas comuns:

| Porta | Protocolo | Uso |
|---|---|---|
| 22 | TCP | SSH (acesso remoto) |
| 80 | TCP | HTTP (sites) |
| 443 | TCP | HTTPS (sites seguros) |
| 5432 | TCP | PostgreSQL |
| 3306 | TCP | MySQL |

---

## 3. Conectando via SSH

### 3.1 Ajustando permissões da chave (Windows PowerShell)

Na primeira vez, o Windows pode reclamar que a chave `.pem` tem permissões muito abertas. Se necessário:

```powershell
# Verificar se o arquivo .pem está acessível
dir chave-linux.pem
```

### 3.2 Navegando até a pasta da chave

```powershell
cd C:\Users\seu-usuario\Downloads
# ou onde você salvou o arquivo .pem
```

### 3.3 Conectando à instância

```powershell
ssh -i "chave-linux.pem" ubuntu@18.231.250.104
```

Anatomia do comando:

| Parte | Significado |
|---|---|
| `ssh` | Protocolo de conexão segura |
| `-i "chave-linux.pem"` | Usa este arquivo como chave de identidade |
| `ubuntu` | Usuário no servidor remoto |
| `18.231.250.104` | Endereço IP da instância |

**Alternativa com DNS (equivalente):**
```powershell
ssh -i "chave-linux.pem" ubuntu@ec2-18-231-250-104.sa-east-1.compute.amazonaws.com
```

### 3.4 Primeira conexão: verificação de identidade

Na primeira vez conectando, o SSH não conhece o servidor e pergunta:

```
The authenticity of host '18.231.250.104' can't be established.
ECDSA key fingerprint is SHA256:xxxxxxxxxxx.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

Digite `yes`. O SSH salva a "impressão digital" do servidor no arquivo `~/.ssh/known_hosts`. Nas próximas conexões, ele verifica automaticamente — se a impressão mudar (o que pode indicar um ataque), o SSH alerta.

### 3.5 Conexão bem-sucedida

Seu terminal muda de:
```
PS C:\Users\seu-usuario>
```
Para:
```
ubuntu@ip-172-31-9-94:~$
```

Você agora está executando comandos em São Paulo, no servidor da AWS. Tudo que você digitar roda lá — não no seu computador.

---

## 4. Entendendo o Prompt e as Informações do Sistema

```
ubuntu@ip-172-31-9-94:~$
  │        │          │ └── $ indica usuário comum (# seria root)
  │        │          └──── ~ é o diretório home (/home/ubuntu)
  │        └─────────────── ip-172-31-9-94 é o hostname interno da instância
  └──────────────────────── nome do usuário
```

Ao conectar, o sistema exibe informações úteis:

```
System information as of Mon Mar 30 14:22:18 UTC 2026

System load:  0.08               Processes:             98
Usage of /:   28.4% of 7.69GB   Users logged in:       0
Memory usage: 18%                IPv4 address for eth0: 172.31.9.94
Swap usage:   0%
```

| Campo | O que significa |
|---|---|
| System load | Uso médio da CPU nos últimos minutos (0.08 = praticamente ociosa) |
| Usage of / | Percentual do disco usado |
| Memory usage | RAM utilizada |
| IPv4 eth0 | IP interno da instância (privado, dentro da AWS) |

---

## 5. Comandos Essenciais no Linux

### Navegação e arquivos

```bash
# Ver arquivos na pasta atual
ls

# Ver com detalhes (permissões, tamanho, data)
ls -la

# Diretório atual
pwd

# Entrar em uma pasta
cd nome-da-pasta

# Voltar uma pasta
cd ..

# Ir para o diretório home
cd ~

# Criar pasta
mkdir nome-da-pasta

# Ver conteúdo de um arquivo
cat arquivo.txt

# Ver as últimas linhas de um arquivo (útil para logs)
tail -f arquivo.log
```

### Sistema

```bash
# Atualizar lista de pacotes disponíveis
sudo apt update

# Instalar atualizações pendentes
sudo apt upgrade -y

# Instalar um pacote
sudo apt install nome-do-pacote -y

# Ver processos em execução
ps aux

# Ver IP público da instância
curl ifconfig.me

# Ver uso de disco
df -h

# Ver uso de memória
free -h
```

### Gerenciamento de serviços

```bash
# Ver status de um serviço
sudo systemctl status nginx

# Iniciar um serviço
sudo systemctl start nginx

# Parar um serviço
sudo systemctl stop nginx

# Reiniciar um serviço
sudo systemctl restart nginx

# Fazer o serviço iniciar automaticamente na inicialização
sudo systemctl enable nginx
```

---

## 6. Fluxo Típico de Uso do EC2

```
1. CRIAR INSTÂNCIA
   └── Escolher AMI (Ubuntu, Amazon Linux, etc.)
   └── Escolher tipo (t3.micro, t3.medium, etc.)
   └── Configurar Security Group (abrir portas necessárias)
   └── Criar/selecionar par de chaves

2. CONECTAR VIA SSH
   └── ssh -i "chave.pem" ubuntu@IP

3. PREPARAR O AMBIENTE
   └── sudo apt update && sudo apt upgrade -y
   └── Instalar Docker, Node, Python, etc.

4. DEPLOY DA APLICAÇÃO
   └── Clonar repositório (git clone)
   └── Configurar variáveis de ambiente
   └── Subir containers (docker run / docker compose up)

5. ACESSAR A APLICAÇÃO
   └── http://IP-público:porta
```

---

## 7. Tipos de Instância EC2

A AWS oferece centenas de tipos de instância. Os mais comuns para aprendizado e projetos pequenos são da família **t** (burstable — podem usar mais CPU em picos):

| Tipo | vCPUs | RAM | Uso típico |
|---|---|---|---|
| t3.nano | 2 | 0.5 GB | Testes, demos |
| t3.micro | 2 | 1 GB | Free tier, dev |
| t3.small | 2 | 2 GB | Apps pequenas |
| t3.medium | 2 | 4 GB | Apps de médio porte |
| t3.large | 2 | 8 GB | Apps com mais memória |

> **Free Tier:** a AWS oferece 750 horas/mês de t3.micro (ou t2.micro) gratuitamente para novos usuários durante 12 meses. É suficiente para manter uma instância rodando continuamente.

---

## 8. IAM — Identity and Access Management

O **IAM** é o sistema de controle de acesso da AWS. Toda ação na AWS — criar instâncias, acessar S3, invocar Lambdas — é autorizada pelo IAM.

**Conceitos fundamentais:**

| Conceito | O que é |
|---|---|
| **User** | Identidade para pessoas (desenvolvedor, admin) |
| **Role** | Identidade para serviços (EC2 acessa S3, Lambda acessa DynamoDB) |
| **Policy** | Documento JSON que define permissões (o que pode fazer em quais recursos) |
| **Group** | Conjunto de Users com as mesmas políticas |

```
Exemplo de Policy — permite apenas leitura em um bucket S3 específico:
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:ListBucket"],
    "Resource": "arn:aws:s3:::meu-bucket/*"
  }]
}
```

**Regra de ouro do IAM: Princípio do Menor Privilégio.** Dê apenas as permissões estritamente necessárias. Uma instância EC2 que só lê do S3 não deve ter permissão de escrever ou deletar.

> **Nunca use credenciais de root para operações do dia a dia.** A conta root tem acesso irrestrito a tudo. Crie um usuário IAM com MFA habilitado para uso cotidiano.

---

## 9. Boas Práticas de Segurança

**Proteja o arquivo .pem:**
- Nunca compartilhe ou cometa em repositórios
- Adicione ao `.gitignore`
- Faça backup em local seguro

**Configure o Security Group corretamente:**
```
❌ ERRADO: Abrir porta 22 para 0.0.0.0/0 (qualquer IP pode tentar conectar)
✅ CERTO:  Abrir porta 22 apenas para o seu IP ou range corporativo
```

**Mantenha o sistema atualizado:**
```bash
# Rodar regularmente
sudo apt update && sudo apt upgrade -y
```

**Use usuários específicos por função:**
- `ubuntu`: acesso humano/administrativo
- `deploy`: acesso para automações e CI/CD (veremos no guia de CD)

**Não use a conta root diretamente:** o prefixo `sudo` já concede privilégios temporários quando necessário.

---

## 9. Arquitetura de Conexão

```
Seu Computador (Windows/Mac/Linux)
        │
        │  SSH com chave privada (.pem)
        │  Porta 22, tráfego criptografado
        ▼
Internet
        │
        ▼
AWS Security Group
        │  (filtra: apenas porta 22 do seu IP)
        ▼
Instância EC2 (Ubuntu)
        │  IP interno: 172.31.9.94
        │  IP externo: 18.231.250.104
        ▼
Aplicação / Docker / Banco de Dados
```
