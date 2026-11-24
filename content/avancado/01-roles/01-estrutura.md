+++
title = "Estrutura de uma role"
weight = 1
description = "O que é uma role, os diretórios padrão e como usá-la em um playbook."
+++

Uma **role** é um pacote com **tudo** o que é preciso para configurar uma função — tarefas,
handlers, templates, arquivos e variáveis — em uma estrutura de diretórios padronizada.
Com roles, o playbook vira uma lista do que cada servidor **é**:

```yaml
- name: Servidores web
  hosts: web
  become: true
  roles:
    - comum
    - nginx
    - php_fpm
    - wordpress
```

## Os diretórios

```text
roles/
└── nginx/
    ├── defaults/
    │   └── main.yml        # variáveis com valores padrão (precedência baixíssima)
    ├── vars/
    │   └── main.yml        # variáveis internas (precedência alta — não são para sobrescrever)
    ├── tasks/
    │   └── main.yml        # ponto de entrada das tarefas
    ├── handlers/
    │   └── main.yml        # handlers
    ├── templates/          # arquivos .j2 usados pelo módulo template
    ├── files/              # arquivos usados por copy e script
    ├── meta/
    │   ├── main.yml        # dependências e informações para o Galaxy
    │   └── argument_specs.yml  # validação das variáveis de entrada
    ├── tests/              # inventário e playbook de teste
    └── README.md
```

Só o que for usado precisa existir — uma role pode ter apenas `tasks/main.yml`.

Dentro da role, os caminhos são **relativos aos diretórios padrão**: um `template` com
`src: site.conf.j2` procura em `roles/nginx/templates/`, um `copy` com `src: index.html`
procura em `roles/nginx/files/`.

## Criando o esqueleto

```bash
ansible-galaxy role init roles/nginx
```

## Onde o Ansible procura roles

1. Em `roles/` ao lado do playbook.
2. Nos caminhos de `roles_path` do `ansible.cfg` (padrão: `~/.ansible/roles`,
   `/usr/share/ansible/roles`, `/etc/ansible/roles`).
3. Dentro de collections (`namespace.colecao.role`).

## Três formas de usar

```yaml
# 1. roles: — executadas ANTES das tasks da play (depois das pre_tasks)
- hosts: web
  roles:
    - nginx
    - role: wordpress
      vars:
        wp_versao: "6.8.3"
      tags: wordpress

# 2. import_role — estática, no meio das tarefas
  tasks:
    - name: Configura o nginx
      ansible.builtin.import_role:
        name: nginx

# 3. include_role — dinâmica: aceita loop e when avaliado na hora
    - name: Instala o banco escolhido
      ansible.builtin.include_role:
        name: "{{ banco_escolhido }}"
```

## Ordem de execução de uma play

```text
pre_tasks → handlers notificados
roles     → handlers notificados
tasks     → handlers notificados
post_tasks→ handlers notificados
```
