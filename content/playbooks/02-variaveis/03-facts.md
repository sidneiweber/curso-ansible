+++
title = "Facts"
weight = 3
description = "Informações coletadas dos hosts, ansible_facts, custom facts e cache de facts."
+++

No início de cada play, a tarefa **Gathering Facts** executa o módulo `setup` em cada host
e guarda o resultado na variável **`ansible_facts`**.

## Os facts mais usados

| Fact | Exemplo |
|---|---|
| `ansible_facts['hostname']` | `web01` |
| `ansible_facts['fqdn']` | `web01.curso.lan` |
| `ansible_facts['os_family']` | `Debian`, `RedHat`, `Windows` |
| `ansible_facts['distribution']` | `Debian`, `Ubuntu`, `Rocky` |
| `ansible_facts['distribution_major_version']` | `13` |
| `ansible_facts['distribution_release']` | `trixie` |
| `ansible_facts['default_ipv4']['address']` | `10.0.2.15` |
| `ansible_facts['all_ipv4_addresses']` | lista de IPs |
| `ansible_facts['enp0s8']['ipv4']['address']` | `192.168.56.21` |
| `ansible_facts['memtotal_mb']` | `983` |
| `ansible_facts['processor_vcpus']` | `1` |
| `ansible_facts['mounts']` | lista de pontos de montagem com tamanho e uso |
| `ansible_facts['service_mgr']` | `systemd` |

```yaml
- name: Relatório
  ansible.builtin.debug:
    msg: >-
      {{ inventory_hostname }}: {{ ansible_facts['distribution'] }}
      {{ ansible_facts['distribution_version'] }},
      {{ ansible_facts['memtotal_mb'] }} MB de RAM,
      {{ ansible_facts['processor_vcpus'] }} vCPU
```

{{% notice warning "ansible_facts['x'] × ansible_x" %}}
Você vai encontrar muito código usando `ansible_distribution`, `ansible_hostname` etc. Esses
nomes ainda funcionam, mas a injeção dos facts como variáveis de topo está **obsoleta** nas
versões recentes do ansible-core. Use sempre `ansible_facts['distribution']`.
{{% /notice %}}

## Desligando ou limitando a coleta

Coletar facts leva alguns segundos por host. Se a play não usa, desligue:

```yaml
- name: Tarefa rápida
  hosts: all
  gather_facts: false
```

Ou colete só um subconjunto:

```yaml
- name: Só rede
  hosts: all
  gather_facts: false
  tasks:
    - name: Coleta facts de rede
      ansible.builtin.setup:
        gather_subset:
          - "!all"
          - network
```

## Cache de facts

Para não coletar a cada execução, guarde os facts em cache (`ansible.cfg`):

```ini
[defaults]
gathering = smart
fact_caching = jsonfile
fact_caching_connection = .cache/facts
fact_caching_timeout = 7200
```

Com cache, `hostvars['db01']['ansible_facts']` fica disponível mesmo quando `db01` não está
na play atual — útil para montar configurações que dependem de outros hosts.

## Custom facts

Qualquer arquivo em `/etc/ansible/facts.d/*.fact` no managed node vira um fact em
`ansible_facts['ansible_local']`. Pode ser um INI, um JSON ou um **executável** que imprime JSON:

```yaml
- name: Cria o diretório de facts locais
  ansible.builtin.file:
    path: /etc/ansible/facts.d
    state: directory
    mode: "0755"

- name: Registra a função do servidor
  ansible.builtin.copy:
    dest: /etc/ansible/facts.d/papel.fact
    content: |
      [servidor]
      papel=web
      responsavel=equipe-infra
    mode: "0644"
```

Na próxima coleta: `ansible_facts['ansible_local']['papel']['servidor']['papel']` → `web`.
