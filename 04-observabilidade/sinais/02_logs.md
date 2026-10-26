# Logs

**Pré-requisito:** [01_metricas.md](01_metricas.md)

> Segue a documentação do Grafana Loki (*LogQL*, *OpenTelemetry ingestion*), do OpenTelemetry (*Logs data model*, instrumentação do módulo `logging` do Python) e o fator XI (*Logs*) do *The Twelve-Factor App*. O conceito de **identificador de correlação** aparece em **Fundamentals of Software Architecture** (Mark Richards & Neal Ford), que no Capítulo 9 também aponta o logging distribuído como uma das dificuldades centrais das arquiteturas distribuídas. As regras sobre dados pessoais seguem a LGPD (Lei 13.709/2018).

---

## Por que este arquivo existe

O painel de métricas do capítulo anterior mostrou: às 18h04, o número de checkouts com falha saltou de 2 para 37 por minuto. A métrica cumpriu seu papel, que é avisar **que** algo está errado e **quanto**. Mas a equipe precisa saber agora **o que** aconteceu em cada falha: qual foi a mensagem de erro? Em qual serviço? Qual pedido? A operadora recusou ou não respondeu? Qual foi o motivo da recusa?

Essas são perguntas sobre **eventos individuais, com detalhes**, e esse é o papel dos **logs**. Este capítulo trata de como escrevê-los para que sejam úteis (estruturados, com o nível certo e conectados aos traces), para onde vão quando a aplicação roda em containers, como consultá-los, e do cuidado que eles exigem: logs são o sinal mais caro e o mais perigoso para a privacidade.

---

## 1. O que é um log

Um **log** é um **registro de um evento discreto**, com horário: algo aconteceu, em um momento, em algum lugar do sistema. Diferente da métrica, ele não resume nada, e por isso guarda o detalhe que a métrica descarta.

### 1.1 Texto livre e logs estruturados

O jeito mais comum de começar é com texto livre:

```
2026-10-26 18:04:12 ERROR pagamento recusado para o pedido 4512 (cartao expirado), cliente 881, valor 99.80
```

Para uma pessoa lendo, está ótimo. Para uma **máquina**, é difícil: para encontrar "todos os pagamentos recusados por cartão expirado com valor acima de 50", seria preciso escrever uma expressão regular que entendesse essa frase específica, e ela quebraria no dia em que alguém mudasse o texto da mensagem.

Um **log estruturado** registra o mesmo evento como **campos com nomes**, normalmente em JSON:

```json
{
  "timestamp": "2026-10-26T18:04:12.381Z",
  "nivel": "ERROR",
  "servico": "pagamentos",
  "mensagem": "pagamento recusado",
  "pedido_id": 4512,
  "cliente_id": 881,
  "valor": 99.80,
  "motivo": "cartao_expirado",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7"
}
```

Agora a pergunta vira um filtro sobre campos (`motivo = "cartao_expirado" and valor > 50`), que funciona independentemente do texto da mensagem. Os campos também permitem **agrupar**: quantas recusas por motivo, por bandeira, por serviço.

| | Texto livre | Estruturado (JSON) |
|---|---|---|
| Leitura por pessoas | Fácil | Boa (as ferramentas formatam) |
| Busca por um campo | Expressão regular frágil | Filtro direto |
| Agrupar e contar | Difícil | Natural |
| Mudar o texto da mensagem | Quebra as buscas | Não afeta os campos |

A regra prática: **a mensagem descreve o evento em poucas palavras fixas** (`"pagamento recusado"`), e **tudo o que varia vai em campos** (`pedido_id`, `motivo`, `valor`). Uma mensagem como `f"pagamento {id} recusado por {motivo}"` cria uma mensagem diferente para cada pedido, e torna impossível agrupar "todas as recusas".

### 1.2 Logs estruturados em Python

Com o módulo `logging` e a instrumentação do OpenTelemetry ([capítulo de telemetria](../fundamentos/02_telemetria_e_opentelemetry.md)), os campos extras viram **atributos** do log, e o `trace_id` e o `span_id` são acrescentados automaticamente:

```python
import logging

log = logging.getLogger("pagamentos")

log.error(
    "pagamento recusado",
    extra={"pedido_id": pedido.id, "motivo": resultado.motivo, "valor": pagamento.valor},
)
```

Quando os logs são lidos da saída padrão (seção 4), em vez de enviados por OTLP, o formato JSON é produzido por uma biblioteca como `structlog` ou `python-json-logger`. O resultado é o mesmo: um objeto JSON por linha.

---

## 2. Níveis de log

O **nível** (ou severidade) indica a importância do evento, e permite filtrar o volume: em produção, normalmente só se registra de `INFO` para cima.

| Nível | Quando usar | Exemplo na loja |
|---|---|---|
| `DEBUG` | Detalhes para investigar um problema durante o desenvolvimento. Desligado em produção | "carrinho com 3 itens, aplicando o cupom X" |
| `INFO` | Eventos normais e relevantes do funcionamento | "pedido 4512 criado", "serviço iniciado na versão 1.4.2" |
| `WARNING` | Algo inesperado, que **não** impediu a operação, mas merece atenção | "operadora respondeu em 1,8 s (limite: 2 s)", "nova tentativa de conexão com o Redis" |
| `ERROR` | A operação **falhou** | "pagamento do pedido 4512 não pôde ser processado" |
| `CRITICAL` | O serviço inteiro não consegue funcionar | "não foi possível conectar ao banco na inicialização" |

> **Confusão comum:** "na dúvida, uso `ERROR`: é melhor sobrar do que faltar". ✅ **Mais preciso:** quando tudo é `ERROR`, nada é. Um cliente que digitou o CEP errado (um `422` esperado, tratado pela aplicação) não é um erro do sistema, e registrá-lo como `ERROR` enche os painéis de falsos problemas: depois de algumas semanas, a equipe se acostuma a ver centenas de "erros" por minuto e deixa de perceber quando os erros de verdade começam. A pergunta que decide o nível é: **alguém precisa agir?** Se o sistema se comportou como deveria diante de uma entrada inválida, é `INFO` (ou nem precisa de log). Se algo falhou e alguém da equipe deveria saber, é `ERROR`. Também vale o inverso: um `except` que captura uma exceção e a registra como `DEBUG` esconde um defeito real em produção, onde o `DEBUG` está desligado.

Dois cuidados de volume:

- **Nada de log dentro de laços** que processam milhares de itens: um `log.info` por item vira milhões de linhas. Registre o início, o fim e um resumo (`"lote processado: 48.211 itens, 3 falhas"`).
- **O nível deve poder mudar sem deploy**, por uma variável de ambiente (`LOG_LEVEL=DEBUG`) ou por uma flag, para investigar um problema e voltar ao normal depois.

---

## 3. Contexto: ligando os logs de uma mesma requisição

### 3.1 O problema

Um checkout passa pela `loja`, por `pedidos` e por `pagamentos`. Cada serviço registra seus próprios logs, com várias réplicas, atendendo centenas de requisições ao mesmo tempo. Quando uma falha acontece, é preciso juntar **os logs daquela requisição**, nos três serviços, e separá-los dos logs das outras centenas de requisições que aconteciam no mesmo segundo.

```
 18:04:12.301 loja        INFO   checkout iniciado
 18:04:12.302 loja        INFO   checkout iniciado               ← qual é qual?
 18:04:12.305 pedidos     INFO   pedido criado
 18:04:12.306 pedidos     INFO   pedido criado
 18:04:12.371 pagamentos  ERROR  pagamento recusado              ← de qual checkout?
 18:04:12.372 pagamentos  INFO   pagamento aprovado
```

Ordenar pelo horário não resolve: as requisições se intercalam, e os relógios de máquinas diferentes nunca estão perfeitamente sincronizados.

### 3.2 A solução: um identificador comum

A solução clássica é um **identificador de correlação** (*correlation ID*): um valor único gerado quando a requisição entra no sistema, repassado a cada serviço chamado e incluído em **todo** log. Richards e Ford descrevem o mesmo mecanismo para mensagens assíncronas: a resposta carrega o identificador da mensagem original, para que se saiba a qual pedido ela pertence.

Com o OpenTelemetry, esse identificador já existe: é o **`trace_id`** do trace da requisição ([capítulo de traces](03_traces.md)). A instrumentação o propaga entre os serviços automaticamente, e a instrumentação do módulo `logging` o anexa a cada log escrito durante a requisição. Os mesmos logs, agora com o contexto:

```
 18:04:12.301 loja        INFO   checkout iniciado     trace_id=4bf92f35...  span_id=d027df1c...
 18:04:12.302 loja        INFO   checkout iniciado     trace_id=a3ce929d...  span_id=5f1e0c22...
 18:04:12.305 pedidos     INFO   pedido criado         trace_id=4bf92f35...  span_id=50654611...
 18:04:12.306 pedidos     INFO   pedido criado         trace_id=a3ce929d...  span_id=8b2d40aa...
 18:04:12.371 pagamentos  ERROR  pagamento recusado    trace_id=4bf92f35...  span_id=8e8a32b0...
 18:04:12.372 pagamentos  INFO   pagamento aprovado    trace_id=a3ce929d...  span_id=c41a9e07...
```

Filtrar por `trace_id=4bf92f35...` mostra a história completa do checkout que falhou, nos três serviços, e nada mais. E o mesmo `trace_id` abre o trace dessa requisição, com a cascata de tempos. É a ligação **log ↔ trace** do fluxo de investigação do [capítulo de fundamentos](../fundamentos/01_monitoramento_e_observabilidade.md).

Além do `trace_id`, vale incluir nos logs os identificadores **do domínio** (`pedido_id`, `cliente_id`), porque são eles que chegam pelo atendimento: o cliente reclama do "pedido 4512", e não de um `trace_id`.

---

## 4. Para onde vão os logs de um container

### 4.1 Escreva na saída padrão

O fator XI do *Twelve-Factor App* diz que a aplicação **não deve se preocupar com onde os logs são guardados**: ela escreve seus eventos na **saída padrão** (stdout e stderr), um por linha, e o ambiente de execução cuida do resto. Em containers, essa regra deixa de ser uma recomendação e vira uma necessidade:

- Um arquivo de log gravado **dentro** do container fica na camada de escrita, que some com o container ([capítulo de imagens](../../02-containers/docker/02_imagens_e_dockerfile.md)), e enche o disco dela.
- O Docker e o Kubernetes já capturam a saída padrão de cada container. É de lá que vêm o `docker logs` e o `kubectl logs`.

O caminho de um log em cada ambiente:

```
 DOCKER
   app ──stdout──▶ dockerd ──▶ /var/lib/docker/containers/<id>/<id>-json.log ──▶ docker logs

 KUBERNETES
   app ──stdout──▶ container runtime ──▶ /var/log/pods/<namespace>_<pod>_<uid>/<container>/0.log
                                                     │                          │
                                              kubectl logs              agente de coleta
                                                                      (DaemonSet, um por node)
                                                                                │
                                                                                ▼
                                                                    Loki / Elasticsearch / ...
```

No Docker, o driver padrão (`json-file`) **não limita** o tamanho do arquivo: um container que registra muito pode encher o disco do servidor. Em servidores como a EC2 do [capítulo de EC2](../../01-infraestrutura/aws/01_ec2_servidores_na_nuvem.md), configure a rotação, no Compose ou no daemon:

```yaml
services:
  api:
    logging:
      driver: json-file
      options:
        max-size: "10m"    # cada arquivo até 10 MB
        max-file: "5"      # guarda no máximo 5 arquivos
```

### 4.2 Por que `kubectl logs` não basta

> **Confusão comum:** "não preciso de uma ferramenta de logs: quando algo dá errado, uso `kubectl logs`". ✅ **Mais preciso:** o `kubectl logs` lê o arquivo do container **no node em que ele está rodando**, e esse arquivo tem a vida do container. Quando o Pod é apagado (em um deploy, em uma escala para baixo, quando o node é substituído), os logs vão junto; e de um container reiniciado só se consegue ver a execução anterior (`--previous`), nunca as mais antigas. Na prática, as perguntas de um incidente quase sempre são sobre o **passado** ("o que aconteceu às 18h04?") e sobre **muitos Pods ao mesmo tempo** ("todos os erros de pagamento, em todas as réplicas"), e é justamente isso que o `kubectl logs` não responde. Um agente de coleta em cada node (Fluent Bit, Grafana Alloy ou o OpenTelemetry Collector com o receiver `filelog`) lê esses arquivos continuamente, acrescenta o namespace, o Pod e o container a cada linha, e envia para um armazenamento central, onde os logs sobrevivem aos Pods. O [capítulo de Kubernetes](../pratica/03_observabilidade_no_kubernetes.md) monta esse agente.

### 4.3 Saída padrão ou OTLP?

Há dois caminhos para levar os logs ao armazenamento:

| Caminho | Como | Vantagem | Cuidado |
|---|---|---|---|
| **Saída padrão + agente** | A aplicação escreve JSON no stdout; o agente do node lê e envia | Funciona com qualquer aplicação, inclusive as que não têm OpenTelemetry; os logs aparecem também no `kubectl logs` | O `trace_id` precisa ser escrito no JSON pela aplicação |
| **OTLP direto** | O SDK do OpenTelemetry envia os logs ao Collector, junto com traces e métricas | Os logs já chegam com `trace_id`, `span_id` e os atributos do recurso (`service.name`) | Se o Collector estiver fora do ar, os logs dependem da fila do SDK |

Os dois podem conviver, e a [parte prática](../pratica/01_stack_de_observabilidade.md) usa o OTLP.

---

## 5. Armazenando e consultando

### 5.1 Duas filosofias de armazenamento

| | Elasticsearch / OpenSearch (+ Kibana) | Grafana Loki |
|---|---|---|
| **Como indexa** | **Todo o conteúdo** de cada log (índice invertido, como um buscador) | Só os **rótulos** de cada fluxo de logs (serviço, namespace, nível); o conteúdo é comprimido e varrido na hora da consulta |
| **Busca de texto livre em grandes volumes** | Muito rápida | Mais lenta (varre os dados), mas eficiente quando os rótulos já restringem bem |
| **Custo de armazenamento e operação** | Alto: o índice ocupa muito espaço e exige um cluster bem dimensionado | Baixo: guarda os dados comprimidos, normalmente em object storage (S3) |
| **Linguagem** | Lucene / KQL | LogQL (inspirada na PromQL) |

Os provedores de nuvem oferecem serviços próprios (na AWS, o CloudWatch Logs). Esta pasta usa o Loki, que se integra ao Grafana junto com o Prometheus e o Tempo.

### 5.2 LogQL

Uma consulta LogQL começa **selecionando fluxos por rótulo** (como a PromQL seleciona séries) e depois **filtra o conteúdo**:

```logql
# Todos os logs do serviço de pagamentos
{service_name="pagamentos"}

# ...que contêm um texto
{service_name="pagamentos"} |= "recusado"

# ...só os de nível ERROR (campo do log recebido por OTLP)
{service_name="pagamentos"} | severity_text="ERROR"

# Para logs em JSON vindos do stdout: extrair os campos e filtrar por eles
{namespace="loja", container="pagamentos"} | json | motivo="cartao_expirado" and valor > 50

# Toda a história de uma requisição, em todos os serviços
{service_name=~".+"} | trace_id="4bf92f3577b34da6a3ce929d0e0e4736"
```

A LogQL também transforma logs em números, com funções parecidas com as da PromQL:

```logql
# Erros por minuto, por serviço, CALCULADOS A PARTIR DOS LOGS
sum by (service_name) (count_over_time({service_name=~".+"} | severity_text="ERROR" [1m]))
```

Isso é útil para investigar ("quantas recusas por cartão expirado houve hoje?", uma pergunta que ninguém previu ao criar as métricas). Mas não deve ser a base de painéis permanentes e alertas, pelo motivo da confusão a seguir.

> **Confusão comum:** "já temos todos os erros nos logs, então os alertas podem contar os logs de erro". ✅ **Mais preciso:** funciona, mas é a forma mais cara e frágil de obter um número que uma métrica entrega quase de graça. Para contar, o sistema de logs precisa **ler** cada linha de cada minuto, o tempo todo; a métrica só lê um contador. A contagem por logs também depende de detalhes que mudam sem aviso: se alguém altera o texto ou o nível de uma mensagem, o alerta deixa de disparar, em silêncio; se os logs forem amostrados para economizar, a contagem fica errada. A divisão saudável é: **métricas para contar e alertar** (a pergunta "quantas falhas por minuto?" do [capítulo de métricas](01_metricas.md)), **logs para explicar** cada falha. A contagem por logs fica reservada para as perguntas novas, até que, se forem recorrentes, virem uma métrica.

### 5.3 Rótulos no Loki: a mesma regra das métricas

No Loki, cada combinação de rótulos é um **fluxo** (*stream*) separado, e muitos fluxos pequenos degradam o sistema exatamente como a cardinalidade alta degrada o Prometheus ([capítulo de métricas](01_metricas.md), seção 7). Rótulos devem ter **poucos valores**: serviço, namespace, ambiente, nível. Identificadores (`pedido_id`, `cliente_id`, `trace_id`) ficam **no conteúdo** do log ou como *metadados estruturados* (que o Loki guarda junto da linha, sem criar fluxos novos), e são filtrados na consulta, como nos exemplos acima.

---

## 6. O que não registrar

Logs são copiados para vários lugares (o agente, o armazenamento, backups, a ferramenta de quem investiga), guardados por semanas e acessados por muita gente. Tudo o que entra em um log deve ser tratado como **potencialmente público dentro da empresa**.

### 6.1 Segredos e dados sensíveis

**Nunca registre:**

- **senhas, tokens, chaves de API, cabeçalhos `Authorization` e cookies de sessão.** O equivalente, nos logs, da access key no Git do [capítulo de IAM](../../01-infraestrutura/aws/03_iam_identidades_e_permissoes.md);
- **dados de cartão** (número, código de segurança). Além do risco, as regras do setor de cartões (PCI DSS) proíbem guardá-los;
- **dados pessoais sem necessidade**: CPF, e-mail, endereço, telefone. A LGPD exige finalidade e necessidade para tratar dados pessoais, e um log de depuração raramente justifica guardá-los;
- **corpos inteiros de requisições e respostas**, que costumam conter todos os itens acima.

```python
# ✘ ruim: registra o cartão e o corpo inteiro da requisição
log.info(f"pagamento recebido: {request_body}")

# ✔ bom: registra o necessário para investigar, com identificadores internos
log.info(
    "pagamento recebido",
    extra={"pedido_id": pedido.id, "bandeira": pagamento.bandeira, "final_cartao": pagamento.numero[-4:]},
)
```

Como defesa adicional, o agente de coleta ou o Collector pode **mascarar** padrões conhecidos (números de cartão, tokens) antes de enviar. Mas a defesa principal é não registrar.

### 6.2 Volume e custo

O custo de logs cresce com o tráfego, e é comum que ele ultrapasse o da própria aplicação. Algumas práticas:

- **`INFO` em produção**, com `DEBUG` ligado só durante investigações.
- **Não registrar o que já é medido**: um log `INFO` para cada requisição bem-sucedida repete, de forma cara, o que a métrica de requisições e o trace já mostram. Muitas equipes registram só os eventos relevantes e os erros.
- **Amostrar eventos repetitivos**: se o mesmo aviso aparece mil vezes por segundo, registrar um a cada cem, com a contagem.
- **Retenção por camadas**: alguns dias em armazenamento rápido para investigação, e um período maior em armazenamento barato (como o S3) quando há exigência legal.
- **Logs de auditoria** (quem acessou o quê, quem alterou a configuração) são uma categoria à parte: têm requisitos próprios de retenção e integridade e não devem ser descartados pela política de economia dos logs de aplicação.

---

## 7. Onde os logs param

Os logs explicam **o que aconteceu em cada serviço**, com todos os detalhes. Mas cada log é o ponto de vista de **um** serviço, em **um** instante. Com o `trace_id`, é possível juntar os logs de uma requisição, mas eles continuam sendo uma lista de eventos, e não respondem bem a uma pergunta que aparece em todo sistema com vários serviços: **dos 1,6 segundo que o checkout levou, quanto foi gasto em cada etapa, e em qual chamada está a demora?**

Calcular isso a partir dos horários dos logs é frágil: os relógios das máquinas não coincidem, nem toda etapa registra seu início e seu fim, e chamadas em paralelo se misturam. Essa pergunta tem um sinal próprio.

---

## Próximo passo

O [03_traces.md](03_traces.md) apresenta o rastreamento distribuído: traces e spans, como ler uma cascata de tempos para encontrar a chamada mais lenta, como o contexto atravessa os serviços, como acrescentar spans do domínio e como amostrar sem perder os traces que importam. E responde à segunda pergunta clássica: em qual dos três microsserviços está a demora.
