# API Gateway

**Pré-requisito:** `load_balancer.md`, `nginx.md`, `../system-design-pt1/api_gateway.md`

> Baseado no Capítulo 17 (*Microservices Architecture*, seções **API Layer** e **Operational Reuse**) de **Fundamentals of Software Architecture** e no Capítulo 8 (*Reuse Patterns* — **Sidecars and Service Mesh**, **Orthogonal Coupling**) de **Software Architecture: The Hard Parts**.

---

## Por que este arquivo existe

`../system-design-pt1/api_gateway.md` cobre o gateway de forma prática e extensa: o ciclo de vida da requisição, autenticação centralizada, rate limiting, circuit breaker, o padrão BFF e um tutorial completo construindo um gateway em FastAPI. **Aquele arquivo continua sendo a referência de implementação** — nada aqui o substitui.

Este arquivo faz outra coisa: examina o gateway como **decisão de arquitetura**. Por que ele existe do ponto de vista de acoplamento, o que os livros dizem explicitamente que ele **não** deve fazer, e quais confusões conceituais fazem equipes transformarem o gateway no componente mais problemático do sistema. É o preparo direto para o arquivo seguinte, que compara os três componentes de borda desta trilha.

---

## 1. O problema real: acoplamento ortogonal

A justificativa habitual para o gateway é a mais visível: sem ele, o cliente precisaria conhecer o endereço de cada serviço.

```
SEM GATEWAY                          COM GATEWAY

App ──▶ usuarios.loja.com            App ──▶ api.loja.com ──┬──▶ Serviço Usuários
App ──▶ pedidos.loja.com                                     ├──▶ Serviço Pedidos
App ──▶ produtos.loja.com                                    ├──▶ Serviço Produtos
App ──▶ pagamentos.loja.com                                  └──▶ Serviço Pagamentos

4 endereços, 4 configurações de CORS,   1 endereço. A topologia interna
4 certificados, e o cliente conhece      deixa de ser conhecimento do cliente.
a topologia interna do backend.
```

Isso é verdade, mas é a parte fácil. A motivação arquitetural mais profunda aparece em *The Hard Parts*, que dá nome ao problema: **acoplamento ortogonal**.

O argumento é o seguinte. Microsserviços são organizados por **domínio** — cada serviço é dono do seu contexto delimitado, e o estilo prefere duplicação a acoplamento ("*duplication is preferable to coupling*"). Só que existe uma classe de capacidades que **atravessa todos os domínios** e que se beneficia enormemente de ser uniforme: autenticação, rate limiting, observabilidade, TLS, circuit breaker. São preocupações **necessárias mas independentes** do domínio — ortogonais a ele, no sentido matemático de intersectar em ângulo reto.

O livro coloca a pergunta que dá origem ao gateway: *se cada equipe implementa monitoramento por conta própria, como garantir que todas implementaram? E como coordenar uma atualização dessa biblioteca em toda a organização?*

```
        Domínios (o eixo dos microsserviços)
        ────────────────────────────────────▶
        Usuários │ Pedidos │ Produtos │ Pagamentos
      ┌──────────┼─────────┼──────────┼───────────┐
 Auth │    ×     │    ×    │    ×     │     ×     │  ← preocupações que cortam
 Rate │    ×     │    ×    │    ×     │     ×     │    TODOS os domínios:
 Logs │    ×     │    ×    │    ×     │     ×     │    acoplamento ortogonal
 TLS  │    ×     │    ×    │    ×     │     ×     │
      └──────────┴─────────┴──────────┴───────────┘
                          │
                          ▼
      Duas soluções para o mesmo problema, em posições diferentes:
      • API Gateway  → concentra na BORDA (tráfego que entra)
      • Sidecar/Mesh → distribui JUNTO A CADA serviço (tráfego interno)
```

Essa é a leitura que torna gateway e service mesh compreensíveis de uma vez: **são duas respostas ao mesmo problema de acoplamento ortogonal, aplicadas a eixos de tráfego diferentes** — norte-sul (externo→interno) e leste-oeste (interno→interno). O detalhamento dessa comparação está em `../system-design-pt1/api_gateway.md`, seção 3.1.

---

## 2. Indireção: o que o gateway é, estruturalmente

*Fundamentals of Software Architecture* descreve a camada de API em duas palavras precisas: ela oferece um bom lugar na arquitetura para executar tarefas úteis "**via indireção, como um proxy**".

Indireção é o conceito estrutural aqui. O gateway insere um nível de "quem responde por este endereço" entre o cliente e os serviços, e é esse nível extra que torna possível:

- **Trocar a topologia interna sem quebrar clientes.** Dividir um serviço em dois, renomeá-lo, migrá-lo de máquina — nada disso vaza para o cliente.
- **Versionar a API independentemente dos serviços.** `/v1/pedidos` e `/v2/pedidos` podem apontar para o mesmo serviço com transformações distintas, ou para serviços diferentes.
- **Migrar gradualmente** (padrão *Strangler Fig*): o gateway roteia `/api/pedidos` para o monolito e `/api/produtos` para o microsserviço novo, e a fronteira se move rota a rota, sem *big bang*.
- **Aplicar políticas em um só lugar** em vez de N implementações potencialmente divergentes.

Uma observação importante do mesmo livro: a camada de API é **opcional**. Ela aparece em quase todo diagrama de microsserviços, mas o livro é explícito ao classificá-la como opcional — comum porque é conveniente, não porque o estilo a exige.

> **Confusão comum:** "toda arquitetura de microsserviços precisa de um API Gateway". ✅ **Mais preciso:** a camada de API é descrita como **opcional** em *Fundamentals of Software Architecture* — ela é comum porque oferece um lugar conveniente para tarefas transversais, não porque o estilo arquitetural a torne obrigatória. Um sistema com três serviços consumidos por um único frontend próprio pode perfeitamente viver sem gateway: um proxy reverso com regras de roteamento resolve, e cada serviço valida seu próprio token. Adicionar um gateway antes de existirem consumidores diversos, políticas distintas ou serviços em número relevante significa pagar por um salto de rede a mais, um componente a operar e um ponto central de falha — em troca de um problema de coordenação que ainda não apareceu.

---

## 3. O que o gateway legitimamente faz

| Responsabilidade | Por que faz sentido centralizar |
|---|---|
| **Roteamento** | O mapa de "qual path pertence a qual serviço" é conhecimento da borda, não do cliente |
| **Terminação de TLS** | Um certificado para gerenciar e renovar, em vez de N |
| **Autenticação** (validar o token) | Rejeitar tráfego não autenticado o mais cedo possível, antes de gastar recursos internos |
| **Rate limiting e quotas** | Só a borda enxerga o consumidor inteiro; um serviço isolado vê apenas sua fatia do tráfego |
| **Observabilidade** | Um ponto onde toda requisição externa passa e pode ser correlacionada com um *trace id* |
| **Circuit breaker** | Impedir que um serviço degradado seja martelado até cair de vez |
| **Versionamento e depreciação de API** | Manter contratos antigos vivos sem congelar os serviços |
| **Transformação de protocolo** | Expor REST para fora, falar gRPC internamente |
| **Service discovery** | O livro observa que a camada de API é frequentemente usada para hospedar descoberta de serviços |

E o item que costuma ser subestimado: o gateway é onde a **API vira produto**. Chave de API por consumidor, planos de uso, portal de documentação, métricas por cliente, cobrança por volume — nada disso pertence a um serviço de domínio, e nenhum load balancer oferece.

---

## 4. O que o gateway não deve fazer

Este é o ponto em que *Fundamentals of Software Architecture* é mais categórico, e vale citar a formulação:

> A camada de API **não deve ser usada como mediador ou ferramenta de orquestração** se o arquiteto quiser permanecer fiel à filosofia dessa arquitetura: toda lógica interessante deve ocorrer **dentro de um contexto delimitado**, e colocar orquestração ou outra lógica em um mediador viola essa regra.

O raciocínio: microsserviços é uma arquitetura **particionada por domínio**. Mediadores e orquestradores são característicos de arquiteturas **particionadas tecnicamente**. Colocar lógica de negócio na camada de API mistura os dois modelos e desfaz a propriedade que dá valor ao estilo — a de que uma mudança de regra de negócio se resolve dentro de um serviço, por uma equipe, com um deploy.

O sintoma da violação tem nome informal na indústria: **"o ESB renascido"**. O *Enterprise Service Bus* da era SOA acumulou roteamento, transformação, orquestração e regras de negócio em um componente central, e o resultado foi um gargalo organizacional — toda mudança precisava passar pela equipe do barramento. Um gateway que ganha lógica de negócio percorre exatamente o mesmo caminho.

```
LEGÍTIMO                                PROBLEMÁTICO
─────────────────────────────────────────────────────────────────────────
"Requisições para /pedidos vão          "Se o valor do pedido > R$ 500,
 para o serviço de pedidos"              chame também o serviço antifraude
                                         antes de encaminhar"

"Rejeite se o token for inválido"       "Se o usuário for do plano free,
                                         limite o campo X da resposta"

"Adicione o header X-Trace-Id"          "Junte a resposta de 3 serviços e
                                         calcule o total com desconto"
```

O terceiro caso do lado direito é o mais sutil, porque agregação é útil de verdade — e é exatamente por isso que existe o padrão **BFF (Backend for Frontend)**, detalhado em `../system-design-pt1/api_gateway.md`, seção 4. A diferença não é técnica, é de **posicionamento e propriedade**: um BFF é um serviço próprio, versionado e mantido pela equipe daquele cliente, que pode conter lógica de composição; o gateway é infraestrutura compartilhada, e lógica de negócio dentro dele não tem dono claro.

> **Confusão comum:** "o gateway resolve o problema de o mobile precisar chamar 5 endpoints para montar uma tela". ✅ **Mais preciso:** resolve **se** ele agregar respostas — e agregar é precisamente a responsabilidade que os livros recomendam manter **fora** do gateway genérico, porque agregação carrega regra de negócio (o que juntar, em que ordem, o que fazer se um dos serviços falhar, qual resposta parcial é aceitável). A solução recomendada é um **BFF**: um serviço de composição por tipo de cliente, atrás do gateway, com dono, ciclo de vida e testes próprios. O gateway continua fazendo o que sabe (auth, rate limit, roteamento) e encaminha para o BFF como para qualquer outro serviço.

> **Confusão comum:** "o gateway é o lugar certo para aplicar as regras de acesso do sistema". ✅ **Mais preciso:** convém separar **autenticação** de **autorização**, exatamente como faz `../system-design-pt1/seguranca.md`. Verificar que o token é válido, não expirou e tem assinatura correta — **autenticação** — é trabalho de borda por excelência: é uma checagem genérica, idêntica para todas as rotas, e barata de fazer cedo. Já decidir se *este usuário específico* pode alterar *este pedido específico* — **autorização** — depende de dados e regras que só o serviço de domínio possui (quem é o dono do pedido, em que estado ele está, qual o papel do usuário naquele contexto). Empurrar autorização fina para o gateway significa levar conhecimento de domínio para a infraestrutura, e é uma das formas mais comuns de o gateway virar um ESB por acidente.

---

## 5. O gateway não é uma fronteira de confiança

Um padrão mental frequente: "o gateway autentica tudo que entra, logo os serviços atrás dele podem confiar em qualquer requisição que recebam". Isso é o modelo de **segurança perimetral**, e ele falha por vários caminhos:

```
                    ┌──────────────┐
   Internet ───────▶│   Gateway    │───┐
                    └──────────────┘   │
                                       ▼
   ┌───────────────────────────────────────────────────────┐
   │  REDE INTERNA                                          │
   │                                                        │
   │   Serviço A ──────▶ Serviço B      ← não passa pelo    │
   │                                       gateway          │
   │   Job/cron  ──────▶ Serviço B      ← idem              │
   │                                                        │
   │   Atacante que comprometeu QUALQUER coisa aqui dentro   │
   │   fala com todo mundo sem apresentar credencial         │
   └───────────────────────────────────────────────────────┘
```

O gateway controla apenas o tráfego **norte-sul**. Chamadas entre serviços, jobs agendados, ferramentas internas e qualquer coisa que já esteja dentro da rede **não passam por ele**. É por isso que o modelo *zero trust* estabelece que cada serviço deve validar identidade e permissão por conta própria — e é por isso que o service mesh existe: ele aplica mTLS e políticas de autorização no tráfego leste-oeste, onde o gateway não alcança.

> **Confusão comum:** "com o gateway validando os tokens, meus serviços internos não precisam se preocupar com autenticação". ✅ **Mais preciso:** essa premissa transforma qualquer brecha na rede interna em acesso total. O gateway vê só o tráfego que entra pela borda; comunicação serviço-a-serviço, jobs e ferramentas internas passam por fora dele. Se um serviço de baixo risco for comprometido, ele fala com o serviço de pagamentos sem apresentar credencial nenhuma. A prática consolidada é o gateway fazer a **primeira** validação (rejeitar cedo o que é obviamente inválido) e cada serviço fazer a validação **que lhe cabe** — identidade do chamador e permissão sobre o recurso concreto. Defesa em profundidade, não perímetro.

---

## 6. O gateway é um SPOF e um gargalo

Toda requisição externa passa por ele. Isso significa:

- **Se ele cai, tudo cai** — mesmo que todos os serviços estejam perfeitamente saudáveis.
- **A latência dele entra em toda requisição.** Validar JWT, consultar rate limit no Redis, resolver rota, reescrever headers: alguns milissegundos multiplicados por 100% do tráfego.
- **Ele precisa escalar tanto quanto a soma dos serviços atrás dele.**
- **Ele vira ponto de coordenação organizacional.** Se toda equipe precisa abrir um chamado para a equipe de plataforma a fim de publicar uma rota, o gateway virou o gargalo que os microsserviços deveriam ter eliminado.

Os quatro problemas têm respostas conhecidas: várias instâncias do gateway atrás de um load balancer (é *stateless* por natureza — o estado de rate limit fica no Redis); cache de chaves públicas JWKS para não consultar o servidor de identidade a cada token; configuração declarativa versionada em Git, com pipeline próprio, para que cada equipe publique suas rotas sem intermediário humano.

> **Confusão comum:** "o gateway é um serviço gerenciado da nuvem, então não tem limites nem pontos de atenção". ✅ **Mais preciso:** gateways gerenciados trocam a operação do componente por **restrições de plataforma** que aparecem tarde e doem: limites de tamanho de payload, tempo máximo de resposta antes de o gateway abortar a requisição, cotas de requisições por segundo por conta, latência adicional de *cold start* em integrações serverless, e um custo por milhão de requisições que se torna significativo em alto volume. Nada disso é defeito — são as fronteiras do serviço, e a hora de conhecê-las é ao escolher, não quando um upload de 20 MB começa a falhar em produção.

---

## 7. Implementações

| Categoria | Exemplos | Observação |
|---|---|---|
| **Gerenciado** | AWS API Gateway, Google Apigee, Azure API Management | Operação zero; limites de plataforma; custo por requisição |
| **Self-hosted, orientado a API** | Kong, Tyk, KrakenD, Gravitee | Kong é construído sobre OpenResty (Nginx + Lua) — ver `nginx.md`, seção 8 |
| **Proxy moderno com papel de gateway** | Envoy, Traefik, HAProxy | Envoy é também o *data plane* mais comum de service meshes |
| **Nginx configurado** | Nginx open source / Plus | Cobre roteamento, TLS, rate limit; não traz gestão de consumidores, portal, plugins de auth |
| **Kubernetes** | Ingress NGINX, Gateway API | Interface declarativa; a implementação por baixo é um desses proxies |
| **Construído sob medida** | FastAPI, Express | Ver o tutorial em `../system-design-pt1/api_gateway.md`, seção 5 |

> **Confusão comum:** "um Ingress do Kubernetes é o API Gateway do meu cluster". ✅ **Mais preciso:** um Ingress resolve **roteamento HTTP de entrada e terminação de TLS** — ou seja, o subconjunto do gateway que um proxy reverso já cobria. Ele não traz, de fábrica, gestão de consumidores e chaves de API, quotas por cliente, transformação de requisição, versionamento de contrato, portal de documentação ou catálogo de plugins de autenticação. Na prática, muitas equipes complementam o Ingress com anotações e plugins até reconstruir um gateway parcial — o que funciona, mas convém ser uma escolha consciente. A **Gateway API** do Kubernetes foi criada justamente para expressar de forma nativa boa parte do que o Ingress não expressava.

---

## Resumo do arquivo

- A justificativa profunda do gateway é o **acoplamento ortogonal**: preocupações (auth, rate limit, TLS, observabilidade) que cortam todos os domínios e se beneficiam de ser uniformes. Gateway e service mesh são duas respostas ao mesmo problema, em eixos de tráfego diferentes.
- Estruturalmente ele é **indireção**: um nível a mais entre cliente e serviços, que permite mudar topologia, versionar API e migrar gradualmente sem quebrar consumidores.
- A camada de API é **opcional** — comum por conveniência, não por exigência do estilo arquitetural.
- **Não deve mediar nem orquestrar.** Lógica de negócio no gateway reproduz o ESB da era SOA, com o mesmo desfecho: gargalo técnico e organizacional. Agregação de respostas pertence a um **BFF**, não ao gateway.
- **Autenticação na borda, autorização no domínio.** Validar o token é genérico; decidir se este usuário pode alterar este recurso depende de conhecimento que só o serviço tem.
- O gateway **não é uma fronteira de confiança**: ele só vê tráfego norte-sul, e tudo que já está na rede interna passa por fora dele.
- Ele **é SPOF, gargalo de latência e potencial gargalo organizacional** — todos com solução conhecida, todos exigindo decisão explícita.

**Próxima leitura:** `load_balancer_vs_nginx_vs_api_gateway.md` — os três componentes desta trilha lado a lado, e por que compará-los diretamente é, antes de tudo, um erro de categoria.
