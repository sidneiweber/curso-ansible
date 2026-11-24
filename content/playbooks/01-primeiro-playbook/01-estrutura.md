+++
title = "Estrutura de um playbook"
weight = 1
description = "Playbook, play, tarefa e módulo — e as palavras-chave de cada nível."
+++

## Os níveis

```text
playbook (arquivo .yml)
└── play: "em QUAIS hosts, COMO"        ← lista de plays
    └── tarefa: "O QUÊ"                 ← lista de tarefas
        └── módulo + parâmetros
```

- Um **playbook** é um arquivo YAML que contém uma **lista de plays**.
- Uma **play** associa um grupo de hosts a uma lista de tarefas.
- Uma **tarefa** chama **um módulo** com parâmetros.

## O primeiro playbook

Crie `~/ansible/basico.yml`:

```yaml
---
- name: Configuração básica dos servidores
  hosts: debian
  become: true

  tasks:
    - name: Atualiza o cache do apt
      ansible.builtin.apt:
        update_cache: true
        cache_valid_time: 3600

    - name: Instala pacotes básicos
      ansible.builtin.apt:
        name:
          - vim
          - htop
          - curl
          - git
        state: present

    - name: Define o fuso horário
      community.general.timezone:
        name: America/Sao_Paulo

    - name: Cria a mensagem de boas-vindas
      ansible.builtin.copy:
        dest: /etc/motd
        content: |
          Servidor gerenciado pelo Ansible.
          Alterações manuais serão sobrescritas.
        owner: root
        group: root
        mode: "0644"
```

## Palavras-chave da play

| Palavra-chave | Para que serve |
|---|---|
| `name` | Descrição — aparece na saída. Sempre preencha. |
| `hosts` | Padrão de hosts (grupo, host, `all`, `web:&producao`...). |
| `become` | Executa as tarefas com sudo. |
| `remote_user` | Usuário SSH (sobrescreve o `ansible.cfg`). |
| `gather_facts` | Coleta os facts no início (padrão `true`). |
| `vars`, `vars_files` | Variáveis da play. |
| `tasks` | A lista de tarefas. |
| `handlers` | Tarefas executadas só quando notificadas. |
| `pre_tasks`, `post_tasks` | Tarefas antes e depois das roles e tarefas. |
| `roles` | Roles a aplicar. |
| `serial` | Quantos hosts por vez (*rolling update*). |

## Palavras-chave da tarefa

| Palavra-chave | Para que serve |
|---|---|
| `name` | Descrição da tarefa. |
| *módulo* | Exatamente **um** módulo por tarefa, com seus parâmetros. |
| `become` | sudo só nesta tarefa. |
| `when` | Condição para executar. |
| `loop` | Repete a tarefa para cada item. |
| `register` | Guarda o resultado em uma variável. |
| `notify` | Aciona um handler se a tarefa mudar algo. |
| `tags` | Rótulos para executar só parte do playbook. |
| `ignore_errors`, `failed_when`, `changed_when` | Controle de erro e de status. |

{{% notice tip "Nomeie tudo" %}}
O `name` das plays e tarefas é a documentação do playbook e o que aparece na saída.
Escreva como uma frase que descreve o **estado desejado**: "Instala o nginx",
"Garante o serviço rodando".
{{% /notice %}}

## Vários plays em um playbook

Um playbook pode orquestrar grupos diferentes, em ordem:

```yaml
---
- name: Banco de dados
  hosts: db
  become: true
  tasks:
    - name: Instala o PostgreSQL
      ansible.builtin.apt:
        name: postgresql
        state: present

- name: Servidores web
  hosts: web
  become: true
  tasks:
    - name: Instala o nginx
      ansible.builtin.apt:
        name: nginx
        state: present
```
