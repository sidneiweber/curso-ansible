+++
title = "Dividindo playbooks: import e include"
weight = 6
description = "import_playbook, import_tasks e include_tasks — estático × dinâmico."
+++

Playbooks longos ficam difíceis de ler. O Ansible permite dividi-los em arquivos menores.

## import_playbook

Um playbook "principal" que junta outros — normalmente chamado `site.yml`:

```yaml
# site.yml
- name: Configuração comum
  ansible.builtin.import_playbook: comum.yml

- name: Banco de dados
  ansible.builtin.import_playbook: db.yml

- name: Servidores web
  ansible.builtin.import_playbook: web.yml
```

## Arquivos de tarefas

Um arquivo de tarefas é só a **lista de tarefas**, sem `hosts` nem `tasks:`:

```yaml
# tarefas/usuarios.yml
- name: Cria o grupo de administradores
  ansible.builtin.group:
    name: admins

- name: Cria os usuários
  ansible.builtin.user:
    name: "{{ item }}"
    groups: admins
    append: true
  loop: "{{ admins }}"
```

```yaml
- name: Configuração comum
  hosts: all
  become: true
  tasks:
    - name: Usuários
      ansible.builtin.import_tasks: tarefas/usuarios.yml

    - name: Tarefas específicas da distribuição
      ansible.builtin.include_tasks: "tarefas/{{ ansible_facts['distribution'] | lower }}.yml"
```

## import × include

| | `import_*` (estático) | `include_*` (dinâmico) |
|---|---|---|
| Quando é processado | Ao **carregar** o playbook | Ao **chegar** na tarefa, durante a execução |
| Nome do arquivo com variáveis de host/facts | Não | **Sim** |
| `loop` | Não | **Sim** |
| `when` | Copiado para **cada** tarefa importada | Avaliado **uma vez**, decide se inclui |
| `tags` | Herdadas pelas tarefas importadas | Aplicadas só à própria inclusão (use `apply`) |
| Aparece no `--list-tasks` | Sim | Não |
| `--start-at-task` | Funciona | Não encontra tarefas incluídas |

**Regra prática:** use `import_*` por padrão (é previsível) e `include_*` quando precisar de
algo dinâmico — nome de arquivo com variável ou `loop`.

```yaml
# include com loop: um arquivo de tarefas para cada site
- name: Configura cada site
  ansible.builtin.include_tasks: tarefas/site.yml
  loop: "{{ sites }}"
  loop_control:
    loop_var: site
```

## include_vars

Carrega variáveis durante a execução — útil para variáveis por distribuição:

```yaml
- name: Carrega variáveis da distribuição
  ansible.builtin.include_vars: "{{ item }}"
  with_first_found:
    - "vars/{{ ansible_facts['distribution'] }}-{{ ansible_facts['distribution_major_version'] }}.yml"
    - "vars/{{ ansible_facts['distribution'] }}.yml"
    - vars/padrao.yml
```

O próximo passo natural na organização é transformar esses arquivos em
[roles]({{% relref "/avancado/01-roles" %}}).
