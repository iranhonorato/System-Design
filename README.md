# System Design

Este repositório é um guia de estudo sobre como sistemas de software são projetados para funcionar, escalar e se manter — da infraestrutura que sustenta uma aplicação até as decisões que definem como seus componentes se organizam, se comunicam e continuam confiáveis sob carga. A proposta é organizar esses conceitos de forma progressiva e didática, com diagramas, tabelas e exemplos práticos em Python/FastAPI, permitindo compreender não só *o que* cada peça faz, mas *por que* ela existe e qual problema motivou sua criação.

Boa parte do material — especialmente os capítulos de arquitetura, escalabilidade e a comunicação entre serviços — foi revisada e ampliada com base em três referências centrais na área: *Fundamentals of Software Architecture* (Mark Richards & Neal Ford), *Software Architecture: The Hard Parts* (Neal Ford, Mark Richards, Pramod Sadalage & Zhamak Dehghani) e *System Design Interview* (Alex Xu). Onde essas obras não cobrem o assunto diretamente (Docker, AWS, hands-on de CI/CD, Auth0 — temas mais operacionais do que de arquitetura em si), o conteúdo segue práticas e documentações consolidadas de mercado, mantendo o mesmo padrão de profundidade e clareza.

## Natureza do projeto

Este projeto não é uma aplicação de software em execução, mas sim uma base de conhecimento estruturada em arquivos de estudo, organizada na pasta [system-design-pt1/](system-design-pt1/). Ele reúne temas centrais como:

- infraestrutura de software (camadas física, lógica e cloud, evolução de bare metal a containers, Infraestrutura como Código);
- arquitetura de software (características de arquitetura, estilos clássicos e modernos, acoplamento e quantum, o Teorema CAP, consistência de dados e o padrão Saga);
- Docker e containers (imagens, camadas, namespaces, cgroups, multi-stage builds);
- AWS na prática (EC2, SSH, Security Groups, IAM);
- concorrência e comunicação em rede (sockets, threads, event loop, condições de corrida);
- testes de software (pirâmide de testes, test doubles, pipeline de CI);
- CI/CD (conceitos, estratégias de deploy, métricas DORA, hands-on de principio a fim);
- escalabilidade de sistemas distribuídos (load balancer, replicação, CDN, sharding, consistent hashing, rate limiting);
- Redis (estruturas de dados, cache, sessões, filas, Pub/Sub vs. Streams);
- segurança (autenticação, tokens, armazenamento de senhas, autorização, ataques comuns, OWASP);
- Auth0 e OAuth 2.0/OpenID Connect;
- API Gateway (roteamento, rate limiting, circuit breaker, Service Mesh, padrão BFF).

A ideia central é construir uma visão integral de como um sistema nasce simples e vai adquirindo, camada por camada, a infraestrutura, a arquitetura, os processos e as ferramentas que permitem que ele cresça sem desmoronar.

## Objetivo

O objetivo principal é fornecer uma trilha de aprendizagem clara para quem deseja entender:

- o que sustenta um software antes de qualquer linha de código rodar, e como essa base evoluiu até os containers;
- por que monolito, microsserviços, arquitetura orientada a eventos e as demais formas de organizar um sistema existem — cada estilo resolve um trade-off que o anterior não resolvia;
- quais técnicas concretas (cache, replicação, balanceamento, filas, gateways) permitem que um sistema aguente crescimento real de usuários e dados sem reescrever tudo do zero;
- quais mecanismos de autenticação, autorização e segurança tornam um sistema em produção confiável.

Não é pré-requisito nenhum conhecimento prévio de arquitetura: o primeiro arquivo parte do zero, e cada leitura seguinte assume apenas o vocabulário já apresentado nas anteriores.

## Como utilizar

A leitura recomendada começa por [system-design-pt1/\_\_sumario\_\_.md](system-design-pt1/__sumario__.md), que organiza os temas em uma ordem lógica de aprendizado, com um mapa de dependências entre arquivos e uma tabela de referência rápida por assunto. O material foi pensado para ser lido sequencialmente — cada arquivo assume o vocabulário dos anteriores — mas também pode ser consultado por assunto conforme a necessidade.

## Estrutura do conteúdo

O repositório está dividido em três blocos temáticos, na ordem de leitura sugerida:

**Fundamentos**
- [system-design-pt1/infraestrutura.md](system-design-pt1/infraestrutura.md) — o que sustenta qualquer software antes do código rodar.
- [system-design-pt1/arquitetura.md](system-design-pt1/arquitetura.md) — como organizar os componentes de um sistema e por quê.
- [system-design-pt1/docker.md](system-design-pt1/docker.md) — imagens, containers e o mecanismo por trás deles.
- [system-design-pt1/aws.md](system-design-pt1/aws.md) — da EC2 ao servidor rodando em produção.
- [system-design-pt1/threads_e_sockets.md](system-design-pt1/threads_e_sockets.md) — como programas conversam pela rede e executam em paralelo.

**Qualidade e Entrega**
- [system-design-pt1/testes_de_software.md](system-design-pt1/testes_de_software.md) — por que testar e como implementar cada camada da pirâmide.
- [system-design-pt1/ci_cd_introduction.md](system-design-pt1/ci_cd_introduction.md) — o mapa conceitual da entrega contínua.
- [system-design-pt1/ci_hannds_on.md](system-design-pt1/ci_hannds_on.md) — CI na prática com GitHub Actions.
- [system-design-pt1/cd_hands_on.md](system-design-pt1/cd_hands_on.md) — deploy automático até uma instância EC2.

**Ferramentas e Especialização**
- [system-design-pt1/escalabilidade.md](system-design-pt1/escalabilidade.md) — as técnicas concretas para levar um sistema de um usuário a milhões.
- [system-design-pt1/redis.md](system-design-pt1/redis.md) — cache, sessões, filas e comunicação entre serviços.
- [system-design-pt1/seguranca.md](system-design-pt1/seguranca.md) — autenticação, senhas, autorização e os ataques mais comuns.
- [system-design-pt1/auth0.md](system-design-pt1/auth0.md) — autenticação como serviço, na prática.
- [system-design-pt1/api_gateway.md](system-design-pt1/api_gateway.md) — o componente que unifica o acesso a um sistema de microsserviços.

## Conclusão

Este projeto representa uma tentativa de transformar conceitos de infraestrutura, arquitetura e escalabilidade — muitas vezes aprendidos soltos e fora de ordem — em uma trilha coesa, onde cada arquivo explica o problema que motivou o próximo.
