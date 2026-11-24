+++
title = "Handlers"
weight = 3
description = "Reiniciando serviços apenas quando a configuração muda: notify, handlers, listen e flush_handlers."
+++

## O problema

```yaml
- name: Configura o nginx
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf

- name: Reinicia o nginx
  ansible.builtin.service:
    name: nginx
    state: restarted
```

Assim o nginx é reiniciado **em toda execução**, mesmo quando nada mudou — derrubando
conexões sem motivo e quebrando a idempotência.

## A solução: notify + handlers

Um **handler** é uma tarefa que só roda quando **notificada** por outra tarefa que teve
status `changed`:

```yaml
- name: Servidor web
  hosts: web
  become: true

  tasks:
    - name: Configura o nginx
      ansible.builtin.template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        validate: nginx -t -c %s
      notify: Recarrega o nginx

    - name: Publica o site
      ansible.builtin.template:
        src: site.conf.j2
        dest: /etc/nginx/sites-available/default
      notify: Recarrega o nginx

  handlers:
    - name: Recarrega o nginx
      ansible.builtin.service:
        name: nginx
        state: reloaded
```

Regras:

- O handler roda **uma única vez**, no **fim da play**, mesmo que notificado várias vezes.
- Se nenhuma tarefa notificou (nada mudou), ele **não roda**.
- Handlers rodam na **ordem em que estão definidos** em `handlers`, não na ordem das notificações.
- O `notify` usa o **nome** do handler — exatamente igual.

## listen: vários handlers com um nome

```yaml
handlers:
  - name: Valida a configuração
    ansible.builtin.command: nginx -t
    changed_when: false
    listen: nginx alterado

  - name: Recarrega o nginx
    ansible.builtin.service:
      name: nginx
      state: reloaded
    listen: nginx alterado
```

```yaml
notify: nginx alterado      # aciona os dois
```

## flush_handlers: executar antes do fim

Às vezes o serviço precisa ser reiniciado **antes** das próximas tarefas (por exemplo, para
testar se a aplicação responde):

```yaml
- name: Aplica a nova configuração agora
  ansible.builtin.meta: flush_handlers

- name: Testa a aplicação
  ansible.builtin.uri:
    url: http://localhost/
```

## E se o playbook falhar?

Se uma tarefa falhar **depois** de uma notificação, os handlers pendentes daquele host
**não rodam** — e na próxima execução a tarefa já não terá `changed`, então o serviço
ficará com a configuração nova sem ter sido reiniciado. Para evitar:

```bash
ansible-playbook site.yml --force-handlers
```

ou, no `ansible.cfg`, `force_handlers = True`.

{{% notice tip "reload × restart" %}}
Prefira `state: reloaded` quando o serviço suporta (nginx, apache, sshd, postgresql para a
maioria dos parâmetros): aplica a configuração sem derrubar conexões.
{{% /notice %}}
