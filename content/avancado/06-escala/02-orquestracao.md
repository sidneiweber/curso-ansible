+++
title = "Orquestração e rolling updates"
weight = 2
description = "serial, max_fail_percentage, delegate_to, run_once e atualizações sem downtime."
+++

## serial: em lotes

Por padrão, cada tarefa roda em **todos** os hosts antes da próxima. Para atualizar um
cluster sem derrubar tudo ao mesmo tempo, processe os hosts em **lotes**:

```yaml
- name: Atualização dos servidores web
  hosts: web
  serial: 1                       # um host por vez
  # serial: "25%"                 # um quarto por vez
  # serial: [1, 5, "50%"]         # 1 primeiro (canário), depois 5, depois metade por vez
  max_fail_percentage: 0          # qualquer falha interrompe os próximos lotes
```

Cada lote executa a play **inteira** (incluindo handlers) antes do próximo começar.

## delegate_to: executar em outro host

Uma tarefa sobre o host atual, mas **executada em outro** — clássico para balanceadores:

```yaml
- name: Rolling update dos servidores web
  hosts: web
  serial: 1
  become: true

  pre_tasks:
    - name: Tira o servidor do balanceador
      community.general.haproxy:
        state: disabled
        host: "{{ inventory_hostname }}"
        backend: web
        socket: /run/haproxy/admin.sock
      delegate_to: "{{ item }}"
      loop: "{{ groups['balanceadores'] }}"

  roles:
    - wordpress

  post_tasks:
    - name: Aguarda a aplicação responder
      ansible.builtin.uri:
        url: "http://{{ ansible_host }}/"
      register: resposta
      until: resposta.status == 200
      retries: 20
      delay: 3
      delegate_to: localhost
      become: false

    - name: Devolve o servidor ao balanceador
      community.general.haproxy:
        state: enabled
        host: "{{ inventory_hostname }}"
        backend: web
        socket: /run/haproxy/admin.sock
      delegate_to: "{{ item }}"
      loop: "{{ groups['balanceadores'] }}"
```

`delegate_to: localhost` (ou `ansible.builtin.local_action`) roda no control node — útil para
chamadas de API, testes HTTP e notificações.

## run_once

Executa a tarefa **uma única vez**, no primeiro host do lote, e o resultado vale para todos:

```yaml
- name: Aplica as migrações do banco (só uma vez)
  ansible.builtin.command: /srv/app/bin/migrate
  run_once: true
  register: migracao
  changed_when: "'Applied' in migracao.stdout"

- name: Avisa no chat
  ansible.builtin.uri:
    url: "{{ webhook_chat }}"
    method: POST
    body_format: json
    body:
      text: "Deploy concluído em {{ ansible_play_hosts | join(', ') }}"
  run_once: true
  delegate_to: localhost
```

{{% notice warning %}}
`run_once` com `serial` roda **uma vez por lote**, não uma vez no total. Para algo realmente
único, coloque em uma play separada antes do rolling update.
{{% /notice %}}

## throttle

Limita quantos hosts executam **uma tarefa específica** ao mesmo tempo, sem mudar o resto:

```yaml
- name: Baixa o artefato do servidor interno (que não aguenta 50 downloads simultâneos)
  ansible.builtin.get_url:
    url: https://artefatos.curso.lan/app.tar.gz
    dest: /tmp/app.tar.gz
    mode: "0644"
  throttle: 5
```
