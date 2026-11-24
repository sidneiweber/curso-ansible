+++
title = "Desempenho"
weight = 1
description = "forks, pipelining, ControlPersist, estratégias, cache de facts, async e profiling."
+++

## Medindo primeiro

Antes de otimizar, descubra **onde** está o tempo:

```ini
# ansible.cfg
[defaults]
callbacks_enabled = ansible.posix.profile_tasks, ansible.posix.timer
```

```text
Monday 23 November 2026  10:15:02 -0300 (0:00:00.412)       0:01:47.201 ******
===============================================================================
Instala nginx e PHP ------------------------------------------------------ 41.32s
Baixa e extrai o WordPress ----------------------------------------------- 22.10s
Atualiza o cache do apt -------------------------------------------------- 12.87s
Gathering Facts ---------------------------------------------------------- 4.02s
...
```

## Ajustes de conexão

```ini
[defaults]
forks = 50                     # hosts em paralelo (padrão: 5)

[ssh_connection]
pipelining = True              # menos operações SSH por tarefa
ssh_args = -o ControlMaster=auto -o ControlPersist=60s   # reaproveita conexões
```

`forks` é limitado pela CPU e memória do control node: cada fork é um processo Python.

## Facts

```ini
[defaults]
gathering = smart              # só coleta se não estiver em cache
fact_caching = jsonfile
fact_caching_connection = .cache/facts
fact_caching_timeout = 86400
```

E `gather_facts: false` nas plays que não usam facts.

## Estratégias

| Estratégia | Comportamento |
|---|---|
| `linear` (padrão) | Todos os hosts terminam a tarefa N antes de qualquer um começar a N+1. |
| `free` | Cada host avança no seu ritmo, sem esperar os outros. |
| `host_pinned` | Como `free`, mas mantém cada fork no mesmo host até o fim. |

```yaml
- name: Atualiza estações de trabalho
  hosts: estacoes
  strategy: free
```

## Tarefas longas: async

Para tarefas demoradas (atualizações grandes, backups), rode em segundo plano e consulte depois:

```yaml
- name: Atualiza o sistema em segundo plano
  ansible.builtin.apt:
    upgrade: dist
  async: 3600          # tempo máximo (s)
  poll: 0              # não espera
  register: atualizacao

- name: Outras tarefas enquanto isso...
  ansible.builtin.debug:
    msg: "trabalhando"

- name: Aguarda a atualização terminar
  ansible.builtin.async_status:
    jid: "{{ atualizacao.ansible_job_id }}"
  register: job
  until: job.finished
  retries: 120
  delay: 30
```

## Menos tarefas, menos módulos

- Passe **listas** para módulos que aceitam (`apt: name: [...]`) em vez de `loop`.
- Use `synchronize` em vez de `copy` para muitos arquivos.
- Use `cache_valid_time` no `apt` para não atualizar o cache toda vez.
