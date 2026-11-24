+++
title = "Jinja2"
weight = 1
description = "Expressões, filtros, testes e lookups — a linguagem por trás de {{ }}."
+++

O **Jinja2** é o motor de templates do Ansible. Ele é usado em dois lugares: nos valores dos
playbooks (`"{{ variavel }}"`) e nos arquivos `.j2` do módulo `template`.

## Delimitadores

| Sintaxe | Uso |
|---|---|
| `{{ ... }}` | **Expressão**: imprime um valor |
| `{% ... %}` | **Instrução**: `if`, `for`, `set` |
| `{# ... #}` | Comentário (não aparece no resultado) |

## Expressões

```jinja
{{ http_port + 1 }}
{{ "Olá, " ~ usuario }}                {# ~ concatena como string #}
{{ app.nome | upper }}
{{ servidores | length }}
{{ 'web' in group_names }}
{{ ansible_facts['memtotal_mb'] // 1024 }} GB
{{ 'sim' if habilitado else 'não' }}
```

## Filtros

Filtros transformam valores com `|` e podem ser encadeados:

| Filtro | Exemplo | Resultado |
|---|---|---|
| `default` | `{{ porta \| default(80) }}` | 80 se `porta` indefinida |
| `upper`, `lower`, `capitalize` | `{{ 'web' \| upper }}` | `WEB` |
| `replace` | `{{ 'a-b' \| replace('-', '_') }}` | `a_b` |
| `int`, `float`, `string`, `bool` | `{{ '8' \| int + 1 }}` | `9` |
| `join` | `{{ ['a','b'] \| join(',') }}` | `a,b` |
| `split` | `{{ 'a,b' \| split(',') }}` | `['a', 'b']` |
| `first`, `last`, `min`, `max`, `sum` | `{{ [3,1,2] \| max }}` | `3` |
| `unique`, `sort`, `reverse` | `{{ [2,1,2] \| unique \| sort }}` | `[1, 2]` |
| `union`, `intersect`, `difference` | `{{ [1,2] \| union([2,3]) }}` | `[1, 2, 3]` |
| `map` | `{{ usuarios \| map(attribute='nome') }}` | lista de nomes |
| `selectattr`, `rejectattr` | `{{ usuarios \| selectattr('ativo') }}` | só os ativos |
| `combine` | `{{ padrao \| combine(custom) }}` | mescla dicionários |
| `dict2items`, `items2dict` | `{{ {'a':1} \| dict2items }}` | `[{'key':'a','value':1}]` |
| `to_yaml`, `to_nice_json`, `from_json` | `{{ config \| to_nice_json }}` | JSON formatado |
| `regex_replace`, `regex_search` | `{{ 'v1.2' \| regex_replace('^v', '') }}` | `1.2` |
| `password_hash` | `{{ senha \| password_hash('sha512') }}` | hash para `/etc/shadow` |
| `b64encode`, `b64decode`, `hash('sha256')` | | codificação e hashes |
| `ansible.utils.ipaddr` | `{{ '192.168.56.0/24' \| ansible.utils.ipaddr('netmask') }}` | `255.255.255.0` |
| `mandatory` | `{{ versao \| mandatory }}` | erro se indefinida |

```yaml
# exemplo: lista de nomes de usuários ativos
- ansible.builtin.debug:
    msg: "{{ usuarios | selectattr('ativo', 'equalto', true) | map(attribute='nome') | join(', ') }}"
```

## Testes

Testes retornam verdadeiro ou falso e são usados com `is`:

```jinja
{{ variavel is defined }}
{{ variavel is undefined }}
{{ resultado is failed }}         {# succeeded, changed, skipped #}
{{ caminho is file }}             {# directory, link, exists — no control node #}
{{ versao is version('2.0', '>=') }}
{{ nome is match('^web\d+') }}
{{ lista is iterable }}
```

## Lookups

*Lookups* buscam dados **no control node** — arquivos, variáveis de ambiente, senhas:

```yaml
chave_publica: "{{ lookup('ansible.builtin.file', '~/.ssh/id_ed25519.pub') }}"
usuario_local: "{{ lookup('ansible.builtin.env', 'USER') }}"
linhas: "{{ lookup('ansible.builtin.file', 'lista.txt').splitlines() }}"
senha_app: "{{ lookup('ansible.builtin.password', 'credenciais/app chars=ascii_letters,digits length=24') }}"
```

O lookup `password` gera uma senha aleatória na primeira execução e a guarda no arquivo
indicado; nas seguintes, reutiliza a mesma.

## Testando expressões rapidamente

```bash
ansible localhost -m ansible.builtin.debug -a "msg={{ [3,1,2] | sort | join('-') }}"
ansible web01 -m ansible.builtin.debug -a "msg={{ ansible_facts['memtotal_mb'] // 1024 }}"
```

{{% notice note %}}
O segundo comando funciona sem coletar facts? Não: comandos ad-hoc não executam o
*Gathering Facts*. Sem cache de facts, ele falha com variável indefinida — ótimo momento
para revisar o [cache de facts]({{% relref "/playbooks/02-variaveis/03-facts" %}}).
{{% /notice %}}
