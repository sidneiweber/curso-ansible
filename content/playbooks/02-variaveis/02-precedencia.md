+++
title = "Precedência"
weight = 2
description = "Qual valor vence quando a mesma variável é definida em vários lugares."
+++

O Ansible tem **22 níveis** de precedência de variáveis. Não é preciso decorar; basta
entender a lógica e conhecer os mais usados.

## A ordem (da mais fraca para a mais forte)

```text
 1. valores na linha de comando (ex.: -u usuario) — não são variáveis
 2. defaults de role (roles/x/defaults/main.yml)          ← mais fraca entre as variáveis
 3. group_vars/all do inventário
 4. group_vars/all do playbook
 5. group_vars/<grupo> do inventário
 6. group_vars/<grupo> do playbook
 7. host_vars do inventário
 8. host_vars do playbook
 9. facts do host
10. vars da play
11. vars_prompt
12. vars_files
13. vars de role (roles/x/vars/main.yml)
14. vars de bloco
15. vars de tarefa
16. include_vars
17. set_fact / register
18. parâmetros de role
19. parâmetros de include
20. extra vars (-e)                                        ← sempre vence
```

(Lista simplificada; a completa está na documentação oficial, em *Variable precedence*.)

## A lógica

- **Mais específico vence o mais genérico:** host > grupo > all.
- **Mais perto da execução vence:** tarefa > bloco > play > inventário.
- **Defaults de role existem para ser sobrescritos**; vars de role, não.
- **`-e` sempre vence** — use para valores pontuais, nunca como configuração permanente.

## Na prática

Restrinja-se a poucos lugares e tenha uma regra para cada um:

| Onde | O que colocar |
|---|---|
| `roles/x/defaults/main.yml` | Todo parâmetro configurável da role, com um valor sensato. |
| `group_vars/all` | O que é igual em toda a infraestrutura (DNS, NTP, fuso horário). |
| `group_vars/<grupo>` | O que muda por função ou ambiente. |
| `host_vars/<host>` | Exceções de um host específico. |
| `-e` | Valores de uma execução (versão a implantar, flags de depuração). |

{{% notice tip "De onde veio esse valor?" %}}
Para descobrir o valor final de uma variável em um host:
`ansible web01 -m ansible.builtin.debug -a "var=http_port"`
ou `ansible-inventory --host web01 --yaml` para as variáveis de inventário.
{{% /notice %}}

## Exercício

1. Defina `http_port: 80` em `group_vars/web.yml`, `8080` em `host_vars/web02.yml` e
   imprima o valor em todos os hosts web com `debug`.
2. Repita passando `-e http_port=9000`. O que acontece?
3. Coloque `http_port: 1234` nos `vars` da play. Ela vence o `host_vars`? E o `-e`?
