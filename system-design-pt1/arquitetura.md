# Arquitetura de Software

## O que é Arquitetura de Software?

Arquitetura de software é a organização fundamental de um sistema, definindo seus componentes, como eles se relacionam entre si, e os princípios que guiam seu design e evolução ao longo do tempo.

Em termos mais simples: é o conjunto de decisões técnicas de alto nível que moldam como um sistema funciona, cresce e se mantém. Diferente do código em si (que resolve problemas específicos), a arquitetura resolve **trade-offs estruturais** — escolhas que afetam o sistema inteiro e são difíceis de reverter depois.

### Por que a arquitetura importa?

Imagine construir um prédio. Você pode reformar um cômodo depois, mas não pode facilmente mover os pilares estruturais sem um custo enorme. O mesmo vale para software: decisões arquiteturais incorretas geram **dívida técnica** — um custo crescente que se paga com lentidão de desenvolvimento, bugs difíceis de rastrear e sistemas que quebram sob pressão.

### Os Trade-offs fundamentais

Toda decisão arquitetural envolve equilibrar forças concorrentes:

| Atributo de Qualidade | Pergunta que responde |
|---|---|
| **Performance** | O sistema responde rápido o suficiente? |
| **Escalabilidade** | Aguenta crescimento de usuários e dados? |
| **Disponibilidade** | Fica no ar mesmo quando partes falham? |
| **Segurança** | Protege dados e resiste a ataques? |
| **Manutenibilidade** | É fácil de modificar, testar e evoluir? |
| **Custo** | É viável economicamente operar e escalar? |
| **Testabilidade** | Dá pra verificar o comportamento de forma isolada? |

> **Exemplo prático de trade-off:** Um sistema que prioriza performance máxima (cache agressivo, pouca sincronização) pode sacrificar consistência dos dados. Um sistema que prioriza segurança máxima (múltiplas camadas de validação) pode sacrificar performance. Não existe arquitetura perfeita — existe a arquitetura **mais adequada para o contexto**.

---

## 1. Características de Arquitetura

A tabela acima já apresentou, de forma informal, os atributos que toda decisão arquitetural precisa equilibrar. Antes de avançar, vale formalizar esse vocabulário — ele é usado constantemente pela literatura de arquitetura de software e vai reaparecer em todos os capítulos seguintes.

**Características de arquitetura** (também chamadas de "-ilities", em referência aos sufixos em inglês de *scalability*, *availability* etc.) são os requisitos que **não pertencem ao domínio de negócio**, mas que são críticos para o sucesso do sistema. Formalmente, um atributo só é considerado uma característica de arquitetura se atender a três critérios simultaneamente:

1. **Especifica uma consideração de design que não é do domínio.** O requisito de negócio diz *o que* o sistema deve fazer (ex.: "processar pagamentos"); a característica de arquitetura diz *como* isso deve ser feito para ter sucesso (ex.: "processar pagamentos em menos de 200ms, mesmo sob pico de tráfego").
2. **Influencia algum aspecto estrutural do design.** Se oferecer suporte a essa característica não exige nenhuma decisão estrutural especial, ela não chega a ser uma característica de arquitetura — é apenas uma boa prática de higiene técnica.
3. **É crítica ou importante para o sucesso da aplicação.** Um sistema poderia, teoricamente, suportar dezenas de características — mas cada uma adicionada aumenta a complexidade do design. Parte do trabalho do arquiteto é escolher **as menos possíveis**, não as mais possíveis.

### Características implícitas vs. explícitas

- **Explícitas:** aparecem documentadas em requisitos ("o sistema deve suportar 10 mil usuários simultâneos").
- **Implícitas:** raramente estão escritas em lugar nenhum, mas são presumidas — disponibilidade, segurança e confiabilidade estão nessa categoria na maioria dos sistemas. Ninguém escreve "o sistema não deve vazar dados de cartão de crédito" em um documento de requisitos, mas é um requisito inegociável mesmo assim.

### As três categorias

| Categoria | O que cobre | Exemplos |
|---|---|---|
| **Operacionais** | Capacidades relacionadas a como o sistema roda em produção | Performance, escalabilidade, elasticidade, disponibilidade, confiabilidade, recuperabilidade |
| **Estruturais** | Qualidade do código e da organização interna | Modularidade, configurabilidade, extensibilidade, manutenibilidade, portabilidade |
| **Cross-cutting** | Atravessam o sistema inteiro, não pertencem a uma camada específica | Autenticação, autorização, privacidade, conformidade legal (LGPD, GDPR) |

> **A regra da "menos pior arquitetura":** nunca existe uma arquitetura que maximize todas as características ao mesmo tempo — melhorar segurança quase sempre custa performance; melhorar consistência quase sempre custa disponibilidade (veremos isso formalmente no **Teorema CAP**, seção 5). O trabalho do arquiteto não é encontrar a *melhor* arquitetura, mas a *menos pior* para o conjunto de características que o negócio realmente precisa.

---

## 2. Arquiteturas Clássicas

Antes de entender os modelos modernos, é fundamental conhecer as arquiteturas que os antecederam. Elas não são apenas "história" — muitas aplicações ainda as usam, e os problemas que elas geravam foram exatamente o que motivou a criação dos padrões modernos.

### 2.1 Arquitetura Monolítica

**O que é:** Um sistema monolítico é aquele onde **todas as funcionalidades vivem em um único processo, um único código-base e fazem um único deploy**. A interface, as regras de negócio, o acesso ao banco de dados e as integrações estão todos entrelaçados.

```
┌─────────────────────────────────────────┐
│            MONOLITO                     │
│  ┌──────────┐  ┌──────────┐            │
│  │   UI     │  │ Pagament │            │
│  └──────────┘  └──────────┘            │
│  ┌──────────┐  ┌──────────┐            │
│  │ Usuários │  │ Estoque  │  → DB      │
│  └──────────┘  └──────────┘            │
│  ┌──────────┐  ┌──────────┐            │
│  │Notificaç.│  │Relatório │            │
│  └──────────┘  └──────────┘            │
└─────────────────────────────────────────┘
         Um deploy. Um processo.
```

**Características:**
- Deploy único: você sobe ou derruba tudo junto
- Código-base único: todos os times trabalham no mesmo repositório
- Banco de dados compartilhado por todos os módulos
- Comunicação entre módulos via chamadas de função (sem rede)

**Quando faz sentido:**
- Startups e projetos no início (a simplicidade acelera o desenvolvimento)
- Times pequenos (1-5 devs)
- Domínio ainda mal compreendido (evita separações prematuras e incorretas)
- Protótipos e MVPs

**Vantagens:**
- Simples de desenvolver no começo
- Fácil de debugar (tudo está num lugar só)
- Sem latência de rede entre módulos
- Deploy e rollback simples
- Transações de banco de dados fáceis (tudo no mesmo contexto)

**Desvantagens e por que ela "quebra" com o crescimento:**
- **Acoplamento alto:** uma mudança em pagamentos pode quebrar notificações, porque o código está entrelaçado
- **Escalabilidade limitada:** você é obrigado a escalar o sistema inteiro, mesmo que só o módulo de relatórios esteja sobrecarregado
- **Uma falha derruba tudo:** um bug de memória em um módulo mata o processo inteiro
- **Deploy arriscado:** qualquer mudança pequena exige um novo deploy de tudo
- **Times grandes trombam:** 10 devs num mesmo código-base geram conflitos de merge constantes

> **Observação:** Monolito não é sinônimo de código ruim. Um monolito bem estruturado internamente (com módulos separados, interfaces claras) é chamado de **monolito modular** e pode ser uma excelente escolha por muito tempo.

---

### 2.2 Arquitetura em Camadas (N-Tier)

**O que é:** Divide a aplicação em camadas horizontais, onde **cada camada tem uma responsabilidade específica e só se comunica com a camada imediatamente abaixo ou acima dela**.

A variante mais comum é a **arquitetura de 3 camadas (3-Tier)**:

```
┌─────────────────────────────────────────┐
│    CAMADA DE APRESENTAÇÃO (UI)          │
│    React, Angular, HTML, Mobile         │
└──────────────────┬──────────────────────┘
                   │ HTTP / REST
┌──────────────────▼──────────────────────┐
│    CAMADA DE NEGÓCIO (Backend)          │
│    Regras, Validações, Cálculos         │
└──────────────────┬──────────────────────┘
                   │ SQL / ORM
┌──────────────────▼──────────────────────┐
│    CAMADA DE DADOS (Banco)              │
│    PostgreSQL, MySQL, Oracle            │
└─────────────────────────────────────────┘
```

**Regra fundamental:** a camada de apresentação **nunca** acessa o banco diretamente. Toda leitura/escrita passa pela camada de negócio. Isso parece burocrático, mas é o que garante que as regras de negócio sejam aplicadas consistentemente.

**Vantagens:**
- Separação clara de responsabilidades
- A UI pode ser substituída (trocar React por Vue) sem tocar na lógica de negócio
- O banco pode ser migrado sem mexer na UI
- Facilita testes: você pode testar a camada de negócio de forma isolada

**Desvantagens:**
- Pode gerar camadas "anêmicas" que apenas repassam dados sem agregar valor
- Chamadas em cascata introduzem latência
- Se mal projetada, vira um monolito com camadas — mesmos problemas, mais complexidade

#### Conceito importante: Pool de Conexões (Connection Pooling)

Na arquitetura em camadas, toda requisição do usuário resulta em uma operação no banco de dados. Abrir uma conexão TCP com o banco a cada requisição é caro (handshakes, autenticação, alocação de recursos). Com muitas requisições simultâneas, isso vira gargalo.

**A solução é o pool de conexões:** um conjunto de conexões abertas e mantidas prontas para uso. Quando uma requisição chega, ela "pega emprestado" uma conexão do pool, usa-a, e a devolve ao terminar.

```
Requisições (100 simultâneas)
        │
        ▼
┌─────────────────────┐
│   Pool de Conexões  │  ← 10 conexões abertas e reutilizadas
│  [C1][C2]...[C10]  │
└────────┬────────────┘
         │
         ▼
    Banco de Dados
```

**Por que isso importa arquiteturalmente?** O banco de dados tem um limite de conexões simultâneas. Sem pool, 1000 usuários = 1000 conexões abertas, o que pode derrubar o banco. Com pool, você pode servir 1000 usuários com 10-20 conexões, porque cada uma é reutilizada rapidamente.

> **Concorrência vs. Paralelismo:** É importante distinguir os dois conceitos:
> - **Concorrência** é lidar com várias tarefas de forma intercalada (um único núcleo alterna entre tarefas rapidamente). O Node.js é concorrente, mas não paralelo por padrão.
> - **Paralelismo** é executar múltiplas tarefas literalmente ao mesmo tempo (múltiplos núcleos de CPU). O Python com multiprocessing ou Go com goroutines podem ser paralelos.

---

### 2.3 Arquitetura Cliente-Servidor

**O que é:** O modelo mais fundamental de sistemas distribuídos. Um **cliente** faz uma requisição, um **servidor** processa e retorna uma resposta. Simples assim.

```
┌──────────┐   Requisição    ┌──────────────┐
│  CLIENTE │ ─────────────→  │   SERVIDOR   │
│          │ ←─────────────  │              │
└──────────┘   Resposta      └──────────────┘
```

**Exemplos no dia a dia:**
- Navegador (cliente) → Servidor web
- Aplicativo mobile → API REST
- Cliente SQL (DBeaver) → Banco de dados
- Jogo online → Servidor de jogo

**Por que estudar isso?** Porque praticamente toda arquitetura moderna — microserviços, serverless, APIs — é uma extensão ou variação desse modelo. Entender cliente-servidor é entender a base de tudo.

**Vantagens:**
- Conceito simples e universal
- Separação clara entre quem consome e quem provê
- Servidor pode atender múltiplos clientes simultâneos

**Desvantagens:**
- O servidor pode virar gargalo único (single point of failure)
- Clientes dependem fortemente da disponibilidade do servidor
- Latência de rede é sempre um fator

---

## 3. Acoplamento e Quantum de Arquitetura

Antes de entrar nas arquiteturas modernas, falta uma peça conceitual: como medir, de forma objetiva, o quão "distribuído" um sistema realmente é? Dividir um monolito em cinco serviços não necessariamente produz cinco unidades independentes — se todos ainda escrevem no mesmo banco de dados, você tem cinco processos, mas continua tendo **um** sistema, no sentido que importa para deploy e disponibilidade.

O conceito que resolve essa ambiguidade é o de **quantum de arquitetura**: uma unidade independentemente implantável, com alta coesão funcional e alto acoplamento estático, conectada ao resto do sistema por acoplamento dinâmico síncrono. Um microsserviço bem desenhado é o exemplo típico de um quantum.

Para aplicar essa definição na prática, é preciso distinguir dois tipos de acoplamento:

| Tipo | O que mede | Exemplos |
|---|---|---|
| **Acoplamento estático** | Como as dependências se resolvem *antes* da execução — o que é preciso existir para o serviço sequer funcionar | Banco de dados, bibliotecas, frameworks, contratos de API |
| **Acoplamento dinâmico** | Como os componentes se chamam *durante* a execução — o tráfego real entre eles | Uma chamada REST síncrona; uma mensagem publicada em uma fila |

**A implicação prática mais importante:** qualquer sistema que compartilha um único banco de dados tem, por definição, **um único quantum** — não importa em quantos processos ou containers ele esteja fisicamente dividido. Isso porque nenhum desses processos pode ser implantado, escalado ou recuperado de uma falha de forma verdadeiramente independente: todos dependem do mesmo recurso estático (o banco).

```
Monolito clássico              "Microsserviços" com banco compartilhado
┌──────────────────┐           ┌────────┐  ┌────────┐  ┌────────┐
│  Todo o sistema   │           │Serviço │  │Serviço │  │Serviço │
│                   │           │   A    │  │   B    │  │   C    │
└─────────┬─────────┘           └───┬────┘  └───┬────┘  └───┬────┘
          ▼                          └──────────┼──────────┘
      1 banco                                   ▼
                                             1 banco
   → 1 quantum                          → também 1 quantum!
```

> **Por que isso importa?** É o critério mais objetivo para responder "isso aqui é microsserviços de verdade, ou é um monolito distribuído disfarçado?". Se um bug no schema do banco derruba todos os "serviços" ao mesmo tempo, e nenhum deles pode ser implantado sem coordenar com os outros, o sistema tem um quantum — a divisão em processos separados só adicionou a complexidade operacional da rede, sem entregar a independência que motivaria essa complexidade em primeiro lugar. Esse é exatamente o problema que a seção 4.7 (Service-Based Architecture) discute em detalhe.

---

## 4. Arquiteturas Modernas

As arquiteturas modernas surgiram para resolver os problemas que as arquiteturas clássicas não conseguiam resolver de forma eficiente: escalabilidade elástica, resiliência a falhas, evolução independente de partes do sistema e times grandes trabalhando sem se bloquear.

### 4.1 Microsserviços (Microservices)

**O que é:** Uma aplicação é decomposta em **serviços pequenos e independentes**, cada um responsável por uma capacidade de negócio específica. Cada serviço é autônomo: tem seu próprio processo, pode ter seu próprio banco de dados e é deployado de forma independente.

```
                    ┌─────────────────┐
     Usuário        │   API Gateway   │
       │            │  (entrada única)│
       └──────────→ └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        ▼                    ▼                    ▼
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│  Serviço de  │   │  Serviço de  │   │  Serviço de  │
│   Usuários   │   │  Pagamentos  │   │   Catálogo   │
│   [DB: PG]   │   │  [DB: PG]   │   │ [DB: Mongo]  │
└──────────────┘   └──────────────┘   └──────────────┘
        ▼
┌──────────────┐
│  Serviço de  │
│ Notificações │
│  [DB: Redis] │
└──────────────┘
```

**Princípio fundamental:** cada serviço deve poder ser desenvolvido, testado, deployado e escalado de forma completamente independente dos outros. Se o serviço de pagamentos precisa de uma atualização urgente, você faz o deploy **só dele**, sem tocar nos outros. Note que isso é a definição de quantum da seção 3 aplicada na prática: cada serviço com seu próprio banco é seu próprio quantum.

**Vantagens:**
- **Escalabilidade granular:** só o serviço sobrecarregado escala (economiza custo)
- **Autonomia de times:** times diferentes podem trabalhar em serviços diferentes sem conflito
- **Resiliência:** se o serviço de recomendações cair, o e-commerce continua funcionando
- **Liberdade tecnológica:** serviço de ML em Python, API em Go, front em TypeScript — cada um usa o que é mais adequado

**Desvantagens e complexidades:**
- **Comunicação distribuída:** chamadas entre serviços passam pela rede (latência, falhas, timeouts)
- **Consistência de dados:** sem um banco compartilhado, garantir consistência exige padrões como Saga e Event Sourcing (aprofundamos isso na seção 6)
- **Observabilidade:** debugar um erro que passa por 5 serviços exige rastreamento distribuído (Jaeger, Zipkin)
- **Overhead operacional:** dezenas de serviços para monitorar, atualizar e escalar

**Quando usar:**
- Produto com domínio bem entendido e estável
- Times grandes (10+ devs) que precisam de autonomia
- Partes do sistema com necessidades de escala muito diferentes
- Empresas com maturidade de DevOps (CI/CD, monitoramento, containers)

> **Atenção:** microsserviços são uma solução para problemas de **escala organizacional e técnica**. Começar um projeto do zero com microsserviços é, na maioria dos casos, um erro — você paga toda a complexidade operacional sem ter ainda os problemas que eles resolvem.

---

### 4.2 Arquitetura Orientada a Eventos (Event-Driven)

**O que é:** Os componentes do sistema se comunicam através de **eventos** — fatos que aconteceram no sistema. Em vez de o serviço A chamar diretamente o serviço B, ele publica um evento ("PedidoRealizado") e qualquer serviço interessado reage a esse evento de forma assíncrona.

```
Serviço de Pedidos
        │
        │  publica: "PedidoRealizado"
        ▼
┌─────────────────┐
│   Message Broker│  (Kafka, RabbitMQ, SQS)
└────────┬────────┘
         │
  ┌──────┼───────────┐
  ▼      ▼           ▼
Estoque Pagamento Notificação
(reage) (reage)   (reage)
```

**Vantagens:**
- **Desacoplamento real:** o produtor não sabe quem são os consumidores
- **Escalabilidade:** consumidores processam no seu próprio ritmo
- **Auditoria natural:** eventos são um log imutável do que aconteceu

**Desafios:**
- Debugging mais difícil (fluxo não é linear)
- Consistência eventual (dados podem estar temporariamente inconsistentes)
- Requer infraestrutura de mensageria (Kafka, RabbitMQ)

Existem duas topologias distintas para implementar uma arquitetura orientada a eventos, e a escolha entre elas tem impacto direto na testabilidade e na capacidade de recuperação de erros do sistema.

#### 4.2.1 Topologia Broker

Não existe um componente central controlando o fluxo. Cada serviço publica os eventos que gera em um broker de mensagens (RabbitMQ, ActiveMQ) usando o modelo publish/subscribe, e qualquer outro serviço interessado assina esse evento. Cada evento processado gera, por sua vez, um novo evento anunciando o que foi feito — mesmo que ninguém esteja ouvindo naquele momento. Isso é proposital: mantém o sistema extensível, porque um novo consumidor pode ser plugado no futuro sem alterar nada nos serviços existentes.

```
PedidoCriado → [Broker] ┬→ Estoque      → decrementa produto
                         ├→ Pagamento    → cobra cartão
                         └→ Notificação  → envia e-mail
                              (cada um publica seu próprio evento de conclusão)
```

| Vantagens | Desvantagens |
|---|---|
| Altamente desacoplado | Sem visibilidade do workflow como um todo |
| Alta escalabilidade | Tratamento de erro difícil (ninguém sabe que algo falhou) |
| Alta responsividade | Sem capacidade de reiniciar a transação de negócio |
| Alta tolerância a falhas | Inconsistência de dados mais difícil de corrigir |

#### 4.2.2 Topologia Mediator

Um **mediador de eventos** central conhece os passos do processo e orquestra explicitamente cada serviço envolvido, geralmente através de filas ponto a ponto dedicadas. Diferente da topologia broker, os serviços não anunciam o que fizeram para o sistema todo — eles respondem diretamente ao mediador.

```
PedidoCriado → [Mediador] → chama Estoque      (aguarda confirmação)
                          → chama Pagamento    (aguarda confirmação)
                          → chama Notificação  (aguarda confirmação)
                          → sabe exatamente em que passo o processo está
```

| Vantagens | Desvantagens |
|---|---|
| Controle centralizado do workflow | Menor responsividade (tudo passa pelo mediador) |
| Tratamento de erro mais simples | Ponto único de falha potencial |
| Capacidade de recuperação (retry) | Menor escalabilidade que a topologia broker |
| Estado do processo é consultável | Maior acoplamento dos serviços ao mediador |

> **Qual escolher?** Broker quando o fluxo é simples e o desacoplamento importa mais do que a visibilidade do processo (ex.: notificações, atualizações de cache). Mediator quando o workflow tem múltiplos passos com regras de erro complexas e alguém precisa saber, a qualquer momento, "em que pé está esse pedido" (ex.: processamento de pagamento, fulfillment). Voltamos a esse tema com mais profundidade na seção 6, ao discutir Sagas.

---

### 4.3 Serverless

**O que é:** Você escreve funções que são executadas sob demanda, sem precisar gerenciar servidores. A infraestrutura é completamente gerenciada pelo provedor de nuvem (AWS Lambda, Google Cloud Functions, Azure Functions).

```
Requisição → AWS Lambda → Executa função → Retorna resultado
            (escala 0 a N automaticamente)
```

**Vantagens:**
- Sem gerenciamento de infraestrutura
- Pagamento por execução (não por servidor ligado 24h)
- Escala automática do zero ao infinito

**Desvantagens:**
- Cold start (primeira execução pode ser lenta)
- Limite de tempo de execução
- Vendor lock-in (difícil migrar entre provedores)
- Debugging local é mais complexo

---

### 4.4 Arquitetura Orientada a APIs

**O que é:** Todo o sistema é projetado em torno de **APIs bem definidas como contratos públicos**. Cada capacidade do sistema é exposta como uma API, permitindo que diferentes clientes (web, mobile, parceiros) consumam os mesmos serviços.

É o modelo do "API-first": você define a API antes de implementar o serviço.

**Por que isso importa:** em empresas como Stripe, Twilio e PagSeguro, a API *é* o produto. A qualidade arquitetural da API determina a experiência dos desenvolvedores que a usam.

---

### 4.5 Arquitetura em Pipeline (Pipes and Filters)

**O que é:** Um dos estilos mais antigos e ainda mais usados em processamento de dados. O sistema é organizado como uma sequência de **filtros** conectados por **pipes** (canais de comunicação unidirecionais). Cada filtro recebe dados de um pipe de entrada, processa, e envia o resultado para um pipe de saída — sem conhecer o que veio antes ou o que vem depois na cadeia.

É o modelo por trás dos pipes do terminal Unix/Linux (`cat arquivo.txt | grep erro | sort | uniq`) e da ideia central por trás do MapReduce.

```
Fonte → [Filtro 1] → [Filtro 2] → [Filtro 3] → Destino
        transforma   filtra      agrega
```

**Características dos filtros:**
- **Autocontidos e sem estado (stateless):** um filtro não sabe nada sobre os outros filtros do pipeline
- **Fazem uma única tarefa:** tarefas compostas são resolvidas encadeando vários filtros simples, não criando um filtro complexo
- **Reutilizáveis:** o mesmo filtro pode aparecer em pipelines diferentes

**Exemplos de uso real:**
- Pipelines de ETL (Extract, Transform, Load) em engenharia de dados
- Processamento de imagem em etapas (redimensionar → comprimir → aplicar watermark)
- Compiladores (análise léxica → sintática → semântica → geração de código)

**Vantagens:**
- Altíssima modularidade — cada filtro pode ser testado e substituído isoladamente
- Fácil de raciocinar sobre o fluxo de dados (é sempre unidirecional)
- Filtros são reutilizáveis entre pipelines diferentes

**Desvantagens:**
- Não é adequado para fluxos com ramificações complexas ou decisões condicionais elaboradas
- Baixa adequação para sistemas que exigem resposta interativa de baixa latência (cada etapa adiciona overhead sequencial)
- Compartilhar estado entre filtros distantes no pipeline é difícil por design

---

### 4.6 Arquitetura Microkernel (Plug-in)

**O que é:** Também chamada de arquitetura de plug-ins, é composta por dois tipos de componentes: um **sistema núcleo (core system)**, contendo a funcionalidade mínima necessária para o sistema funcionar, e **componentes de plug-in**, independentes entre si, que estendem esse núcleo com lógica customizada.

O exemplo clássico é uma IDE como o Eclipse: o núcleo é apenas um editor de texto capaz de abrir, editar e salvar arquivos. Tudo o que faz o Eclipse parecer uma IDE completa — suporte a Java, Git, debugger — chega através de plug-ins.

```
                 ┌─────────────────┐
                 │   Core System    │
                 │ (fluxo principal,│
                 │  sem lógica      │
                 │  customizada)    │
                 └────────┬─────────┘
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
    ┌──────────┐     ┌──────────┐     ┌──────────┐
    │ Plug-in A │     │ Plug-in B │     │ Plug-in C │
    └──────────┘     └──────────┘     └──────────┘
     (independentes entre si — sem comunicação direta)
```

**Quando faz sentido:** sistemas onde a lógica principal é estável, mas regras específicas variam muito por cliente ou contexto. Exemplo prático: um sistema de reciclagem eletrônica que precisa de uma rotina de avaliação diferente para cada modelo de aparelho — em vez de um `if/else` gigante no núcleo, cada avaliação vira um plug-in independente, registrado e invocado dinamicamente pelo núcleo.

**Vantagens:**
- Extensibilidade alta: adicionar uma nova regra de negócio é só adicionar um novo plug-in, sem tocar no núcleo
- Isolamento de complexidade: lógica customizada e volátil fica separada da lógica estável
- Plug-ins podem ser testados isoladamente

**Desvantagens:**
- Normalmente é implantado como um monolito (núcleo + plug-ins compartilham processo e, geralmente, banco de dados) — na terminologia da seção 3, continua sendo **um único quantum**
- Não resolve problemas de escala distribuída — resolve problemas de extensibilidade e manutenibilidade

---

### 4.7 Service-Based Architecture

**O que é:** Um híbrido pragmático entre o monolito e os microsserviços — por isso é frequentemente confundida com microsserviços, mas é estruturalmente diferente. O sistema é dividido em um pequeno número de **serviços de domínio de granularidade grossa** (tipicamente entre 4 e 12), implantados separadamente, mas que **compartilham um único banco de dados monolítico**.

```
              ┌─────────────────┐
              │  Interface (UI)  │
              └────────┬─────────┘
      ┌──────────────┬─┴┬──────────────┐
      ▼               ▼               ▼
┌───────────┐   ┌───────────┐   ┌───────────┐
│ Serviço de │   │ Serviço de │   │ Serviço de │
│  Usuários  │   │  Pedidos   │   │  Catálogo  │
└─────┬─────┘   └─────┬─────┘   └─────┬─────┘
      └─────────────────┼─────────────────┘
                         ▼
                 Banco de Dados
                    (compartilhado)
```

**A diferença crucial em relação a microsserviços não é o número de serviços — é a granularidade e o modelo de transação:**

| | Service-Based | Microsserviços |
|---|---|---|
| Granularidade | Grossa (4-12 serviços) | Fina (dezenas a centenas) |
| Banco de dados | Compartilhado | Um por serviço |
| Transações | ACID tradicionais dentro de cada serviço | BASE / consistência eventual entre serviços |
| Quantum (seção 3) | Geralmente 1 (banco compartilhado) | Um por serviço |

Pense no exemplo de um checkout de loja: se o pagamento falhar, um `ROLLBACK` de banco tradicional desfaz tudo instantaneamente, porque tudo está na mesma transação ACID. No modelo de microsserviços, o mesmo cenário exige que cada serviço reverta manualmente o que já fez — o problema que a seção 6 (Sagas) resolve.

**Vantagens:**
- Ganha independência de deploy por serviço sem pagar o custo total da consistência distribuída
- Mantém a simplicidade de transações ACID — menos código de tratamento de erro
- Pragmático: costuma ser um bom "próximo passo" quando um monolito começa a doer, antes de saltar direto para microsserviços

**Desvantagens:**
- Uma mudança de schema no banco compartilhado ainda exige coordenar e testar todos os serviços que o usam
- Menos elástico: como vimos na seção 3, um banco compartilhado tende a reduzir o sistema a um único quantum real, mesmo com múltiplos serviços implantados separadamente
- Não resolve o problema de escalar serviços de forma verdadeiramente independente

> **Quando escolher Service-Based em vez de Microsserviços?** Quando o domínio ainda não está maduro o suficiente para justificar bancos separados por serviço, mas o time já sofre com os problemas de deploy de um monolito. É, na prática, o degrau intermediário mais comum entre os dois mundos.

---

## 5. O Teorema CAP

Em sistemas distribuídos, toda decisão arquitetural é constrangida pelo **Teorema CAP** (Brewer, 2000): um sistema distribuído pode garantir no máximo **dois** dos três atributos simultaneamente.

| Atributo | O que significa |
|---|---|
| **C — Consistency** | Toda leitura retorna o dado mais recente (ou um erro) |
| **A — Availability** | Toda requisição recebe uma resposta (pode ser desatualizada) |
| **P — Partition Tolerance** | O sistema continua funcionando mesmo se nós perderem comunicação entre si |

Como falhas de rede (partições) são inevitáveis em sistemas distribuídos, **P é obrigatório** na prática. A escolha real é entre C e A:

```
CP — Consistência + Tolerância a Partições
  Exemplo: HBase, MongoDB (modo estrito), sistemas bancários
  Comportamento: em caso de partição, prefere rejeitar requisições
                 a retornar dados potencialmente desatualizados.

AP — Disponibilidade + Tolerância a Partições
  Exemplo: Cassandra, DynamoDB, DNS
  Comportamento: em caso de partição, continua respondendo
                 mas pode retornar dados ligeiramente desatualizados
                 (consistência eventual).
```

> **Por que isso importa na prática?** O Redis Cluster, por exemplo, sacrifica consistência em favor de disponibilidade durante partições — leituras podem retornar dados stale. Bancos relacionais com replicação sínccrona sacrificam disponibilidade para garantir que todos os nós vejam o mesmo dado. Entender o CAP ajuda a escolher a ferramenta certa para o grau de consistência que o seu negócio exige.

### CAP não é binário: quorum

Na prática, poucos sistemas escolhem "100% C" ou "100% A" — a maioria dos bancos distribuídos modernos (Cassandra, DynamoDB) permite **ajustar** o grau de consistência por operação usando **quorum**. Com `N` réplicas de um dado, cada escrita precisa ser confirmada por `W` réplicas, e cada leitura precisa consultar `R` réplicas:

- **`W + R > N`** → consistência forte garantida (existe pelo menos uma réplica em comum entre quem escreveu e quem leu)
- **`W + R ≤ N`** → mais rápido, mas sem garantia de ler o dado mais recente

Por exemplo, com `N=3`: usar `W=1` otimiza para escrita rápida (não espera confirmação de todas as réplicas); usar `R=N` otimiza para leitura sempre consistente, ao custo de esperar todas as réplicas responderem. Essa é a mesma lógica do CAP, só que configurável por chamada em vez de fixa para o sistema inteiro.

---

## 6. Consistência de Dados em Sistemas Distribuídos

A seção 4.1 mencionou de passagem que, sem um banco compartilhado, microsserviços precisam de padrões como Saga para manter a consistência dos dados. Esta seção detalha o que isso significa na prática — é provavelmente o problema mais difícil de arquitetura distribuída, porque a solução mais óbvia (uma transação de banco tradicional) simplesmente não existe quando os dados estão espalhados entre serviços diferentes.

### 6.1 De quem é o dado? (Ownership)

Ao quebrar um banco monolítico em pedaços menores, alguém precisa ser dono de cada tabela. A regra geral é: **quem escreve em uma tabela, é dono dela.** Isso gera três cenários:

| Cenário | Definição | Como resolver |
|---|---|---|
| **Ownership único** | Só um serviço escreve na tabela | Trivial — esse serviço é o dono |
| **Ownership comum** | Praticamente todos os serviços precisam escrever (ex.: uma tabela de auditoria) | Criar um serviço dedicado, único dono da tabela; os demais enviam a informação a ele (síncrono ou fire-and-forget) |
| **Ownership conjunto** | Um subconjunto de serviços do mesmo domínio escreve na mesma tabela | Dividir a tabela por coluna/domínio, ou delegar a escrita a um único serviço responsável |

### 6.2 Por que não existe "transação distribuída" de verdade

Uma transação de banco tradicional garante as propriedades **ACID** (Atomicidade, Consistência, Isolamento, Durabilidade) — tudo commita junto, ou nada commita. Isso funciona porque tudo acontece dentro do mesmo processo de banco de dados.

Quando uma operação de negócio única precisa gravar dados em **múltiplos serviços** (cada um com seu próprio banco), nenhuma das quatro garantias sobrevive: cada serviço só pode garantir ACID para a própria escrita, não para a operação de negócio como um todo. Se o serviço de pagamento falha depois que o serviço de pedidos já commitou, os dados ficam temporariamente fora de sincronia — não existe um `ROLLBACK` global.

O modelo que substitui ACID em sistemas distribuídos chama-se **BASE**:

| | ACID (transação local) | BASE (transação distribuída) |
|---|---|---|
| **B**asic Availability | — | Todos os serviços envolvidos devem estar disponíveis para participar |
| **S**oft state | — | O estado da operação pode ficar "em progresso" por um tempo — nem sempre se sabe o estado exato |
| **E**ventual consistency | — | Dado tempo suficiente, todos os dados convergem para o estado correto |

### 6.3 Os três padrões de consistência eventual

| Padrão | Como funciona | Tempo até consistência | Complexidade de erro |
|---|---|---|---|
| **Background Synchronization** | Um processo externo varre periodicamente os dados e corrige divergências (batch noturno ou polling) | O mais longo (minutos a horas) | Baixa, mas quebra o *bounded context*: o processo de sincronização precisa de acesso de escrita a tabelas de outros serviços |
| **Orchestrated Request-Based** | Um orquestrador dedicado chama cada serviço envolvido *durante* a requisição, aguardando confirmação de cada um antes de responder ao cliente | Curto (a própria duração da requisição) | Alta — se um passo falhar no meio do caminho, o orquestrador precisa disparar transações compensatórias nos passos já concluídos |
| **Event-Based** | O serviço originador publica um evento; os demais serviços assinam e reagem de forma assíncrona | Curto e paralelo | Média — mensagens não entregues vão para uma *dead letter queue* para correção manual ou automática |

> **Qual escolher?** Background Sync quando a divergência temporária não afeta ninguém (ex.: relatórios). Orchestrated Request-Based quando o cliente precisa de uma resposta definitiva na hora (ex.: "sua compra foi confirmada?"). Event-Based é o mais usado em arquiteturas orientadas a eventos modernas — bom equilíbrio entre responsividade e desacoplamento, ao custo de exigir consumidores duráveis e uma estratégia clara para mensagens que falham.

### 6.4 Padrão Saga: transações compensatórias

Uma **Saga** é uma sequência de transações locais, onde cada serviço confirma sua própria etapa e publica um evento (ou responde a um orquestrador) para acionar a próxima. Se uma etapa falhar, a Saga não faz um `ROLLBACK` — ela executa **transações compensatórias**: operações que desfazem manualmente o efeito das etapas já concluídas (ex.: "estornar pagamento" em vez de um rollback automático).

Toda saga é uma combinação de três decisões independentes, o que gera 8 padrões possíveis:

- **Comunicação:** síncrona ou assíncrona
- **Consistência:** atômica (tenta preservar all-or-nothing) ou eventual
- **Coordenação:** orquestrada (um coordenador central comanda o fluxo, como na topologia Mediator da seção 4.2.2) ou coreografada (serviços reagem em cadeia a eventos uns dos outros, como na topologia Broker da seção 4.2.1)

| Padrão | Comunicação | Consistência | Coordenação | Quando considerar |
|---|---|---|---|---|
| **Epic Saga** | Síncrona | Atômica | Orquestrada | Imita o comportamento de um monolito — mais fácil de entender, mas o mais acoplado de todos os oito padrões |
| **Phone Tag Saga** | Síncrona | Atômica | Coreografada | Raramente compensa: exige o acoplamento síncrono sem ter um coordenador central para lidar com os erros |
| **Fairy Tale Saga** | Síncrona | Eventual | Orquestrada | Resposta imediata ao usuário; um orquestrador cuida da consistência em segundo plano |
| **Time Travel Saga** | Síncrona | Eventual | Coreografada | Serviços reagem em cadeia sem coordenador central; difícil rastrear o estado global do processo |
| **Fantasy Fiction Saga** | Assíncrona | Atômica | Orquestrada | Tentativa de manter atomicidade mesmo com chamadas assíncronas — combinação incomum, geralmente sintoma de tentar acelerar um Epic Saga sem abrir mão de consistência atômica |
| **Horror Story** | Assíncrona | Atômica | Coreografada | A combinação mais complexa e frágil de todas — atomicidade sem coordenador, com comunicação assíncrona. Evitar sempre que possível |
| **Parallel Saga** | Assíncrona | Eventual | Orquestrada | O equilíbrio mais comum em microsserviços modernos: baixo acoplamento, alta escala, e ainda assim um workflow rastreável |
| **Anthology Saga** | Assíncrona | Eventual | Coreografada | Máxima escala e desacoplamento — ideal para alto throughput com erros simples ou raros; ao custo de pouca visibilidade do fluxo geral |

> **Não é preciso decorar os oito nomes.** O que importa é o raciocínio por trás da tabela: cada eixo (comunicação, consistência, coordenação) é um trade-off independente, e apertar um deles geralmente força uma concessão em outro. Na dúvida, **Parallel Saga** é o ponto de partida mais equilibrado para a maioria dos sistemas orientados a eventos — e **Horror Story** é o padrão a evitar ativamente: ele tenta obter a garantia mais rígida (atomicidade) exatamente com as duas ferramentas de coordenação mais frouxas (assincronia e coreografia).

---

## 7. Registrando Decisões: Architecture Decision Records (ADR)

Depois de aplicar tudo isso — características de arquitetura, estilo escolhido, estratégia de consistência — surge um problema silencioso: **seis meses depois, ninguém lembra por que aquela decisão foi tomada.** A discussão se repete, alguém propõe reverter a decisão sem entender o contexto original, e o time perde tempo relitigando o que já havia sido resolvido.

Um **Architecture Decision Record (ADR)** é um documento curto (1-2 páginas, geralmente em Markdown, versionado junto com o código) que registra uma decisão arquitetural específica. A estrutura básica tem cinco seções:

```
# ADR 042: Uso de mensageria assíncrona entre Pedidos e Pagamento

## Status
Aceito

## Contexto
O serviço de Pedidos precisa notificar o serviço de Pagamento quando um
pedido é criado. As opções consideradas foram REST síncrono e mensageria
assíncrona via fila.

## Decisão
Vamos usar mensageria assíncrona (RabbitMQ) entre os dois serviços.

## Consequências
+ Serviço de Pedidos não fica bloqueado esperando o Pagamento responder.
+ Maior resiliência: se o Pagamento cair, as mensagens ficam na fila.
- Consistência passa a ser eventual (ver seção 6) — a tela de confirmação
  do pedido não pode mais afirmar que o pagamento já foi processado.
- Exige infraestrutura de mensageria e tratamento de dead letter queue.
```

**Por que isso funciona:**
- **Status** pode ser `Proposto`, `Aceito` ou `Substituído por ADR-N` — mantendo um histórico completo de como a arquitetura evoluiu, em vez de decisões "perdidas" em threads de e-mail ou chat.
- **Decisão** é escrita em voz afirmativa ("vamos usar X"), não como opinião ("acho que X seria melhor") — elimina ambiguidade sobre se a decisão já foi tomada.
- **Consequências** documenta o trade-off explicitamente — inclusive as desvantagens aceitas conscientemente. É a seção mais valiosa: sem ela, alguém no futuro vê apenas a decisão, não o raciocínio, e corre o risco de revertê-la sem entender o que se perde.

> **Regra prática:** um ADR não precisa de aprovação de comitê para decisões de baixo risco. Reserve o processo formal para decisões caras, difíceis de reverter, ou que afetam múltiplos times — os mesmos critérios de "característica crítica" da seção 1.

---

## 8. Como Escolher a Arquitetura Certa?

Não existe uma resposta universal. A escolha depende de:

| Critério | Considere |
|---|---|
| **Tamanho do time** | Times pequenos → monolito bem estruturado. Times grandes → microsserviços |
| **Maturidade do domínio** | Domínio incerto → monolito (fácil de refatorar). Domínio estável → microsserviços |
| **Escala esperada** | Escala uniforme → monolito ou N-tier. Escala assimétrica → microsserviços |
| **Maturidade de DevOps** | Sem CI/CD maduro → não use microsserviços |
| **Requisitos de resiliência** | Falhas catastróficas são inaceitáveis → distribua e isole |

> **Regra prática:** comece simples. A maioria dos sistemas de sucesso começa como monolito e evolui para microsserviços quando os problemas de escala e organização realmente aparecem — não antes. O caminho mais comum na prática é **monolito → service-based (seção 4.7) → microsserviços**, não um salto direto.
