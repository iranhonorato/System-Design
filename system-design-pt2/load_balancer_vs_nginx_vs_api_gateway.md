# Load Balancer × Nginx × API Gateway: diferenças e semelhanças

**Pré-requisito:** `load_balancer.md`, `nginx.md`, `api_gateway.md`

> Síntese dos três arquivos anteriores. O vocabulário de *acoplamento ortogonal* vem de **Software Architecture: The Hard Parts** (Cap. 8) e o de **camada de API opcional** vem de **Fundamentals of Software Architecture** (Cap. 17).

---

## Por que este arquivo existe

Esta é provavelmente a dúvida mais recorrente de quem estuda arquitetura de borda, e ela costuma ser formulada assim: *"qual dos três eu devo usar?"*

A resposta curta é que **a pergunta está mal formada** — e entender por quê resolve a confusão de forma definitiva, muito melhor do que decorar uma tabela de diferenças.

---

## 1. O erro de categoria

Os três termos não são três opções da mesma lista. Eles pertencem a **categorias diferentes de coisa**:

```
   LOAD BALANCER          API GATEWAY              NGINX
        │                      │                     │
        ▼                      ▼                     ▼
    uma FUNÇÃO           um PAPEL             um PRODUTO
   (o que precisa      ARQUITETURAL          (o software que
    acontecer)        (a posição e o          você instala e
                    conjunto de respon-        configura)
                     sabilidades na
                       arquitetura)
```

Uma analogia torna isso imediato:

| | Analogia | Componente |
|---|---|---|
| **Função** | "cortar" | balancear carga |
| **Papel** | "utensílio de cozinha" | API Gateway |
| **Produto** | "canivete suíço" | Nginx |

Perguntar "devo usar cortar, utensílio de cozinha ou canivete suíço?" não faz sentido — e é literalmente a estrutura da pergunta original. O canivete suíço **corta** (exerce a função) e **pode servir** como utensílio de cozinha (desempenhar o papel), enquanto continua sendo um objeto de uma categoria distinta de ambos.

Traduzindo de volta:

- **Nginx pode exercer a função de load balancer.** (E exerce, muito bem.)
- **Nginx pode desempenhar o papel de API Gateway.** (Parcialmente — a seção 5 diz até onde.)
- **Um API Gateway quase sempre exerce a função de load balancer** internamente, ao escolher entre réplicas de um serviço.
- **Um load balancer não desempenha o papel de gateway**, porque o papel inclui responsabilidades que a função não contempla.

> **Confusão comum:** "preciso escolher entre load balancer, Nginx e API Gateway para a borda do meu sistema". ✅ **Mais preciso:** os três não são alternativas concorrentes — são respostas a perguntas diferentes, e sistemas reais frequentemente têm os três ao mesmo tempo. A sequência correta de perguntas é: **(1)** *quais funções eu preciso?* (distribuir carga, autenticar, limitar taxa, servir estáticos, terminar TLS); **(2)** *que papéis arquiteturais isso configura?* (só balanceamento, ou uma camada de API completa); **(3)** *qual produto implementa esses papéis com o menor custo operacional para minha equipe?* Escolher o produto primeiro é o que produz respostas como "usamos Kong" para um sistema que precisava apenas de um proxy reverso.

---

## 2. Os três lado a lado

| | **Load Balancer** | **Nginx** | **API Gateway** |
|---|---|---|---|
| **Categoria** | Função / papel de infraestrutura | Produto (software) | Papel arquitetural |
| **Pergunta que responde** | Para qual das réplicas envio? | O que eu instalo? | Como exponho minha API ao mundo? |
| **Camada típica** | L4 ou L7 | L4 (`stream`) e L7 (`http`) | Sempre L7 |
| **Unidade de decisão** | Instância dentro de um pool | Depende de como for configurado | Serviço / rota / consumidor |
| **Conhece o cliente?** | Não — trata todos igualmente | Não, por padrão | **Sim** — identidade, plano, quota |
| **Conhece o serviço?** | Não — só endereços do pool | Só o que a config declarar | **Sim** — contrato, versão, dono |
| **Estado** | Mínimo (saúde do pool) | Mínimo (+ cache, se ativado) | Considerável (chaves, quotas, contadores) |
| **Latência adicionada** | Muito baixa | Muito baixa | Baixa a moderada (auth, quota, plugins) |
| **Motivação** | Capacidade e disponibilidade | — (é a ferramenta) | Governança e acoplamento ortogonal |
| **Exemplos** | AWS NLB/ALB, HAProxy, IPVS | Nginx, NGINX Plus, OpenResty | Kong, AWS API Gateway, Apigee, Tyk |

---

## 3. O que se sobrepõe

A confusão não é gratuita: existe um núcleo comum genuíno entre os três. Todos são, no fundo, **proxies reversos** — e por isso herdam o mesmo conjunto de capacidades básicas.

```
   ┌─────────────────────────────────────────────────────────────┐
   │                                                              │
   │   FUNÇÃO: LOAD BALANCER          PAPEL: API GATEWAY          │
   │   ┌──────────────────┐          ┌──────────────────┐        │
   │   │ • distribuir      │          │ • auth de token   │        │
   │   │   entre réplicas  │          │ • quota por       │        │
   │   │ • health check    │  ┌────┐  │   consumidor      │        │
   │   │ • failover        │  │NÚCLEO│ │ • versionamento   │        │
   │   │ • L4 puro         │  │COMUM │ │ • transformação   │        │
   │   │ • protocolos não  │  │      │ │ • portal / chaves │        │
   │   │   HTTP            │  └────┘  │ • circuit breaker │        │
   │   └──────────────────┘     ▲     └──────────────────┘        │
   │                            │                                  │
   │              proxy reverso │ roteamento                       │
   │              TLS           │ rate limiting                    │
   │              observabilidade                                  │
   │                                                              │
   │   PRODUTO: NGINX ── implementa TODO o núcleo comum,          │
   │            toda a coluna da esquerda, e PARTE da direita      │
   └─────────────────────────────────────────────────────────────┘
```

O núcleo comum — proxy reverso, roteamento, terminação de TLS, rate limiting básico, logs — é exatamente o que faz os três parecerem intercambiáveis quando se olha só para uma configuração simples. A diferença aparece nas bordas do diagrama.

---

## 4. O que só um deles faz

**Só o load balancer (na sua forma L4):**
- Balancear tráfego que **não é HTTP**: um banco de dados, um servidor de jogos, MQTT, SMTP, gRPC bruto.
- Sustentar volumes de conexão em que decodificar HTTP seria proibitivo.
- Preservar a criptografia ponta a ponta **sem** decifrar nada no caminho.

**Só o API Gateway:**
- **Conhecer o consumidor** — identidade, chave de API, plano contratado, cota mensal. Um load balancer não tem noção de "quem", só de "quanto".
- **Governança de contrato** — versionar, depreciar, documentar, publicar em um portal.
- **Monetização e analytics por cliente** — quantas chamadas o cliente X fez ao endpoint Y neste mês.
- **Transformação semântica** — expor REST e falar gRPC internamente, remapear campos, agregar (com as ressalvas de `api_gateway.md`, seção 4).
- **Catálogo de plugins de autenticação** — OAuth2, OIDC, JWT, mTLS, HMAC, chave de API, prontos e configuráveis.

**Só o Nginx (entre os produtos citados):**
- **Servir arquivos estáticos direto do disco** com `sendfile`. Nem load balancer nem gateway fazem isso.
- **Cache HTTP de respostas** com `proxy_cache`, incluindo servir conteúdo vencido quando o backend cai.
- **Falar FastCGI/uwsgi/SCGI** com servidores de aplicação (php-fpm, por exemplo).
- Ser, ao mesmo tempo, todas as peças anteriores em **um único processo e um único arquivo de configuração**.

---

## 5. Quando o Nginx basta como gateway — e quando deixa de bastar

Como o Nginx cobre o núcleo comum e boa parte das funções de gateway, uma pergunta prática legítima é: *quando um `nginx.conf` bem escrito é suficiente?*

**Basta quando:**

| Situação | Por quê |
|---|---|
| Poucos serviços, um consumidor (seu próprio frontend) | Não há governança de terceiros a fazer |
| Autenticação simples (validar JWT com chave pública) | Resolvível com `auth_request` ou um módulo |
| Rate limiting por IP | `limit_req` resolve — é Leaky Bucket pronto |
| As rotas mudam com pouca frequência | Editar e recarregar configuração é aceitável |
| A equipe já opera Nginx | Custo operacional marginal ≈ zero |

**Deixa de bastar quando aparece algum destes sinais:**

- **Consumidores externos com identidade própria** — parceiros, clientes B2B, apps de terceiros. Chave por consumidor, quota por plano e revogação individual não são expressáveis em `nginx.conf` sem construir um sistema paralelo.
- **A configuração vira código de negócio.** Quando o arquivo acumula blocos `if`, `map` e reescritas condicionais implementando regra de domínio, a ferramenta foi ultrapassada — e o resultado é um dos formatos de configuração mais difíceis de testar que existem.
- **Cada equipe precisa publicar as próprias rotas.** Configuração centralizada em um arquivo editado por uma equipe é o gargalo organizacional descrito em `api_gateway.md`, seção 6. Gateways oferecem configuração declarativa por rota, com API própria.
- **É preciso mudar política sem reload.** Nginx open source aplica configuração via reload; gateways alteram rotas e políticas em tempo de execução, via API.
- **Health check ativo é requisito.** Recurso de NGINX Plus (ver `nginx.md`, seção 6); presente de fábrica em Kong, Envoy, Traefik e nos balanceadores gerenciados.
- **Autenticação além do básico** — OAuth2 completo, introspecção de token, mTLS por consumidor, rotação de chaves.

> **Confusão comum:** "Nginx é a versão gratuita e limitada de um API Gateway". ✅ **Mais preciso:** eles não estão na mesma escala. Nginx é um **produto de propósito geral** que cobre o núcleo de proxy reverso com desempenho excepcional; API Gateway é um **papel arquitetural** que inclui governança de API — gestão de consumidores, contratos, quotas e ciclo de vida. Tanto que **o Kong é literalmente construído sobre o Nginx** (via OpenResty): não é "Nginx ou Kong", é "Nginx puro ou Nginx com uma camada de gestão de API por cima". A pergunta certa não é "qual é mais completo", é **"eu tenho um problema de governança de API, ou só de roteamento e balanceamento?"** — porque no segundo caso a camada extra é custo sem contrapartida.

> **Confusão comum:** "um load balancer L7 é a mesma coisa que um API Gateway, já que os dois roteiam por path e headers". ✅ **Mais preciso:** os dois roteiam por conteúdo HTTP, mas a **unidade de raciocínio** de cada um é diferente. O LB L7 pensa em **instâncias**: dado um pool de réplicas equivalentes, qual delas atende esta requisição? O gateway pensa em **APIs e consumidores**: qual serviço responde por esta rota, quem é este cliente, ele tem permissão, já estourou a cota, esta versão do contrato ainda é suportada? Um ALB da AWS roteia `/api/*` para um *target group* — e não tem a menor ideia de que existe um "cliente parceiro X com plano premium". Essa noção é justamente o que caracteriza o papel de gateway.

---

## 6. Como os três coexistem em um sistema real

Nada aqui é hipotético — esta é a topologia típica de um sistema de médio a grande porte:

```
                         Internet
                             │
                             ▼
   ①  DNS / CDN                          distribuição geográfica,
      (GeoDNS, Cloudflare, CloudFront)    cache de estáticos na borda
                             │
                             ▼
   ②  Load Balancer L4 gerenciado         absorve volume, protege a rede,
      (AWS NLB, Anycast)                  IP público único e estável
                             │
                             ▼
   ③  API Gateway  ◀── PAPEL              auth, quota por consumidor,
      (Kong, AWS API GW, ou Nginx)         roteamento por API, versão,
      ── frequentemente rodando            observabilidade, circuit breaker
         VÁRIAS instâncias, que
         por isso precisam do ②
                             │
        ┌────────────────────┼────────────────────┐
        ▼                    ▼                    ▼
   ④  LB interno         LB interno           LB interno   ◀── FUNÇÃO
      (ou service           (ou sidecar do        ...           embutida
       discovery)            service mesh)
        │                    │                    │
     ┌──┴──┬─────┐        ┌──┴──┬─────┐        ┌──┴──┬─────┐
     ▼     ▼     ▼        ▼     ▼     ▼        ▼     ▼     ▼
   [Usuários ×3]        [Pedidos ×3]         [Produtos ×3]
        │                    │                    │
        ▼                    ▼                    ▼
   ⑤  Réplicas de leitura do banco ── arquitetura_master_slave.md
```

Observações que amarram os três arquivos anteriores:

- **O balanceamento aparece três vezes** (②, ④ e ⑤), em camadas e granularidades diferentes. É uma **função**, e funções se repetem onde forem necessárias.
- **O gateway (③) precisa de um load balancer na frente dele.** Ele é *stateless*, roda em várias instâncias e é o SPOF mais crítico do desenho — logo, ele próprio precisa ser balanceado. Isso encerra de vez a ideia de que "se tenho gateway, não preciso de load balancer".
- **O Nginx pode ocupar ②, ③ ou ④** — ou os três, em processos distintos. É um produto; a posição no diagrama é decisão de configuração.
- **Sistemas menores colapsam camadas.** Um sistema com três serviços e um frontend pode ter apenas `DNS → LB gerenciado → Nginx (gateway + balanceamento) → serviços`. Isso é uma simplificação legítima, não uma arquitetura incompleta.

> **Confusão comum:** "se eu tenho um API Gateway, não preciso de load balancer". ✅ **Mais preciso:** o gateway **exerce** a função de balancear ao escolher entre as réplicas de cada serviço — mas ele mesmo roda em várias instâncias e precisa de um balanceador **na frente**, ou vira o ponto único de falha de todo o sistema. Portanto, um sistema com gateway normalmente tem balanceamento em **duas** posições: antes do gateway (para o gateway ser redundante) e depois dele (para os serviços serem redundantes). A ideia de substituição vem de confundir função com produto: o que o gateway substitui, quando substitui, é a **caixa** de LB dedicado logo atrás dele — nunca a função em si.

> **Confusão comum:** "o API Gateway substitui o Nginx do meu servidor". ✅ **Mais preciso:** depende de quais papéis aquele Nginx estava desempenhando. Se ele só fazia proxy reverso e TLS para a API, sim — o gateway absorve isso. Mas se ele também **servia os arquivos estáticos do frontend**, fazia **cache HTTP** de respostas, ou conversava por **FastCGI** com um php-fpm, o gateway não cobre nada disso, e remover o Nginx quebra o sistema de formas que só aparecem depois. Vale mapear papel a papel antes de trocar o componente.

---

## 7. Guia de decisão

| Preciso de... | Componente | Observação |
|---|---|---|
| Distribuir entre réplicas idênticas | **Load balancer** (função) | Qualquer produto que a implemente |
| Balancear tráfego não-HTTP | **LB L4** | HAProxy TCP, NLB, Nginx `stream` |
| Servir arquivos estáticos | **Nginx** | Gateway e LB não fazem |
| Terminar TLS em um só lugar | Qualquer um dos três | Escolha pelo que já existe no caminho |
| Rate limiting por IP | **Nginx** (`limit_req`) ou LB L7 | Suficiente na esmagadora maioria dos casos |
| Rate limiting por **consumidor**, com planos | **API Gateway** | Exige conhecer identidade — só o papel de gateway cobre |
| Validar JWT antes de chegar aos serviços | **Nginx** (`auth_request`) ou **Gateway** | Nginx basta se a política for única |
| Chave de API, portal, cobrança por uso | **API Gateway** | Não é expressável em configuração de proxy |
| Roteamento por path para serviços distintos | **Nginx**, LB L7 ou **Gateway** | Nginx é a opção mais barata |
| Versionar e depreciar contratos de API | **API Gateway** | Governança é a razão de ser do papel |
| Cache de respostas HTTP | **Nginx** ou CDN | Gateways oferecem, mas raramente tão bem |
| Segurança entre serviços internos | **Nenhum dos três** | É service mesh — ver `api_gateway.md`, seção 1 |

---

## Resumo do arquivo

- Comparar os três diretamente é um **erro de categoria**: load balancer é uma **função**, API Gateway é um **papel arquitetural**, Nginx é um **produto**. Um produto exerce funções e desempenha papéis.
- Todos os três são, na base, **proxies reversos** — daí o núcleo comum (roteamento, TLS, rate limiting, observabilidade) que os faz parecer intercambiáveis em configurações simples.
- **Só o LB L4** balanceia tráfego não-HTTP e evita decodificar a requisição. **Só o gateway** conhece o *consumidor* — chave, plano, quota, contrato. **Só o Nginx** (dos três) serve arquivos estáticos, faz cache HTTP e fala FastCGI.
- **Nginx basta como gateway** em sistemas com poucos serviços, um consumidor e políticas homogêneas. Deixa de bastar quando aparecem consumidores externos com identidade própria, publicação de rotas descentralizada, ou políticas que precisam mudar sem reload.
- **Kong é Nginx com uma camada de gestão de API por cima** — a melhor prova de que a relação entre produto e papel é de composição, não de concorrência.
- Em um sistema real, **balanceamento aparece em várias camadas**, e **o gateway precisa de um load balancer na frente dele** para não ser o SPOF do sistema inteiro.

**Próxima leitura:** `arquitetura_master_slave.md` — descendo para a última camada do caminho da requisição, a única que não pode ser tornada *stateless*: o banco de dados.
