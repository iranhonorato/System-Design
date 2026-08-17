# Load Balancer

**Pré-requisito:** `escalabilidade_horizontal_vs_vertical.md`, `../system-design-pt1/threads_e_sockets.md`

> Baseado na seção *Load balancer* do Capítulo 1 de **System Design Interview** (Alex Xu) e no Capítulo 15 (*Space-Based Architecture*) de **Fundamentals of Software Architecture**, que descreve o *messaging grid* — o componente de distribuição de requisições — e observa que ele é "normalmente implementado com um servidor web com capacidade de balanceamento de carga, como HAProxy e Nginx".

---

## Por que este arquivo existe

O arquivo anterior terminou em um impasse: escala horizontal significa N instâncias intercambiáveis, mas o cliente tem **um** endereço para acessar. Alguém precisa ficar no meio e decidir, requisição por requisição, para qual instância cada uma vai.

Esse componente é o **load balancer** — e ele é bem mais do que um distribuidor de tráfego. Na prática, ele é quem **torna a escala horizontal invisível para o cliente**, e quem transforma a queda de um servidor de incidente em não-evento.

`../system-design-pt1/escalabilidade.md`, seção 2, apresentou o conceito e três algoritmos. Aqui a peça é aberta por inteiro: o que ela vê, o que ela consegue decidir com o que vê, como sabe que um servidor está vivo, e por que ela própria não pode ser um ponto único de falha.

---

## 1. O que um load balancer realmente é

Um load balancer é um **proxy reverso que escolhe o destino**. Ele fica na frente de um conjunto de servidores (o *pool*, ou *upstream*), recebe todo o tráfego em um endereço público único, decide para qual membro do pool encaminhar cada requisição, e devolve a resposta ao cliente como se ele mesmo a tivesse produzido.

```
                       ┌──────────────────────┐
  Cliente ───────────▶ │    Load Balancer     │
  (conhece 1 endereço) │  IP público: 1 só    │
                       └──────────┬───────────┘
           ┌──────────────────────┼──────────────────────┐
           ▼                      ▼                      ▼
   ┌───────────────┐      ┌───────────────┐      ┌───────────────┐
   │  Instância 1  │      │  Instância 2  │      │  Instância 3  │
   │  10.0.1.11    │      │  10.0.1.12    │      │  10.0.1.13    │
   │  (IP privado) │      │  (IP privado) │      │  (IP privado) │
   └───────────────┘      └───────────────┘      └───────────────┘
        rede privada — inalcançável diretamente pela internet
```

Repare em um detalhe que Alex Xu destaca e que costuma passar batido: **as instâncias usam IPs privados**. Depois que o load balancer entra em cena, os servidores de aplicação deixam de ser acessíveis pela internet. Isso não é um efeito colateral — é um dos benefícios: a superfície de ataque exposta cai de N máquinas para uma.

### Proxy reverso vs. proxy direto

O termo "proxy" aparece nos dois sentidos, e a diferença é **de quem ele age em nome**:

```
PROXY DIRETO (forward proxy)          PROXY REVERSO (reverse proxy)

 Cliente ──▶ [Proxy] ──▶ Internet      Internet ──▶ [Proxy] ──▶ Servidores

 Age em nome do CLIENTE.               Age em nome do SERVIDOR.
 O servidor não sabe quem              O cliente não sabe quantos
 é o cliente real.                     servidores existem.
 Ex.: proxy corporativo, VPN.          Ex.: load balancer, CDN, Nginx.
```

> **Confusão comum:** "load balancer e proxy reverso são a mesma coisa". ✅ **Mais preciso:** todo load balancer é um proxy reverso, mas nem todo proxy reverso é um load balancer. **Proxy reverso** descreve a *posição* (fica na frente de servidores, em nome deles) e traz benefícios que existem mesmo com **um único** backend: terminação de TLS, cache, compressão, ocultação da topologia interna, servir arquivos estáticos. **Load balancer** descreve uma *função* específica — distribuir entre **múltiplos** backends equivalentes. Um Nginx na frente de uma única aplicação é um proxy reverso pleno e não está balanceando nada. A distinção importa porque explica por que uma equipe com um só servidor ainda ganha bastante colocando um proxy reverso na frente.

---

## 2. As quatro funções, além de distribuir

Distribuir tráfego é a função que dá nome ao componente, mas normalmente é a menos crítica das quatro:

| Função | O que acontece sem ela |
|---|---|
| **Distribuição** | Uma instância satura enquanto as outras ficam ociosas |
| **Failover** | Uma instância morta continua recebendo tráfego → erros para uma fração dos usuários |
| **Escala transparente** | Adicionar uma instância exigiria mudar algo do lado do cliente |
| **Isolamento de rede** | Cada instância precisaria de IP público e de suas próprias regras de firewall |

O **failover** merece destaque: é a função que converte a queda de uma máquina em um evento invisível. O load balancer testa continuamente a saúde de cada membro do pool (seção 5) e, ao detectar uma falha, simplesmente para de rotear para ela. Nenhum usuário vê erro; nenhum humano precisa ser acordado às 3h.

A **escala transparente** é o que fecha o ciclo do arquivo anterior: como Alex Xu resume, se o tráfego crescer, "basta adicionar mais servidores ao pool, e o load balancer automaticamente começa a enviar requisições para eles" — sem nenhuma alteração no cliente, no DNS ou na aplicação.

---

## 3. Camada 4 vs. Camada 7

Esta é a distinção mais importante do arquivo, e ela decorre diretamente do modelo de camadas visto em redes: **um load balancer só consegue decidir com base no que ele consegue enxergar**, e o que ele enxerga depende de até que camada ele decodifica o pacote.

```
              O QUE CADA UM VÊ DE UMA MESMA REQUISIÇÃO

  ┌─────────────────────────────────────────────────────────┐
  │ POST /api/pagamentos/123  HTTP/1.1                       │  ← só o L7 vê
  │ Host: loja.com                                           │     esta parte
  │ Authorization: Bearer eyJhbGc...                         │
  │ Cookie: sessao=abc123                                    │
  │ { "valor": 199.90 }                                      │
  ├─────────────────────────────────────────────────────────┤
  │ TCP  porta origem: 51234 → porta destino: 443            │  ← o L4 vê
  │ IP   origem: 200.1.2.3  → destino: 10.0.1.11             │     só isto
  └─────────────────────────────────────────────────────────┘
```

| | **L4 (transporte)** | **L7 (aplicação)** |
|---|---|---|
| **Decide com base em** | IP e porta | Path, método, headers, cookies, corpo |
| **Entende o protocolo?** | Não — repassa bytes | Sim — decodifica HTTP inteiro |
| **Custo por requisição** | Muito baixo | Maior (parsing, buffering) |
| **Granularidade** | Por **conexão** | Por **requisição** |
| **Termina TLS?** | Normalmente não (repassa cifrado) | Sim — precisa decifrar para ler |
| **Funciona com** | Qualquer protocolo TCP/UDP | Protocolos que ele saiba interpretar |
| **Exemplos** | AWS NLB, HAProxy em modo TCP, IPVS | AWS ALB, Nginx, HAProxy em modo HTTP, Envoy, Traefik |

**A consequência prática mais importante** é a granularidade. Um LB de camada 4 escolhe um destino **quando a conexão é aberta**, e todas as requisições daquela conexão vão para o mesmo servidor — o que importa muito, porque HTTP/1.1 usa conexões persistentes e HTTP/2 multiplexa centenas de requisições em uma única conexão (ver `../system-design-pt1/threads_e_sockets.md`). Um LB de camada 7 decide **a cada requisição**, e por isso distribui de forma muito mais uniforme sob esses protocolos.

E há decisões que simplesmente **não são expressáveis** em L4, porque a informação não está visível naquela camada:

```
Roteamento por path       → /api/pagamentos/* vai para o serviço de pagamentos
Roteamento por header     → X-Versao: beta vai para o pool canário
Roteamento por cookie     → usuários com cookie de teste A/B vão para o pool B
Reescrita de URL          → /v1/users vira /users no backend
Retry de requisição       → repetir um GET que falhou em outro servidor
```

> **Confusão comum:** "camada 7 é melhor que camada 4, porque é uma camada mais alta". ✅ **Mais preciso:** camada mais alta significa **mais informação disponível para decidir**, e mais informação custa mais trabalho por requisição. Um LB de camada 4 não é uma versão inferior — é um componente com propriedades diferentes: sustenta um volume muito maior de conexões com menos recursos, preserva o IP de origem com mais facilidade, funciona com **qualquer** protocolo sobre TCP (um banco de dados, um servidor de jogos, MQTT, gRPC bruto) e nunca precisa decifrar o TLS — o tráfego permanece criptografado de ponta a ponta até a aplicação. Camada 7 é escolha certa quando as decisões dependem do conteúdo HTTP; camada 4 é escolha certa quando não dependem, ou quando o protocolo nem é HTTP. Arquiteturas grandes usam os dois **empilhados**: um NLB de camada 4 na borda absorvendo volume, e um ALB ou Nginx de camada 7 atrás dele tomando as decisões de roteamento.

> **Confusão comum:** "um load balancer de camada 4 é mais seguro porque não decifra o TLS". ✅ **Mais preciso:** não decifrar significa que aquele componente **não pode inspecionar** o tráfego — o que corta possibilidades nos dois sentidos. Ele de fato não é um ponto onde os dados aparecem em texto claro, mas também não consegue aplicar WAF, bloquear por padrão de requisição, validar token, nem sequer registrar qual URL foi chamada. Além disso, a criptografia continua terminando em algum lugar: se ela termina em cada instância da aplicação, o certificado precisa estar em N máquinas, e a renovação vira um problema distribuído. Terminar TLS no LB (*TLS termination*) centraliza a gestão de certificados e habilita a inspeção; re-cifrar a partir dali até o backend (*TLS re-encryption*) recupera a confidencialidade interna ao custo de mais CPU. São três desenhos com trade-offs distintos, não uma escala de "mais seguro" para "menos seguro".

---

## 4. Algoritmos de balanceamento

| Algoritmo | Como decide | Bom para | Cuidado |
|---|---|---|---|
| **Round Robin** | Sequência circular: 1, 2, 3, 1, 2, 3... | Servidores homogêneos e requisições de custo parecido | Ignora completamente a carga real de cada servidor |
| **Weighted Round Robin** | Round robin com pesos (peso 3 recebe 3× mais) | Pool heterogêneo (máquinas de tamanhos diferentes), *canary deploy* | Pesos são estáticos: não reagem à realidade |
| **Least Connections** | Envia para quem tem menos conexões ativas | Requisições de duração muito variável | Conexão aberta ≠ trabalho; conexões ociosas contam igual |
| **Least Response Time** | Menos conexões **e** menor latência média recente | Melhor aproximação de "quem está menos ocupado" | Pode concentrar tráfego em um servidor que responde rápido *por estar falhando rápido* |
| **IP Hash** | `hash(IP do cliente)` decide o destino, sempre o mesmo | Afinidade sem cookie; funciona em L4 | Clientes atrás de um mesmo NAT caem todos no mesmo servidor |
| **Consistent Hash** | Anel de hash sobre uma chave (IP, header, path) | Cache local por servidor; minimiza remapeamento ao mudar o pool | Ver `../system-design-pt1/escalabilidade.md`, seção 5 |
| **Random / Two Choices** | Sorteia dois servidores e manda para o menos carregado | Excelente equilíbrio a custo quase zero, em escala | Menos previsível para depurar |
| **Least Bandwidth / Least Time** | Menor tráfego ou menor tempo total servido | Streaming, downloads grandes | Requer contabilidade adicional |

O algoritmo **Two Random Choices** é digno de nota: sortear dois candidatos ao acaso e escolher o menos carregado entre eles produz uma distribuição drasticamente melhor que o sorteio simples, sem exigir estado global sobre todos os servidores. É o algoritmo padrão em vários balanceadores modernos justamente por esse custo/benefício.

> **Confusão comum:** "round robin distribui a carga igualmente entre os servidores". ✅ **Mais preciso:** round robin distribui **requisições** igualmente, e requisições não têm custo igual. Se um servidor receber, por sorte da rotação, três relatórios pesados de 8 segundos enquanto o vizinho recebe três consultas de 20 ms, os dois receberam "a mesma carga" pela contabilidade do round robin e cargas absurdamente diferentes pela contabilidade da CPU. É por isso que `least_conn` e suas variantes existem: elas aproximam a **ocupação real** em vez do **número de despachos**. Round robin continua sendo um padrão perfeitamente razoável quando as requisições são homogêneas — o erro é assumir homogeneidade sem verificar.

> **Confusão comum:** "*least connections* manda a requisição para o servidor menos ocupado". ✅ **Mais preciso:** manda para o servidor com **menos conexões abertas**, que é uma aproximação — às vezes ruim — de ocupação. Um servidor pode ter 200 conexões ociosas em *keep-alive* consumindo quase nada de CPU, enquanto outro tem 5 conexões executando consultas pesadíssimas. Existe ainda um efeito perverso conhecido: um servidor que entrou em falha e passou a **recusar conexões instantaneamente** aparece como o menos carregado do pool, e passa a atrair tráfego desproporcional — o chamado problema do *"servidor mais rápido a falhar"*. Health checks bem configurados (seção 5) são o que impede esse cenário.

---

## 5. Health checks: como o LB sabe quem está vivo

Todo o valor de failover depende dessa mecânica, e ela é mais sutil do que parece.

**Health check ativo** — o LB envia periodicamente uma requisição própria a cada membro do pool:

```
A cada 5s:   GET /health  ──▶  Instância 2
             ◀── 200 OK              → saudável, continua no pool
             ◀── 503 / timeout       → falha contabilizada

Após 3 falhas consecutivas  →  removido do pool
Após 2 sucessos consecutivos →  devolvido ao pool
```

**Health check passivo** — o LB não testa nada por conta própria; ele observa o resultado do **tráfego real** e marca como indisponível o servidor que acumular erros ou timeouts em uma janela de tempo. É mais barato (zero requisições extras) e mais lento para detectar, porque exige que usuários reais tomem erro primeiro. É o único modo disponível no Nginx open source (ver `nginx.md`, seção 6).

Os parâmetros que importam são sempre os mesmos três, e a calibração é um trade-off direto:

| Parâmetro | Curto demais | Longo demais |
|---|---|---|
| **Intervalo** | Carga extra e ruído | Servidor morto atende usuários por muito tempo |
| **Limiar de falhas** | *Flapping*: servidor entra e sai do pool a cada oscilação | Detecção lenta |
| **Timeout** | Servidor lento porém sadio é expulso, agravando a carga nos demais | Cliente espera pelo timeout junto com o LB |

> **Confusão comum:** "o health check está verde, então o servidor está saudável". ✅ **Mais preciso:** o health check afirma exatamente uma coisa — que **aquele endpoint específico** respondeu. Um `/health` que apenas retorna `{"status":"ok"}` sem tocar em nada prova somente que o processo está de pé e aceitando conexões; ele continuará verde com o banco de dados fora do ar, o Redis inacessível, o disco cheio e o pool de conexões esgotado. Enquanto isso, 100% das requisições reais falham e o LB segue roteando tráfego confiante. A correção é um *health check* que exercite as dependências críticas — mas com uma ressalva importante, na confusão seguinte.

> **Confusão comum:** "então o health check deve verificar todas as dependências, quanto mais completo melhor". ✅ **Mais preciso:** um health check profundo demais transforma uma falha parcial em queda total. Se o `/health` de todas as instâncias consulta o mesmo banco, uma lentidão momentânea nesse banco reprova **todas** as instâncias ao mesmo tempo — o LB esvazia o pool inteiro e o sistema cai 100%, quando na verdade apenas as rotas que dependiam daquele banco estavam degradadas. A prática consolidada é separar dois endpoints com semânticas diferentes: **`liveness`** ("o processo está sadio ou precisa ser reiniciado?" — checagem rasa) e **`readiness`** ("esta instância está pronta para receber tráfego agora?" — checagem das dependências de que ela realmente precisa, com timeout curto e tolerância a falhas parciais). É a distinção que o Kubernetes formalizou, e ela vale para qualquer load balancer.

---

## 6. O problema da sessão

Já visto no arquivo anterior pela ótica da aplicação; aqui pela ótica do LB, que é quem oferece o mecanismo:

| Mecanismo | Como funciona | Limitação |
|---|---|---|
| **IP Hash** | `hash(IP)` fixa o destino | Quebra quando o IP do cliente muda (rede móvel); agrupa todos atrás de um NAT |
| **Cookie de afinidade** | O LB injeta um cookie próprio identificando o servidor escolhido | Exige L7; cliente precisa aceitar cookies |
| **Sessão externalizada** | O LB não faz nada — a aplicação busca a sessão no Redis | Requer mudar a aplicação; é a solução de fato |

O ponto: **afinidade de sessão é uma configuração do load balancer para compensar uma limitação da aplicação**. Ela funciona, tem custo, e mantém intactos os problemas listados no arquivo anterior — perda de sessão quando o servidor cai, desbalanceamento, e atrito com *auto scaling*.

---

## 7. Balanceamento acontece em várias camadas ao mesmo tempo

Um pedido de página em um sistema grande é balanceado várias vezes antes de chegar ao código de negócio:

```
              Usuário digita loja.com
                        │
                        ▼
   ① DNS  ─────────────────────────────  devolve IPs diferentes por
      (round robin de DNS, GeoDNS)       região ou em rotação
                        │
                        ▼
   ② Balanceamento de rede (Anycast) ──  o mesmo IP anunciado em vários
                                          datacenters; a rota mais curta vence
                        │
                        ▼
   ③ Load balancer de borda (L4) ───────  absorve volume, protege a rede
                        │
                        ▼
   ④ Load balancer / gateway (L7) ──────  decide por path, header, cookie
                        │
                        ▼
   ⑤ Balanceamento interno ─────────────  entre réplicas de cada serviço
      (service discovery / sidecar)       — o "leste-oeste" do service mesh
                        │
                        ▼
   ⑥ Réplicas de leitura do banco ──────  arquitetura_master_slave.md
```

O **round robin de DNS** (nível ①) merece um comentário, porque é a forma mais antiga e mais mal compreendida de balanceamento: o servidor DNS devolve vários registros `A` para o mesmo nome e alterna a ordem entre as respostas. É gratuito e não exige componente algum — mas o DNS **não conhece a saúde de nada**: um servidor morto continua sendo devolvido até alguém editar a zona, e o cache de DNS em cada camada (navegador, sistema operacional, resolvedor do provedor) faz com que a correção demore o TTL inteiro para propagar. Serve como distribuição geográfica grosseira; não serve como mecanismo de failover.

---

## 8. Quem balanceia o balanceador?

O load balancer resolve o SPOF da camada de aplicação e, ingenuamente configurado, se torna o novo SPOF do sistema. As três respostas padrão:

**Ativo/passivo com IP virtual (VIP)** — dois LBs, um ativo e um em espera, compartilhando um endereço IP flutuante. O passivo monitora o ativo por *heartbeat* e assume o VIP se ele parar de responder (é o que fazem protocolos como VRRP, via `keepalived`).

```
            VIP: 203.0.113.10   ← o IP que o DNS aponta
                    │
        ┌───────────┴───────────┐
        ▼                       ▼
   ┌─────────┐  heartbeat  ┌─────────┐
   │ LB-1    │ ◀─────────▶ │ LB-2    │
   │ ATIVO   │             │ PASSIVO │  ← assume o VIP se o LB-1 sumir
   └─────────┘             └─────────┘
```

**Ativo/ativo com DNS ou Anycast** — vários LBs recebendo tráfego simultaneamente, com a distribuição feita no nível de DNS ou de roteamento IP. Aproveita todo o hardware, mas depende de uma camada acima para retirar de circulação um LB defeituoso.

**Load balancer gerenciado** — a opção da nuvem (AWS ELB/ALB/NLB, GCP Cloud Load Balancing, Azure Load Balancer). O provedor opera um serviço que já é internamente redundante e elástico; você configura o pool e as regras. É o motivo pelo qual essa preocupação praticamente desapareceu da rotina da maioria das equipes — mas o SPOF não sumiu, apenas foi terceirizado, e vale saber disso quando o provedor tem um incidente regional.

> **Confusão comum:** "com um load balancer na frente, meu sistema tem alta disponibilidade". ✅ **Mais preciso:** o load balancer **remove** o SPOF das instâncias de aplicação e **cria** um novo em si mesmo, a menos que ele também seja redundante. E mesmo com o LB redundante, o caminho segue tão disponível quanto seu elo mais fraco: um banco de dados único atrás de três instâncias e dois LBs continua derrubando o sistema inteiro quando cai. Alta disponibilidade é uma propriedade do **caminho completo** da requisição, não de um componente.

> **Confusão comum:** "o load balancer protege contra DDoS". ✅ **Mais preciso:** ele ajuda de duas formas indiretas — absorve mais tráfego do que uma instância sozinha e esconde os IPs reais dos servidores — mas não é uma defesa contra DDoS. Um ataque volumétrico satura o próprio LB (ou o link de rede antes dele), e um ataque de camada 7 (requisições legítimas em aparência, caras de processar) passa direto e chega à aplicação. Defesa real contra DDoS exige capacidade de absorção na borda da internet e filtragem por comportamento: CDN/scrubbing, WAF, rate limiting agressivo (`../system-design-pt1/escalabilidade.md`, seção 6), *challenge* de cliente. O LB é onde algumas dessas defesas são **aplicadas** — não é a defesa em si.

---

## 9. Implementações

| Categoria | Exemplos | Característica |
|---|---|---|
| **Software, L7** | Nginx, HAProxy, Envoy, Traefik, Caddy | Roda em qualquer máquina; máxima flexibilidade de configuração |
| **Software, L4** | HAProxy (modo TCP), IPVS/LVS, Nginx (`stream`) | Altíssimo volume, protocolo-agnóstico |
| **Gerenciado (nuvem)** | AWS ALB (L7) / NLB (L4), GCP LB, Azure LB | Redundância e elasticidade inclusas; menos controle fino |
| **Hardware** | F5 BIG-IP, Citrix ADC | Desempenho extremo; caro; comum em ambientes on-premise legados |
| **Client-side** | gRPC balancer, Ribbon, service mesh (`sidecar`) | Sem salto extra na rede; exige *service discovery* e lógica no cliente |

O **balanceamento client-side** é conceitualmente o mais diferente: em vez de um intermediário, o próprio chamador descobre a lista de instâncias disponíveis (via *service discovery*) e escolhe uma. Elimina um salto de rede e o gargalo central, ao custo de embutir a lógica de balanceamento em cada cliente — problema que o padrão *Sidecar* resolve ao delegar essa lógica a um proxy local, exatamente como descrito em `api_gateway.md`, seção 1.

---

## Resumo do arquivo

- Um load balancer é um **proxy reverso que escolhe o destino**. Todo LB é um proxy reverso; nem todo proxy reverso balanceia.
- Ele entrega quatro coisas — **distribuição, failover, escala transparente e isolamento de rede**. O failover costuma valer mais que a distribuição.
- **L4 vs. L7 é sobre o que o componente enxerga**: L4 decide por IP/porta, por conexão, e é rápido e protocolo-agnóstico; L7 decide por path/header/cookie, por requisição, e habilita roteamento de conteúdo ao custo de decodificar tudo. Não é uma escala de qualidade — arquiteturas grandes empilham os dois.
- **Round robin distribui requisições, não carga.** `least_conn` aproxima ocupação, e `two random choices` entrega quase o mesmo resultado com custo muito menor.
- O **health check** é o que sustenta o failover. Raso demais mantém servidores quebrados no pool; profundo demais transforma uma falha de dependência em queda total. A separação `liveness` / `readiness` resolve.
- **Afinidade de sessão é um remendo do LB para uma limitação da aplicação** — funciona, mas preserva os problemas de fundo.
- O balanceamento acontece em **várias camadas simultâneas** (DNS, Anycast, borda, aplicação, banco), cada uma com uma granularidade e um mecanismo de failover próprios.
- O LB **é um SPOF** até que seja explicitamente redundante (ativo/passivo com VIP, ativo/ativo, ou gerenciado pela nuvem).

**Próxima leitura:** `nginx.md` — a implementação concreta mais difundida de tudo o que este arquivo descreveu em abstrato, e o produto que costuma ser o primeiro contato de qualquer pessoa com proxy reverso e balanceamento.
