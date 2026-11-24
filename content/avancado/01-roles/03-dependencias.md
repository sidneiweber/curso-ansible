+++
title = "Defaults, vars e dependências"
weight = 3
description = "Quando usar defaults ou vars, variáveis por distribuição e dependências entre roles."
+++

## defaults × vars

| | `defaults/main.yml` | `vars/main.yml` |
|---|---|---|
| Precedência | A **mais baixa** de todas | Alta (vence inventário e `vars` da play) |
| Propósito | Parâmetros que o usuário da role **deve** poder mudar | Constantes internas da role |
| Exemplos | portas, versões, listas de sites, flags | nomes de pacotes, caminhos por distribuição |

Regra prática: **quase tudo vai em `defaults`**. Use `vars` só para valores que não fazem
sentido mudar de fora.

## Variáveis por distribuição

Para uma role que funciona em Debian e RHEL, carregue as diferenças em `vars/`:

```yaml
# roles/nginx/vars/Debian.yml
nginx_usuario: www-data
nginx_dir_sites: /etc/nginx/sites-available

# roles/nginx/vars/RedHat.yml
nginx_usuario: nginx
nginx_dir_sites: /etc/nginx/conf.d
```

```yaml
# início de roles/nginx/tasks/main.yml
- name: Carrega as variáveis da distribuição
  ansible.builtin.include_vars: "{{ ansible_facts['os_family'] }}.yml"
```

## Dependências

Uma role pode exigir que outra rode antes. Declare em `meta/main.yml`:

```yaml
# roles/wordpress/meta/main.yml
dependencies:
  - role: php_fpm
  - role: nginx
    vars:
      nginx_sites:
        - nome: wordpress
          root: /var/www/wordpress
          index: index.php
          php_socket: /run/php/php8.4-fpm.sock
```

Ao aplicar `wordpress`, o Ansible executa antes `php_fpm` e `nginx`.

Regras:

- Uma role dependente roda **uma vez só** por play, mesmo que várias roles dependam dela —
  a menos que seja chamada com parâmetros diferentes ou tenha `allow_duplicates: true`.
- Dependências são executadas **antes** da role que as declara, recursivamente.

{{% notice tip "Dependências com moderação" %}}
Dependências escondem o que vai rodar e acoplam as roles. Muitas equipes preferem
**listar as roles explicitamente** no playbook e usar dependências só quando uma role
realmente não funciona sem a outra.
{{% /notice %}}

## Executando só um pedaço da role

```yaml
- name: Só os handlers e templates de configuração
  ansible.builtin.include_role:
    name: nginx
    tasks_from: config        # roles/nginx/tasks/config.yml em vez de main.yml
    defaults_from: main
```

Útil para dividir uma role em `instalacao.yml`, `config.yml`, `servico.yml` e chamar só a
parte necessária.
