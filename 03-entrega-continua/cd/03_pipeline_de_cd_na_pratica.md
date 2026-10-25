# Pipeline de CD na Prática

**Pré-requisito:** [02_estrategias_de_deploy.md](02_estrategias_de_deploy.md), [../ci/02_github_actions_na_pratica.md](../ci/02_github_actions_na_pratica.md), [../../01-infraestrutura/aws/01_ec2_servidores_na_nuvem.md](../../01-infraestrutura/aws/01_ec2_servidores_na_nuvem.md). As partes 3 e 4 assumem [../../01-infraestrutura/aws/03_iam_identidades_e_permissoes.md](../../01-infraestrutura/aws/03_iam_identidades_e_permissoes.md) e a parte de Kubernetes.

> Segue a documentação do GitHub Actions (*Reusing workflows*, *Using environments for deployment*, *Publishing Docker images*), do AWS Systems Manager (*Run Command*), do Amazon EKS e do Argo CD. Reescreve e amplia o antigo `cd_hands_on.md` do repositório Arquitetura de Software, corrigindo duas práticas inseguras dele (a porta 22 aberta para a internet e a tag `latest`).

---

## Por que este arquivo existe

Os dois capítulos anteriores definiram o que um pipeline de CD deve fazer. Este o constrói, em quatro versões que vão do mais simples ao mais robusto:

| Parte | Destino | Como o pipeline chega lá | Credenciais guardadas no GitHub |
|---|---|---|---|
| **1** | Registry | Publica a imagem com a tag do commit | Nenhuma (usa o token automático) |
| **2** | EC2 | SSH com uma chave dedicada | Chave SSH e endereço do servidor |
| **3** | EC2 | OIDC + Session Manager, sem SSH | **Nenhuma** |
| **4** | Kubernetes (EKS) | `kubectl` (push) ou Argo CD (GitOps, pull) | **Nenhuma** |

A parte 5 junta tudo em um pipeline com dois ambientes: homologação automática e produção com aprovação. A pergunta que guia o capítulo é: *como levar a imagem validada pelo CI até produção, sem nenhum passo manual além de uma aprovação, sem chaves de longo prazo e com rollback automático se algo der errado?*

---

## 0. A estrutura dos workflows

O pipeline de CI do [capítulo de GitHub Actions](../ci/02_github_actions_na_pratica.md) roda em pull requests e na `main`. O CD só faz sentido na `main`, e precisa **reaproveitar** o CI, e não copiá-lo. O GitHub Actions resolve isso com **workflows reutilizáveis**: o `ci.yml` passa a aceitar ser chamado por outro workflow.

```yaml
# .github/workflows/ci.yml  (só o bloco "on" muda)
on:
  pull_request:
    branches: [main]
  workflow_call:          # permite que outro workflow o execute como um job
```

```yaml
# .github/workflows/cd.yml  (construído ao longo do capítulo)
on:
  push:
    branches: [main]

jobs:
  ci:
    uses: ./.github/workflows/ci.yml     # roda o CI inteiro como um job deste workflow
  # ... publicar, deploy-homologacao, deploy-producao
```

Repare que o `ci.yml` deixou de reagir ao `push` na `main`: quem cuida disso agora é o `cd.yml`, que chama o CI. Sem essa mudança, cada merge rodaria o CI duas vezes.

```
 pull request ─────▶ ci.yml                       (só verifica)

 push na main ─────▶ cd.yml ─▶ ci ─▶ publicar ─▶ homologação ─▶ [aprovação] ─▶ produção
                                │
                                └─ o MESMO ci.yml, chamado como job
```

---

## Parte 1: Publicando a imagem

### 1.1 Escolhendo um registry

A imagem precisa ficar em um **registry** acessível pelo ambiente de destino. Três opções comuns:

| Registry | Endereço | Autenticação no pipeline |
|---|---|---|
| **GitHub Container Registry** (GHCR) | `ghcr.io/seu-usuario/minha-api` | O `GITHUB_TOKEN` automático: nenhum secret extra |
| **Docker Hub** | `docker.io/seu-usuario/minha-api` | Um token de acesso guardado em secret |
| **Amazon ECR** | `123456789012.dkr.ecr.sa-east-1.amazonaws.com/minha-api` | OIDC com uma role (o exemplo do [capítulo de IAM](../../01-infraestrutura/aws/03_iam_identidades_e_permissoes.md)) |

As partes 1 e 2 usam o GHCR, porque ele não exige nenhum segredo. As partes 3 e 4 usam o ECR, que fica dentro da AWS, junto do destino.

### 1.2 O job de publicação

```yaml
  publicar:
    name: Publicar imagem
    needs: ci
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write                 # permite ao GITHUB_TOKEN publicar no GHCR
    outputs:
      imagem: ${{ steps.nome.outputs.imagem }}
    steps:
      - uses: actions/checkout@v7

      - id: nome
        run: echo "imagem=ghcr.io/${GITHUB_REPOSITORY,,}:${GITHUB_SHA}" >> "$GITHUB_OUTPUT"

      - uses: docker/login-action@v4
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/setup-buildx-action@v4

      - uses: docker/build-push-action@v7
        with:
          context: .
          push: true
          tags: ${{ steps.nome.outputs.imagem }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

Três detalhes:

- **`${GITHUB_REPOSITORY,,}`** converte o nome do repositório para minúsculas (uma sintaxe do bash), porque nomes de imagem não aceitam maiúsculas. O resultado é algo como `ghcr.io/seu-usuario/minha-api:a1b2c3d4...`.
- **`outputs:`** expõe o nome completo da imagem para os jobs seguintes, que o leem com `${{ needs.publicar.outputs.imagem }}`. É assim que **a mesma imagem** chega a homologação e a produção, a regra *construa uma vez* do [primeiro capítulo de CD](01_entrega_e_implantacao_continua.md).
- **A tag é o hash do commit**, e não `latest`. O antigo hands-on usava `app_users:latest`, o que impedia saber qual versão estava rodando e tornava o rollback impossível: a "versão anterior" da `latest` simplesmente deixava de existir. Com o hash, toda versão publicada continua no registry e pode ser reimplantada.

Para que **o resto do mundo** consiga baixar a imagem sem autenticação, mude a visibilidade do pacote para pública nas configurações do pacote no GitHub. Para imagens privadas, o servidor de destino precisa se autenticar, como na parte 2.

---

## Parte 2: Deploy na EC2 via SSH

Esta é a versão revisada do antigo hands-on: o pipeline entra na EC2 do [capítulo de EC2](../../01-infraestrutura/aws/01_ec2_servidores_na_nuvem.md) por SSH e atualiza o Compose. É a forma mais simples de começar, e tem limitações que a parte 3 resolve.

### 2.1 Preparando o servidor

**Um usuário só para o deploy.** O pipeline não deve entrar com o usuário `ubuntu`, que tem `sudo` sem senha. Crie um usuário dedicado:

```bash
# Na EC2, como ubuntu:
sudo adduser --disabled-password --gecos "" deploy
sudo usermod -aG docker deploy
sudo mkdir -p /opt/minha-loja && sudo chown deploy:deploy /opt/minha-loja
```

| | Pipeline entra como `ubuntu` | Pipeline entra como `deploy` |
|---|---|---|
| `sudo` | Sem senha: controle total da máquina | Não tem |
| Se a chave do pipeline vazar | O atacante é dono do servidor | O atacante controla os containers |

Um aviso sobre a última linha: o [capítulo de arquitetura do Docker](../../02-containers/docker/01_arquitetura_e_isolamento.md) mostrou que pertencer ao grupo `docker` **equivale a ter root**, porque é possível montar o disco do host em um container. O usuário dedicado reduz o risco de erros e deixa o acesso auditável, mas não é uma fronteira de segurança forte. A parte 3 elimina o problema na raiz, porque não há chave nenhuma para vazar.

**O Compose do servidor.** Em `/opt/minha-loja`, o `docker-compose.yml` de produção não tem `build:`, porque o servidor nunca constrói imagens; ele só baixa a que o pipeline indicar, por uma variável:

```yaml
# /opt/minha-loja/docker-compose.yml
services:
  api:
    image: ${IMAGEM_API}             # definida pelo pipeline a cada deploy
    ports:
      - "80:8000"
    env_file: .env
    restart: unless-stopped
    depends_on:
      postgres:
        condition: service_healthy

  postgres:
    image: postgres:16-alpine
    env_file: .env
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER}"]
      interval: 5s
      retries: 10
    restart: unless-stopped

volumes:
  postgres_data:
```

O `.env` com as senhas é criado **no servidor**, uma vez, à mão ou pela automação de configuração, e nunca passa pelo pipeline nem pelo Git. (O `$$` em `$${POSTGRES_USER}` escapa o `$` para o Compose: a variável é resolvida dentro do container, que recebeu o `.env`.)

### 2.2 Uma chave SSH só para o pipeline

Na **sua máquina** (não no servidor), gere um par de chaves exclusivo para o pipeline:

```bash
ssh-keygen -t ed25519 -C "github-actions-deploy" -f deploy_minha_loja -N ""
# deploy_minha_loja       ← chave privada: vai para o secret do GitHub
# deploy_minha_loja.pub   ← chave pública: vai para o servidor
```

No servidor, autorize a chave pública para o usuário `deploy`:

```bash
sudo mkdir -p /home/deploy/.ssh
sudo tee -a /home/deploy/.ssh/authorized_keys < deploy_minha_loja.pub
sudo chown -R deploy:deploy /home/deploy/.ssh
sudo chmod 700 /home/deploy/.ssh && sudo chmod 600 /home/deploy/.ssh/authorized_keys
```

As permissões importam: o servidor SSH ignora o `authorized_keys` se outros usuários puderem alterá-lo.

Capture também a **impressão digital** do servidor, para que o pipeline possa conferir que está falando com a máquina certa (o mesmo mecanismo do `known_hosts` do [capítulo de EC2](../../01-infraestrutura/aws/01_ec2_servidores_na_nuvem.md)):

```bash
ssh-keyscan -t ed25519 18.231.250.104 | ssh-keygen -lf - | awk '{print $2}'
# SHA256:Xk3p...
```

### 2.3 Environments e secrets

No GitHub, em **Settings → Environments → New environment**, crie o ambiente `producao`. Um **environment** agrupa os secrets de um destino e pode exigir aprovação antes de qualquer job usá-lo:

```
Environment: producao
  ☑ Required reviewers: (você e/ou alguém da equipe)
  Deployment branches: Selected branches → main

  Environment secrets:
    EC2_HOST         18.231.250.104
    EC2_USER         deploy
    EC2_SSH_KEY      (conteúdo inteiro de deploy_minha_loja, do BEGIN ao END)
    EC2_FINGERPRINT  SHA256:Xk3p...
```

Guardar os secrets **no environment**, e não no repositório, faz com que só jobs que declaram `environment: producao` os recebam, e só depois da aprovação. Um job de CI de um pull request nunca os vê.

### 2.4 O job de deploy

```yaml
  deploy-producao:
    name: Deploy em produção (EC2)
    needs: publicar
    runs-on: ubuntu-latest
    environment:
      name: producao
      url: http://18.231.250.104
    concurrency:
      group: deploy-producao           # nunca dois deploys ao mesmo tempo no mesmo destino
      cancel-in-progress: false        # e nunca interromper um deploy no meio
    permissions:
      contents: read
      packages: read                   # o GITHUB_TOKEN repassado ao servidor só pode BAIXAR imagens
    steps:
      - name: Atualizar a aplicação no servidor
        uses: appleboy/ssh-action@v1.2.0
        env:
          IMAGEM: ${{ needs.publicar.outputs.imagem }}
          GHCR_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GHCR_USER: ${{ github.actor }}
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          fingerprint: ${{ secrets.EC2_FINGERPRINT }}
          envs: IMAGEM,GHCR_TOKEN,GHCR_USER
          script: |
            set -euo pipefail
            cd /opt/minha-loja

            # Guarda a versão atual para um eventual rollback
            ANTERIOR=$(cat .imagem_atual 2>/dev/null || echo "")

            echo "$GHCR_TOKEN" | docker login ghcr.io -u "$GHCR_USER" --password-stdin
            export IMAGEM_API="$IMAGEM"
            docker compose pull api
            docker compose up -d api

            # Smoke test: até 10 tentativas, 3s entre elas
            for i in $(seq 1 10); do
              if curl --fail --silent http://localhost/healthz > /dev/null; then
                echo "$IMAGEM" > .imagem_atual
                echo "deploy ok: $IMAGEM"
                docker logout ghcr.io
                exit 0
              fi
              sleep 3
            done

            echo "smoke test falhou; voltando para: $ANTERIOR"
            if [ -n "$ANTERIOR" ]; then
              export IMAGEM_API="$ANTERIOR"
              docker compose up -d api
            fi
            docker logout ghcr.io
            exit 1
```

O que mudou em relação ao antigo hands-on:

| Antes | Agora | Por quê |
|---|---|---|
| `:latest` | `:<hash do commit>` | Saber o que está rodando e conseguir voltar |
| `docker stop` + `rm` + `run` | `docker compose up -d api` | O Compose recria só o que mudou, mantendo rede, volumes e configuração do arquivo |
| Nenhuma verificação | Smoke test com novas tentativas | O pipeline só fica verde se a aplicação responde |
| Nenhum rollback | Volta para a imagem anterior se o smoke test falhar | Recuperação automática |
| Secrets no repositório | Secrets no environment, com aprovação | Só o job de produção, aprovado, os recebe |
| Sem conferência do servidor | `fingerprint` | Impede que o pipeline entregue a chave a um servidor impostor |
| `@master` na action | `@v1.2.0` (ou o hash do commit) | O risco das tags móveis do [capítulo de GitHub Actions](../ci/02_github_actions_na_pratica.md) |

O `GITHUB_TOKEN` usado no `docker login` do servidor vale só enquanto o job roda, e o `docker logout` o remove do servidor no fim.

Durante o `docker compose up -d api`, o container antigo é substituído pelo novo, e há alguns segundos sem resposta: é a estratégia **recreate** do [capítulo de estratégias](02_estrategias_de_deploy.md). Para eliminar esse intervalo em uma única máquina, seria preciso subir o novo container ao lado do antigo e trocar o tráfego em um proxy reverso (um blue-green com Nginx). Com várias instâncias atrás de um load balancer, ou com Kubernetes (parte 4), o rolling update resolve isso naturalmente.

### 2.5 O problema da porta 22

O antigo hands-on terminava com uma recomendação perigosa: abrir a porta 22 do Security Group para `0.0.0.0/0` "ou pelo menos para os IPs do GitHub". As duas opções têm problemas:

- **Para `0.0.0.0/0`:** é o cenário que o [capítulo de EC2](../../01-infraestrutura/aws/01_ec2_servidores_na_nuvem.md) desaconselha, com o servidor recebendo tentativas de login do mundo inteiro.
- **Para "os IPs do GitHub":** os runners hospedados rodam na nuvem da Microsoft e usam **milhares** de faixas de IP, que mudam com frequência (a lista é publicada em `https://api.github.com/meta`, na chave `actions`). Liberar todas é quase o mesmo que liberar a internet inteira, e a faixa citada no guia antigo nem é a dos runners.

As saídas reais são:

| Alternativa | Como funciona |
|---|---|
| **Liberar só o IP do runner, só durante o job** | O job descobre o próprio IP, adiciona uma regra no Security Group com a CLI da AWS, faz o deploy e remove a regra no fim (exige credenciais AWS, via OIDC) |
| **Runner auto-hospedado dentro da VPC** | Uma máquina sua, na mesma rede, executa os jobs; a porta 22 só aceita conexões internas |
| **Não usar SSH** | O pipeline pede à AWS que execute o comando na instância, pelo Session Manager. A porta 22 pode ficar **fechada** |

A última é a mais simples e a mais segura, e é a parte 3.

---

## Parte 3: Deploy na EC2 sem chaves (OIDC + Session Manager)

A ideia: em vez de o pipeline **entrar** no servidor, ele pede à **API da AWS** que execute um script na instância. O agente do Systems Manager, que já roda na instância, recebe o comando por uma conexão **de saída** que ele mesmo abriu (o mecanismo descrito no [capítulo de VPC](../../01-infraestrutura/aws/02_vpc_redes_na_aws.md), seção 9). E o pipeline se autentica na AWS por **OIDC**, como no [capítulo de IAM](../../01-infraestrutura/aws/03_iam_identidades_e_permissoes.md), seção 9.

```
 GitHub Actions                       AWS                                  EC2 (porta 22 FECHADA)
┌───────────────┐ ① token OIDC  ┌──────────────┐                    ┌────────────────────────┐
│ job de deploy │──────────────▶│ STS          │                    │ agente SSM             │
│               │◀── ② cred. ───│              │                    │ (conexão de SAÍDA para │
│               │   temporárias └──────────────┘                    │  o serviço SSM)        │
│               │                                                   │                        │
│               │ ③ ssm send-command ┌──────────────┐  ④ entrega o   │ ⑤ executa o script:    │
│               │───────────────────▶│ Systems      │───────────────▶│   docker compose pull  │
│               │◀── ⑥ resultado ────│ Manager      │◀── resultado ──│   docker compose up    │
└───────────────┘                    └──────────────┘                └────────────────────────┘
```

Nenhuma chave SSH, nenhum IP liberado, nenhum secret de longo prazo no GitHub. Toda execução fica registrada no CloudTrail e no histórico do *Run Command*.

### 3.1 O que preparar na AWS

**Na instância** (uma vez):

- uma **instance role** com as políticas gerenciadas `AmazonSSMManagedInstanceCore` (para o agente SSM) e `AmazonEC2ContainerRegistryReadOnly` (para baixar imagens do ECR sem senha);
- uma **tag** que identifique a instância como alvo do deploy, por exemplo `app = loja`;
- a **AWS CLI**, usada pelo script para autenticar no ECR. Ela vem no Amazon Linux, mas não no Ubuntu: instale com `sudo snap install aws-cli --classic` (ou inclua o comando no *user data* do [capítulo de EC2](../../01-infraestrutura/aws/01_ec2_servidores_na_nuvem.md)).

**Para o pipeline**, a role `github-deploy-loja`, com a **trust policy** do capítulo de IAM (aceitando só o seu repositório e a `main`) e esta política de permissões:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicarNoEcr",
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload",
        "ecr:PutImage"
      ],
      "Resource": "*"
    },
    {
      "Sid": "UsarSoOScriptPadrao",
      "Effect": "Allow",
      "Action": "ssm:SendCommand",
      "Resource": "arn:aws:ssm:sa-east-1::document/AWS-RunShellScript"
    },
    {
      "Sid": "SoNasInstanciasDaLoja",
      "Effect": "Allow",
      "Action": "ssm:SendCommand",
      "Resource": "arn:aws:ec2:sa-east-1:123456789012:instance/*",
      "Condition": {
        "StringEquals": { "ssm:resourceTag/app": "loja" }
      }
    },
    {
      "Sid": "AcompanharOResultado",
      "Effect": "Allow",
      "Action": ["ssm:GetCommandInvocation", "ssm:ListCommandInvocations"],
      "Resource": "*"
    }
  ]
}
```

(Para simplificar, as ações de ECR estão com `"Resource": "*"`. Em produção, restrinja as de upload ao ARN do repositório, como no capítulo de IAM.)

A condição `ssm:resourceTag/app` é o que impede o pipeline de executar comandos em **qualquer** instância da conta: ele só consegue enviar comandos para máquinas com a tag `app = loja`. Mesmo que alguém altere o workflow, a AWS recusa um comando para o servidor de outro sistema.

### 3.2 O job

```yaml
  deploy-producao:
    name: Deploy em produção (EC2 via SSM)
    needs: publicar
    runs-on: ubuntu-latest
    environment: producao
    concurrency:
      group: deploy-producao
      cancel-in-progress: false
    permissions:
      id-token: write                 # permite pedir o token OIDC
      contents: read
    steps:
      - uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-deploy-loja
          aws-region: sa-east-1

      - name: Enviar o comando de deploy
        id: enviar
        env:
          IMAGEM: ${{ needs.publicar.outputs.imagem }}
          INSTANCIA: ${{ vars.EC2_INSTANCE_ID }}
        run: |
          COMANDOS=$(cat <<EOF
          set -euo pipefail
          cd /opt/minha-loja
          aws ecr get-login-password --region sa-east-1 | docker login --username AWS --password-stdin ${IMAGEM%%/*}
          export IMAGEM_API=$IMAGEM
          docker compose pull api
          docker compose up -d api
          for i in \$(seq 1 10); do curl -fs http://localhost/healthz && exit 0; sleep 3; done
          exit 1
          EOF
          )
          ID=$(aws ssm send-command \
            --instance-ids "$INSTANCIA" \
            --document-name AWS-RunShellScript \
            --comment "deploy ${GITHUB_SHA::7}" \
            --parameters "$(jq -n --arg c "$COMANDOS" '{commands: [$c]}')" \
            --query Command.CommandId --output text)
          echo "id=$ID" >> "$GITHUB_OUTPUT"

      - name: Aguardar e mostrar o resultado
        env:
          INSTANCIA: ${{ vars.EC2_INSTANCE_ID }}
        run: |
          aws ssm wait command-executed --command-id "${{ steps.enviar.outputs.id }}" --instance-id "$INSTANCIA" || true
          aws ssm get-command-invocation --command-id "${{ steps.enviar.outputs.id }}" --instance-id "$INSTANCIA" \
            --query '{status: Status, saida: StandardOutputContent, erro: StandardErrorContent}'
          STATUS=$(aws ssm get-command-invocation --command-id "${{ steps.enviar.outputs.id }}" \
            --instance-id "$INSTANCIA" --query Status --output text)
          test "$STATUS" = "Success"
```

Dois detalhes do script. O heredoc `<<EOF` (sem aspas) faz o shell do **runner** substituir `$IMAGEM` antes do envio, e `${IMAGEM%%/*}` extrai dele só o endereço do registry. Já o `\$(seq 1 10)` está escapado de propósito, para chegar intacto à instância e ser executado **lá**.

O job da parte 1 também muda: em vez de publicar no GHCR, ele assume a mesma role e publica no ECR, com o `aws-actions/amazon-ecr-login` mostrado no [capítulo de IAM](../../01-infraestrutura/aws/03_iam_identidades_e_permissoes.md).

Repare no que **não** existe mais: nenhum secret no environment `producao` (o ID da instância é uma *variable*, não um segredo), nenhuma regra de porta 22 no Security Group, nenhuma chave para rotacionar. O environment continua útil pela **aprovação obrigatória**.

O script desta versão não tem o rollback automático da parte 2, para manter o exemplo curto. Ele pode ser acrescentado da mesma forma (guardar a imagem atual em um arquivo e voltar para ela se o smoke test falhar), ou feito por um segundo `send-command` disparado por um step com `if: failure()`.

---

## Parte 4: Deploy no Kubernetes

Com o sistema rodando no EKS (como no [capítulo do Compose ao cluster](../../02-containers/kubernetes/04_do_compose_ao_cluster.md)), o deploy deixa de ser "executar comandos em um servidor" e passa a ser "mudar o estado desejado no cluster". O loop de reconciliação e o rolling update fazem o resto. Há duas formas de o pipeline mudar esse estado.

### 4.1 Push: o pipeline aplica a mudança

O pipeline se autentica no cluster e manda o Deployment usar a imagem nova:

```yaml
  deploy-producao:
    name: Deploy em produção (EKS)
    needs: publicar
    runs-on: ubuntu-latest
    environment: producao
    concurrency:
      group: deploy-producao
      cancel-in-progress: false
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-deploy-loja
          aws-region: sa-east-1

      - name: Configurar o kubectl
        run: aws eks update-kubeconfig --name loja --region sa-east-1

      - name: Atualizar a imagem
        run: |
          kubectl set image deployment/minha-api \
            api=${{ needs.publicar.outputs.imagem }} -n loja
          kubectl annotate deployment/minha-api -n loja --overwrite \
            kubernetes.io/change-cause="commit ${GITHUB_SHA::7} por ${GITHUB_ACTOR}"

      - name: Aguardar o rollout
        run: kubectl rollout status deployment/minha-api -n loja --timeout=5m

      - name: Rollback se o rollout falhar
        if: failure()
        run: kubectl rollout undo deployment/minha-api -n loja
```

Como cada peça se liga aos capítulos anteriores:

- A role precisa de `eks:DescribeCluster` (para o `update-kubeconfig`) e de uma **access entry** no cluster que lhe dê permissão **só no namespace `loja`**, e só para atualizar Deployments: o IAM responde quem é, o RBAC responde o que pode fazer ([capítulo de IAM](../../01-infraestrutura/aws/03_iam_identidades_e_permissoes.md), seção 11).
- O `rollout status` espera até as réplicas novas estarem **prontas**, usando as readiness probes ([capítulo de workloads](../../02-containers/kubernetes/02_workloads_rede_e_escala.md)). Se elas não ficarem prontas em 5 minutos, o comando falha, o job falha, e o step `if: failure()` reverte.
- A anotação `change-cause` faz o `kubectl rollout history` mostrar qual commit e qual pessoa geraram cada revisão.

### 4.2 Pull: GitOps com Argo CD

No modelo **GitOps**, o pipeline **não tem acesso ao cluster**. O estado desejado vive em um repositório Git (os manifestos com Kustomize do capítulo do Compose ao cluster), e uma ferramenta **dentro do cluster**, o **Argo CD**, observa esse repositório e aplica qualquer mudança. O deploy passa a ser um **commit**:

```
 ┌────────────── fora do cluster ───────────────┐      ┌────────── dentro do cluster ──────────┐
 │                                              │      │                                       │
 │ repo da aplicação            repo de config  │      │   Argo CD                             │
 │ ┌──────────────┐  ① CI      ┌──────────────┐ │      │   ┌────────────────────────────────┐  │
 │ │ código       │──publica──▶│ k8s/overlays/│ │◀─────┼───│ ② observa o repo de config     │  │
 │ │ Dockerfile   │  imagem e  │ producao/    │ │ pull │   │ ③ compara com o cluster        │  │
 │ └──────────────┘  faz commit│ (tag nova)   │ │      │   │ ④ aplica a diferença           │  │
 │                   da tag    └──────────────┘ │      │   └────────────────────────────────┘  │
 └──────────────────────────────────────────────┘      └───────────────────────────────────────┘
```

O job de deploy do pipeline vira uma alteração em um arquivo:

```yaml
  atualizar-config:
    name: Atualizar a versão no repositório de configuração
    needs: publicar
    runs-on: ubuntu-latest
    environment: producao               # a aprovação continua valendo
    steps:
      - uses: actions/checkout@v7
        with:
          repository: seu-usuario/loja-config
          token: ${{ secrets.CONFIG_REPO_TOKEN }}   # token com escrita SÓ nesse repositório

      - name: Trocar a tag da imagem
        run: |
          cd k8s/overlays/producao
          kustomize edit set image minha-api=${{ needs.publicar.outputs.imagem }}
          git config user.name  "pipeline"
          git config user.email "pipeline@users.noreply.github.com"
          git commit -am "producao: minha-api ${GITHUB_SHA::7}"
          git push
```

E, no cluster, o Argo CD é configurado uma vez com uma **Application**:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: loja-producao
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/seu-usuario/loja-config
    targetRevision: main
    path: k8s/overlays/producao
  destination:
    server: https://kubernetes.default.svc     # o próprio cluster
    namespace: loja
  syncPolicy:
    automated:
      prune: true        # remove do cluster o que foi removido do Git
      selfHeal: true     # desfaz mudanças feitas à mão no cluster
```

Com `selfHeal: true`, se alguém rodar um `kubectl edit` em produção, o Argo CD desfaz a mudança em instantes: o Git é a **única** fonte da verdade. Um rollback é um `git revert` do commit que trocou a tag, e o histórico do repositório de configuração vira o registro completo de tudo o que já esteve em produção, com autor e horário.

| | Push (`kubectl` no pipeline) | Pull (GitOps, Argo CD) |
|---|---|---|
| **Quem acessa o cluster** | O pipeline, de fora | Só o Argo CD, de dentro |
| **Credenciais do cluster fora dele** | Sim (a role do pipeline) | Não |
| **Mudança manual no cluster** | Fica lá até o próximo deploy | É desfeita automaticamente |
| **Rollback** | `kubectl rollout undo` | `git revert` |
| **Registro de auditoria** | Logs do pipeline | O histórico do Git |
| **Complexidade inicial** | Menor | Maior (instalar e operar o Argo CD) |

> **Confusão comum:** "fazemos GitOps: nossos manifestos estão no Git". ✅ **Mais preciso:** guardar YAML no Git é pré-requisito, não GitOps. O que define o GitOps é **quem aplica** e **em que direção**: um agente dentro do cluster **puxa** continuamente o estado do Git e **reconcilia** o cluster com ele, desfazendo divergências. É o loop de reconciliação do [capítulo de fundamentos do Kubernetes](../../02-containers/kubernetes/01_fundamentos_e_arquitetura.md) estendido até o repositório. Se os manifestos estão no Git, mas quem os aplica é um `kubectl apply` no pipeline, e qualquer pessoa pode mudar o cluster à mão sem que nada aconteça, o Git é só uma cópia, possivelmente desatualizada, do que está em produção. O teste prático: se alguém apagar um Deployment de produção à mão, ele volta sozinho?

---

## Parte 5: O pipeline completo, com homologação e produção

Juntando as peças, com o destino EKS em modo push (parte 4.1) e dois ambientes:

```yaml
# .github/workflows/cd.yml
name: CD

on:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  ci:
    uses: ./.github/workflows/ci.yml

  publicar:
    name: Publicar imagem no ECR
    needs: ci
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    outputs:
      imagem: ${{ steps.nome.outputs.imagem }}
    steps:
      - uses: actions/checkout@v7
      - uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-publicar-minha-api
          aws-region: sa-east-1
      - id: ecr
        uses: aws-actions/amazon-ecr-login@v2
      - id: nome
        run: echo "imagem=${{ steps.ecr.outputs.registry }}/minha-api:${GITHUB_SHA}" >> "$GITHUB_OUTPUT"
      - uses: docker/setup-buildx-action@v4
      - uses: docker/build-push-action@v7
        with:
          context: .
          push: true
          tags: ${{ steps.nome.outputs.imagem }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy-homologacao:
    name: Deploy em homologação
    needs: publicar
    runs-on: ubuntu-latest
    environment:
      name: homologacao                  # sem aprovação: automático
      url: https://homolog.loja.exemplo.com
    concurrency:
      group: deploy-homologacao
      cancel-in-progress: false
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v7
      - uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-deploy-homologacao
          aws-region: sa-east-1
      - run: aws eks update-kubeconfig --name loja-homologacao --region sa-east-1
      - run: kubectl set image deployment/minha-api api=${{ needs.publicar.outputs.imagem }} -n loja
      - run: kubectl rollout status deployment/minha-api -n loja --timeout=5m
      - name: Smoke tests
        run: ./scripts/smoke-test.sh https://homolog.loja.exemplo.com
      - if: failure()
        run: kubectl rollout undo deployment/minha-api -n loja

  deploy-producao:
    name: Deploy em produção
    needs: [publicar, deploy-homologacao]
    runs-on: ubuntu-latest
    environment:
      name: producao                     # com "required reviewers": espera aprovação
      url: https://loja.exemplo.com
    concurrency:
      group: deploy-producao
      cancel-in-progress: false
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v7
      - uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-deploy-producao
          aws-region: sa-east-1
      - run: aws eks update-kubeconfig --name loja --region sa-east-1
      - run: |
          kubectl set image deployment/minha-api api=${{ needs.publicar.outputs.imagem }} -n loja
          kubectl annotate deployment/minha-api -n loja --overwrite \
            kubernetes.io/change-cause="commit ${GITHUB_SHA::7} por ${GITHUB_ACTOR}"
      - run: kubectl rollout status deployment/minha-api -n loja --timeout=5m
      - name: Smoke tests
        run: ./scripts/smoke-test.sh https://loja.exemplo.com
      - if: failure()
        run: kubectl rollout undo deployment/minha-api -n loja
```

O fluxo, como aparece na página da execução:

```
 ci ──▶ publicar ──▶ deploy-homologacao ──▶ ⏸ aguardando aprovação ──▶ deploy-producao
 (≈3min)  (≈1min)     (≈2min + smoke)         (alguém clica em            (≈2min + smoke)
                                               "Review deployments")
```

Pontos que valem notar:

- **Uma role por ambiente** (`github-deploy-homologacao`, `github-deploy-producao`), e a trust policy da role de produção pode exigir, além do repositório e do branch, o **environment**: o `sub` do token OIDC passa a ser `repo:seu-usuario/minha-api:environment:producao`. Assim, só um job que passou pela aprovação do environment consegue assumir a role de produção.
- **A mesma imagem** (`needs.publicar.outputs.imagem`) vai para os dois ambientes. Produção recebe exatamente o que homologação validou.
- O script `smoke-test.sh` é o do [capítulo de estratégias](02_estrategias_de_deploy.md), seção 8, guardado no repositório.
- Para **implantação contínua** (sem aprovação), basta remover os *required reviewers* do environment `producao`. Nada no workflow muda: é a diferença de uma etapa descrita no [primeiro capítulo de CD](01_entrega_e_implantacao_continua.md).

---

## 6. Quando algo dá errado

| Sintoma | Causa provável | Onde olhar |
|---|---|---|
| `denied: permission_denied` ao publicar no GHCR | Falta `packages: write` nas permissões do job | Bloco `permissions` do job |
| `Not authorized to perform sts:AssumeRoleWithWebIdentity` | A condição `sub` da trust policy não bate (outro branch, outro environment, outro repositório) | O `sub` exato aparece no log do `configure-aws-credentials`; compare com a trust policy |
| `ssh: handshake failed` / `i/o timeout` | Porta 22 bloqueada no Security Group, ou fingerprint diferente | Security Group; `EC2_FINGERPRINT` |
| `permission denied ... docker.sock` no servidor | Usuário `deploy` fora do grupo `docker`, ou sem novo login após entrar no grupo | `groups deploy` no servidor |
| SSM: `InvalidInstanceId` | Agente SSM sem conexão, ou instância sem a instance role | *Systems Manager → Fleet Manager*: a instância aparece como gerenciada? |
| SSM: `AccessDeniedException` no `send-command` | A instância não tem a tag exigida pela política | Tags da instância versus a condição `ssm:resourceTag` |
| `error: You must be logged in to the server (Unauthorized)` no `kubectl` | A role não tem access entry no cluster | `aws eks list-access-entries --cluster-name loja` |
| `rollout status` expira | Pods novos não ficam prontos: imagem errada, crash, readiness falhando | O roteiro de diagnóstico do [capítulo do Compose ao cluster](../../02-containers/kubernetes/04_do_compose_ao_cluster.md) |
| O deploy "funcionou" mas nada mudou | A tag não mudou (uso de `latest`) | A imagem que o Deployment está usando: `kubectl get deploy minha-api -o jsonpath='{..image}'` |

---

## 7. Checklist do pipeline de CD

- [ ] O CI é reaproveitado (workflow reutilizável), e não copiado.
- [ ] A imagem é construída **uma vez** e identificada pelo hash do commit.
- [ ] A mesma imagem passa por homologação e produção.
- [ ] Configuração e segredos de cada ambiente ficam no ambiente (environments, Secrets do Kubernetes), e não na imagem.
- [ ] Nenhuma credencial de longo prazo no GitHub: OIDC para a AWS, `GITHUB_TOKEN` para o GHCR.
- [ ] Produção exige aprovação (entrega contínua) ou tem verificação automática suficiente para dispensá-la (implantação contínua).
- [ ] Nunca dois deploys simultâneos no mesmo ambiente (`concurrency`).
- [ ] Todo deploy termina com uma verificação (rollout status, smoke tests), e uma falha dispara rollback automático.
- [ ] O rollback usa o mesmo mecanismo do deploy e foi testado.
- [ ] Nenhuma porta 22 aberta para a internet.
- [ ] As actions de terceiros estão fixadas por versão ou hash.

---

## Próximo passo

Os blocos de CI e CD usaram o GitHub Actions, uma plataforma em que o GitHub opera tudo: os servidores, os runners, a interface, as atualizações. Em muitas empresas, porém, a automação roda em um servidor próprio, dentro da rede da empresa, com acesso a sistemas internos que nunca estarão na internet. Na grande maioria desses casos, esse servidor é o **Jenkins**.

O [../jenkins/01_fundamentos_do_jenkins.md](../jenkins/01_fundamentos_do_jenkins.md) apresenta o Jenkins: o que ele é, como funciona por dentro e como instalá-lo. Os dois capítulos seguintes reconstroem neles os pipelines de CI e CD deste bloco.
