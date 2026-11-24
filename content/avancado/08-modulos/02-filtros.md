+++
title = "Filtros Jinja2 próprios"
weight = 2
description = "Filter plugins em Python para simplificar expressões complexas nos templates e playbooks."
+++

Módulos rodam **no managed node**. Já os **plugins** — filtros, lookups, callbacks,
inventários — rodam **no control node** e estendem o próprio Ansible. O mais simples e útil
de escrever é o **filtro**.

## O problema

```yaml
# monta "usuario:uid" para os ativos, ordenado pelo uid
lista: "{{ usuarios | selectattr('ativo') | sort(attribute='uid') | map(attribute='nome') | zip(usuarios | selectattr('ativo') | sort(attribute='uid') | map(attribute='uid')) | map('join', ':') | list }}"
```

Funciona, mas ninguém consegue manter isso.

## O filtro

Crie `filter_plugins/curso.py` ao lado do playbook (ou `plugins/filter/` em uma collection):

```python
# filter_plugins/curso.py


def usuarios_ativos(usuarios, separador=":"):
    """Retorna 'nome<sep>uid' dos usuários ativos, ordenados pelo uid."""
    ativos = sorted((u for u in usuarios if u.get("ativo", True)), key=lambda u: u["uid"])
    return [f"{u['nome']}{separador}{u['uid']}" for u in ativos]


def tamanho_humano(num_bytes):
    """Converte bytes em uma string legível: 1536 -> '1.5 KiB'."""
    for unidade in ["B", "KiB", "MiB", "GiB", "TiB"]:
        if abs(num_bytes) < 1024:
            return f"{num_bytes:.1f} {unidade}"
        num_bytes /= 1024
    return f"{num_bytes:.1f} PiB"


class FilterModule:
    def filters(self):
        return {
            "usuarios_ativos": usuarios_ativos,
            "tamanho_humano": tamanho_humano,
        }
```

## Usando

```yaml
- name: Relatório
  hosts: all
  gather_facts: true
  vars:
    usuarios:
      - {nome: maria, uid: 1002}
      - {nome: joao, uid: 1001}
      - {nome: ana, uid: 1003, ativo: false}
  tasks:
    - name: Usuários ativos
      ansible.builtin.debug:
        msg: "{{ usuarios | usuarios_ativos }}"        # ['joao:1001', 'maria:1002']

    - name: Disco raiz
      ansible.builtin.debug:
        msg: >-
          Livre em /:
          {{ (ansible_facts['mounts'] | selectattr('mount', 'eq', '/') | first).size_available | tamanho_humano }}
```

{{% notice tip "Onde colocar plugins" %}}
Diretórios `filter_plugins/`, `lookup_plugins/`, `callback_plugins/` e `library/` ao lado do
playbook (ou dentro de uma role) são carregados automaticamente. Para compartilhar entre
projetos, empacote tudo em uma [collection]({{% relref "/avancado/02-galaxy/02-collection-propria" %}})
e use pelo FQCN: `{{ usuarios | sidnei.infra.usuarios_ativos }}`.
{{% /notice %}}

## Fim do curso

Você saiu do primeiro `ansible all -m ping` e chegou a roles testadas, segredos no Vault,
Windows, deploys sem downtime e plugins próprios. O próximo passo é aplicar no seu ambiente:
comece pelo **baseline** (usuários, SSH, pacotes, NTP) dos servidores que você já tem e
automatize um pouco mais a cada semana.
