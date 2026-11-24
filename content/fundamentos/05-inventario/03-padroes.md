+++
title = "Padrões e --limit"
weight = 3
description = "Selecionando hosts com padrões: união, interseção, exclusão, curingas e --limit."
+++

Todo comando `ansible` recebe um **padrão** dizendo em quais hosts rodar. Nos playbooks, o
mesmo padrão vai na linha `hosts:`.

| Padrão | Significado |
|---|---|
| `all` ou `*` | Todos os hosts |
| `web01` | Um host |
| `web` | Um grupo |
| `web:db` | **União**: hosts em `web` **ou** em `db` |
| `web:&producao` | **Interseção**: hosts em `web` **e** em `producao` |
| `debian:!db` | **Exclusão**: hosts em `debian`, **menos** os de `db` |
| `web0*` | Curinga |
| `~web0[12]` | Expressão regular (começa com `~`) |
| `web[0]` | O primeiro host do grupo `web` |
| `web[0:1]` | Os dois primeiros hosts do grupo |

```bash
ansible 'web:&producao' --list-hosts
ansible 'debian:!db' --list-hosts
ansible 'web0*' --list-hosts
```

{{% notice warning %}}
Use **aspas simples** em padrões com `!`, `&` ou `*` para o shell não interpretar esses
caracteres antes do Ansible.
{{% /notice %}}

## --limit

O `--limit` (`-l`) restringe ainda mais os hosts escolhidos. É muito usado com playbooks,
cujo `hosts:` está fixo no arquivo:

```bash
ansible-playbook site.yml --limit web01
ansible-playbook site.yml -l 'web:!web02'
ansible-playbook site.yml -l @falhas.txt      # hosts listados em um arquivo
```

{{% notice tip %}}
Antes de executar algo arriscado, confira os alvos com `--list-hosts`:
`ansible-playbook site.yml -l producao --list-hosts`.
{{% /notice %}}
