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

## 1. Arquiteturas Clássicas

Antes de entender os modelos modernos, é fundamental conhecer as arquiteturas que os antecederam. Elas não são apenas "história" — muitas aplicações ainda as usam, e os problemas que elas geravam foram exatamente o que motivou a criação dos padrões modernos.

### 1.1 Arquitetura Monolítica

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

### 1.2 Arquitetura em Camadas (N-Tier)

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

### 1.3 Arquitetura Cliente-Servidor

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

## 2. Arquiteturas Modernas

As arquiteturas modernas surgiram para resolver os problemas que as arquiteturas clássicas não conseguiam resolver de forma eficiente: escalabilidade elástica, resiliência a falhas, evolução independente de partes do sistema e times grandes trabalhando sem se bloquear.

### 2.1 Microsserviços (Microservices)

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

**Princípio fundamental:** cada serviço deve poder ser desenvolvido, testado, deployado e escalado de forma completamente independente dos outros. Se o serviço de pagamentos precisa de uma atualização urgente, você faz o deploy **só dele**, sem tocar nos outros.

**Vantagens:**
- **Escalabilidade granular:** só o serviço sobrecarregado escala (economiza custo)
- **Autonomia de times:** times diferentes podem trabalhar em serviços diferentes sem conflito
- **Resiliência:** se o serviço de recomendações cair, o e-commerce continua funcionando
- **Liberdade tecnológica:** serviço de ML em Python, API em Go, front em TypeScript — cada um usa o que é mais adequado

**Desvantagens e complexidades:**
- **Comunicação distribuída:** chamadas entre serviços passam pela rede (latência, falhas, timeouts)
- **Consistência de dados:** sem um banco compartilhado, garantir consistência exige padrões como Saga e Event Sourcing
- **Observabilidade:** debugar um erro que passa por 5 serviços exige rastreamento distribuído (Jaeger, Zipkin)
- **Overhead operacional:** dezenas de serviços para monitorar, atualizar e escalar

**Quando usar:**
- Produto com domínio bem entendido e estável
- Times grandes (10+ devs) que precisam de autonomia
- Partes do sistema com necessidades de escala muito diferentes
- Empresas com maturidade de DevOps (CI/CD, monitoramento, containers)

> **Atenção:** microsserviços são uma solução para problemas de **escala organizacional e técnica**. Começar um projeto do zero com microsserviços é, na maioria dos casos, um erro — você paga toda a complexidade operacional sem ter ainda os problemas que eles resolvem.

---

### 2.2 Arquitetura Orientada a Eventos (Event-Driven)

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

---

### 2.3 Serverless

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

### 2.4 Arquitetura Orientada a APIs

**O que é:** Todo o sistema é projetado em torno de **APIs bem definidas como contratos públicos**. Cada capacidade do sistema é exposta como uma API, permitindo que diferentes clientes (web, mobile, parceiros) consumam os mesmos serviços.

É o modelo do "API-first": você define a API antes de implementar o serviço.

**Por que isso importa:** em empresas como Stripe, Twilio e PagSeguro, a API *é* o produto. A qualidade arquitetural da API determina a experiência dos desenvolvedores que a usam.

---

## 3. Como Escolher a Arquitetura Certa?

Não existe uma resposta universal. A escolha depende de:

| Critério | Considere |
|---|---|
| **Tamanho do time** | Times pequenos → monolito bem estruturado. Times grandes → microsserviços |
| **Maturidade do domínio** | Domínio incerto → monolito (fácil de refatorar). Domínio estável → microsserviços |
| **Escala esperada** | Escala uniforme → monolito ou N-tier. Escala assimétrica → microsserviços |
| **Maturidade de DevOps** | Sem CI/CD maduro → não use microsserviços |
| **Requisitos de resiliência** | Falhas catastróficas são inaceitáveis → distribua e isolie |

> **Regra prática:** comece simples. A maioria dos sistemas de sucesso começa como monolito e evolui para microsserviços quando os problemas de escala e organização realmente aparecem — não antes.
