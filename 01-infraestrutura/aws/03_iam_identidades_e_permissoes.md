# AWS IAM: Identidades e Permissões

**Pré-requisito:** [02_vpc_redes_na_aws.md](02_vpc_redes_na_aws.md), [01_ec2_servidores_na_nuvem.md](01_ec2_servidores_na_nuvem.md). Os conceitos de autenticação, autorização e menor privilégio vêm de [Arquitetura de Software: seguranca.md](https://github.com/iranhonorato/Arquitetura-de-Software/blob/main/system-design-pt1/seguranca.md).

> Segue o *AWS IAM User Guide* (em especial *Policy evaluation logic* e *Security best practices in IAM*) e a documentação de OIDC do GitHub Actions e do Amazon EKS.

---

## Por que este arquivo existe

O [01_ec2_servidores_na_nuvem.md](01_ec2_servidores_na_nuvem.md) apresentou o IAM em uma tabela de quatro linhas (User, Role, Policy, Group) e uma política de exemplo. Isso basta para começar, mas deixa de fora justamente o que causa os incidentes reais: chaves de acesso esquecidas em um repositório, uma política com `"*"` onde deveria haver um recurso específico, uma role que qualquer repositório do GitHub consegue assumir.

O IAM é o assunto deste capítulo porque, na AWS, **toda** ação é uma chamada de API: criar uma instância, ler um arquivo do S3, abrir uma sessão no Session Manager, baixar uma imagem do ECR. A VPC do [capítulo anterior](02_vpc_redes_na_aws.md) decide por onde os **pacotes** podem passar; o IAM decide quais **chamadas de API** são permitidas, e para quem. Um erro na VPC expõe um servidor; um erro no IAM pode expor a conta inteira.

A pergunta central: *como cada pessoa, cada servidor e cada pipeline recebe exatamente as permissões de que precisa, sem nenhuma senha ou chave guardada onde não deveria?*

---

## 1. As duas perguntas de toda requisição

Toda chamada à API da AWS, seja do console, da CLI, de um SDK ou do Terraform, passa por duas verificações, as mesmas que [Arquitetura de Software: seguranca.md](https://github.com/iranhonorato/Arquitetura-de-Software/blob/main/system-design-pt1/seguranca.md) separa:

1. **Autenticação: quem é você?** A requisição vem assinada com uma credencial, e a AWS verifica a assinatura e identifica o **principal**: o usuário, a role ou o serviço que está chamando.
2. **Autorização: você pode fazer isto?** A AWS reúne todas as **políticas** que se aplicam àquela requisição e decide se ela é permitida.

Para decidir, o IAM olha quatro elementos de cada requisição:

```
  aws s3 cp s3://loja-backups/2026-10-25.sql .

  Principal   arn:aws:iam::123456789012:role/servidor-backup   ← quem
  Ação        s3:GetObject                                     ← o quê
  Recurso     arn:aws:s3:::loja-backups/2026-10-25.sql         ← em quê
  Contexto    IP de origem, horário, MFA usado, região...      ← em que condições
```

Todo o resto do capítulo trata de como esses quatro elementos são escritos nas políticas e combinados na decisão.

Um comando útil para saber "quem eu sou" para a AWS neste momento:

```bash
aws sts get-caller-identity
```
```json
{
    "UserId": "AIDAEXEMPLO1234567890",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/ana"
}
```

---

## 2. O usuário root

Ao criar uma conta AWS, você recebe o **usuário root**: o e-mail e a senha usados no cadastro. Ele tem **acesso irrestrito e não limitável** a tudo na conta. Nenhuma política do IAM consegue restringi-lo dentro da própria conta.

Por isso, a regra é usá-lo **o mínimo possível**:

- ative **MFA** no root no primeiro dia;
- **nunca** crie chaves de acesso (access keys) para ele;
- use-o só para as poucas tarefas que exigem root (alterar dados da conta, fechá-la, alguns ajustes de cobrança);
- para todo o resto, inclusive administração, use identidades do IAM.

Uma analogia: o root é a escritura do imóvel. Você a guarda no cofre e só a tira de lá para vender a casa. Para entrar e sair todos os dias, usa a chave da porta.

---

## 3. As identidades: users, groups e roles

### 3.1 IAM User: uma identidade permanente

Um **IAM User** representa uma pessoa (ou, no passado, uma aplicação) com **credenciais de longo prazo**: uma senha para o console e, opcionalmente, **access keys** (um par `AccessKeyId` + `SecretAccessKey`) para a CLI e os SDKs. Essas credenciais valem até alguém as revogar.

### 3.2 IAM Group: permissões por função

Um **Group** é um conjunto de users que recebem as mesmas políticas. Em vez de dar permissões a cada pessoa, você as dá ao grupo `desenvolvedores` e adiciona ou remove pessoas dele. Groups não são identidades: não é possível "entrar como" um grupo.

### 3.3 IAM Role: uma identidade que se **assume**

Uma **Role** é uma identidade com permissões, mas **sem credenciais próprias**. Ninguém tem a senha de uma role. Em vez disso, uma identidade autorizada **assume** a role e recebe, em troca, **credenciais temporárias**, que expiram sozinhas (por padrão em 1 hora).

A analogia é o crachá de visitante de uma empresa. Você se identifica na portaria (autenticação), recebe um crachá que abre só certas portas (as permissões da role) e que deixa de funcionar no fim do dia (a expiração). Quem decide quem pode receber aquele crachá é uma regra da portaria: a **política de confiança** (*trust policy*) da role.

Toda role tem, portanto, **duas** políticas:

| Política | Pergunta que responde |
|---|---|
| **Trust policy** (política de confiança) | **Quem** pode assumir esta role? |
| **Permission policies** (políticas de permissão) | O que quem assumiu pode **fazer**? |

Roles são usadas por:

- **serviços da AWS**: uma EC2 que lê do S3, uma Lambda que grava no DynamoDB (seção 8);
- **sistemas externos**: o GitHub Actions publicando no ECR (seção 9);
- **outras contas**: uma conta de auditoria lendo logs desta conta;
- **pessoas**: no modelo recomendado hoje, pessoas também assumem roles (veja a seguir).

### 3.4 E as pessoas?

Para pessoas, a AWS recomenda hoje o **IAM Identity Center** (antigo AWS SSO) no lugar de IAM Users. Cada pessoa faz login com a identidade da empresa (Google Workspace, Microsoft Entra ID, Okta), e o Identity Center a deixa assumir roles nas contas em que ela trabalha, com credenciais temporárias:

```bash
aws configure sso          # configura uma vez
aws sso login              # abre o navegador para o login da empresa
aws s3 ls --profile dev    # usa credenciais temporárias, renovadas automaticamente
```

Quando alguém sai da empresa e sua conta corporativa é desativada, o acesso à AWS acaba junto. Nenhuma access key fica esquecida no notebook antigo dessa pessoa.

| Identidade | Credencial | Validade | Uso recomendado |
|---|---|---|---|
| **Root** | E-mail + senha (+ MFA) | Permanente | Quase nunca |
| **IAM User** | Senha e/ou access keys | Permanente, até revogar | Casos residuais (sistemas antigos que não suportam roles) |
| **Role** | Credenciais temporárias, obtidas ao assumir | Minutos a horas | Serviços, pipelines, outras contas e pessoas (via Identity Center) |

---

## 4. Políticas: a linguagem das permissões

### 4.1 A anatomia de uma política

Uma **política** é um documento JSON com uma ou mais **declarações** (*statements*). Cada declaração diz: **permito** (ou **nego**) **estas ações** **nestes recursos**, **sob estas condições**.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "LerBackupsDaLoja",
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::loja-backups/*",
      "Condition": {
        "IpAddress": { "aws:SourceIp": "203.0.113.0/24" }
      }
    }
  ]
}
```

| Campo | Significado |
|---|---|
| `Version` | A versão da **linguagem** de políticas. É sempre `"2012-10-17"`, não a data de hoje. |
| `Sid` | Um nome opcional para a declaração, para facilitar a leitura. |
| `Effect` | `Allow` (permitir) ou `Deny` (negar). |
| `Action` | As chamadas de API, no formato `serviço:Operação`. Aceita curingas: `s3:Get*`. |
| `Resource` | **Em quais** recursos, identificados por ARN. |
| `Condition` | **Em que condições**: IP de origem, uso de MFA, tags, região, horário. Opcional. |

A declaração acima se lê assim: "permitir baixar objetos do bucket `loja-backups`, mas só se a requisição vier da rede `203.0.113.0/24`". O CIDR é o mesmo da notação do capítulo [AWS: VPC](02_vpc_redes_na_aws.md).

### 4.2 ARN: o endereço de cada recurso

Todo recurso da AWS tem um **ARN** (*Amazon Resource Name*), um identificador único no formato:

```
arn:aws:serviço:região:conta:recurso

arn:aws:ec2:sa-east-1:123456789012:instance/i-0abc123def456
arn:aws:iam::123456789012:role/servidor-backup         ← IAM é global: sem região
arn:aws:s3:::loja-backups                              ← o bucket (S3: sem região nem conta)
arn:aws:s3:::loja-backups/*                            ← todos os objetos DENTRO do bucket
```

As duas últimas linhas parecem quase iguais, mas identificam **coisas diferentes**, e isso causa um dos erros mais comuns ao escrever políticas.

> **Confusão comum:** "para dar leitura em um bucket, basta `s3:GetObject` e `s3:ListBucket` em `arn:aws:s3:::meu-bucket/*`". ✅ **Mais preciso:** cada ação do S3 se aplica a um tipo de recurso específico. `s3:GetObject` age sobre um **objeto**, e por isso casa com `meu-bucket/*`. `s3:ListBucket` age sobre o **bucket** (listar o conteúdo é uma operação do bucket, não de um objeto), e só casa com `arn:aws:s3:::meu-bucket`, **sem** o `/*`. A política de exemplo do [01_ec2_servidores_na_nuvem.md](01_ec2_servidores_na_nuvem.md) tem esse problema: o download de um arquivo conhecido funciona, mas `aws s3 ls s3://meu-bucket` falha com `AccessDenied`, e a mensagem não diz qual parte da política está errada. A versão correta separa os dois recursos:
> ```json
> {
>   "Version": "2012-10-17",
>   "Statement": [
>     { "Effect": "Allow", "Action": "s3:ListBucket", "Resource": "arn:aws:s3:::meu-bucket" },
>     { "Effect": "Allow", "Action": "s3:GetObject",  "Resource": "arn:aws:s3:::meu-bucket/*" }
>   ]
> }
> ```
> A tabela *Actions, resources, and condition keys* da documentação de cada serviço mostra a qual tipo de recurso cada ação se aplica.

### 4.3 Exemplos que aparecem com frequência

**Exigir MFA para ações destrutivas.** Um `Deny` com condição bloqueia a ação mesmo que outra política a permita:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "SemMfaNaoApagaInstancias",
      "Effect": "Deny",
      "Action": ["ec2:TerminateInstances"],
      "Resource": "*",
      "Condition": {
        "BoolIfExists": { "aws:MultiFactorAuthPresent": "false" }
      }
    }
  ]
}
```

**Limitar a uma região.** Útil para impedir que recursos sejam criados por engano (ou por um atacante) em regiões que a empresa não usa:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "SoSaoPaulo",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": { "aws:RequestedRegion": "sa-east-1" }
      }
    }
  ]
}
```

(Em uma política real, os serviços globais, como IAM e CloudFront, precisam ser excluídos dessa negação com um `NotAction`, ou deixam de funcionar.)

---

## 5. Onde as políticas ficam

### 5.1 Baseadas em identidade e baseadas em recurso

Uma política pode estar presa a dois lugares diferentes:

- **Baseada em identidade** (*identity-based*): anexada a um user, group ou role. Diz o que **aquela identidade** pode fazer. Não tem o campo `Principal`, porque o principal é a própria identidade.
- **Baseada em recurso** (*resource-based*): anexada ao **recurso**, como um bucket S3, uma fila SQS ou uma chave KMS. Diz **quem** pode fazer o quê **com aquele recurso**, e por isso tem o campo `Principal`. A **trust policy** de uma role é uma política desse tipo: o recurso é a própria role.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ContaDeAuditoriaLeLogs",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::999988887777:root" },
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": ["arn:aws:s3:::loja-logs", "arn:aws:s3:::loja-logs/*"]
    }
  ]
}
```

Essa é uma **bucket policy** (baseada em recurso) que deixa outra conta, a `999988887777`, ler os logs. O `:root` no ARN não significa o usuário root daquela conta: significa "a conta como um todo", que ainda precisa dar a seus próprios usuários a permissão correspondente.

### 5.2 Gerenciadas pela AWS, gerenciadas por você e inline

As políticas baseadas em identidade vêm em três formas:

| Tipo | Quem mantém | Quando usar |
|---|---|---|
| **AWS managed** (ex.: `AmazonS3ReadOnlyAccess`) | A AWS, que atualiza quando surgem ações novas | Para começar e explorar; costumam ser amplas demais para produção |
| **Customer managed** | Você; reutilizável em várias identidades e versionada | O padrão em produção |
| **Inline** | Embutida em uma única identidade; some quando ela é apagada | Raramente: casos estritamente individuais |

### 5.3 Os limites acima das políticas

Duas camadas extras **restringem** o que as políticas podem conceder, sem conceder nada por si só:

- **Permissions boundary**: um teto aplicado a um user ou role específico. Mesmo que uma política dê `"Action": "*"`, a identidade só faz o que também estiver dentro do boundary. É usada para deixar equipes criarem suas próprias roles sem que possam criar uma role mais poderosa que elas mesmas.
- **SCP** (*Service Control Policy*): um teto aplicado a **contas inteiras**, a partir do AWS Organizations. Vale até para o usuário root das contas-membro. A política "só São Paulo" da seção 4.3 costuma ser aplicada como SCP.

---

## 6. Como a AWS decide: a lógica de avaliação

Com políticas em vários lugares, a AWS segue uma ordem fixa para decidir. Simplificada para o caso de uma única conta:

```
                     requisição chega
                            │
                            ▼
         ┌─────────────────────────────────────┐
         │ Alguma política tem um Deny          │── sim ──▶  NEGADO
         │ explícito que casa?                  │            (nada consegue reverter)
         └──────────────────┬──────────────────┘
                            │ não
                            ▼
         ┌─────────────────────────────────────┐
         │ SCPs e permissions boundary (se      │── não ──▶  NEGADO
         │ existirem) permitem a ação?          │
         └──────────────────┬──────────────────┘
                            │ sim (ou não existem)
                            ▼
         ┌─────────────────────────────────────┐
         │ Alguma política de identidade OU de  │── não ──▶  NEGADO
         │ recurso tem um Allow que casa?       │            (negação implícita)
         └──────────────────┬──────────────────┘
                            │ sim
                            ▼
                        PERMITIDO
```

Três regras resumem o diagrama:

1. **Tudo começa negado.** Sem um `Allow` que case, a resposta é não. Isso se chama **negação implícita**, e é o menor privilégio por padrão.
2. **Um `Deny` explícito sempre vence**, não importa quantos `Allow` existam. É por isso que as políticas de MFA e de região da seção 4.3 usam `Deny`: elas funcionam como travas que nenhuma outra política consegue abrir.
3. **Entre contas diferentes, os dois lados precisam permitir.** Para a conta de auditoria ler o bucket da seção 5.1, a bucket policy precisa permitir aquela conta **e** uma política de identidade dentro da conta de auditoria precisa permitir o usuário dela.

---

## 7. Credenciais de longo prazo e temporárias

### 7.1 Access keys: o problema

Access keys de IAM Users começam com `AKIA` e valem **até alguém as revogar**. Elas aparecem em arquivos `~/.aws/credentials`, variáveis de ambiente, Secrets de pipelines e, com frequência, onde não deviam.

> **Confusão comum:** "o repositório é privado, então não tem problema deixar a access key no código". ✅ **Mais preciso:** uma access key no código fica em **todo o histórico do Git** (apagar o arquivo depois não a remove dos commits antigos), em toda cópia do repositório, em todo fork, em todo notebook de quem clonou e em toda ferramenta de CI que já o leu. Repositórios privados ficam públicos por engano, notebooks são roubados e integrações de terceiros são comprometidas. Robôs varrem o GitHub público o tempo todo atrás de strings que comecem com `AKIA`, e chaves vazadas costumam ser usadas em **minutos**, normalmente para criar instâncias de mineração de criptomoedas por conta do dono. Se uma chave vazar, **desative-a imediatamente** no console (antes mesmo de investigar) e depois verifique no CloudTrail (seção 10) o que foi feito com ela. A solução definitiva é não ter chaves de longo prazo: usar roles em tudo o que roda na AWS (seção 8) e federação para o que roda fora dela (seção 9).

### 7.2 STS: credenciais temporárias

O **STS** (*Security Token Service*) é o serviço que emite credenciais temporárias quando uma identidade assume uma role:

```bash
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/leitor-de-backups \
  --role-session-name ana-investigando
```
```json
{
    "Credentials": {
        "AccessKeyId": "ASIAEXEMPLO1234567890",
        "SecretAccessKey": "wJalr...",
        "SessionToken": "IQoJb3JpZ2luX2VjE...",
        "Expiration": "2026-10-25T19:30:00Z"
    },
    "AssumedRoleUser": {
        "Arn": "arn:aws:sts::123456789012:assumed-role/leitor-de-backups/ana-investigando"
    }
}
```

Três diferenças em relação a uma access key comum: o prefixo é `ASIA` (temporária), há um terceiro elemento, o `SessionToken`, sem o qual a credencial não funciona, e há uma **data de expiração**. Se essas credenciais vazarem, o estrago tem prazo. O nome da sessão (`ana-investigando`) aparece nos registros de auditoria, de modo que se sabe quem usou a role e para quê.

Na prática, quase nunca se chama o `assume-role` à mão: os SDKs, a CLI e as integrações das seções seguintes fazem isso automaticamente e renovam as credenciais antes de expirarem.

---

## 8. Roles para serviços da AWS

### 8.1 EC2: o instance profile

A API do capítulo [Docker: Compose e produção](../../02-containers/docker/04_compose_e_producao.md), rodando em uma EC2, precisa gravar arquivos no S3. A forma errada é configurar uma access key no servidor. A forma certa é anexar uma **role** à instância:

1. Crie uma role cuja **trust policy** permita que o serviço EC2 a assuma:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Principal": { "Service": "ec2.amazonaws.com" },
         "Action": "sts:AssumeRole"
       }
     ]
   }
   ```
2. Anexe a ela uma política de permissão com apenas o necessário (`s3:PutObject` em `arn:aws:s3:::loja-uploads/*`).
3. Associe a role à instância (o console chama isso de **instance profile**).

A partir daí, o código **não configura credencial nenhuma**:

```python
import boto3

s3 = boto3.client("s3")             # sem chave, sem senha
s3.upload_file("foto.jpg", "loja-uploads", "produtos/123.jpg")
```

O SDK encontra as credenciais sozinho, consultando o **serviço de metadados da instância** (IMDS), um endereço especial disponível só de dentro da EC2. Na versão atual (IMDSv2), a consulta exige primeiro um token de sessão:

```bash
# De dentro da instância (é isto que o SDK faz por baixo):
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 300")
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/api-loja
# { "AccessKeyId": "ASIA...", "SecretAccessKey": "...", "Token": "...", "Expiration": "..." }
```

As credenciais são temporárias e renovadas automaticamente pela AWS. A mesma ideia vale para outros serviços: uma **Lambda** tem uma *execution role*, um container no **ECS** tem uma *task role*.

Exija **IMDSv2** em todas as instâncias (é uma opção da instância e o padrão nas mais recentes). Na versão anterior, um simples `GET` devolvia as credenciais, e uma falha de SSRF na aplicação (um formulário que busca uma URL informada pelo usuário, por exemplo) bastava para um atacante obtê-las. Foi esse o caminho de um dos maiores vazamentos de dados bancários já registrados, em 2019. O token do IMDSv2, obtido com um `PUT`, bloqueia esse tipo de ataque.

### 8.2 A role do Session Manager

O acesso sem porta 22 do capítulo [AWS: VPC](02_vpc_redes_na_aws.md) (seção 9) depende do mesmo mecanismo: a instância precisa de uma role com a política gerenciada `AmazonSSMManagedInstanceCore`, e a pessoa que abre a sessão precisa de uma política que permita `ssm:StartSession` naquela instância. A rede (VPC) e as permissões (IAM) trabalham juntas: sem a saída para o serviço do SSM, a role não adianta; sem a role, a saída não adianta.

---

## 9. Exemplo completo: GitHub Actions sem chaves guardadas

### 9.1 O problema

O pipeline de [../../03-entrega-continua/cd/03_pipeline_de_cd_na_pratica.md](../../03-entrega-continua/cd/03_pipeline_de_cd_na_pratica.md) guarda credenciais de longo prazo nos Secrets do GitHub: a senha do Docker Hub e a chave SSH da EC2. Para publicar imagens no **ECR** (o registry da AWS) e fazer deploy em uma conta AWS, a solução ingênua seria acrescentar mais um par de access keys aos Secrets, com todos os problemas da seção 7.1, agravados: um pipeline é um alvo valioso.

### 9.2 A solução: federação com OIDC

O GitHub Actions emite, para cada execução de workflow, um **token OIDC** assinado, que diz "sou uma execução do repositório `minha-org/minha-loja`, no branch `main`". A AWS pode ser configurada para **confiar nesses tokens** e trocá-los por credenciais temporárias de uma role. Nenhuma chave fica guardada em lugar nenhum.

```
 GitHub Actions                                         AWS
┌────────────────────┐  ① token OIDC assinado      ┌───────────────────────┐
│ workflow em        │  "repo:minha-org/minha-     │ STS                   │
│ minha-org/         │──── loja:ref:refs/heads/ ──▶│ ② a assinatura é do   │
│ minha-loja (main)  │      main"                  │   GitHub? o repo e o  │
│                    │                             │   branch batem com a  │
│                    │◀── ③ credenciais ASIA... ───│   trust policy?       │
│                    │     (valem ~1 hora)         └───────────────────────┘
│ ④ docker push para │
│   o ECR, deploy... │
└────────────────────┘
```

**Passo 1: registrar o GitHub como provedor de identidade na conta** (uma vez por conta), em IAM → *Identity providers*, com a URL `https://token.actions.githubusercontent.com` e o público (*audience*) `sts.amazonaws.com`.

**Passo 2: criar a role, com uma trust policy que aceita só o seu repositório e branch:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:minha-org/minha-loja:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

**Passo 3: dar à role só o que o pipeline faz**, por exemplo publicar imagens em um repositório específico do ECR:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "LoginNoEcr",
      "Effect": "Allow",
      "Action": "ecr:GetAuthorizationToken",
      "Resource": "*"
    },
    {
      "Sid": "PublicarImagensDaApi",
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload",
        "ecr:PutImage"
      ],
      "Resource": "arn:aws:ecr:sa-east-1:123456789012:repository/minha-api"
    }
  ]
}
```

(`ecr:GetAuthorizationToken` é uma das poucas ações que não se aplicam a um recurso específico, e por isso exige `"*"`.)

**Passo 4: usar no workflow:**

```yaml
# .github/workflows/deploy.yml
name: deploy
on:
  push:
    branches: [main]

permissions:
  id-token: write      # permite ao workflow pedir o token OIDC
  contents: read

jobs:
  publicar:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7

      - uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-deploy-minha-api
          aws-region: sa-east-1

      - id: ecr
        uses: aws-actions/amazon-ecr-login@v2

      - run: |
          IMAGEM=${{ steps.ecr.outputs.registry }}/minha-api:${{ github.sha }}
          docker build -t "$IMAGEM" .
          docker push "$IMAGEM"
```

A imagem recebe como tag o hash do commit (`github.sha`), seguindo a recomendação de tags únicas dos capítulos [Docker: imagens e Dockerfile](../../02-containers/docker/02_imagens_e_dockerfile.md) e [Kubernetes: workloads, rede e escala](../../02-containers/kubernetes/02_workloads_rede_e_escala.md). E não há nenhum segredo da AWS configurado no repositório.

> **Confusão comum:** "basta confiar no provedor OIDC do GitHub, o resto é detalhe". ✅ **Mais preciso:** o provedor `token.actions.githubusercontent.com` é o **mesmo para todo o GitHub**. Uma trust policy que confia nele, mas não verifica a condição `sub`, aceita tokens de **qualquer repositório de qualquer pessoa** no GitHub: bastaria um atacante criar um repositório próprio, escrever um workflow com o ARN da sua role e assumi-la. Esse erro já foi encontrado em muitas contas reais. A condição `sub` é o que prende a role ao **seu** repositório, e vale restringi-la ao máximo: `repo:minha-org/minha-loja:ref:refs/heads/main` (só o branch principal) é bem mais seguro que `repo:minha-org/*` (qualquer repositório da organização, inclusive os de teste). O mesmo raciocínio vale para qualquer federação: confiar no emissor do token **não** basta, é preciso verificar **de quem** é o token.

---

## 10. Menor privilégio na prática

O princípio é simples: cada identidade recebe só as permissões de que precisa ([Arquitetura de Software: seguranca.md](https://github.com/iranhonorato/Arquitetura-de-Software/blob/main/system-design-pt1/seguranca.md)). A dificuldade é saber **quais** são elas. Ninguém acerta de primeira a lista de ações que uma aplicação usa. O caminho prático é iterativo, e a AWS oferece ferramentas para cada etapa:

| Etapa | Ferramenta | O que ela faz |
|---|---|---|
| **Registrar** | **CloudTrail** | Grava cada chamada de API da conta: quem, o quê, quando, de onde, e se foi negada |
| **Gerar** | **IAM Access Analyzer** (geração de política) | Lê o CloudTrail de um período e **propõe** uma política com só as ações que a identidade realmente usou |
| **Podar** | *Last accessed* (na página de cada identidade) | Mostra quais serviços uma identidade nunca usou ou não usa há meses |
| **Validar** | **Access Analyzer** (validação) e *Policy Simulator* | Aponta erros e permissões excessivas em políticas e simula se uma ação seria permitida |
| **Vigiar** | **Access Analyzer** (acesso externo) | Avisa quando um bucket, role ou chave fica acessível de fora da conta ou da organização |

Um ciclo comum para uma aplicação nova: começar em um ambiente de testes com uma política gerenciada mais ampla, rodar a aplicação por alguns dias, gerar a política a partir do CloudTrail, revisar, aplicar e só então levar para produção.

### Checklist da conta

- [ ] Root com MFA, sem access keys, guardado para emergências.
- [ ] Pessoas entram pelo IAM Identity Center (ou, no mínimo, IAM Users com MFA obrigatório).
- [ ] Nenhuma access key de longo prazo em servidores: EC2, ECS e Lambda usam roles.
- [ ] Pipelines usam federação OIDC, com a condição `sub` restrita ao repositório e branch.
- [ ] IMDSv2 exigido em todas as instâncias.
- [ ] Políticas de produção são *customer managed*, com recursos específicos em vez de `"*"`.
- [ ] CloudTrail ativo em todas as regiões, com os logs em um bucket protegido.
- [ ] Access Analyzer ativo, revisando acessos externos.
- [ ] Revisão periódica de identidades e permissões sem uso.

---

## 11. Juntando tudo: EKS, VPC e IAM

O cluster do capítulo [Kubernetes: do Compose ao cluster](../../02-containers/kubernetes/04_do_compose_ao_cluster.md), levado para a AWS, depende das duas peças vistas nestes últimos capítulos. Cada camada de acesso tem um mecanismo próprio:

```
  Pessoas e pipelines                      Conta AWS
 ┌─────────────────┐   ① IAM (Identity    ┌──────────────────────────── VPC (cap. 11) ──────┐
 │ kubectl / CI    │──── Center ou OIDC)──▶│  EKS control plane  ◀── role do cluster        │
 └─────────────────┘   + access entry      │        │                 (criar LBs, discos)    │
                       ② RBAC (cap. 09)    │        ▼                                        │
                                           │  sub-redes privadas                            │
                                           │  ┌──────────── node (EC2) ◀── role do node ──┐ │
                                           │  │                            (ECR, rede)    │ │
                                           │  │  [Pod API] ◀── ③ Pod Identity: role da API │ │
                                           │  │      │          (só s3:PutObject no bucket)│ │
                                           │  └──────┼────────────────────────────────────┘ │
                                           └─────────┼──────────────────────────────────────┘
                                                     ▼
                                                  S3, Secrets Manager...
```

**① Quem pode usar o cluster.** O acesso de pessoas e pipelines ao API server do EKS começa no IAM: o `kubectl` obtém um token a partir das credenciais AWS de quem o usa. O comando abaixo prepara o kubeconfig (capítulo *Kubernetes: fundamentos e arquitetura*) para isso:

```bash
aws eks update-kubeconfig --name loja --region sa-east-1
```

No cluster, uma **access entry** associa aquela identidade IAM a permissões do Kubernetes. A partir daí, quem decide o que ela pode fazer **dentro** do cluster é o **RBAC** do capítulo [Kubernetes: configuração, estado e segurança](../../02-containers/kubernetes/03_configuracao_estado_e_seguranca.md). São duas camadas: o IAM responde "quem é você", e o RBAC responde "o que você pode fazer no cluster".

**② O que o cluster e os nodes podem fazer na AWS.** O próprio EKS tem uma **role do cluster**, usada para criar load balancers nas sub-redes públicas e discos EBS para os PVCs. Cada node tem uma **role do node** (via instance profile, seção 8), com permissão para baixar imagens do ECR e gerenciar os IPs dos Pods na VPC.

**③ O que cada Pod pode fazer na AWS.** Se a API precisa gravar no S3, a tentação é usar a role do node. Mas aí **todo** Pod daquele node herdaria a permissão, inclusive o Redis e qualquer Pod comprometido. A solução é dar uma role **a cada ServiceAccount** do Kubernetes, com o **EKS Pod Identity** (ou o mecanismo anterior, IRSA, baseado em OIDC, como o GitHub da seção 9):

```bash
aws eks create-pod-identity-association \
  --cluster-name loja \
  --namespace loja \
  --service-account minha-api \
  --role-arn arn:aws:iam::123456789012:role/api-loja-s3
```

Os Pods que usam a ServiceAccount `minha-api` recebem credenciais temporárias da role `api-loja-s3`, e só eles. O código continua sem configurar credencial nenhuma, como na EC2 da seção 8. É o menor privilégio aplicado na menor unidade possível: o Pod.

| Pergunta | Quem responde | Onde ver |
|---|---|---|
| Por onde os pacotes podem passar? | VPC: sub-redes, rotas, Security Groups | AWS 02 (VPC) |
| Quem pode chamar a API da AWS? | IAM: identidades, políticas, roles | AWS 03 (este) |
| Quem pode usar o cluster? | IAM (autenticação) + access entries | AWS 03 (este) |
| O que se pode fazer dentro do cluster? | RBAC do Kubernetes | Kubernetes 03 |
| Qual Pod pode falar com qual? | NetworkPolicy | Kubernetes 03 |
| Qual Pod pode acessar qual recurso da AWS? | Pod Identity: uma role por ServiceAccount | AWS 03 (este) |

---

## Próximo passo

A parte de infraestrutura e containers começou com a pergunta "o que precisa existir para uma linha de código chegar até um usuário?" e passou pelo hardware, pela virtualização, pelos containers, pelo Docker, pelo Kubernetes, pela rede da nuvem e, por fim, pelas permissões. Com os capítulos de VPC e IAM, cada peça tem um lugar: a imagem roda em Pods, os Pods em nodes, os nodes em sub-redes privadas de uma VPC, e cada acesso, de pessoas, pipelines e Pods, passa por uma identidade com permissões mínimas.

Falta fazer tudo isso acontecer sozinho, a cada commit: testar o código, construir a imagem, publicá-la e atualizar o ambiente, sem ninguém digitando comandos. Esse é o assunto da última parte da trilha, que começa em [../../03-entrega-continua/ci/01_fundamentos_de_integracao_continua.md](../../03-entrega-continua/ci/01_fundamentos_de_integracao_continua.md). Os pipelines de lá usam diretamente o que foi visto aqui: federação OIDC (seção 9) no lugar de chaves guardadas, publicação no ECR com tags por commit e deploy tanto em uma EC2 quanto em um cluster EKS.
