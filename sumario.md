# Sumário — Ordem de Leitura

Este projeto é um guia progressivo de arquitetura de software e desenvolvimento backend. Os arquivos foram organizados de forma que cada leitura constrói sobre o conhecimento da anterior — comece do início se você estiver começando, ou pule direto para o tema que precisar.

---

## Trilha de Leitura

```
FUNDAMENTOS
  ① infraestrutura.md
  ② arquitetura.md
  ③ docker.md
  ④ aws.md
  ⑤ threads_e_sockets.md
         │
         ▼
QUALIDADE E ENTREGA
  ⑥ testes_de_software.md
  ⑦ ci_cd_introduction.md
  ⑧ ci_hannds_on.md
  ⑨ cd_hands_on.md
         │
         ▼
FERRAMENTAS E ESPECIALIZAÇÃO
  ⑩ redis.md
  ⑪ seguranca.md
  ⑫ auth0.md
  ⑬ api_gateway.md
```

---

## ① [infraestrutura.md](infraestrutura.md)

**Pré-requisito:** nenhum — é o ponto de partida.

Explica o que sustenta qualquer software antes de uma linha de código rodar. Cobre as três camadas da infraestrutura (física, lógica e cloud), a evolução histórica do bare metal até containers (por que cada era surgiu e qual problema resolveu), o conceito de Infraestrutura como Código (IaC) com Terraform, Ansible e Kubernetes, e por que desenvolvedores modernos precisam entender infraestrutura.

> Leia primeiro porque todos os outros arquivos assumem que você sabe o que é um servidor, uma VM e um container.

---

## ② [arquitetura.md](arquitetura.md)

**Pré-requisito:** ① infraestrutura.md

Apresenta os padrões de organização de sistemas: arquiteturas clássicas (monolítica, em camadas N-Tier, cliente-servidor), arquiteturas modernas (microsserviços, orientada a eventos, serverless, orientada a APIs) e como escolher a certa para cada contexto. Inclui o **Teorema CAP** — a lei fundamental dos sistemas distribuídos que governa a escolha entre consistência e disponibilidade.

> Leia antes de qualquer discussão sobre microsserviços ou sistemas distribuídos. Sem esse mapa mental, decisões arquiteturais parecem arbitrárias.

---

## ③ [docker.md](docker.md)

**Pré-requisito:** ① infraestrutura.md

Desmonta o Docker de dentro para fora: imagens, containers, Dockerfile, camadas e cache. Explica os mecanismos do kernel Linux que tornam containers possíveis (namespaces e cgroups). Cobre **Docker Compose** para orquestrar múltiplos serviços localmente e **multi-stage builds** para criar imagens de produção enxutas.

> Leia antes de AWS e CI/CD — deploy moderno é quase sempre deploy de containers.

---

## ④ [aws.md](aws.md)

**Pré-requisito:** ③ docker.md

Guia prático de EC2 (servidores virtuais na AWS): como funciona o acesso via SSH com chave `.pem`, o que são Security Groups, tipos de instância e o fluxo completo do "zero ao servidor rodando". Inclui uma seção sobre **IAM** (o sistema de permissões da AWS) e boas práticas de segurança no ambiente cloud.

> Leia antes dos hands-on de CI/CD — o pipeline de CD faz deploy em uma EC2.

---

## ⑤ [threads_e_sockets.md](threads_e_sockets.md)

**Pré-requisito:** ② arquitetura.md

Explica os dois mecanismos fundamentais de todo sistema em rede: **sockets** (como programas se comunicam pela rede — TCP handshake, portas, canais bidirecionais) e **threads** (como um programa executa múltiplas tarefas — single vs multi-thread, condições de corrida, locks). Cobre também os modelos de concorrência que aparecem em todo lugar: Node.js (event loop), Nginx (multi-processo com event loop por worker), Redis (single-thread genuíno) e Python asyncio.

> Leia para entender *por que* ferramentas como Redis e Nginx são projetadas da forma que são — e por que o Redis ser single-threaded é uma vantagem, não uma limitação.

---

## ⑥ [testes_de_software.md](testes_de_software.md)

**Pré-requisito:** ③ docker.md

Explica por que testes existem (a "Regra dos 10x"), a pirâmide de testes (unitários → integração → E2E), e como implementar cada camada em Python com FastAPI. Inclui **Test Doubles** (mocks, stubs, fakes, spies), um pipeline de CI completo com GitHub Actions, e boas práticas como o padrão AAA e testes independentes entre si.

> Leia antes de CI/CD — um pipeline sem testes é apenas um script de deploy automatizado. Os testes são o coração do CI.

---

## ⑦ [ci_cd_introduction.md](ci_cd_introduction.md)

**Pré-requisito:** ⑥ testes_de_software.md

Apresenta os conceitos de Continuous Integration, Continuous Delivery e Continuous Deployment, a diferença entre eles, e por que o modelo tradicional de releases mensais é problemático. Cobre estratégias de deploy sem downtime (**Blue-Green**, **Canary**, **Rolling Update**) e as métricas DORA para medir a saúde de um processo de entrega.

> Leia antes dos hands-on. É o mapa conceitual — os hands-on seguintes são a implementação prática.

---

## ⑧ [ci_hannds_on.md](ci_hannds_on.md)

**Pré-requisito:** ⑦ ci_cd_introduction.md, ③ docker.md

Hands-on de CI: configura um pipeline no **GitHub Actions** que roda lint, testes e build de imagem Docker automaticamente a cada `git push`. Mostra o arquivo `.github/workflows/ci.yml` completo e explica cada passo — desde o checkout do código até o report de cobertura.

> Leia e pratique: clone o projeto, configure o workflow e veja o pipeline rodar no GitHub.

---

## ⑨ [cd_hands_on.md](cd_hands_on.md)

**Pré-requisito:** ⑧ ci_hannds_on.md, ④ aws.md

Hands-on de CD: estende o pipeline do arquivo anterior para fazer deploy automático em uma instância EC2. O fluxo completo: testes passam → imagem Docker buildada → push para o Docker Hub → SSH automático na EC2 → container atualizado em produção. Inclui a configuração de usuário de deploy com mínimo privilégio e armazenamento seguro de credenciais nos Secrets do GitHub.

> A conclusão prática de toda a trilha de fundamentos: código no git → aplicação em produção, sem intervenção manual.

---

## ⑩ [redis.md](redis.md)

**Pré-requisito:** ⑤ threads_e_sockets.md, ② arquitetura.md

Cobre o Redis de ponta a ponta: por que dados em RAM são ordens de magnitude mais rápidos, as estruturas de dados nativas (String, Hash, List, Set, Sorted Set), persistência (RDB vs AOF), TTL e expiração de chaves. Implementa cinco casos de uso reais em Python/FastAPI: cache de produtos, sessões de usuário, rate limiting, leaderboard em tempo real e filas de processamento assíncrono. Inclui a distinção crítica entre Pub/Sub e **Redis Streams** para sistemas orientados a eventos.

> Leia quando precisar entender cache, sessões, filas leves ou comunicação entre serviços. O Redis aparece como componente em quase todo sistema de médio porte.

---

## ⑪ [seguranca.md](seguranca.md)

**Pré-requisito:** ② arquitetura.md, ⑤ threads_e_sockets.md, ⑩ redis.md

O guia mais denso do projeto. Separa e aprofunda os três conceitos: **autenticação** (fatores, MFA, TOTP), **tokens** (JWT vs session, Access + Refresh Token, onde armazenar no frontend), **armazenamento de senhas** (por que MD5 é inseguro, rainbow tables, salt, BCrypt vs Argon2id), **autorização** (RBAC vs ABAC, princípio do menor privilégio), **ataques** (SQL Injection, XSS, CSRF, Brute Force, IDOR) com prevenção em código, HTTPS/TLS, OWASP Top 10 e um checklist de produção.

> Leia antes de implementar qualquer sistema com login. Os erros cobertos aqui são responsáveis pela maioria dos vazamentos de dados reais.

---

## ⑫ [auth0.md](auth0.md)

**Pré-requisito:** ⑪ seguranca.md

Aplica os conceitos do arquivo anterior em uma solução concreta: o Auth0, plataforma de identidade como serviço. Explica o que o Auth0 resolve (e o que não resolve), os fluxos OAuth 2.0 e OpenID Connect, como o Authorization Code Flow funciona passo a passo, e como implementar autenticação completa em React (frontend), Node.js/Express e Python/FastAPI. Cobre recursos avançados: Actions para customizar tokens, RBAC no painel, e SSO entre aplicações.

> Leia quando precisar implementar autenticação em um projeto real sem construir do zero. O Auth0 abstrai a complexidade do arquivo anterior.

---

## ⑬ [api_gateway.md](api_gateway.md)

**Pré-requisito:** ② arquitetura.md, ⑩ redis.md, ⑫ auth0.md

Fecha o ciclo de microsserviços com o componente que unifica tudo: o API Gateway. Explica o problema de expor múltiplos serviços diretamente (clientes precisariam conhecer cada endereço), as responsabilidades que o Gateway centraliza (roteamento, autenticação, rate limiting, SSL termination, circuit breaker), a diferença entre Gateway, Load Balancer e Service Mesh, e o padrão **BFF** (Backend for Frontend). Implementa com Nginx, Kong e Node.js/Express, e mostra como o Gateway e o Auth0 se complementam.

> Leia para entender como as peças se conectam em uma arquitetura de microsserviços real: Auth0 emite tokens → Gateway valida → serviços internos processam.

---

## Mapa de Dependências

```
infraestrutura
      │
      ├──▶ arquitetura ──▶ threads_e_sockets ──▶ redis ──▶ api_gateway
      │         │                                  │
      │         └──────────────────────────────────┴──▶ autenticação
      │                                                       │
      ├──▶ docker ──▶ aws ──▶ testes ──▶ ci_intro            └──▶ auth0
      │                           │          │
      │                           └──────────┴──▶ ci_hands_on ──▶ cd_hands_on
      │
      └── (todos os outros dependem implicitamente deste)
```

---

## Referência Rápida por Assunto

| Quero entender... | Leia |
|---|---|
| O que é infraestrutura e como a cloud funciona | ①, ④ |
| Monolito vs microsserviços vs serverless | ② |
| Docker e containers | ③ |
| Como programas se comunicam em rede | ⑤ |
| Como testar meu código corretamente | ⑥ |
| Como automatizar deploys | ⑦, ⑧, ⑨ |
| Cache, filas e sessões rápidas | ⑩ |
| Login, senhas e segurança | ⑪ |
| Implementar autenticação com Auth0 | ⑫ |
| Centralizar e proteger APIs | ⑬ |
