+++
title = "Arquitetura"
weight = 3
description = "Control node, managed nodes, inventário, módulos e o que acontece em cada execução."
+++

## Os componentes

```text
             ┌───────────────────────────────┐
             │         Control node          │
             │  ansible / ansible-playbook   │
             │                               │
             │  inventário   playbooks       │
             │  ansible.cfg  roles/collections│
             └───────┬───────────┬───────────┘
                SSH  │           │  SSH          WinRM / SSH
          ┌──────────┘           └──────────┐        │
   ┌──────┴──────┐                   ┌──────┴──────┐ │ ┌─────────────┐
   │    web01    │                   │    db01     │ └─┤    win01    │
   │ Python + SSH│                   │ Python + SSH│   │ PowerShell  │
   └─────────────┘                   └─────────────┘   └─────────────┘
              managed nodes (nós gerenciados)
```

| Componente | O que é |
|---|---|
| **Control node** | A máquina onde o Ansible está instalado e de onde os comandos são executados. Precisa ser Linux, macOS ou WSL (Windows **não** pode ser control node). |
| **Managed nodes** | As máquinas gerenciadas. Precisam de SSH e Python (Linux) ou WinRM/SSH e PowerShell (Windows). |
| **Inventário** | A lista dos managed nodes, organizados em grupos. |
| **Módulo** | A unidade de trabalho: `apt`, `copy`, `service`, `user`... Cada tarefa chama um módulo. |
| **Tarefa (*task*)** | Uma chamada a um módulo com parâmetros. |
| **Playbook** | Arquivo YAML com uma ou mais *plays*; cada play aplica tarefas a um grupo de hosts. |
| **Role** | Um pacote reutilizável de tarefas, templates, arquivos e variáveis. |
| **Collection** | Forma de distribuição de módulos, plugins e roles (ex.: `community.general`). |
| **Plugin** | Estende o próprio Ansible: conexão, filtros, inventário dinâmico, *callbacks*. |

## O que acontece em uma tarefa

Quando o Ansible executa a tarefa "instalar o nginx" em `web01`:

1. Lê o inventário e as variáveis do host.
2. Abre uma conexão **SSH** com `web01` (reaproveitada nas próximas tarefas).
3. Gera um pequeno programa **Python** com o código do módulo `apt` e os parâmetros da tarefa.
4. Copia esse programa para um diretório temporário em `web01` (ou envia direto pela
   conexão, com *pipelining*).
5. Executa o programa (com `sudo`, se `become: true`).
6. O módulo verifica o estado, faz a mudança se necessário e devolve um **JSON**:
   `{"changed": true, ...}`.
7. Remove os arquivos temporários e mostra o resultado: `ok`, `changed`, `failed`, `skipped`.

Esse processo acontece **em paralelo** em vários hosts (5 por padrão, configurável com
`forks`), e **tarefa por tarefa**: todos os hosts terminam a tarefa 1 antes de começarem a 2.

{{% notice note %}}
O módulo roda **no managed node**, não no control node. Por isso o Python precisa estar
instalado lá. A exceção são as tarefas com `delegate_to: localhost` e alguns módulos de
rede e nuvem, que rodam localmente e conversam com uma API.
{{% /notice %}}

## Os comandos

| Comando | Para que serve |
|---|---|
| `ansible` | Executa **um** módulo contra um grupo de hosts (comandos *ad-hoc*). |
| `ansible-playbook` | Executa playbooks. |
| `ansible-inventory` | Mostra o inventário como o Ansible o entende. |
| `ansible-doc` | Documentação de módulos e plugins, offline. |
| `ansible-config` | Mostra e valida as configurações em uso. |
| `ansible-galaxy` | Instala e cria roles e collections. |
| `ansible-vault` | Criptografa arquivos e variáveis com segredos. |
| `ansible-console` | Console interativo para executar módulos. |
| `ansible-pull` | Inverte o modelo: o próprio host baixa o repositório e se configura. |
