# AWS VPC: a Rede Privada na Nuvem

**Pré-requisito:** [01_ec2_servidores_na_nuvem.md](01_ec2_servidores_na_nuvem.md), [../../02-containers/docker/03_volumes_e_redes.md](../../02-containers/docker/03_volumes_e_redes.md). Para a última seção, [../../02-containers/kubernetes/04_do_compose_ao_cluster.md](../../02-containers/kubernetes/04_do_compose_ao_cluster.md).

> Segue o *Amazon VPC User Guide* e o *AWS Well-Architected Framework* (pilares de segurança e confiabilidade). A arquitetura em camadas usada como exemplo é a do Capítulo 1 de **System Design Interview** (Alex Xu): load balancer na frente, servidores web atrás e banco de dados separado, replicado entre data centers.

---

## Por que este arquivo existe

No [01_ec2_servidores_na_nuvem.md](01_ec2_servidores_na_nuvem.md), a instância EC2 parecia uma máquina solta na internet: tinha um IP público, um Security Group e uma porta 22 aberta. Mas o prompt dela já denunciava outra coisa:

```
ubuntu@ip-172-31-9-94:~$
```

`172.31.9.94` é um endereço **privado**, que não existe na internet. Ele pertence a uma rede que a AWS criou automaticamente quando a conta foi aberta: a **VPC padrão**. Toda instância, todo banco RDS, todo node de um cluster EKS vive dentro de uma VPC, e é ela que decide o que pode chegar a cada recurso e o que cada recurso pode alcançar.

Este capítulo responde à pergunta: *como construir, na AWS, uma rede em que o load balancer fique exposto à internet, a aplicação fique acessível só pelo load balancer e o banco de dados fique completamente inacessível de fora?* Para chegar lá, começa pelos fundamentos de endereçamento IP, porque sem eles uma VPC é só uma lista de números.

---

## 1. Fundamentos: endereços IP e CIDR

### 1.1 Endereços públicos e privados

Um endereço IPv4 tem 4 números de 0 a 255 (`18.231.250.104`). Eles se dividem em dois grupos:

- **Públicos:** únicos na internet inteira, roteáveis de qualquer lugar. São escassos e controlados.
- **Privados:** reservados para uso **interno** de redes (RFC 1918). Podem se repetir em milhões de redes diferentes, porque nunca saem para a internet. A rede Wi-Fi da sua casa usa um deles.

| Faixa privada | Exemplo de uso |
|---|---|
| `10.0.0.0` a `10.255.255.255` | Redes corporativas e VPCs (a mais usada) |
| `172.16.0.0` a `172.31.255.255` | A VPC padrão da AWS (`172.31.x.x`) e as redes do Docker (`172.17.x.x`, capítulo [Docker: arquitetura e isolamento](../../02-containers/docker/01_arquitetura_e_isolamento.md)) |
| `192.168.0.0` a `192.168.255.255` | Roteadores domésticos |

### 1.2 Notação CIDR

Redes são descritas na notação **CIDR**: um endereço seguido de uma barra e um número, como `10.0.0.0/16`. O número depois da barra diz **quantos bits iniciais são fixos**, ou seja, compartilhados por todos os endereços da rede. Os bits restantes variam livremente.

Um IPv4 tem 32 bits. Com `/16`, os primeiros 16 bits (os dois primeiros números) são fixos e os outros 16 variam:

```
10.0.0.0/16
└┬─┘ └┬┘
fixo  livre        → endereços de 10.0.0.0 a 10.0.255.255
                   → 2^(32−16) = 2^16 = 65.536 endereços

10.0.1.0/24
└──┬───┘ └┬┘
  fixo   livre     → endereços de 10.0.1.0 a 10.0.1.255
                   → 2^(32−24) = 2^8 = 256 endereços
```

Quanto **maior** o número depois da barra, **menor** a rede:

| CIDR | Endereços | Analogia |
|---|---|---|
| `/16` | 65.536 | Um bairro inteiro |
| `/20` | 4.096 | Uma rua grande |
| `/24` | 256 | Um quarteirão |
| `/28` | 16 | Um prédio pequeno |
| `/32` | 1 | Um único endereço (usado em regras: "só o IP `200.1.2.3/32`") |

Duas regras práticas aparecem o tempo todo:

- `0.0.0.0/0` significa **"qualquer endereço"**: nenhum bit fixo. Em uma regra de firewall, é "o mundo inteiro". Em uma tabela de rotas, é "qualquer destino não listado", a **rota padrão**.
- Redes que precisam se comunicar **não podem se sobrepor**. Se duas VPCs usam `10.0.0.0/16`, elas nunca poderão ser conectadas diretamente. Por isso o planejamento de endereços se faz **antes** de criar a rede.

---

## 2. O que é uma VPC

Uma **VPC** (*Virtual Private Cloud*) é uma **rede privada e isolada, só sua, dentro de uma região da AWS**. Você escolhe a faixa de endereços (um CIDR entre `/16` e `/28`), divide em sub-redes, define as rotas e os firewalls. Nada entra nem sai dela sem que você configure.

Uma analogia ajuda a acompanhar o resto do capítulo. Uma VPC é um **condomínio fechado**:

| Condomínio | VPC | Seção |
|---|---|---|
| O terreno murado | A VPC, com seu CIDR | 2 |
| As ruas internas, cada uma em um bloco | As **sub-redes**, cada uma em uma zona de disponibilidade | 3 |
| As placas de trânsito em cada rua | As **tabelas de rotas** | 4 |
| O portão principal, por onde entra e sai gente | O **Internet Gateway** | 5 |
| Uma saída de serviço, que só permite sair | O **NAT Gateway** | 6 |
| A fechadura de cada casa | O **Security Group** | 7 |
| A cancela no início de cada rua | A **Network ACL** | 7 |

### Regiões e zonas de disponibilidade

Uma VPC pertence a **uma região** (por exemplo, `sa-east-1`, São Paulo). Cada região tem várias **zonas de disponibilidade** (*Availability Zones*, AZs): `sa-east-1a`, `sa-east-1b`, `sa-east-1c`. Cada AZ é um ou mais data centers **fisicamente separados**, com energia, refrigeração e rede próprias, a alguns quilômetros das outras e ligados a elas por fibra de baixa latência.

```
┌──────────────────────── Região sa-east-1 (São Paulo) ──────────────────────────┐
│                                                                                │
│  ┌──────── VPC 10.0.0.0/16 ─────────────────────────────────────────────────┐  │
│  │                                                                          │  │
│  │   ┌── AZ sa-east-1a ──┐   ┌── AZ sa-east-1b ──┐   ┌── AZ sa-east-1c ──┐  │  │
│  │   │  sub-redes...     │   │  sub-redes...     │   │  sub-redes...     │  │  │
│  │   └───────────────────┘   └───────────────────┘   └───────────────────┘  │  │
│  │                                                                          │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────────────┘
```

A VPC se estende por **todas** as AZs da região. Distribuir a aplicação por pelo menos duas AZs é o que permite sobreviver à queda de um data center inteiro. É a versão na nuvem da recomendação de Alex Xu de replicar entre data centers.

### A VPC padrão

Toda conta tem, em cada região, uma **VPC padrão** já criada: CIDR `172.31.0.0/16`, uma sub-rede **pública** em cada AZ, e instâncias que recebem IP público automaticamente. Ela existe para que seja possível criar uma EC2 em cinco minutos, como no [01_ec2_servidores_na_nuvem.md](01_ec2_servidores_na_nuvem.md), sem entender nada deste capítulo.

O preço dessa conveniência é que **tudo nela é público por padrão**. Um banco criado lá, sem atenção, fica com um IP alcançável da internet, protegido apenas pelo Security Group. Para produção, cria-se uma VPC própria, desenhada com o que vem a seguir.

---

## 3. Sub-redes

Uma **sub-rede** (*subnet*) é uma fatia do CIDR da VPC, sempre contida em **uma única AZ**. Todo recurso com placa de rede (uma EC2, um RDS, um node do EKS) é criado **dentro de uma sub-rede**, e é dela que recebe seu IP privado.

```
VPC 10.0.0.0/16  (65.536 endereços)
 ├── 10.0.1.0/24   sub-rede em sa-east-1a   (256 endereços)
 ├── 10.0.2.0/24   sub-rede em sa-east-1b
 ├── 10.0.11.0/24  sub-rede em sa-east-1a
 └── ...
```

Em cada sub-rede, a AWS **reserva 5 endereços** para uso próprio. Em `10.0.1.0/24`:

| Endereço | Uso |
|---|---|
| `10.0.1.0` | Endereço da rede |
| `10.0.1.1` | Roteador da VPC |
| `10.0.1.2` | Servidor DNS da AWS |
| `10.0.1.3` | Reservado para uso futuro |
| `10.0.1.255` | Endereço de broadcast |

Sobram, portanto, **251** endereços utilizáveis em um `/24`, e não 256.

### Pública ou privada

As sub-redes são classificadas em dois tipos, e essa é a decisão de rede mais importante:

- **Pública:** os recursos nela **podem** se comunicar diretamente com a internet, nos dois sentidos (se tiverem IP público). É onde ficam o load balancer e o NAT Gateway.
- **Privada:** os recursos nela **não podem** ser alcançados a partir da internet. É onde ficam a aplicação e o banco.

> **Confusão comum:** "uma sub-rede pública é uma sub-rede marcada como pública na criação". ✅ **Mais preciso:** não existe nenhuma opção "pública" ou "privada" na sub-rede. O que torna uma sub-rede pública é **uma linha na tabela de rotas** associada a ela: uma rota de `0.0.0.0/0` apontando para um **Internet Gateway** (seção 4). Sem essa rota, a sub-rede é privada, mesmo que as instâncias nela tenham IP público: os pacotes simplesmente não têm caminho para a internet. A configuração "atribuir IP público automaticamente" que existe na sub-rede é outra coisa (só decide se as instâncias novas ganham um IP público) e, sozinha, não torna nada acessível. Entender isso evita os dois erros clássicos: a instância "pública" que não responde (falta a rota) e o banco "privado" que está exposto (alguém associou a tabela de rotas errada à sub-rede dele).

---

## 4. Tabelas de rotas

Cada sub-rede está associada a uma **tabela de rotas** (*route table*), que diz, para cada destino, **por onde** o pacote deve sair. A regra de escolha é a da **rota mais específica**: se um destino casa com várias linhas, vence a de CIDR maior (mais bits fixos).

Toda tabela começa com uma rota que não pode ser removida, a **rota local**, que permite a comunicação entre todas as sub-redes da VPC:

**Tabela da sub-rede pública:**

| Destino | Alvo | Significado |
|---|---|---|
| `10.0.0.0/16` | `local` | Qualquer endereço da VPC: entrega direto |
| `0.0.0.0/0` | `igw-0a1b2c...` (Internet Gateway) | Qualquer outro destino: sai pela internet |

**Tabela da sub-rede privada:**

| Destino | Alvo | Significado |
|---|---|---|
| `10.0.0.0/16` | `local` | Qualquer endereço da VPC: entrega direto |
| `0.0.0.0/0` | `nat-0d4e5f...` (NAT Gateway) | Qualquer outro destino: sai pelo NAT (seção 6) |

Um exemplo de decisão: uma instância na sub-rede privada envia um pacote para `10.0.21.15`. As duas linhas casam (`0.0.0.0/0` casa com tudo), mas `10.0.0.0/16` é mais específica, e o pacote é entregue internamente. Se o destino for `142.250.79.46` (um servidor do Google), só a segunda linha casa, e o pacote vai para o NAT.

A diferença entre as duas sub-redes está **inteira** na segunda linha da tabela.

---

## 5. Internet Gateway: a porta da VPC para a internet

O **Internet Gateway** (IGW) é o componente que conecta a VPC à internet. Ele é gerenciado pela AWS, redundante e sem limite prático de banda, e existe um por VPC.

Para que uma instância seja alcançável da internet, **três** condições precisam estar presentes ao mesmo tempo:

1. Um **Internet Gateway** associado à VPC.
2. A sub-rede da instância com uma rota `0.0.0.0/0 → igw` (ou seja, uma sub-rede pública).
3. Um **IP público** na instância (atribuído automaticamente ou um **Elastic IP**, que é fixo).

E, mesmo assim, o Security Group (seção 7) ainda precisa permitir o tráfego.

A instância **não sabe** que tem um IP público. Rode `ip addr` nela e verá só o `10.0.1.25`. Quem faz a tradução é o Internet Gateway: pacotes que chegam para o IP público têm o destino reescrito para o IP privado, e as respostas fazem o caminho inverso. É por isso que o prompt da EC2 do [01_ec2_servidores_na_nuvem.md](01_ec2_servidores_na_nuvem.md) mostrava `172.31.9.94`, e não o `18.231.250.104`.

Vale saber que, desde 2024, a AWS **cobra por todo IPv4 público** (cerca de US$ 0,005 por hora por endereço, ou uns US$ 3,60 por mês), inclusive dos atribuídos automaticamente. É mais um motivo para manter IPs públicos só onde são necessários.

---

## 6. NAT Gateway: saída sem entrada

### 6.1 O problema

A aplicação está em uma sub-rede privada, inalcançável da internet, como deve ser. Mas ela **precisa** sair para a internet: baixar atualizações de pacotes (`apt update`), chamar a API de um meio de pagamento, baixar imagens de um registry externo. Sem rota para a internet, nada disso funciona.

### 6.2 A solução

O **NAT Gateway** (*Network Address Translation*) permite que recursos em sub-redes privadas **iniciem** conexões com a internet, enquanto continua impossível para a internet iniciar conexões com eles. Ele fica em uma **sub-rede pública** (porque precisa ele próprio chegar à internet), com um Elastic IP, e a tabela de rotas das sub-redes privadas aponta `0.0.0.0/0` para ele.

```
 sub-rede PRIVADA                 sub-rede PÚBLICA
┌────────────────┐             ┌──────────────────┐
│ app 10.0.11.20 │──① sai ───▶│ NAT Gateway      │──② troca a origem para ──▶ Internet
│                │             │ (EIP 54.x.x.x)   │    o IP público do NAT       │
│                │◀── ④ ─────│                  │◀── ③ resposta para 54.x.x.x ─┘
└────────────────┘             └──────────────────┘
       ▲
       ✘  uma conexão NOVA vinda da internet não tem como chegar aqui:
          o NAT só sabe devolver respostas de conexões que ele mesmo abriu.
```

Para o servidor externo, quem fez a requisição foi o IP público do NAT. O endereço da aplicação nunca aparece.

### 6.3 Custo e disponibilidade

O NAT Gateway é um dos itens que mais surpreendem na conta da AWS: ele é cobrado **por hora** (algo em torno de US$ 0,045 por hora em `us-east-1`, um pouco mais em São Paulo), **mesmo parado**, e mais uma taxa **por GB** processado. E ele vive em **uma AZ**: se aquela AZ cair, as sub-redes privadas de todas as AZs que apontam para ele perdem a saída para a internet. Em produção, o padrão é **um NAT Gateway por AZ**, cada sub-rede privada apontando para o NAT da própria zona. Em ambientes de estudo, um só (ou nenhum, como na seção 10) costuma bastar.

> **Confusão comum:** "coloquei um NAT Gateway, então consigo acessar minha instância privada pela internet". ✅ **Mais preciso:** o NAT funciona em **um único sentido**: de dentro para fora. Ele só reconhece pacotes que são **respostas** a conexões que um recurso interno abriu. Uma conexão nova vinda da internet, como um SSH seu, não corresponde a nada que ele conheça e é descartada. Para **receber** tráfego da internet, o recurso precisa estar atrás de algo que fique em uma sub-rede pública e encaminhe para ele: um **load balancer** (para tráfego de usuários) ou, para acesso administrativo, as opções da seção 9.

---

## 7. Security Groups e Network ACLs: os dois firewalls

A VPC tem dois níveis de firewall, que funcionam de formas bem diferentes.

### 7.1 Security Group: o firewall de cada recurso

Um **Security Group** (SG) é associado à **placa de rede de cada recurso**: uma instância, um load balancer, um banco RDS. Ele foi apresentado no [01_ec2_servidores_na_nuvem.md](01_ec2_servidores_na_nuvem.md), e tem três características que importam:

1. **Só tem regras de permissão.** Tudo o que não está explicitamente permitido é bloqueado. Não existe regra de "negar".
2. **É *stateful* (guarda estado).** Se uma conexão de entrada foi permitida, a **resposta** sai automaticamente, sem precisar de regra de saída. O mesmo vale no sentido inverso.
3. **Pode referenciar outros Security Groups.** Em vez de "permitir a porta 5432 para `10.0.11.0/24`", você escreve "permitir a porta 5432 **para qualquer recurso que tenha o SG `sg-app`**".

A terceira característica permite montar uma **cadeia de confiança** que não depende de IPs:

```
 Internet ──443──▶ [ sg-alb ] ──8000──▶ [ sg-app ] ──5432──▶ [ sg-db ]
                    load balancer        aplicação            banco

sg-alb   entrada: 443 de 0.0.0.0/0          ← o mundo pode falar com o LB
sg-app   entrada: 8000 de sg-alb            ← só o LB fala com a aplicação
sg-db    entrada: 5432 de sg-app            ← só a aplicação fala com o banco
```

Se a aplicação escalar de 2 para 20 instâncias, com IPs novos, nenhuma regra precisa mudar: toda instância nova recebe `sg-app` e passa automaticamente a falar com o banco. É a mesma lógica das **NetworkPolicies** do capítulo [Kubernetes: configuração, estado e segurança](../../02-containers/kubernetes/03_configuracao_estado_e_seguranca.md), que selecionam Pods por label em vez de IP.

### 7.2 Network ACL: o firewall de cada sub-rede

Uma **Network ACL** (NACL) é associada a uma **sub-rede inteira** e avalia todo pacote que entra ou sai dela. Ela é diferente do SG em quase tudo:

| | Security Group | Network ACL |
|---|---|---|
| **Aplicado a** | Cada recurso (placa de rede) | Uma sub-rede inteira |
| **Tipos de regra** | Só permitir | Permitir **e negar** |
| **Estado** | *Stateful*: a resposta volta sozinha | ***Stateless***: a resposta precisa de regra própria |
| **Avaliação** | Todas as regras; se alguma permite, passa | Em ordem numérica; a primeira que casar decide |
| **Uso típico** | O controle principal, sempre | Bloqueios amplos: negar uma faixa de IPs maliciosa, isolar sub-redes |

A NACL padrão de uma VPC permite tudo nos dois sentidos, e a maioria das arquiteturas a deixa assim, fazendo todo o controle nos Security Groups. Quando se usa NACLs restritivas, o ponto delicado é o *stateless*.

> **Confusão comum:** "liberei a porta 443 de entrada na NACL, mas os usuários não conseguem acessar o site". ✅ **Mais preciso:** como a NACL não guarda estado, ela trata a **resposta** como um pacote novo, que precisa passar pelas regras de **saída**. E a resposta não sai pela porta 443: ela vai para a **porta efêmera** que o cliente escolheu ao abrir a conexão, um número alto e aleatório (no Linux, entre 32768 e 60999; no Windows, a partir de 49152). Sem uma regra de saída permitindo as portas `1024-65535`, a requisição entra e a resposta é descartada. O cliente vê apenas um *timeout*. No Security Group isso não acontece, porque ele lembra que a conexão foi permitida e libera a resposta automaticamente.

---

## 8. Juntando tudo: uma VPC de três camadas

Com as peças apresentadas, dá para montar a arquitetura que a pergunta do início pedia. Ela segue o desenho de Alex Xu (load balancer, camada web e banco separados, replicados entre data centers) traduzido em rede: três camadas de sub-redes, repetidas em duas AZs.

### 8.1 O plano de endereços

| Camada | sa-east-1a | sa-east-1b | Tabela de rotas | O que roda |
|---|---|---|---|---|
| **Pública** | `10.0.1.0/24` | `10.0.2.0/24` | `0.0.0.0/0 → IGW` | Load balancer, NAT Gateways |
| **Aplicação** (privada) | `10.0.11.0/24` | `10.0.12.0/24` | `0.0.0.0/0 → NAT da mesma AZ` | EC2 da API, nodes do EKS |
| **Dados** (privada) | `10.0.21.0/24` | `10.0.22.0/24` | só a rota `local` | RDS primário e réplica |

A VPC usa `10.0.0.0/16`, e as seis sub-redes ocupam só uma pequena parte dela. Isso é proposital: sobra espaço para criar novas sub-redes depois (outra AZ, outra camada) sem precisar refazer a rede. A camada de dados nem tem rota para o NAT: o banco não tem motivo nenhum para falar com a internet.

### 8.2 O diagrama

```
                               Internet
                                  │
                         ┌────────▼─────────┐
                         │ Internet Gateway │
                         └────────┬─────────┘
┌─────────────────────── VPC 10.0.0.0/16 ┼──────────────────────────────────────┐
│                                        │                                      │
│        AZ sa-east-1a                   │                 AZ sa-east-1b        │
│  ┌─────────────────────────────────────┼──────────────────────────────────┐   │
│  │ PÚBLICAS   10.0.1.0/24     ┌────────▼────────┐        10.0.2.0/24      │   │
│  │  [NAT-a]                   │  Load Balancer  │              [NAT-b]    │   │
│  │                            │  (sg-alb)       │                         │   │
│  └────────────────────────────└───┬─────────┬───┘─────────────────────────┘   │
│                                   │ 8000    │ 8000                            │
│  ┌────────────────────────────────▼──┐  ┌───▼──────────────────────────────┐  │
│  │ APLICAÇÃO 10.0.11.0/24            │  │ APLICAÇÃO 10.0.12.0/24           │  │
│  │   [API]  [API]   (sg-app)         │  │   [API]  [API]   (sg-app)        │  │
│  └────────────────┬──────────────────┘  └──────────────┬───────────────────┘  │
│                   │ 5432                               │ 5432                 │
│  ┌────────────────▼──────────────────┐  ┌──────────────▼───────────────────┐  │
│  │ DADOS 10.0.21.0/24                │  │ DADOS 10.0.22.0/24               │  │
│  │   [RDS primário] (sg-db) ─────────┼──┼──▶ [RDS réplica] (sg-db)         │  │
│  └───────────────────────────────────┘  └──────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────────────┘
```

### 8.3 Seguindo os pacotes

**Uma requisição de usuário:**

1. O usuário acessa `https://loja.exemplo.com`. O DNS aponta para o load balancer.
2. O pacote entra pelo **Internet Gateway** e chega ao load balancer, na sub-rede pública. O `sg-alb` permite a porta 443 de qualquer origem.
3. O load balancer escolhe uma instância da API (em qualquer AZ) e envia para a porta 8000. O pacote viaja pela **rota local**. O `sg-app` aceita, porque a origem tem o `sg-alb`.
4. A API consulta o banco na porta 5432, de novo pela rota local. O `sg-db` aceita, porque a origem tem o `sg-app`.
5. As respostas fazem o caminho inverso, liberadas automaticamente porque os Security Groups são *stateful*.

**A API chamando um serviço externo:**

1. A API abre uma conexão para a API do meio de pagamento.
2. A tabela de rotas da sub-rede de aplicação manda `0.0.0.0/0` para o **NAT** da mesma AZ.
3. O NAT troca a origem para o seu IP público e envia pelo Internet Gateway. A resposta volta pelo mesmo caminho.

**Um atacante tentando falar com o banco:**

Não há caminho. O banco não tem IP público, a sub-rede de dados não tem rota para a internet e o `sg-db` só aceita conexões vindas do `sg-app`. São três barreiras independentes. Essa sobreposição de proteções se chama **defesa em profundidade**: se uma falhar por um erro de configuração, as outras continuam valendo.

---

## 9. Acesso administrativo sem abrir a porta 22

Se as instâncias estão em sub-redes privadas, como acessá-las por SSH para investigar um problema? Há duas abordagens.

**Bastion host (o modelo tradicional).** Uma instância pequena em uma sub-rede pública, com a porta 22 aberta **só para o IP da sua empresa**. Você entra nela e, de lá, salta para as instâncias privadas:

```bash
ssh -J ubuntu@bastion.exemplo.com ubuntu@10.0.11.20
```

Funciona, mas o bastion é uma porta de entrada permanente que precisa ser protegida, atualizada e auditada, e as chaves `.pem` continuam espalhadas entre as pessoas.

**AWS Systems Manager Session Manager (o modelo recomendado hoje).** Um agente que já vem instalado nas AMIs oficiais abre uma conexão **de saída** da instância até o serviço da AWS. Você se conecta pelo console ou pela CLI, e a sessão passa por esse canal:

```bash
aws ssm start-session --target i-0abc123def456
```

Não há porta 22 aberta, nem bastion, nem chave `.pem`. Quem pode abrir uma sessão é decidido por uma **política IAM** (veja o [03_iam_identidades_e_permissoes.md](03_iam_identidades_e_permissoes.md)), e cada sessão fica registrada. A instância precisa de uma **IAM Role** que permita falar com o Systems Manager, e de uma saída para ele (pelo NAT ou por endpoints, seção 10).

---

## 10. Conectando a VPC a outras redes e serviços

### VPC Endpoints: falar com serviços da AWS sem passar pela internet

Por padrão, quando uma instância privada acessa o S3, o tráfego sai pelo NAT, vai à internet e volta para a AWS, pagando a taxa por GB do NAT. Um **VPC Endpoint** cria um caminho **privado** da VPC até o serviço:

| Tipo | Serviços | Como funciona | Custo |
|---|---|---|---|
| **Gateway Endpoint** | S3 e DynamoDB | Uma rota na tabela de rotas | Gratuito |
| **Interface Endpoint** (PrivateLink) | A maioria dos outros (ECR, Secrets Manager, SSM...) | Uma placa de rede com IP privado na sua sub-rede | Por hora e por GB |

Um gateway endpoint para o S3 é uma das otimizações mais simples de fazer: não tem custo e tira de dentro do NAT o tráfego de backups e arquivos. Com endpoints suficientes, é possível ter sub-redes privadas **sem NAT nenhum**, ainda acessando ECR, S3 e SSM.

### Conectando VPCs entre si e à rede da empresa

| Necessidade | Solução | Observação |
|---|---|---|
| Duas VPCs se comunicarem | **VPC Peering** | Não é transitivo: se A↔B e B↔C, A **não** fala com C |
| Muitas VPCs se comunicarem | **Transit Gateway** | Um roteador central, ligado a todas as VPCs |
| A VPC falar com o escritório | **Site-to-Site VPN** (pela internet, cifrada) ou **Direct Connect** (link físico dedicado) | Os CIDRs não podem se sobrepor com a rede da empresa |

A última observação da tabela é o motivo do planejamento da seção 1: a VPC criada às pressas com `192.168.0.0/16` vai colidir com metade das redes domésticas e corporativas no dia em que precisar de uma VPN.

### VPC Flow Logs: o registro do tráfego

Os **Flow Logs** registram cada conexão que passa por uma VPC, sub-rede ou placa de rede: origem, destino, porta, protocolo e se foi **aceita ou rejeitada**. Quando um serviço "não alcança" outro, um registro `REJECT` mostra se o bloqueio veio de um Security Group ou de uma NACL, em vez de deixar você adivinhando.

---

## 11. A VPC como código

Criar uma VPC pelo console envolve dezenas de cliques fáceis de errar. Na prática, ela é descrita com IaC, como o capítulo [Virtualização, containers e IaC](../fundamentos/02_virtualizacao_containers_e_iac.md) defendeu. Um trecho da VPC da seção 8 em Terraform (só uma AZ, para ficar curto):

```hcl
resource "aws_vpc" "principal" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  tags = { Name = "loja-vpc" }
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.principal.id
}

# ── Sub-rede pública ─────────────────────────────────────
resource "aws_subnet" "publica_a" {
  vpc_id            = aws_vpc.principal.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "sa-east-1a"
}

resource "aws_route_table" "publica" {
  vpc_id = aws_vpc.principal.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id    # ← é ISTO que a torna pública
  }
}

resource "aws_route_table_association" "publica_a" {
  subnet_id      = aws_subnet.publica_a.id
  route_table_id = aws_route_table.publica.id
}

# ── NAT Gateway (na sub-rede pública) ───────────────────
resource "aws_eip" "nat_a" {
  domain = "vpc"
}

resource "aws_nat_gateway" "a" {
  allocation_id = aws_eip.nat_a.id
  subnet_id     = aws_subnet.publica_a.id
}

# ── Sub-rede privada de aplicação ───────────────────────
resource "aws_subnet" "app_a" {
  vpc_id            = aws_vpc.principal.id
  cidr_block        = "10.0.11.0/24"
  availability_zone = "sa-east-1a"
}

resource "aws_route_table" "app_a" {
  vpc_id = aws_vpc.principal.id
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.a.id         # ← saída só pelo NAT
  }
}

resource "aws_route_table_association" "app_a" {
  subnet_id      = aws_subnet.app_a.id
  route_table_id = aws_route_table.app_a.id
}

# ── Security Groups em cadeia ───────────────────────────
resource "aws_security_group" "app" {
  vpc_id = aws_vpc.principal.id
}

resource "aws_security_group" "db" {
  vpc_id = aws_vpc.principal.id
  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]   # ← "de quem tem sg-app", não um IP
  }
}
```

Cada bloco corresponde a uma seção deste capítulo, e o arquivo inteiro mostra toda a topologia da rede. Em projetos reais, costuma-se usar o módulo da comunidade `terraform-aws-modules/vpc/aws`, que cria a estrutura completa (sub-redes em várias AZs, NAT por zona, tabelas de rotas) a partir de poucas linhas.

---

## 12. A VPC de um cluster Kubernetes (EKS)

O cluster do capítulo [Kubernetes: do Compose ao cluster](../../02-containers/kubernetes/04_do_compose_ao_cluster.md), levado para o EKS, se encaixa na VPC da seção 8 assim:

- **Os nodes ficam nas sub-redes privadas de aplicação**, distribuídos pelas AZs. Nunca recebem IP público.
- **Os load balancers ficam nas sub-redes públicas.** Quando você cria um Service `LoadBalancer` ou um Ingress (capítulo *Kubernetes: workloads, rede e escala*), o controller da AWS precisa saber em quais sub-redes colocá-lo, e descobre isso por **tags**: `kubernetes.io/role/elb = 1` nas públicas (load balancers voltados para a internet) e `kubernetes.io/role/internal-elb = 1` nas privadas (load balancers internos).
- **Os Pods recebem IPs da própria VPC.** O plugin de rede padrão do EKS, o *Amazon VPC CNI*, dá a cada Pod um IP real da sub-rede, e não de uma rede virtual separada como no kind. Isso simplifica a rede (um Pod é alcançável como qualquer instância, e Security Groups podem ser aplicados a ele), mas **consome endereços muito rápido**: um node pode hospedar dezenas de Pods, e cada um ocupa um IP da sub-rede.

> **Confusão comum:** "um `/24` por AZ é mais do que suficiente para o meu cluster pequeno". ✅ **Mais preciso:** com o VPC CNI, a conta não é "quantos nodes", e sim "quantos **Pods**, mais os IPs que cada node reserva antecipadamente para Pods futuros". Um `/24` tem 251 endereços utilizáveis; dez nodes com 25 Pods cada já esgotam a sub-rede, e a partir daí os Pods novos ficam presos em `ContainerCreating` com erros de alocação de IP, mesmo com CPU e memória sobrando. Para EKS, o padrão é reservar sub-redes de aplicação bem maiores (`/19` ou `/20` por AZ, com milhares de endereços), e esse é mais um motivo para começar a VPC com um `/16` e planejar os endereços antes de criar qualquer coisa.

---

## Próximo passo

A VPC decide **por onde** o tráfego pode passar. Mas metade das interações na AWS não passa por ela: criar uma instância, ler um arquivo do S3, abrir uma sessão no Session Manager, publicar uma imagem no ECR. São chamadas à **API da AWS**, e quem decide se cada uma é permitida não é um Security Group, e sim o sistema de identidades e permissões.

O [03_iam_identidades_e_permissoes.md](03_iam_identidades_e_permissoes.md) explica como esse sistema funciona e fecha a trilha juntando VPC, IAM e EKS.
