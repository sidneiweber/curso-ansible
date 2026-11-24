+++
title = "Blocks e tratamento de erros"
weight = 5
description = "block, rescue e always; ignore_errors, failed_when, changed_when e any_errors_fatal."
+++

## block: agrupando tarefas

Um `block` agrupa tarefas para aplicar `when`, `become`, `tags` ou variáveis a todas de uma vez:

```yaml
- name: Configuração específica do Debian
  when: ansible_facts['os_family'] == 'Debian'
  become: true
  tags: pacotes
  block:
    - name: Instala o unattended-upgrades
      ansible.builtin.apt:
        name: unattended-upgrades

    - name: Habilita as atualizações automáticas
      ansible.builtin.copy:
        dest: /etc/apt/apt.conf.d/20auto-upgrades
        content: |
          APT::Periodic::Update-Package-Lists "1";
          APT::Periodic::Unattended-Upgrade "1";
        mode: "0644"
```

## rescue e always: try/catch/finally

```yaml
- name: Atualização da aplicação
  block:
    - name: Coloca em manutenção
      ansible.builtin.file:
        path: /srv/app/MANUTENCAO
        state: touch
        mode: "0644"

    - name: Aplica as migrações do banco
      ansible.builtin.command: /srv/app/bin/migrate
      register: migracao
      changed_when: "'Nenhuma migração pendente' not in migracao.stdout"

  rescue:
    - name: Restaura o backup do banco
      ansible.builtin.command: /srv/app/bin/restore-ultimo-backup

    - name: Avisa a equipe
      ansible.builtin.debug:
        msg: "Falha em {{ ansible_failed_task.name }}: {{ ansible_failed_result.msg | default('') }}"

  always:
    - name: Remove o modo de manutenção
      ansible.builtin.file:
        path: /srv/app/MANUTENCAO
        state: absent
```

| Seção | Quando roda |
|---|---|
| `block` | Sempre (as tarefas principais). |
| `rescue` | Só se alguma tarefa do `block` falhar. Se o `rescue` terminar bem, o host **não** é marcado como falho (aparece em `rescued`). |
| `always` | Sempre, com ou sem falha. |

Dentro do `rescue`, `ansible_failed_task` e `ansible_failed_result` dizem o que falhou.

## changed_when

Diz ao Ansible **quando** considerar a tarefa como alteração. Essencial em `command` e `shell`:

```yaml
- name: Consulta a versão
  ansible.builtin.command: nginx -v
  register: versao
  changed_when: false                       # consulta: nunca altera nada

- name: Cria o banco se não existir
  ansible.builtin.command: /usr/local/bin/cria-banco loja
  register: cria
  changed_when: "'criado' in cria.stdout"
```

Ou use `creates`/`removes`, que pulam o comando se o arquivo existir/não existir:

```yaml
- name: Inicializa a aplicação
  ansible.builtin.command:
    cmd: /srv/app/bin/init
    creates: /srv/app/.inicializado
```

## failed_when

Define **quando** a tarefa é considerada falha:

```yaml
- name: Verifica o espaço em disco
  ansible.builtin.command: df --output=pcent /
  register: disco
  changed_when: false
  failed_when: disco.stdout_lines[-1] | trim | replace('%', '') | int > 90

- name: Comando que retorna 1 quando "não encontrado" (não é erro)
  ansible.builtin.command: grep -q swap /etc/fstab
  register: tem_swap
  changed_when: false
  failed_when: tem_swap.rc not in [0, 1]
```

## ignore_errors e ignore_unreachable

```yaml
- name: Tenta parar um serviço que talvez não exista
  ansible.builtin.service:
    name: servico-antigo
    state: stopped
  ignore_errors: true
```

{{% notice tip %}}
`ignore_errors` esconde problemas. Antes de usá-lo, veja se `failed_when` (dizer exatamente
o que é erro) ou um `when` (não executar quando não faz sentido) resolvem melhor.
{{% /notice %}}

## Parando tudo

Por padrão, um host que falha sai da execução e os demais continuam. Para mudar isso:

```yaml
- name: Cluster de banco
  hosts: db
  any_errors_fatal: true          # qualquer falha para a play em TODOS os hosts
  max_fail_percentage: 20         # ou: para se mais de 20% dos hosts falharem
```
