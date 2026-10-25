# Do Bare Metal ao Container

**Pré-requisito:** [01_o_que_e_infraestrutura.md](01_o_que_e_infraestrutura.md)

> O Prefácio de **Fundamentals of Software Architecture** (Mark Richards & Neal Ford) usa a ascensão da conteinerização como exemplo de como o ecossistema de software muda: ferramentas como o Kubernetes não existiam uma década antes da publicação do livro e hoje têm conferências inteiras dedicadas a elas. Este capítulo conta essa mudança.

---

## Por que este arquivo existe

É possível aprender Docker decorando comandos sem entender por que ele existe. O problema é que, sem o "porquê", fica difícil tomar decisões: quando usar container e quando usar VM? Por que um container sobe em milissegundos e uma VM leva minutos? Por que um container não isola tanto quanto uma VM?

Todas essas respostas aparecem quando se olha para a história. A forma de rodar aplicações em servidores passou por **três eras**, e cada uma surgiu para resolver o problema principal da anterior, criando um problema novo no lugar. Este capítulo percorre as três e termina com a ideia que liga tudo ao Docker: **Infraestrutura como Código**.

---

## 1. Era 1: Bare Metal (1980 até meados de 1990)

"Bare metal" (metal nu) significa rodar o sistema operacional **diretamente no hardware físico**, sem nenhuma camada intermediária.

```
┌─────────────────────────────────────┐
│         Aplicação                   │
├─────────────────────────────────────┤
│         Sistema Operacional         │
├─────────────────────────────────────┤
│         Hardware Físico             │
└─────────────────────────────────────┘
```

**Como funcionava:** a empresa comprava um servidor Dell ou HP, instalava Windows Server ou Red Hat e colocava suas aplicações dentro dele.

**Um exemplo do problema.** Imagine uma empresa com duas aplicações internas:

- o sistema de RH, escrito para o **Java 8**;
- o sistema financeiro, que exige o **Java 11**.

Instalar as duas versões no mesmo sistema operacional é possível, mas frágil: variáveis de ambiente, bibliotecas do sistema e scripts de inicialização começam a se misturar, e uma atualização feita para o financeiro quebra o RH. A saída "segura" era **um servidor físico por aplicação**. Dessa escolha nasceram os quatro problemas da era:

1. **Conflito de dependências (*dependency hell*).** Aplicações com requisitos incompatíveis não coexistem bem no mesmo sistema.
2. **Desperdício de recursos.** Com uma aplicação por servidor, máquinas de 32 núcleos operavam com 5% a 10% de uso. O resto ficava ocioso, mas pago.
3. **Escalabilidade lenta e cara.** Se o site crescia, era preciso comprar outro servidor, esperar semanas pela entrega, montar no rack e configurar.
4. **Ambientes inconsistentes.** Configurar dois servidores "iguais" à mão sempre gerava diferenças sutis: uma versão de biblioteca aqui, um parâmetro de kernel ali. O clássico *"funciona na minha máquina"* nasce aqui.

O problema número 2 era o mais caro, e foi o primeiro a ser atacado.

---

## 2. Era 2: Máquinas Virtuais (final dos anos 1990 até meados de 2010)

A VMware popularizou a **virtualização**: uma camada de software, o **hypervisor**, que simula hardware. Com ele, um único servidor físico passa a rodar vários sistemas operacionais completos ao mesmo tempo, cada um achando que tem uma máquina só para si.

```
┌──────────┬──────────┬──────────┐
│  App A   │  App B   │  App C   │
├──────────┼──────────┼──────────┤
│  SO      │  SO      │  SO      │  ← cada VM tem seu próprio SO completo
│ (Linux)  │(Windows) │ (Linux)  │
├──────────┴──────────┴──────────┤
│         Hypervisor              │  ← simula o hardware para cada VM
├─────────────────────────────────┤
│         Hardware Físico         │
└─────────────────────────────────┘
```

Voltando ao exemplo: o RH ganha uma VM com Java 8, o financeiro ganha outra com Java 11, e as duas rodam no **mesmo** servidor físico, sem interferir uma na outra.

**O que melhorou:**

- **Consolidação:** um servidor físico rodando 10 VMs no lugar de 10 servidores físicos. O uso do hardware sobe e o custo cai.
- **Isolamento forte:** cada VM tem seu próprio kernel, e um travamento em uma não afeta as outras.
- **Portabilidade:** uma VM é, no fim, um conjunto de arquivos, e pode ser copiada e movida entre servidores.

**Dois tipos de hypervisor:**

| Tipo | Onde roda | Exemplos | Uso típico |
|---|---|---|---|
| **Tipo 1** (*bare-metal*) | Direto no hardware, sem SO por baixo | VMware ESXi, Microsoft Hyper-V, KVM | Data centers e nuvem. Mais eficiente. |
| **Tipo 2** (*hosted*) | Como um programa comum, sobre um SO | VirtualBox, VMware Workstation | Desktops de desenvolvedores. Mais fácil de usar. |

**O novo problema: cada VM carrega um sistema operacional inteiro.**

1. **Overhead do SO convidado.** Para rodar uma API de 50 MB, você precisa de um Ubuntu de 2 GB por baixo, com kernel, drivers e serviços de sistema. Em 10 VMs, são 10 sistemas operacionais consumindo memória e CPU só para existir.
2. **Boot lento.** Iniciar uma VM é ligar um computador: o kernel carrega, os serviços sobem, e isso pode levar de 1 a 5 minutos.
3. **Portabilidade limitada na prática.** Uma imagem de VM pesa vários gigabytes, e transferi-la entre ambientes é lento.

Repare que o problema de "funciona na minha máquina" ainda não foi resolvido: o desenvolvedor raramente roda a mesma VM de produção no próprio notebook, justamente por ela ser pesada.

---

## 3. Era 3: Containers (2013 até hoje)

O Docker, lançado em 2013 pela dotCloud, partiu de uma percepção simples: **para isolar aplicações, não é preciso virtualizar o hardware. Basta isolar os processos.**

```
┌──────────┬──────────┬──────────┐
│  App A   │  App B   │  App C   │
├──────────┼──────────┼──────────┤
│ Libs A   │ Libs B   │ Libs C   │  ← cada container tem suas próprias bibliotecas
├──────────┴──────────┴──────────┤
│       Docker Engine             │  ← configura o isolamento
├─────────────────────────────────┤
│    Sistema Operacional Host     │  ← UM único kernel, compartilhado por todos
├─────────────────────────────────┤
│         Hardware Físico         │
└─────────────────────────────────┘
```

Compare com o diagrama da VM: a linha "SO" de cada aplicação **desapareceu**. Os containers compartilham o **kernel** do sistema operacional da máquina (o *host*), e cada um tem apenas o seu próprio *espaço de usuário*: arquivos, bibliotecas, processos.

**A analogia do prédio.** Máquinas virtuais são casas separadas: cada uma tem fundação, encanamento e telhado próprios. Containers são apartamentos em um prédio: compartilham a estrutura (o kernel), mas cada um tem seus próprios cômodos e sua própria chave (bibliotecas, arquivos, processos). Construir um apartamento novo num prédio que já existe é muito mais rápido e barato do que construir uma casa.

No exemplo de sempre: o RH vira um container com Java 8, o financeiro um container com Java 11. Cada um carrega só o seu runtime, sem um sistema operacional completo por baixo.

### Comparação direta

| Característica | Máquina virtual | Container |
|---|---|---|
| **Tamanho típico** | 1–10 GB | 10–500 MB |
| **Tempo para iniciar** | 1–5 minutos | Milissegundos a poucos segundos |
| **O que é isolado** | Hardware virtual inteiro (kernel próprio) | Processos (kernel compartilhado) |
| **Overhead** | Alto (um SO por VM) | Mínimo |
| **Portabilidade** | Limitada (arquivos grandes) | Alta (imagens leves) |

### O que torna isso possível

O isolamento de containers usa dois recursos que já existiam no kernel Linux. O [../../02-containers/docker/01_arquitetura_e_isolamento.md](../../02-containers/docker/01_arquitetura_e_isolamento.md) mostra os dois funcionando com comandos reais. Por enquanto, basta a ideia:

- **Namespaces isolam o que o processo enxerga.** Cada container recebe sua própria visão dos processos (dentro dele, a aplicação é o PID 1), da rede (interface e portas próprias), do sistema de arquivos e dos usuários.
- **cgroups (control groups) limitam o quanto o processo consome.** Por exemplo: "este container pode usar no máximo 512 MB de RAM e meia CPU". Se ele tentar passar disso, o kernel impede.

> **Confusão comum:** "um container é uma VM mais leve". ✅ **Mais preciso:** um container **não é uma máquina**, é um **processo comum do Linux** rodando com restrições de visibilidade (namespaces) e de consumo (cgroups). Não existe um segundo kernel nem hardware simulado. Uma consequência prática: se você rodar `ps aux` no host, verá os processos de todos os containers ali, como quaisquer outros. Outra consequência: um container Linux precisa de um kernel Linux. No Mac e no Windows, o Docker Desktop resolve isso rodando, discretamente, **uma VM Linux** por baixo (no Windows, via WSL 2), e os containers rodam dentro dela.

> **Confusão comum:** "containers isolam tanto quanto máquinas virtuais, só que mais rápido". ✅ **Mais preciso:** o isolamento é **mais fraco**, e esse é exatamente o preço da leveza. Como todos os containers compartilham o mesmo kernel, uma falha de segurança no kernel pode permitir que um processo "escape" do container e alcance o host. Em uma VM, o atacante precisaria atravessar também o hypervisor, uma barreira bem menor e mais auditada. Por isso, ambientes que rodam código de terceiros não confiáveis (provedores de nuvem, plataformas de execução de código) costumam colocar containers **dentro** de VMs leves, como o Firecracker usado pelo AWS Lambda. As duas tecnologias se combinam, não competem.

> **Confusão comum:** "o Docker inventou os containers". ✅ **Mais preciso:** os blocos fundamentais são anteriores ao Docker: o `chroot` é de 1979, os *jails* do FreeBSD de 2000, os namespaces e cgroups entraram no kernel Linux entre 2002 e 2008, e o LXC (Linux Containers) já combinava os dois em 2008. O que o Docker trouxe em 2013 foi **usabilidade**: um formato de imagem portátil, o Dockerfile, um registry público para compartilhar imagens e uma linha de comando simples. Foi isso que transformou um recurso obscuro do kernel em uma ferramenta do dia a dia.

---

## 4. Infraestrutura como Código (IaC)

As três eras mudaram **onde** a aplicação roda. Falta uma mudança igualmente importante: **como** o ambiente é descrito.

### Antes: configuração manual

Na era do bare metal, e por muito tempo depois, configurar um servidor era um trabalho manual:

- o administrador acessava o servidor via SSH;
- instalava as dependências uma a uma;
- editava arquivos de configuração "na mão";
- depois de alguns meses, ninguém sabia exatamente o que estava instalado ali;
- reproduzir aquele ambiente em outra máquina era quase impossível.

Esse servidor configurado à mão, único e insubstituível, ganhou o apelido de **"servidor floco de neve"** (*snowflake server*): não existem dois iguais.

### Depois: o ambiente vira um arquivo de texto

**Infraestrutura como Código** é a prática de **descrever a infraestrutura em arquivos de texto versionados**, que uma ferramenta lê e aplica automaticamente. O primeiro exemplo desta trilha é o próprio Dockerfile:

```dockerfile
# Descreve, passo a passo, o ambiente onde a aplicação roda
FROM python:3.11
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "app.py"]
```

Cada linha substitui um passo que antes seria feito à mão via SSH. E, por ser um arquivo de texto, ele é:

- **versionável no Git:** você sabe exatamente como era o ambiente em cada ponto do tempo;
- **reproduzível:** qualquer pessoa com esse arquivo cria exatamente o mesmo ambiente;
- **auditável:** toda mudança no ambiente aparece no histórico de commits, com autor e motivo;
- **automatizável:** pipelines de CI/CD executam esse arquivo sem intervenção humana.

### As ferramentas e o que cada uma descreve

O Dockerfile descreve **o que vai dentro** de um container. Outras ferramentas de IaC descrevem outras camadas da infraestrutura. A distinção mais útil é entre **provisionar** (criar o recurso) e **configurar** (preparar o que roda dentro dele).

**Terraform: provisiona.** Cria recursos de nuvem como máquinas, redes e bancos. Você declara *o que* quer, e o Terraform calcula o que precisa criar, alterar ou destruir para chegar lá:

```hcl
# main.tf: "quero uma máquina t3.micro rodando esta imagem do Ubuntu"
resource "aws_instance" "api" {
  ami           = "ami-0abcdef1234567890"
  instance_type = "t3.micro"

  tags = {
    Name = "api-producao"
  }
}
```

Rodar `terraform apply` com esse arquivo cria a instância. Rodar de novo sem mudar nada **não cria outra**: o Terraform compara o arquivo com o que já existe e conclui que não há nada a fazer.

**Ansible: configura.** Entra em servidores que já existem (via SSH) e garante que eles estejam no estado descrito:

```yaml
# playbook.yml: "nos servidores web, o nginx precisa estar instalado e rodando"
- hosts: servidores_web
  become: true
  tasks:
    - name: Instalar o nginx
      apt:
        name: nginx
        state: present

    - name: Garantir que o nginx está ativo
      service:
        name: nginx
        state: started
```

Repare que nenhum dos dois exemplos diz "instale" ou "crie" como uma ordem cega. Ambos descrevem um **estado desejado** (`state: present`, `state: started`). Executar o playbook dez vezes produz o mesmo resultado que executar uma vez. Essa propriedade se chama **idempotência**, e é ela que torna IaC seguro de automatizar.

| Ferramenta | Camada | O que descreve |
|---|---|---|
| **Terraform** | Provisionamento | Máquinas, redes (VPC), bancos (RDS), DNS na nuvem |
| **Ansible** | Configuração | Pacotes, serviços e arquivos dentro de servidores existentes |
| **Dockerfile** | Empacotamento | O ambiente dentro de uma imagem de container |
| **Docker Compose** | Orquestração local | Vários containers que formam um sistema ([../../02-containers/docker/04_compose_e_producao.md](../../02-containers/docker/04_compose_e_producao.md)) |
| **Manifests do Kubernetes** | Orquestração em cluster | Como containers devem rodar em dezenas de máquinas |

---

## Próximo passo

Este capítulo explicou **por que** containers existem e de onde vem a ideia de descrever o ambiente em um arquivo. Ficou em aberto **como** o isolamento funciona: o que exatamente o Docker faz quando você digita `docker run`, e como namespaces e cgroups aparecem na prática.

O [../../02-containers/docker/01_arquitetura_e_isolamento.md](../../02-containers/docker/01_arquitetura_e_isolamento.md) abre o Docker e mostra cada peça funcionando com comandos que você pode rodar na sua máquina.
