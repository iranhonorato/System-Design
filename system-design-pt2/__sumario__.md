# Sumário — Parte 2: A Borda e a Camada de Dados

Esta parte aprofunda os componentes que ficam **entre o usuário e a aplicação** — e a única camada que não pode ser tornada *stateless*, o banco de dados. Onde a parte 1 apresentou esses temas em visão geral, aqui cada um é aberto por inteiro.

Além do conteúdo técnico, cada arquivo marca explicitamente as **confusões comuns de aprendizado** no formato `Confusão comum → Mais preciso`, porque boa parte da dificuldade nesses temas não vem de desconhecer o conceito, e sim de tê-lo aprendido com uma imprecisão que só aparece em produção.

---

## Trilha de Leitura

```
A DECISÃO
  ① escalabilidade_horizontal_vs_vertical.md
         │
         ▼
OS COMPONENTES DE BORDA
  ② load_balancer.md                       ← a função
  ③ nginx.md                               ← o produto
  ④ api_gateway.md                         ← o papel arquitetural
  ⑤ load_balancer_vs_nginx_vs_api_gateway.md ← a síntese
         │
         ▼
A CAMADA DE DADOS
  ⑥ arquitetura_master_slave.md
```

---

## ① [escalabilidade_horizontal_vs_vertical.md](escalabilidade_horizontal_vs_vertical.md)

**Pré-requisito:** `../system-design-pt1/arquitetura.md`, `../system-design-pt1/escalabilidade.md`

A decisão que condiciona todas as outras desta trilha. Cobre o que significa "não aguentar mais" (a curva de degradação e o recurso que saturou), os prós e contras reais de cada estratégia, o **pré-requisito escondido da escala horizontal** — tornar a aplicação *stateless*, com o inventário de todo estado que costuma estar escondido em cada instância — e a distinção formal entre **escalabilidade, performance, elasticidade e disponibilidade**, que informalmente são tratadas como sinônimos e não são. Fecha explicando por que dobrar as máquinas não dobra a capacidade, e como decidir na prática.

> **Confusões tratadas:** "meu sistema está lento, preciso escalar"; "escala vertical é ultrapassada"; "escalar horizontalmente é só subir mais instâncias"; "*sticky sessions* resolvem o problema de estado"; "*auto scaling* resolve picos"; "escalar deixa o sistema mais rápido"; "escala horizontal elimina o SPOF"; "replicação e sharding são a mesma coisa".

---

## ② [load_balancer.md](load_balancer.md)

**Pré-requisito:** ①, `../system-design-pt1/threads_e_sockets.md`

O componente que a escala horizontal torna obrigatório. Explica o que um LB **é** estruturalmente (um proxy reverso que escolhe o destino) e a diferença entre proxy direto e reverso; as quatro funções que ele entrega além de distribuir; **L4 vs. L7 em profundidade** — o que cada um enxerga, o que consegue decidir com isso, e por que a granularidade (conexão vs. requisição) importa tanto sob HTTP/2; oito algoritmos de balanceamento com suas armadilhas; **health checks** e a calibração entre raso demais e profundo demais; o problema da sessão; as seis camadas em que balanceamento acontece simultaneamente; e como o próprio LB deixa de ser um ponto único de falha.

> **Confusões tratadas:** "LB e proxy reverso são a mesma coisa"; "L7 é melhor que L4"; "L4 é mais seguro porque não decifra TLS"; "round robin distribui a carga igualmente"; "*least connections* manda para o menos ocupado"; "health check verde = servidor saudável"; "com LB tenho alta disponibilidade"; "o LB protege contra DDoS".

---

## ③ [nginx.md](nginx.md)

**Pré-requisito:** ②, `../system-design-pt1/threads_e_sockets.md`

O primeiro **produto** da trilha — os anteriores eram conceitos. Cobre os quatro papéis que o Nginx acumula, seu modelo master/worker com event loop, o **modelo de configuração em contextos com herança** e a ordem real de avaliação dos blocos `location` (que não é a ordem do arquivo). Traz configurações comentadas para os quatro usos: servidor de estáticos, proxy reverso (com a regra da barra final do `proxy_pass` e os headers `X-Forwarded-*`), load balancer (bloco `upstream`, algoritmos, health check passivo) e TLS + cache + rate limiting. Fecha com as variantes (NGINX Plus, OpenResty, Ingress NGINX).

> **Confusões tratadas:** "Nginx é um load balancer"; "Nginx é single-threaded"; "o *master* do Nginx é o mesmo *master* do banco"; "os `location` são testados em ordem"; "`root` e `alias` são equivalentes"; "a barra do `proxy_pass` é só estilo"; "o Nginx serve minha aplicação Python"; "`max_fails` é health check"; "editei o `nginx.conf`, já está valendo".

---

## ④ [api_gateway.md](api_gateway.md)

**Pré-requisito:** ②, ③, `../system-design-pt1/arquitetura.md`, `../system-design-pt1/seguranca.md`

O gateway examinado como **decisão de arquitetura**: a motivação profunda (o **acoplamento ortogonal** de *The Hard Parts*), o padrão **Sidecar e o Service Mesh** como a resposta complementar para o tráfego interno, o gateway como mecanismo de **indireção**, suas responsabilidades legítimas, e — o ponto mais categórico dos livros — **o que ele não deve fazer**, com o alerta contra reproduzir o ESB da era SOA. Trata ainda do padrão **BFF** como o lugar onde agregação legitimamente mora, da separação autenticação/autorização, de por que o gateway **não é uma fronteira de confiança**, e de seu papel como SPOF e gargalo.

> **Confusões tratadas:** "todo microsserviço precisa de gateway"; "o gateway resolve o problema de chamar 5 endpoints"; "o gateway é o lugar das regras de acesso"; "com o gateway validando tokens, os serviços não precisam se preocupar"; "gateway gerenciado não tem limites"; "Ingress do Kubernetes é um API Gateway".

---

## ⑤ [load_balancer_vs_nginx_vs_api_gateway.md](load_balancer_vs_nginx_vs_api_gateway.md)

**Pré-requisito:** ②, ③, ④

A síntese dos três anteriores, e a resposta à pergunta mais recorrente da área. Começa mostrando que compará-los diretamente é um **erro de categoria** — load balancer é uma **função**, API Gateway é um **papel arquitetural**, Nginx é um **produto** — e que a pergunta correta tem três etapas: quais funções preciso, que papel isso configura, qual produto implementa. Traz a tabela dos três lado a lado, o diagrama do que se sobrepõe e do que é exclusivo de cada um, os sinais concretos de quando o Nginx deixa de bastar como gateway, o diagrama de como os três coexistem em um sistema real, e um guia de decisão por necessidade.

> **Confusões tratadas:** "preciso escolher entre os três"; "Nginx é a versão gratuita do API Gateway"; "LB L7 é a mesma coisa que gateway"; "se tenho gateway não preciso de LB"; "o gateway substitui o Nginx".

---

## ⑥ [arquitetura_master_slave.md](arquitetura_master_slave.md)

**Pré-requisito:** ①, `../system-design-pt1/arquitetura.md` (Teorema CAP)

A última camada do caminho da requisição — e a única que não pode ser tornada *stateless*. Explica o modelo master/slave e a **terminologia atual** de cada banco (source/replica, primary/standby, leader/follower), como a replicação funciona por baixo (log de transações, física vs. lógica, comando vs. linha), o espectro **síncrona / semissíncrona / assíncrona** como materialização direta do Teorema CAP, e **replication lag** — a origem de uma classe de bugs que não reproduz em desenvolvimento, com as cinco estratégias conhecidas de mitigação. Detalha o **failover** passo a passo (detecção, escolha, fencing, promoção, reconfiguração) e por que sem quorum e fencing ele produz split-brain. Fecha com os limites do modelo e as variantes (cascata, multi-master, quorum, sharding, CQRS).

> **Confusões tratadas:** "síncrona elimina todos os problemas"; "o lag é irrelevante na prática"; "meu ORM já distribui a carga"; "failover automático é sempre melhor"; "mais réplicas = mais capacidade de leitura, sempre"; **"réplica é backup"**; "se o master cair não perco nada"; "master/slave, quorum e sharding são gerações da mesma ideia".

---

## Mapa de Dependências

```
   PARTE 1                                  PARTE 2
   ───────                                  ───────

   arquitetura.md ─────────┐
   escalabilidade.md ──────┼──▶ ① escalabilidade_horizontal_vs_vertical
                           │              │
   threads_e_sockets.md ───┼──▶ ② load_balancer ──▶ ⑥ arquitetura_master_slave
                           │              │
                           │              ▼
                           └──▶ ③ nginx ──┐
                                          ├──▶ ⑤ load_balancer_vs_nginx_vs_api_gateway
   seguranca.md ───────────────▶ ④ api_gateway ──┘
   redis.md ──────────────────────┘
```

---

## Referência Rápida por Assunto

| Quero entender... | Leia |
|---|---|
| Quando escalar para cima e quando escalar para os lados | ① |
| Por que minha aplicação quebrou ao subir a segunda instância | ① (seção 3) |
| A diferença entre escalabilidade, elasticidade e performance | ① (seção 4) |
| Como um load balancer decide o destino de cada requisição | ② (seções 3 e 4) |
| Por que meu health check está verde com o sistema fora do ar | ② (seção 5) |
| Configurar Nginx como proxy reverso ou balanceador | ③ (seções 5 e 6) |
| Por que meu backend recebe 404 depois de passar pelo Nginx | ③ (seção 5) |
| Por que minha aplicação só vê o IP do proxy | ③ (seção 5) |
| O que um API Gateway deve — e não deve — fazer | ④ (seções 3 e 4) |
| Onde fica autenticação e onde fica autorização | ④ (seção 4) |
| **Qual dos três eu devo usar** | ⑤ |
| Quando o Nginx deixa de bastar como gateway | ⑤ (seção 5) |
| Como replicar um banco e distribuir as leituras | ⑥ (seções 2 e 3) |
| Por que o usuário não vê o que acabou de salvar | ⑥ (seção 5) |
| O que acontece de verdade quando o master cai | ⑥ (seção 6) |
| Por que réplica não é backup | ⑥ (seção 8) |

---

## Relação com a Parte 1

Esta parte **aprofunda** temas que a parte 1 apresentou em visão geral. Onde há sobreposição, a divisão é:

| Tema | Parte 1 | Parte 2 |
|---|---|---|
| Escalabilidade | Panorama das técnicas (CDN, sharding, consistent hashing, rate limiting) | A decisão vertical vs. horizontal em profundidade |
| Load balancer | Conceito e três algoritmos | Estrutura, L4/L7, oito algoritmos, health checks, redundância |
| Nginx | Modelo de concorrência (`../system-design-pt1/threads_e_sockets.md`) | O produto inteiro: configuração, papéis e armadilhas |
| API Gateway | — (o tema saiu da parte 1) | O arquivo inteiro: decisão arquitetural, Sidecar/Service Mesh, BFF, limites e antipadrões |
| Master/slave | Meia página em `../system-design-pt1/escalabilidade.md` | O arquivo inteiro |
