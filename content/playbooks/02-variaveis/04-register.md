+++
title = "register, debug e set_fact"
weight = 4
description = "Guardando o resultado de tarefas, depurando variáveis e criando variáveis em tempo de execução."
+++

## register

`register` guarda o **resultado** de uma tarefa em uma variável:

```yaml
- name: Verifica a versão do kernel
  ansible.builtin.command: uname -r
  register: kernel
  changed_when: false          # apenas consulta: nunca é "changed"

- name: Mostra o resultado completo
  ansible.builtin.debug:
    var: kernel
```

```yaml
kernel:
  changed: false
  cmd: [uname, -r]
  rc: 0
  stdout: 6.12.48+deb13-amd64
  stdout_lines: [6.12.48+deb13-amd64]
  stderr: ''
  stderr_lines: []
  start: '2026-11-23 10:00:01.123456'
  end: '2026-11-23 10:00:01.130000'
  delta: '0:00:00.006544'
  failed: false
```

| Campo | Conteúdo |
|---|---|
| `changed`, `failed`, `skipped` | Status da tarefa |
| `rc` | Código de saída (command/shell) |
| `stdout`, `stdout_lines` | Saída padrão, inteira ou como lista de linhas |
| `stderr` | Saída de erro |
| `results` | Quando a tarefa tem `loop`: uma lista com o resultado de cada item |

Cada módulo devolve campos próprios — veja **RETURN VALUES** no `ansible-doc`. O `stat`,
por exemplo:

```yaml
- name: Verifica se existe o arquivo de manutenção
  ansible.builtin.stat:
    path: /srv/app/MANUTENCAO
  register: manutencao

- name: Avisa
  ansible.builtin.debug:
    msg: "Servidor em manutenção!"
  when: manutencao.stat.exists
```

## debug

```yaml
- ansible.builtin.debug:
    var: ansible_facts['default_ipv4']        # imprime a variável

- ansible.builtin.debug:
    msg: "IP: {{ ansible_facts['default_ipv4']['address'] }}"

- ansible.builtin.debug:
    var: kernel
    verbosity: 2                               # só aparece com -vv ou mais
```

## set_fact

Cria variáveis **durante a execução**, normalmente calculadas a partir de outras:

```yaml
- name: Calcula memória do PHP-FPM
  ansible.builtin.set_fact:
    fpm_max_children: "{{ (ansible_facts['memtotal_mb'] * 0.7 / 50) | int }}"

- name: Mostra
  ansible.builtin.debug:
    msg: "pm.max_children = {{ fpm_max_children }}"
```

Variáveis criadas com `set_fact` ou `register` valem **para aquele host** até o fim da
execução do playbook (inclusive nas plays seguintes) e podem ser lidas por outros hosts via
`hostvars['web01']['fpm_max_children']`.

## Exercícios

1. Registre a saída de `df -h /` e mostre só a última linha (`stdout_lines[-1]`).
2. Use `ansible.builtin.stat` para verificar se `/etc/nginx/nginx.conf` existe e imprima
   uma mensagem diferente em cada caso.
3. Com `set_fact`, crie a variável `ambiente` com o valor `producao` se o host estiver no
   grupo `producao` e `homologacao` caso contrário (dica: `'producao' in group_names`).
