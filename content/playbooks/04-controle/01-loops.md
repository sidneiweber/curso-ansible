+++
title = "Loops"
weight = 1
description = "loop, with_items, loops sobre dicionários, loop_control e register em loops."
+++

## loop

Em vez de repetir a mesma tarefa:

```yaml
- name: Cria o usuário maria
  ansible.builtin.user:
    name: maria
- name: Cria o usuário joao
  ansible.builtin.user:
    name: joao
```

use `loop` — cada item fica disponível em `item`:

```yaml
- name: Cria os usuários
  ansible.builtin.user:
    name: "{{ item }}"
    shell: /bin/bash
  loop:
    - maria
    - joao
    - ana
```

{{% notice tip %}}
Muitos módulos aceitam listas diretamente, o que é **mais rápido** que um loop (uma chamada
só). O `apt`, por exemplo: `name: [nginx, git, curl]`. Use `loop` quando o módulo não aceita.
{{% /notice %}}

## with_items e outros with_*

Em playbooks mais antigos você verá `with_items`, `with_dict`, `with_fileglob`...
Eles continuam funcionando, mas o `loop` (com filtros) é o padrão atual:

| Antigo | Atual |
|---|---|
| `with_items: "{{ lista }}"` | `loop: "{{ lista \| flatten(levels=1) }}"` |
| `with_dict: "{{ dic }}"` | `loop: "{{ dic \| dict2items }}"` |
| `with_fileglob: "files/*.conf"` | `loop: "{{ query('fileglob', 'files/*.conf') }}"` |
| `with_sequence: start=1 end=5` | `loop: "{{ range(1, 6) \| list }}"` |
| `with_nested` | `loop: "{{ lista1 \| product(lista2) \| list }}"` |

## Lista de dicionários

```yaml
vars:
  usuarios:
    - nome: maria
      grupos: [sudo, www-data]
    - nome: joao
      grupos: [www-data]
    - nome: ana
      grupos: []
      estado: absent

tasks:
  - name: Gerencia os usuários
    ansible.builtin.user:
      name: "{{ item.nome }}"
      groups: "{{ item.grupos }}"
      append: true
      state: "{{ item.estado | default('present') }}"
    loop: "{{ usuarios }}"
```

## Dicionários

```yaml
vars:
  sysctl:
    net.ipv4.ip_forward: 1
    vm.swappiness: 10

tasks:
  - name: Ajusta parâmetros do kernel
    ansible.posix.sysctl:
      name: "{{ item.key }}"
      value: "{{ item.value }}"
      sysctl_set: true
    loop: "{{ sysctl | dict2items }}"
```

## loop_control

```yaml
- name: Cria os usuários
  ansible.builtin.user:
    name: "{{ usuario.nome }}"
  loop: "{{ usuarios }}"
  loop_control:
    loop_var: usuario          # nome da variável em vez de "item"
    label: "{{ usuario.nome }}"  # o que aparece na saída (evita imprimir o dicionário inteiro)
    pause: 2                   # segundos entre os itens
    index_var: i               # posição do item (0, 1, 2...)
```

`loop_var` é obrigatório quando há loops aninhados (um `include_tasks` com loop dentro de
outro loop), para que as variáveis não colidam.

## register em loops

O resultado de cada item fica na lista `results`:

```yaml
- name: Verifica sites
  ansible.builtin.uri:
    url: "https://{{ item }}"
  loop:
    - sidneiweber.com.br
    - debian.org
  register: sites

- name: Mostra o status de cada um
  ansible.builtin.debug:
    msg: "{{ item.item }} → {{ item.status }}"
  loop: "{{ sites.results }}"
  loop_control:
    label: "{{ item.item }}"
```

## until: repetir até dar certo

```yaml
- name: Aguarda a aplicação responder
  ansible.builtin.uri:
    url: http://localhost:8080/health
  register: saude
  until: saude.status == 200
  retries: 10
  delay: 3
```
