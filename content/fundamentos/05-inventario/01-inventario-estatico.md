+++
title = "Hosts e grupos"
weight = 1
description = "Inventário em INI e em YAML, grupos, grupos de grupos e os grupos all e ungrouped."
+++

## Formato INI

O formato mais simples. Crie `~/ansible/inventory.ini`:

```ini
[web]
web01
web02

[db]
db01

[debian:children]
web
db
```

- Cada `[seção]` é um **grupo**.
- `[grupo:children]` cria um **grupo de grupos**.
- Um host pode estar em quantos grupos quiser.

Se os nomes não resolvem por DNS ou `/etc/hosts`, informe o endereço:

```ini
[web]
web01 ansible_host=192.168.56.21
web02 ansible_host=192.168.56.22
```

E faixas economizam linhas:

```ini
[web]
web[01:20].curso.lan      # web01 ... web20
```

## Formato YAML

O mesmo inventário em YAML (`inventory.yml`). É mais verboso, mas se integra melhor com
variáveis complexas:

```yaml
all:
  children:
    web:
      hosts:
        web01:
          ansible_host: 192.168.56.21
        web02:
          ansible_host: 192.168.56.22
    db:
      hosts:
        db01:
          ansible_host: 192.168.56.31
    debian:
      children:
        web:
        db:
```

## Grupos automáticos

Todo inventário tem dois grupos implícitos:

| Grupo | Contém |
|---|---|
| `all` | Todos os hosts. |
| `ungrouped` | Hosts que não estão em nenhum grupo além de `all`. |

## Organizando os grupos

Pense em grupos em mais de uma dimensão; um host costuma estar em várias:

```ini
# o que o host faz
[web]
web01
web02

[db]
db01

# onde ele está
[producao]
web01
db01

[homologacao]
web02

# como ele é
[debian:children]
web
db
```

Assim dá para executar algo em "todos os web de produção" combinando grupos
(veja [padrões]({{% relref "/fundamentos/05-inventario/03-padroes" %}})).

{{% notice tip "Um inventário por ambiente" %}}
Em projetos maiores é comum ter um inventário **por ambiente** — `inventories/producao/`,
`inventories/homologacao/` — para que seja impossível rodar algo em produção por engano.
Veja [boas práticas]({{% relref "/avancado/05-boas-praticas" %}}).
{{% /notice %}}

## Conferindo

O comando `ansible-inventory` mostra o inventário como o Ansible o interpretou:

```bash
ansible-inventory --graph
```

```text
@all:
  |--@ungrouped:
  |--@debian:
  |  |--@web:
  |  |  |--web01
  |  |  |--web02
  |  |--@db:
  |  |  |--db01
```

```bash
ansible-inventory --list           # tudo em JSON, com as variáveis
ansible-inventory --host web01     # variáveis de um host
ansible all --list-hosts           # só os nomes
```
