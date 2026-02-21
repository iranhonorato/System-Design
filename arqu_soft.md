# Arquitetura de Software

**Definição:** Arquitetura de software é a forma como um sistema é organizado:
a estrutura do sistema, os seus componentes, como eles se comunicam e as decisões técnicas fundamentais que determinam como ele funciona, evolui e escala.

Em outras palavras, a arquitetura resolve trade-offs entre:

- Performance: O sistema precisa ser rápido?
- Escalabilidade: Aguenta muitos usuários juntos?
- Segurança: É seguro contra invasões?
- Disponibilidade: Fica no ar sem cair?
- Manutenibilidade: É fácil de modificar?
- Custo: É barato de rodar?

## 1. Arquiteturas clássicas

**Definição:** Essas arquiteturas são fundamentais porque serviram de base para todos os modelos modernos (microserviços, serverless, eventos etc.). Entender isso é como conhecer a história do software — te dá clareza sobre por que as coisas são como são hoje.

### 1.1 Arquitetura Monolítica:

**Definição:** É um sistema único, indivisível, onde:
- Todas as funcionalidades
- Regras de negócio
- Acesso de dados
- Interface
- Integrações

ficam no mesmo projeto, rodando como um único processo.

**Características:**
- Um único deploy
- Um único código-base
- Tudo empacotado como um só serviço

**Vantagens:**
- Simples de entender
- Fácil de desenvolver no início
- Sem complexidade de comunicação entre serviços
- Deploy único (rápido testar/lançar)

**Desvantagens:**
- Fica lento/difícil de escalar quando cresce
- Uma falha derruba tudo
- Mudanças pequenas exigem novo deploy do sistema inteiro
- Cresce acoplado → difícil dar manutenção
- Equipes grandes tropeçam umas nas outras


### 1.2 Arquitetura em camadas (Arquitetura de Três Camadas, 3-Tier, N-Tier)

**Definição:** Divide a aplicação em três grandes blocos, cada um com seu propósito:

- **UI (User Interface):** Onde o usuário interage
- **Lógica de negócio (Backend):** Validações, cálculos, regras, integrações
- **Data Layer (Banco/Armazenamento):**: PostgreSQL, SQL Server, Oracle

**Vantagens:**
- Separação clara de responsabilidades
- Fácil evoluir partes separadamente
- UI pode mudar sem mexer no backend
- Backend pode escalar separado
- Acesso ao banco é centralizado → *pooling* funciona bem

**Desvantagens:**
- Mais complexa que um monolito
- Requer estrutura arquitetural maior
- Depende de APIs/contratos bem definidos


**OBSERVAÇÃO: Entendendo o Pool de Conexões (pooling)**

Imagine uma situação em que sua aplicação precisa realizar várias operações de banco de dados simultaneamente. Criar e fechar uma nova conexão a cada operação pode ser ineficiente, especialmente quando há um número significativo de **requisições concorrentes**.

-------------------------------------------------------------------------
**Concorrência:** Refere-se à capacidade de um sistema de lidar com múltiplas tarefas ao mesmo tempo. Não significa necessariamente que essas tarefas estão sendo executadas simultaneamente, mas sim que o sistema pode trocar rapidamente entre as tarefas, intercalando a execução, de modo que parece que estão ocorrendo ao mesmo tempo.

**Paralelismo:** Por outro lado, paralelismo refere-se à execução simultânea de várias tarefas. Isso requer múltiplos núcleos de CPU, onde cada núcleo pode executar uma tarefa diferente ao mesmo tempo. O paralelismo é sobre fazer várias coisas ao mesmo tempo.

--------------------------------------------------------------------------
É aí que entra o pool de conexões.

O pool de conexões é uma estratégia em que um conjunto pré-definido de conexões com o banco de dados é mantido aberto, mantidas pelo **driver** (biblioteca de comunicação com o banco de dados), e reutilizado conforme necessário, basicamente um **cache** de conexões de banco de dados abertas. Resumindo, em vez de criar uma nova conexão toda vez que uma operação é executada, a aplicação pega uma conexão disponível no pool, utiliza-a e a devolve quando não é mais necessária.

----------------------------------------------------------------------------
**Cache:** No geral, cache é um local de armazenamento temporário que guarda resultados prontos ou recursos prontos para evitar refazer trabalhos caros. No contexto de pooling, o cache é um estoque de conexões já abertas e prontas para uso. Em vez de abrir uma conexão (caro) a cada requisição, você pega uma do cache, usa e devolve.

----------------------------------------------------------------------------


### 1.3 Arquitetura Cliente-Servidor

**Definição:** Essa é a mãe de todas as arquiteturas modernas.
É mais antiga e mais simples que a 3‑tier, mas extremamente importante. Trata-se de um modelo onde o *cliente* faz uma requisição e o *servidor* atende a requisição.

**Exemplos clássicos:**
- Navegador (cliente) → Site (Servidor)
- Aplicativo → API
- App desktop → banco diretamente

**Vantagens:**
- Baixa complexidade
- Fácil de entender
- Fácil de implementar

**Desvantagens:**
- Cliente e servidor podem ficar fortemente acoplados
- O servidor vira gargalo único
- Dificulta escalabilidade

## 2. Arquiteturas modernas
**Definição:** A evolução das aplicações nos últimos anos trouxe novas formas de projetar sistemas escaláveis, resilientes e fáceis de evoluir. As quatro principais abordagens modernas são:

- Microsserviços
- Arquitetura orientada a eventos
- Serverless
- Arquitetura orientada a APIs

Cada uma resolve problemas específicos e pode ser combinada com as outras.

### 2.1 Microservices (Microsserviços)
**Definição:** Microsserviços são uma forma de arquitetura onde a aplicação é dividida em módulos independentes, cada um responsável por uma função de negócio específica. Cada serviço:

- Tem seu próprio código
- Pode ter seu próprio banco de dados
- Escala de forma independente
- Pode ser implementado em linguagens diferentes

**Exemplo:** Um e-commerce pode ter serviços como

- Serviço de pagamentos
- Serviço de catálogo
- Serviço de estoque
- Serviço de notificações
- Serviço de usuários


**Vantagens**

- Escalabilidade individual: só escala o que precisa.
- Desenvolvimento independente: times podem trabalhar sem - bloquear outros.
- Resiliência: se um serviço cai, o sistema inteiro não precisa cair.
- Tecnologias diversas: cada serviço pode usar a melhor tecnologia.

**Desvantagens**

- Aumenta a complexidade operacional.
- Comunicação entre serviços precisa ser muito bem planejada.
- Pode gerar muitos serviços pequenos difíceis de manter.

**Quando usar:**

- Projetos grandes
- Times distribuídos
- Necessidade de escalabilidade pesada
- Empresas que querem evolução contínua sem parar o sistema