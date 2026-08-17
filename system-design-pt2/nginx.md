# Nginx

**Pré-requisito:** `load_balancer.md`, `../system-design-pt1/threads_e_sockets.md`

> Os livros de arquitetura tratam o Nginx como uma **implementação** de conceitos que eles descrevem em abstrato — *Fundamentals of Software Architecture* o cita como uma das formas de materializar o *messaging grid* de uma arquitetura baseada em espaço. Por isso este arquivo segue a documentação oficial e as práticas consolidadas de mercado, mantendo o mesmo padrão de profundidade dos demais.

---

## Por que este arquivo existe

Os dois arquivos anteriores descreveram **conceitos**: escala horizontal é uma estratégia, load balancer é um papel. Este arquivo trata de um **produto** — um binário concreto que você instala, configura e executa.

Essa mudança de natureza é justamente a fonte da maior confusão desta trilha. "Load balancer" responde à pergunta *o que precisa acontecer*; "Nginx" responde à pergunta *o que eu instalo para que aconteça*. Um mesmo Nginx pode desempenhar quatro papéis diferentes ao mesmo tempo — e é exatamente por isso que ele aparece na conversa toda vez que se fala de proxy, balanceamento ou gateway.

---

## 1. O que o Nginx é

O Nginx (lê-se "engine-x") foi escrito por Igor Sysoev e lançado em 2004 para resolver um problema específico e bem documentado da época: o **problema C10k** — como atender dez mil conexões simultâneas em uma única máquina. O modelo dominante até então (o Apache clássico, com um processo ou thread por conexão) esbarrava em memória e em troca de contexto muito antes disso.

Hoje ele acumula quatro papéis, e é essencial entender que são papéis **distintos**, não sinônimos:

| Papel | O que faz | Configuração típica |
|---|---|---|
| **Servidor web** | Serve arquivos do disco (HTML, CSS, JS, imagens) | `root`, `try_files` |
| **Proxy reverso** | Encaminha requisições a outro processo e devolve a resposta | `proxy_pass` |
| **Load balancer** | Distribui entre vários backends equivalentes | `upstream` + `proxy_pass` |
| **Terminador de TLS / cache / limitador** | Decifra HTTPS, guarda respostas, aplica rate limit | `ssl_certificate`, `proxy_cache`, `limit_req` |

```
                    UM ÚNICO PROCESSO NGINX

  Internet ──▶ ┌──────────────────────────────────────────┐
               │  TLS termination (443 → texto claro)      │
               │  ├─ /static/*  → lê do disco              │  ← servidor web
               │  ├─ /api/*     → proxy para a aplicação    │  ← proxy reverso
               │  │              (escolhendo entre 3        │  ← load balancer
               │  │               instâncias do upstream)   │
               │  └─ cache + rate limit em todas as rotas   │
               └──────────────────────────────────────────┘
```

> **Confusão comum:** "Nginx é um load balancer". ✅ **Mais preciso:** Nginx é um **servidor HTTP que também sabe fazer proxy reverso, e cujo proxy reverso também sabe balancear**. A ordem histórica importa: ele nasceu servindo arquivos estáticos, ganhou proxy reverso depois, e balanceamento depois ainda. Chamá-lo de "load balancer" é como chamar um smartphone de "câmera" — verdadeiro, incompleto, e leva a decisões erradas (por exemplo, ignorar que ele resolve o problema de servir estáticos, ou não perceber que existem balanceadores mais adequados para tráfego não-HTTP em altíssimo volume).

---

## 2. Como o Nginx executa

O modelo de concorrência do Nginx foi detalhado em `../system-design-pt1/threads_e_sockets.md`; o resumo necessário aqui:

```
  Processo MASTER   ← lê a configuração, gerencia workers, aplica reload.
        │              Roda como root apenas para abrir portas privilegiadas.
        ├── Worker 1 (núcleo 0)  ── event loop (epoll) ── milhares de conexões
        ├── Worker 2 (núcleo 1)  ── event loop (epoll) ── milhares de conexões
        └── Worker N (núcleo N)  ── event loop (epoll) ── milhares de conexões
```

Cada worker é um **processo independente e single-threaded**, com seu próprio event loop baseado em `epoll` (Linux) ou `kqueue` (BSD/macOS). Nenhuma thread é criada por conexão: um worker monitora todos os seus sockets simultaneamente e só trabalha quando há dados prontos. É daí que vem a característica marcante do Nginx — consumo de memória que cresce muito devagar com o número de conexões.

> **Confusão comum:** "o Nginx é single-threaded, como o Redis". ✅ **Mais preciso:** o Nginx é **multi-processo com um event loop single-threaded por processo** — normalmente um worker por núcleo de CPU (`worker_processes auto;`). Ele usa todos os núcleos da máquina; o que ele não faz é criar uma thread por conexão. O Redis, esse sim, é genuinamente single-threaded no caminho de execução de comandos. Confundir os dois leva ao erro prático de dimensionar um servidor Nginx como se ele não aproveitasse múltiplos núcleos.

> **Confusão comum:** "o processo *master* do Nginx e o *master* de um banco master/slave são a mesma ideia". ✅ **Mais preciso:** é a mesma palavra usada para duas coisas sem relação. No Nginx, *master* é o processo **supervisor** que gerencia os workers da mesma máquina — não existe "worker de reserva" nem promoção de worker a master; se um worker morre, o master simplesmente cria outro. Na replicação de banco (`arquitetura_master_slave.md`), *master* é o nó que **detém a autoridade de escrita** sobre um dado replicado em outras máquinas, e a promoção de um slave a master é justamente o mecanismo central. Um é supervisão local de processos; o outro é autoridade sobre dados distribuídos.

---

## 3. O modelo de configuração

A configuração do Nginx é uma árvore de **contextos** aninhados. Diretivas declaradas em um contexto são herdadas pelos contextos internos, e podem ser sobrescritas neles. Entender essa hierarquia resolve metade dos problemas de configuração.

```nginx
# ── contexto MAIN (global) ───────────────────────────────────────────────
user  www-data;
worker_processes  auto;          # um worker por núcleo de CPU disponível
error_log  /var/log/nginx/error.log warn;

events {
    # ── contexto EVENTS: como cada worker lida com conexões ──────────────
    worker_connections  4096;    # conexões simultâneas POR worker
                                 # capacidade total ≈ workers × worker_connections
}

http {
    # ── contexto HTTP: tudo que vale para todo tráfego HTTP ──────────────
    include       /etc/nginx/mime.types;
    sendfile      on;            # copia arquivo → socket sem passar pelo
                                 # espaço de usuário (chamada sendfile do kernel)
    keepalive_timeout  65;
    gzip          on;

    server {
        # ── contexto SERVER: um "site" (virtual host) ────────────────────
        listen       80;
        server_name  loja.com www.loja.com;

        location /static/ {
            # ── contexto LOCATION: uma rota dentro deste site ────────────
            root /var/www;
        }
    }
}
```

| Contexto | Responde a que pergunta |
|---|---|
| `main` | Como o processo Nginx roda no sistema operacional? |
| `events` | Como cada worker gerencia conexões? |
| `http` | O que vale para todo o tráfego HTTP deste servidor? |
| `server` | Qual configuração se aplica a **este domínio/porta**? |
| `location` | Qual configuração se aplica a **esta rota** dentro do domínio? |
| `upstream` | Quais são os backends de um grupo, e como escolher entre eles? |
| `stream` | (irmão de `http`) Proxy e balanceamento de TCP/UDP puro — camada 4 |

### A ordem de avaliação dos `location` — a armadilha clássica

O Nginx **não** avalia os blocos `location` de cima para baixo até achar o primeiro que combina. Ele segue uma ordem de precedência própria:

```
1. location =  /exato          → correspondência EXATA. Se casar, para aqui.
2. location ^~ /prefixo        → prefixo mais longo que casar; se for ^~,
                                 para aqui e NÃO testa as expressões regulares.
3. location ~  /regex/         → expressões regulares, na ORDEM DO ARQUIVO,
   location ~* /regex-i/         primeira que casar vence (~* = case-insensitive)
4. location    /prefixo        → o prefixo mais longo guardado no passo 2,
                                 usado só se nenhuma regex casou.
```

> **Confusão comum:** "os blocos `location` são testados na ordem em que aparecem no arquivo, como um `if/elif`". ✅ **Mais preciso:** só as **expressões regulares** são avaliadas na ordem do arquivo. Os prefixos são avaliados por **comprimento** — o mais longo vence, esteja ele onde estiver no arquivo — e uma expressão regular colocada no fim do arquivo pode "roubar" uma requisição de um prefixo declarado bem antes. Isso produz o sintoma clássico: você adiciona `location ~ \.php$` no final da configuração e, sem querer, desvia requisições que deveriam ter sido tratadas por um `location /admin/` declarado lá em cima. O modificador `^~` existe justamente para dizer "se este prefixo casar, nem teste as regex".

---

## 4. Nginx como servidor de arquivos estáticos

```nginx
server {
    listen 80;
    server_name loja.com;

    root /var/www/loja;          # diretório base deste site

    location / {
        # try_files tenta cada alternativa, em ordem, e usa a primeira
        # que existir. O último item é o fallback.
        #   $uri       → o caminho pedido, como arquivo
        #   $uri/      → o caminho pedido, como diretório
        #   /index.html → fallback (essencial para SPAs: React, Vue, Angular,
        #                 onde o roteamento acontece no navegador)
        try_files $uri $uri/ /index.html;
    }

    location /assets/ {
        # Arquivos com hash no nome (app.4f3a2b.js) nunca mudam de conteúdo:
        # podem ser cacheados agressivamente pelo navegador.
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

Servir estáticos é onde o Nginx é imbatível: com `sendfile on`, o arquivo vai do cache de páginas do kernel direto para o socket, sem sequer ser copiado para o espaço de usuário do processo. Nenhum servidor de aplicação (Gunicorn, Node, Puma) chega perto desse custo por requisição.

> **Confusão comum:** `root` e `alias` são intercambiáveis. ✅ **Mais preciso:** eles compõem o caminho final de formas diferentes, e trocar um pelo outro produz 404 silenciosos. Com `location /imagens/ { root /var/www; }`, o Nginx **concatena** o caminho da requisição ao `root`: `/imagens/logo.png` vira `/var/www/imagens/logo.png`. Com `location /imagens/ { alias /var/www/fotos/; }`, o Nginx **substitui** o prefixo da location: `/imagens/logo.png` vira `/var/www/fotos/logo.png`. Regra prática: `root` mantém o prefixo da URL no caminho do disco; `alias` o descarta.

---

## 5. Nginx como proxy reverso

```nginx
server {
    listen 80;
    server_name api.loja.com;

    location /api/ {
        proxy_pass http://127.0.0.1:8000;

        # ── Headers que a aplicação PRECISA receber ──────────────────────
        # Sem eles, a aplicação enxerga apenas o Nginx, e não o cliente real.
        proxy_set_header Host              $host;              # domínio original
        proxy_set_header X-Real-IP         $remote_addr;       # IP do cliente
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;            # http ou https

        # ── Timeouts: quanto o Nginx espera pelo backend ─────────────────
        proxy_connect_timeout  5s;    # para ABRIR a conexão
        proxy_send_timeout    60s;    # entre dois envios ao backend
        proxy_read_timeout    60s;    # entre duas respostas do backend

        # ── Buffering ────────────────────────────────────────────────────
        # Ligado (padrão): o Nginx lê a resposta inteira do backend, libera
        # o backend, e só então entrega ao cliente no ritmo dele. Protege o
        # backend de clientes lentos.
        # Desligue apenas para streaming / SSE / respostas incrementais:
        # proxy_buffering off;
    }
}
```

### A regra da barra final em `proxy_pass`

Este é, com folga, o erro de configuração mais frequente do Nginx:

```
CONFIGURAÇÃO                                   REQUISIÇÃO      CHEGA NO BACKEND COMO
────────────────────────────────────────────────────────────────────────────────────
location /api/ { proxy_pass http://app:8000;  }    /api/users   →  /api/users
location /api/ { proxy_pass http://app:8000/; }    /api/users   →  /users
                                          ▲
                                    esta barra muda tudo
```

A regra formal: se o `proxy_pass` contém **um caminho** depois do host (mesmo que seja apenas `/`), o Nginx **substitui** o prefixo casado pela location por esse caminho. Se contém **apenas host e porta**, ele repassa a URI original intacta.

> **Confusão comum:** "a barra final no `proxy_pass` é irrelevante, é só estilo". ✅ **Mais preciso:** ela decide se o prefixo da rota é removido antes do encaminhamento — e o sintoma de errá-la é uma cascata de 404 vindos do backend, com a configuração do Nginx aparentemente correta. O diagnóstico rápido é olhar o log de acesso da **aplicação** (não o do Nginx) e conferir qual caminho ela de fato recebeu.

> **Confusão comum:** "o Nginx serve minha aplicação Python/Node/PHP". ✅ **Mais preciso:** o Nginx **não executa código de aplicação** — ele não tem interpretador embutido para nenhuma linguagem. Ele encaminha a requisição para um processo separado que executa: Gunicorn/Uvicorn (Python), o próprio processo Node, Puma (Ruby), php-fpm (PHP, via protocolo FastCGI em vez de HTTP). Isso é uma diferença estrutural em relação ao Apache com `mod_php`, que carregava o interpretador **dentro** do próprio servidor web. A consequência prática: se a aplicação está fora do ar, o Nginx responde `502 Bad Gateway` — um erro que sempre significa "eu estou vivo, quem está atrás de mim não está", e nunca "o Nginx quebrou".

> **Confusão comum:** "minha aplicação registra o IP do cliente em `request.remote_addr`, então está tudo certo". ✅ **Mais preciso:** com um proxy reverso na frente, `remote_addr` passa a ser o IP **do proxy** — normalmente `127.0.0.1` ou o IP interno do LB. O IP real do cliente só chega se o proxy o injetar em `X-Forwarded-For` **e** a aplicação for configurada para confiar nesse header. E aqui mora um risco de segurança: `X-Forwarded-For` é um header comum, que **qualquer cliente pode forjar**. Confiar nele cegamente permite que um atacante burle rate limiting por IP ou envenene logs de auditoria. A configuração correta é confiar nesse header **apenas** quando a requisição vem de um proxy conhecido (no Nginx, via módulo `real_ip` com `set_real_ip_from`; no framework, via lista de proxies confiáveis).

---

## 6. Nginx como load balancer

```nginx
http {
    # ── O bloco upstream define o pool de backends ───────────────────────
    upstream aplicacao {
        # Algoritmo: sem diretiva, é round robin ponderado.
        # least_conn;                 # menos conexões ativas
        # ip_hash;                    # afinidade por IP do cliente
        # hash $request_uri consistent;  # consistent hashing (ver pt1, seção 5)
        # random two least_conn;      # duas escolhas aleatórias, a menos carregada

        server 10.0.1.11:8000 weight=3;   # recebe 3x mais que os de peso 1
        server 10.0.1.12:8000;
        server 10.0.1.13:8000;

        # backup: só recebe tráfego se TODOS os demais estiverem fora
        server 10.0.1.99:8000 backup;

        # ── Health check PASSIVO ─────────────────────────────────────────
        # max_fails: falhas dentro de fail_timeout para marcar como indisponível
        # fail_timeout: janela de contagem E tempo que o servidor fica fora
        # (declarados por servidor; os padrões são max_fails=1 fail_timeout=10s)

        # ── Pool de conexões keep-alive com os backends ──────────────────
        # Sem isto, o Nginx abre uma conexão TCP nova a cada requisição
        # encaminhada — desperdício considerável sob volume.
        keepalive 32;
    }

    server {
        listen 80;
        location / {
            proxy_pass http://aplicacao;      # aponta para o upstream
            proxy_http_version 1.1;           # necessário para keepalive
            proxy_set_header Connection "";   # idem
        }
    }
}
```

> **Confusão comum:** "configurei `max_fails` e `fail_timeout`, então o Nginx está fazendo health check dos meus servidores". ✅ **Mais preciso:** o Nginx **open source** faz apenas health check **passivo** — ele nunca envia uma requisição própria para testar um backend; ele observa as requisições **de usuários reais** e conta as que falharam. Isso tem duas consequências que costumam surpreender: **(1)** um servidor que caiu só é detectado depois que `max_fails` usuários reais tomaram erro; **(2)** um servidor marcado como indisponível volta ao pool quando `fail_timeout` expira, **sem nenhuma verificação prévia** — e se ainda estiver quebrado, mais usuários tomam erro para que ele seja removido de novo. Health check **ativo** (`health_check`, com intervalo, limiares e URI própria) é recurso exclusivo do **NGINX Plus**, a versão comercial. Alternativas em software livre para obter checagem ativa: HAProxy, Envoy, Traefik, ou o balanceador gerenciado da nuvem.

---

## 7. TLS, cache e rate limiting

```nginx
# ── Zona de rate limiting, declarada no contexto http ────────────────────
# $binary_remote_addr = IP do cliente em formato binário (ocupa menos memória)
# zone=por_ip:10m     = 10 MB de estado compartilhado entre os workers
#                       (~160 mil IPs distintos)
# rate=10r/s          = 10 requisições por segundo por IP
limit_req_zone $binary_remote_addr zone=por_ip:10m rate=10r/s;

# ── Zona de cache de respostas do backend ────────────────────────────────
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=respostas:10m
                 max_size=1g inactive=60m;

server {
    listen 443 ssl;
    http2 on;
    server_name loja.com;

    ssl_certificate     /etc/letsencrypt/live/loja.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/loja.com/privkey.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;

    location /api/ {
        # burst=20 → permite uma rajada de até 20 requisições acima da taxa
        # nodelay  → atende a rajada imediatamente em vez de enfileirar
        limit_req zone=por_ip burst=20 nodelay;
        proxy_pass http://aplicacao;
    }

    location /produtos/ {
        proxy_cache respostas;
        proxy_cache_valid 200 5m;      # respostas 200 ficam 5 min em cache
        proxy_cache_valid 404 1m;
        # Serve conteúdo vencido enquanto revalida em segundo plano — e,
        # crucialmente, também se o backend estiver fora do ar:
        proxy_cache_use_stale error timeout updating http_502 http_503;
        add_header X-Cache-Status $upstream_cache_status;   # HIT / MISS / STALE
        proxy_pass http://aplicacao;
    }
}

# ── Redirecionamento de HTTP para HTTPS ──────────────────────────────────
server {
    listen 80;
    server_name loja.com;
    return 301 https://$host$request_uri;
}
```

Vale conectar o `limit_req` ao vocabulário já estabelecido: ele implementa o algoritmo **Leaky Bucket** descrito em `../system-design-pt1/escalabilidade.md`, seção 6 — as requisições entram em uma fila de tamanho `burst` e saem a uma taxa constante `rate`. O parâmetro `nodelay` altera esse comportamento para atender a rajada de imediato, mantendo a taxa média. E `proxy_cache_use_stale` é, na prática, uma forma simples de degradação graciosa: o site continua servindo conteúdo (vencido, mas útil) mesmo com a aplicação inteira fora do ar.

> **Confusão comum:** "editei o `nginx.conf` e salvei, então a mudança está valendo". ✅ **Mais preciso:** o Nginx lê a configuração **na inicialização e no reload**, nunca a cada requisição. O ciclo correto é sempre o mesmo: `nginx -t` valida a sintaxe (e deve ser executado **antes** de qualquer reload, porque uma configuração inválida impede o Nginx de subir), e `nginx -s reload` aplica. O reload é gracioso — o processo master lê a nova configuração, inicia workers novos com ela e deixa os workers antigos **terminarem as conexões em andamento** antes de encerrar. Não há queda de conexão nem janela de indisponibilidade; a confusão que leva equipes a agendar "janela de manutenção" para trocar configuração de Nginx é infundada.

---

## 8. Variantes e onde o Nginx aparece hoje

| Nome | O que é |
|---|---|
| **Nginx (open source)** | O servidor descrito neste arquivo |
| **NGINX Plus** | Versão comercial: health check ativo, *session persistence* avançada, API de configuração dinâmica, métricas detalhadas |
| **OpenResty** | Nginx + LuaJIT embutido — permite escrever lógica em Lua dentro do ciclo da requisição. É a base sobre a qual o **Kong** (API Gateway) foi construído |
| **NGINX Unit** | Servidor de aplicação (esse sim executa código) da mesma família, projeto distinto |
| **Ingress NGINX** | Controlador de Ingress do Kubernetes que traduz recursos do cluster em configuração de Nginx |

O último item explica por que tanta gente usa Nginx sem nunca ter escrito um `nginx.conf`: em um cluster Kubernetes com Ingress NGINX, você declara regras de roteamento em YAML e um controlador as converte em configuração de Nginx e aplica o reload automaticamente. O Nginx continua sendo a peça que executa o trabalho — apenas a interface de configuração mudou.

---

## Resumo do arquivo

- Nginx é um **produto**, não um conceito — e acumula quatro papéis: servidor web, proxy reverso, load balancer e terminador de TLS/cache/rate limiter.
- Ele executa como **um master supervisor + N workers**, cada worker um processo com event loop próprio. Usa todos os núcleos; não cria thread por conexão.
- A configuração é uma **árvore de contextos com herança** (`main` → `events`/`http` → `server` → `location`), e blocos `location` são resolvidos por **precedência**, não pela ordem do arquivo.
- Como proxy reverso, três detalhes decidem se funciona: a **barra final do `proxy_pass`**, os **headers `X-Forwarded-*`** e os **timeouts**.
- Como load balancer, o open source oferece round robin (padrão), `least_conn`, `ip_hash`, `hash ... consistent` e `random two`, com **health check apenas passivo** — checagem ativa é recurso do NGINX Plus.
- Ele **não executa código de aplicação**: sempre há um servidor de aplicação atrás dele, e `502 Bad Gateway` significa "o backend não respondeu", não "o Nginx quebrou".
- `nginx -t` antes de `nginx -s reload`; o reload é **gracioso**, sem derrubar conexões.

**Próxima leitura:** `api_gateway.md` — o terceiro componente da borda, e o que mais se confunde com o que você acabou de ler.
