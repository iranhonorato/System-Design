# AWS EC2: Servidores na Nuvem

**Pré-requisito:** [../fundamentos/01_o_que_e_infraestrutura.md](../fundamentos/01_o_que_e_infraestrutura.md) e os capítulos de Docker, até [../../02-containers/docker/04_compose_e_producao.md](../../02-containers/docker/04_compose_e_producao.md).

> Segue o *Amazon EC2 User Guide*. Reescreve e amplia o antigo `aws.md` do repositório Arquitetura de Software, no formato desta trilha.

---

## Por que este arquivo existe

Até aqui, tudo rodou na sua máquina: containers, o Compose com API, Postgres e Redis, e até um cluster Kubernetes inteiro dentro do Docker. Mas uma aplicação só serve a alguém quando roda em um servidor **acessível pela internet, ligado 24 horas por dia**. O [capítulo de fundamentos](../fundamentos/01_o_que_e_infraestrutura.md) mostrou que, hoje, esse servidor quase sempre é alugado de um provedor de nuvem.

Este capítulo é o primeiro contato prático com a AWS. A pergunta central é: *como criar um servidor na nuvem, entrar nele com segurança e colocar o sistema do capítulo de Compose para rodar ali?* Ao longo do caminho aparecem as peças (regiões, AMIs, tipos de instância, pares de chaves, Security Groups, discos) que os capítulos seguintes, de [VPC](02_vpc_redes_na_aws.md) e [IAM](03_iam_identidades_e_permissoes.md), aprofundam.

---

## 1. O que é o EC2

O **EC2** (*Elastic Compute Cloud*) é o serviço da AWS que fornece **máquinas virtuais**, chamadas **instâncias**. Em termos do [capítulo de virtualização](../fundamentos/02_virtualizacao_containers_e_iac.md), uma instância EC2 é uma VM rodando sobre o hypervisor da AWS (o *Nitro*), em um servidor físico de um data center da Amazon. É o modelo **IaaS** puro: a AWS entrega a máquina, e tudo do sistema operacional para cima é seu.

O "Elastic" do nome indica que a capacidade é **alugada sob demanda**: você cria uma instância em minutos, aumenta ou diminui o tamanho dela e a desliga quando não precisa mais, pagando pelo tempo de uso.

Usos típicos de uma instância:

- hospedar uma API ou um site;
- rodar containers com Docker ou Docker Compose;
- servir de node para um cluster Kubernetes (os nodes do EKS são instâncias EC2);
- executar tarefas de processamento pesadas e temporárias.

---

## 2. As cinco decisões ao criar uma instância

Criar uma instância é responder a cinco perguntas. Vale conhecê-las antes de abrir o console, porque cada uma tem consequências de custo e de segurança.

```
┌─────────────────────────────────────────────────────────────────────┐
│  ① ONDE?          Região (sa-east-1, us-east-1...)                  │
│  ② QUAL SISTEMA?  AMI (Ubuntu, Amazon Linux, Windows...)            │
│  ③ QUAL TAMANHO?  Tipo de instância (t3.micro, m7i.large...)        │
│  ④ COMO ENTRAR?   Par de chaves (SSH)                               │
│  ⑤ QUEM ACESSA?   Security Group (firewall)                         │
│                   + disco (EBS) e rede (VPC/sub-rede)               │
└─────────────────────────────────────────────────────────────────────┘
```

### ① Região

Uma **região** é uma área geográfica com vários data centers da AWS: `sa-east-1` (São Paulo), `us-east-1` (Virgínia do Norte), `eu-west-1` (Irlanda). A escolha afeta três coisas:

- **Latência:** quanto mais perto dos usuários, mais rápidas as respostas. Para usuários no Brasil, São Paulo costuma responder bem mais rápido que os Estados Unidos.
- **Preço:** os preços variam por região. São Paulo é, em geral, mais cara que `us-east-1`.
- **Leis e dados:** algumas regras de proteção de dados exigem que eles fiquem no país.

Recursos de uma região **não aparecem** nas outras. Um erro clássico de iniciante é criar uma instância em uma região, trocar a região no canto do console sem perceber e concluir que a instância "sumiu".

Cada região é dividida em **zonas de disponibilidade** (`sa-east-1a`, `sa-east-1b`...), data centers fisicamente separados. O [capítulo de VPC](02_vpc_redes_na_aws.md) mostra por que isso importa para a disponibilidade.

### ② AMI: o sistema operacional

Uma **AMI** (*Amazon Machine Image*) é a imagem a partir da qual o disco da instância é criado: o sistema operacional e, opcionalmente, software pré-instalado. A ideia é a mesma de uma imagem Docker ([capítulo de imagens](../../02-containers/docker/02_imagens_e_dockerfile.md)), só que para uma máquina inteira.

| AMI | Usuário padrão para SSH | Observação |
|---|---|---|
| **Ubuntu Server** | `ubuntu` | A mais usada nesta trilha |
| **Amazon Linux 2023** | `ec2-user` | Otimizada pela AWS; substitui o Amazon Linux 2 (fim de suporte em 2026) |
| **Debian** | `admin` | |
| **Rocky Linux / AlmaLinux** | `rocky` / `ec2-user` | Compatíveis com RHEL |
| **Red Hat (RHEL)** | `ec2-user` | Licença paga, inclusa no preço |

O usuário padrão importa porque é com ele que você entra por SSH. Tentar `ssh ubuntu@...` em uma instância Amazon Linux é um dos motivos mais comuns de *Permission denied*.

### ③ Tipo de instância: o tamanho da máquina

O nome de um tipo de instância segue um padrão:

```
  m  7  i  .  large
  │  │  │      └── tamanho: nano, micro, small, medium, large, xlarge, 2xlarge...
  │  │  └───────── variação: i = Intel, g = Graviton (ARM), a = AMD, d = disco local...
  │  └──────────── geração: quanto maior, mais nova (e normalmente melhor custo-benefício)
  └─────────────── família: o perfil de uso
```

| Família | Perfil | Uso típico |
|---|---|---|
| **t** (t3, t4g) | Uso geral, **burstable** | Desenvolvimento, sites e APIs com pouco tráfego |
| **m** (m7i, m7g) | Uso geral, equilibrado | Aplicações web em produção |
| **c** (c7i, c7g) | Mais CPU | Processamento, compressão, servidores de jogos |
| **r** (r7i, r7g) | Mais memória | Bancos de dados, caches grandes |
| **g / p** | GPU | Machine learning, vídeo |

Os tamanhos mais comuns da família `t3`, para começar:

| Tipo | vCPUs | RAM | Uso |
|---|---|---|---|
| `t3.nano` | 2 | 0,5 GB | Testes |
| `t3.micro` | 2 | 1 GB | Estudo, APIs pequenas |
| `t3.small` | 2 | 2 GB | O mínimo confortável para o Compose com Postgres e Redis |
| `t3.medium` | 2 | 4 GB | Aplicações pequenas em produção |

> **Confusão comum:** "uma `t3.micro` tem 2 vCPUs, então aguenta usar 2 núcleos o tempo todo". ✅ **Mais preciso:** a família `t` é **burstable** (de *burst*, rajada). A instância tem uma **linha de base** de CPU bem menor que o total (na `t3.micro`, cerca de 10% de cada vCPU) e acumula **créditos de CPU** enquanto fica abaixo dela. Quando precisa de mais, gasta os créditos e pode usar até 100% por um tempo. Isso é perfeito para cargas que ficam ociosas a maior parte do tempo e têm picos curtos. Para uma carga que usa CPU **continuamente** (um build pesado, um processamento em lote), os créditos acabam. A partir daí, no modo padrão das `t3` (*unlimited*), a AWS cobra pelo uso extra; no modo *standard*, a instância fica limitada à linha de base e tudo fica lento. Se o uso de CPU é sempre alto, uma família `m` ou `c` costuma sair mais barata e previsível.

### ④ Par de chaves

Para entrar na instância por SSH, a AWS usa **criptografia de chave pública** em vez de senha. Na criação, você escolhe ou cria um **par de chaves**:

```
┌──────────────────────────────────────────────────────────────────┐
│                       PAR DE CHAVES SSH                          │
│                                                                  │
│   Chave privada (.pem)               Chave pública               │
│   ┌──────────────────────┐           ┌──────────────────────┐    │
│   │ fica com você        │           │ fica no servidor,    │    │
│   │ NUNCA compartilhe    │           │ em ~/.ssh/           │    │
│   │                      │           │ authorized_keys      │    │
│   └──────────────────────┘           └──────────────────────┘    │
│                                                                  │
│  A pública é a fechadura: qualquer um pode vê-la.                │
│  A privada é a chave: só quem a tem consegue abrir.              │
└──────────────────────────────────────────────────────────────────┘
```

A AWS grava a chave pública na instância e entrega a chave privada (o arquivo `.pem`) para download **uma única vez**. Se você perder esse arquivo, não há como baixá-lo de novo, e recuperar o acesso exige procedimentos trabalhosos (como desanexar o disco e montá-lo em outra instância).

### ⑤ Security Group

Um **Security Group** é o firewall da instância. Por padrão, **todo tráfego de entrada é bloqueado**, e você abre explicitamente o que for necessário:

| Porta | Protocolo | Uso | Origem recomendada |
|---|---|---|---|
| 22 | TCP | SSH | **Só o seu IP** (`200.10.20.30/32`) |
| 80 | TCP | HTTP | `0.0.0.0/0` (qualquer um), se for um site público |
| 443 | TCP | HTTPS | `0.0.0.0/0` |
| 5432 | TCP | PostgreSQL | **Nunca** a internet; só outros Security Groups da aplicação |

O capítulo de [VPC](02_vpc_redes_na_aws.md) trata os Security Groups em detalhe, inclusive como um Security Group pode referenciar outro.

> **Confusão comum:** "deixar a porta 22 aberta para `0.0.0.0/0` não tem problema, porque sem a chave `.pem` ninguém entra". ✅ **Mais preciso:** a chave de fato impede o login, mas a porta aberta continua sendo uma superfície de ataque. Minutos depois de criada, qualquer instância com SSH aberto para a internet começa a receber milhares de tentativas automáticas de login por hora, vindas de robôs que varrem todos os IPs da AWS. Uma falha de segurança no próprio servidor SSH, uma senha habilitada por engano ou uma chave vazada passam a ser exploráveis por qualquer pessoa no mundo. Restringir a origem ao seu IP elimina quase todo esse ruído. Melhor ainda é não abrir a porta 22, usando o Session Manager descrito no capítulo de [VPC](02_vpc_redes_na_aws.md) (seção 9).

---

## 3. Criando a instância, passo a passo

No console da AWS, em **EC2 → Instances → Launch instances**:

1. **Name:** `loja-api`.
2. **AMI:** *Ubuntu Server 24.04 LTS* (64-bit x86).
3. **Instance type:** `t3.small`.
4. **Key pair:** *Create new key pair* → nome `loja-chave`, tipo *ED25519*, formato `.pem`. O download começa na hora; guarde o arquivo.
5. **Network settings:** deixe a VPC padrão (o [próximo capítulo](02_vpc_redes_na_aws.md) explica o que ela é) e, em *Firewall (security groups)*, crie um novo com:
   - SSH (22) com origem **My IP**;
   - HTTP (80) com origem **Anywhere**.
6. **Configure storage:** 20 GiB, tipo `gp3`.
7. **Launch instance.**

Em menos de um minuto, a instância aparece com o estado `Running`, um **IP público** e um **DNS público**.

### Os estados de uma instância

```
            launch
              │
              ▼
          ┌────────┐   stop    ┌─────────┐
          │running │ ────────▶ │ stopped │   ← não cobra CPU/RAM,
          │        │ ◀──────── │         │     mas COBRA o disco (EBS)
          └───┬────┘   start   └────┬────┘
              │ terminate           │ terminate
              ▼                     ▼
          ┌──────────────────────────────┐
          │ terminated                   │   ← a instância deixa de existir;
          └──────────────────────────────┘     o disco raiz é apagado
```

| Ação | O que acontece | Cobrança |
|---|---|---|
| **Reboot** | Reinicia o sistema operacional, na mesma máquina física | Continua |
| **Stop** | Desliga a VM; o disco EBS é preservado | Para a de computação; o disco continua cobrado |
| **Start** | Liga de novo, possivelmente em **outro** servidor físico | Volta a cobrar |
| **Terminate** | Destrói a instância; por padrão, apaga o disco raiz | Para tudo |

`Terminate` é irreversível. Para instâncias importantes, ative a opção **termination protection**, que exige desligá-la antes de permitir a destruição.

---

## 4. Conectando via SSH

### 4.1 Permissões da chave

O cliente SSH se recusa a usar uma chave privada que outros usuários do computador possam ler. No Linux e no Mac:

```bash
chmod 400 ~/Downloads/loja-chave.pem    # leitura só para você
```

No Windows, o OpenSSH do PowerShell geralmente aceita a chave na pasta do seu usuário. Se reclamar de permissões (*UNPROTECTED PRIVATE KEY FILE*), remova o acesso dos outros usuários nas propriedades do arquivo (*Segurança → Avançado → Desabilitar herança*) ou com `icacls`.

### 4.2 O comando

```bash
ssh -i ~/Downloads/loja-chave.pem ubuntu@18.231.250.104
```

| Parte | Significado |
|---|---|
| `ssh` | O cliente SSH |
| `-i ~/Downloads/loja-chave.pem` | Usa este arquivo como identidade (a chave privada) |
| `ubuntu` | O usuário no servidor (o padrão da AMI, seção 2) |
| `18.231.250.104` | O IP público da instância (o DNS público também funciona) |

Para não digitar tudo isso sempre, crie uma entrada em `~/.ssh/config`:

```text
Host loja
    HostName 18.231.250.104
    User ubuntu
    IdentityFile ~/Downloads/loja-chave.pem
```

E a partir daí basta `ssh loja`.

### 4.3 A primeira conexão

Na primeira vez, o SSH ainda não conhece o servidor e pede confirmação:

```
The authenticity of host '18.231.250.104' can't be established.
ED25519 key fingerprint is SHA256:Xk3p...
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

Ao responder `yes`, o SSH grava essa **impressão digital** em `~/.ssh/known_hosts`. Nas próximas conexões, ele confere se o servidor continua apresentando a mesma. Se um dia ela mudar, o SSH exibe um aviso grande (`WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!`) e se recusa a conectar, porque isso pode indicar que alguém está se passando pelo servidor. Na AWS, a explicação mais comum é inofensiva: você destruiu a instância e criou outra que recebeu o mesmo IP. Nesse caso, remova a linha antiga com `ssh-keygen -R 18.231.250.104`.

### 4.4 Lendo o prompt

```
ubuntu@ip-172-31-9-94:~$
  │        │          │ └── $ = usuário comum (# seria root)
  │        │          └──── ~ = diretório home (/home/ubuntu)
  │        └─────────────── hostname: derivado do IP PRIVADO da instância
  └──────────────────────── usuário
```

Repare que o hostname contém `172.31.9.94`, e não o IP público usado no `ssh`. Esse é o IP **privado** da instância, dentro da rede da AWS. O capítulo de [VPC](02_vpc_redes_na_aws.md) começa exatamente por esse detalhe.

Ao entrar, o Ubuntu mostra um resumo da máquina:

```
System load:  0.08               Processes:             98
Usage of /:   18.4% of 19.20GB   Users logged in:       0
Memory usage: 21%                IPv4 address for ens5: 172.31.9.94
```

---

## 5. IP público, DNS e Elastic IP

Uma instância na VPC padrão tem dois endereços externos que apontam para a mesma máquina:

- **IP público:** `18.231.250.104`;
- **DNS público:** `ec2-18-231-250-104.sa-east-1.compute.amazonaws.com` (o IP e a região, escritos como nome).

> **Confusão comum:** "o IP público da instância muda toda vez que ela reinicia". ✅ **Mais preciso:** um **reboot** mantém o IP, porque a instância continua na mesma máquina física. O que troca o IP é **stop seguido de start**: ao parar, a AWS devolve o IP ao seu estoque, e ao ligar de novo a instância recebe outro (e talvez vá para outro servidor físico). O IP **privado**, por outro lado, é mantido mesmo após stop e start. Para ter um IP público que não muda, associe um **Elastic IP**: um endereço reservado na sua conta, que você liga a uma instância e pode mover para outra. Desde 2024, a AWS cobra por todo IPv4 público, em uso ou não. Um Elastic IP reservado e esquecido gera cobrança todo mês. Em produção, o mais comum é nem usar IP fixo na instância: os usuários acessam um nome de domínio que aponta para um **load balancer**, e as instâncias atrás dele podem mudar à vontade.

---

## 6. Preparando o servidor

### 6.1 Comandos essenciais do Linux

Dentro da instância, você está em um Linux comum. Os comandos que mais aparecem:

| Comando | O que faz |
|---|---|
| `ls -la` | Lista arquivos com detalhes (permissões, dono, tamanho) |
| `cd pasta` / `cd ..` / `cd ~` | Entra em uma pasta / sobe um nível / vai para o home |
| `pwd` | Mostra o diretório atual |
| `cat arquivo` / `tail -f arquivo.log` | Mostra um arquivo / acompanha um log em tempo real |
| `sudo apt update && sudo apt upgrade -y` | Atualiza a lista de pacotes e instala as atualizações |
| `sudo apt install -y pacote` | Instala um pacote |
| `df -h` / `free -h` | Uso de disco / uso de memória |
| `ps aux` / `top` | Processos em execução / uso de recursos em tempo real |
| `curl ifconfig.me` | Mostra o IP público visto de fora |

Os serviços do sistema (Docker, Nginx, o próprio SSH) são gerenciados pelo **systemd**:

```bash
sudo systemctl status docker      # estado do serviço
sudo systemctl restart docker     # reinicia
sudo systemctl enable docker      # inicia automaticamente no boot
journalctl -u docker -f           # logs do serviço
```

### 6.2 Instalando o Docker

São os mesmos passos do [capítulo de arquitetura do Docker](../../02-containers/docker/01_arquitetura_e_isolamento.md):

```bash
sudo apt update && sudo apt upgrade -y
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker ubuntu
exit                               # saia e entre de novo para o grupo valer
```

```bash
ssh loja
docker run --rm hello-world
# Hello from Docker!
```

O script oficial também instala o plugin do Compose (`docker compose`) e deixa o serviço habilitado no boot.

---

## 7. Rodando o sistema na instância

Com o Docker instalado, colocar o sistema do capítulo de [Compose](../../02-containers/docker/04_compose_e_producao.md) no ar é o mesmo procedimento da sua máquina:

```bash
git clone https://github.com/seu-usuario/minha-loja.git
cd minha-loja

# O .env não está no Git (de propósito): crie-o no servidor
cat > .env <<'EOF'
POSTGRES_USER=app
POSTGRES_PASSWORD=uma-senha-forte-de-verdade
POSTGRES_DB=loja
EOF

docker compose up -d --build
docker compose ps
```

Para expor a API na porta 80 (a que foi aberta no Security Group), troque o `ports:` do serviço `api` no `docker-compose.yml` de `"8000:8000"` para `"80:8000"`: porta 80 da instância, porta 8000 do container. Da sua máquina:

```bash
curl http://18.231.250.104/
# {"mensagem":"Olá de dentro do container!"}
```

E se a instância reiniciar? O serviço do Docker sobe no boot (seção 6.2), e a política `restart: unless-stopped` do Compose faz os containers subirem junto. É o mesmo mecanismo do capítulo de Compose, agora protegendo um servidor real.

Repare no que este fluxo ainda tem de manual: alguém precisou entrar por SSH, clonar o repositório, criar o `.env` e rodar o Compose. Cada nova versão exigiria repetir parte disso. Automatizar esse caminho é o objetivo da parte de [entrega contínua](../../03-entrega-continua/SUMARIO.md), cujo pipeline de CD faz deploy exatamente nesta instância.

---

## 8. User data: configurando a instância na criação

Os passos da seção 6.2 podem ser executados **automaticamente** na primeira inicialização. No campo **User data** (em *Advanced details*, na criação da instância), você cola um script, que o **cloud-init** executa como root no primeiro boot:

```bash
#!/bin/bash
apt-get update -y
apt-get upgrade -y
curl -fsSL https://get.docker.com | sh
usermod -aG docker ubuntu
```

Quando você entra pela primeira vez, o Docker já está instalado. O log da execução fica em `/var/log/cloud-init-output.log`, útil para investigar quando algo não saiu como esperado.

É uma versão simples da ideia de **Infraestrutura como Código** do [capítulo de virtualização](../fundamentos/02_virtualizacao_containers_e_iac.md): a configuração da máquina vira um arquivo versionável, e uma instância nova fica igual à anterior sem ninguém repetir passos à mão. O passo seguinte nessa direção é criar a própria instância com Terraform, e o [capítulo de VPC](02_vpc_redes_na_aws.md) mostra um exemplo.

---

## 9. Discos: EBS e instance store

O disco de uma instância é, quase sempre, um volume **EBS** (*Elastic Block Store*): um disco **de rede**, que existe independentemente da máquina física. É por isso que uma instância pode ser parada e ligada em outro servidor físico mantendo os dados.

| | EBS | Instance store |
|---|---|---|
| **O que é** | Disco de rede, separado da máquina | Disco físico no próprio servidor |
| **Sobrevive a stop/start?** | Sim | **Não**: os dados se perdem |
| **Tipos de instância** | Todos | Só alguns (com `d` no nome, como `m7gd`) |
| **Uso** | Disco do sistema, bancos, dados permanentes | Cache e dados temporários que precisam de muita velocidade |

Um volume EBS pertence a **uma zona de disponibilidade**, e só pode ser conectado a instâncias da mesma zona, o mesmo limite que o capítulo de [Kubernetes: configuração, estado e segurança](../../02-containers/kubernetes/03_configuracao_estado_e_seguranca.md) mostra para os PVCs.

**Snapshots** são cópias de um volume EBS, guardadas no S3 e cobradas pelo espaço usado. Eles servem de backup e também de modelo: é possível criar um volume novo, em outra zona, a partir de um snapshot. Para o banco do Compose rodando na instância, um snapshot diário do volume (agendado com o *Amazon Data Lifecycle Manager*) é um backup simples. Mesmo assim, como o capítulo de [volumes](../../02-containers/docker/03_volumes_e_redes.md) avisou, um backup lógico do banco (`pg_dump`) continua sendo recomendável.

---

## 10. Custos e cuidados

### Modelos de preço

| Modelo | Como funciona | Quando usar |
|---|---|---|
| **On-Demand** | Paga por segundo de uso, sem compromisso | Estudo, cargas imprevisíveis, começo de um projeto |
| **Savings Plans / Reserved** | Compromisso de uso por 1 ou 3 anos, com desconto grande | Servidores que ficam ligados o tempo todo |
| **Spot** | Usa capacidade ociosa da AWS com desconto alto, mas a AWS pode retomar a instância com 2 minutos de aviso | Tarefas tolerantes a interrupção: builds de CI, processamento em lote |

As regras do **nível gratuito** (*Free Tier*) para contas novas mudaram em 2025, passando a ser baseadas em créditos por um período inicial. Confira as condições atuais na página oficial antes de contar com elas.

### Como não ter surpresas na fatura

- **Crie um alerta de orçamento** no primeiro dia (*AWS Budgets*: por exemplo, avisar por e-mail ao passar de US$ 10 no mês).
- **Pare** (stop) instâncias de estudo quando não estiver usando, e **termine** as que não vai mais usar. Lembre que discos de instâncias paradas continuam cobrados.
- **Libere** Elastic IPs que não estão em uso.
- Confira **todas as regiões** de vez em quando: recursos esquecidos em outra região continuam cobrando.

### Segurança básica

- Guarde o `.pem` fora de repositórios (e no `.gitignore` por precaução) e com permissão `400`.
- SSH só a partir do seu IP, ou nenhum SSH aberto (Session Manager).
- Mantenha o sistema atualizado (`sudo apt update && sudo apt upgrade -y`), ou ative as atualizações automáticas de segurança (`unattended-upgrades`, já instalado no Ubuntu).
- Não use o usuário root da conta AWS no dia a dia. O capítulo de [IAM](03_iam_identidades_e_permissoes.md) explica o que usar no lugar.
- Para automações, como o pipeline de CD, crie um **usuário específico no servidor**, com só o que ele precisa. A parte de [CD](../../03-entrega-continua/cd/03_pipeline_de_cd_na_pratica.md) mostra como.

### Visão geral da conexão

```
Seu computador
      │  SSH (porta 22) com a chave privada .pem
      ▼
Internet
      │
      ▼
Security Group  ── permite 22 só do seu IP; 80 de qualquer um
      │
      ▼
Instância EC2 (Ubuntu)
      │  IP privado: 172.31.9.94   IP público: 18.231.250.104
      ▼
Docker ──▶ containers da API, Postgres e Redis (Compose)
```

---

## Próximo passo

A instância deste capítulo funciona, mas está toda em uma **sub-rede pública** da VPC padrão: o banco de dados roda na mesma máquina exposta à internet, protegido apenas por não ter a porta publicada no Security Group. E o IP privado `172.31.9.94` sugere que existe uma rede inteira, que ainda não foi explicada, em volta da instância.

O [02_vpc_redes_na_aws.md](02_vpc_redes_na_aws.md) mostra como essa rede funciona e como desenhar uma em que o banco fique completamente fora do alcance da internet.
