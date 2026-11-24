+++
title = "Condicionais"
weight = 2
description = "when com facts, variáveis, resultados de tarefas e combinações de condições."
+++

`when` executa a tarefa só se a condição for verdadeira. A condição é uma expressão Jinja2
**sem** `{{ }}`:

```yaml
- name: Instala pacotes no Debian/Ubuntu
  ansible.builtin.apt:
    name: nginx
  when: ansible_facts['os_family'] == 'Debian'

- name: Instala pacotes no Rocky/RHEL
  ansible.builtin.dnf:
    name: nginx
  when: ansible_facts['os_family'] == 'RedHat'
```

## Combinando condições

```yaml
# E: lista de condições (todas precisam ser verdadeiras)
when:
  - ansible_facts['distribution'] == 'Debian'
  - ansible_facts['distribution_major_version'] | int >= 12

# OU
when: ansible_facts['distribution'] == 'Debian' or ansible_facts['distribution'] == 'Ubuntu'

# pertence a uma lista
when: ansible_facts['distribution'] in ['Debian', 'Ubuntu']

# negação
when: not habilitar_firewall
```

## Condições comuns

```yaml
when: minha_variavel is defined
when: minha_variavel | default(false)           # booleano opcional
when: "'web' in group_names"                    # host pertence ao grupo
when: inventory_hostname == groups['db'][0]     # só no primeiro host do grupo
when: ansible_facts['memtotal_mb'] > 2048
when: resultado is changed                      # failed, succeeded, skipped
when: resultado.rc != 0
when: "'erro' in resultado.stdout"
when: arquivo.stat.exists
when: not ansible_check_mode
```

{{% notice warning "Condições precisam ser booleanas" %}}
A partir do ansible-core 2.19, o resultado de `when` precisa ser realmente `true` ou `false`.
Condições que antes funcionavam "por acaso" — como `when: minha_lista` ou `when: "{{ var }}"` —
agora geram erro. Seja explícito: `when: minha_lista | length > 0`, `when: var | bool`.
{{% /notice %}}

## Exemplo: reiniciar só se o kernel mudou

```yaml
- name: Atualiza o sistema
  ansible.builtin.apt:
    upgrade: dist
    update_cache: true

- name: Verifica se é preciso reiniciar
  ansible.builtin.stat:
    path: /var/run/reboot-required
  register: reboot_required

- name: Reinicia
  ansible.builtin.reboot:
  when: reboot_required.stat.exists
```

## when com loop

A condição é avaliada **para cada item**:

```yaml
- name: Cria apenas os usuários ativos
  ansible.builtin.user:
    name: "{{ item.nome }}"
  loop: "{{ usuarios }}"
  when: item.ativo | default(true)
```

## assert e fail

Para **parar** a execução quando algo não está certo:

```yaml
- name: Exige Debian 12 ou superior
  ansible.builtin.assert:
    that:
      - ansible_facts['distribution'] == 'Debian'
      - ansible_facts['distribution_major_version'] | int >= 12
    fail_msg: "Sistema não suportado: {{ ansible_facts['distribution'] }}"

- name: Bloqueia produção sem confirmação
  ansible.builtin.fail:
    msg: "Use -e confirmar=sim para executar em produção"
  when:
    - "'producao' in group_names"
    - confirmar | default('nao') != 'sim'
```
