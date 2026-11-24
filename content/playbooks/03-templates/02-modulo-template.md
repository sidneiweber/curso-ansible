+++
title = "O módulo template"
weight = 2
description = "Arquivos .j2, ansible_managed, laços e condicionais em templates e validação antes de aplicar."
+++

O módulo `template` processa um arquivo **Jinja2** no control node e grava o resultado no
managed node. É a forma recomendada de gerenciar arquivos de configuração.

## copy × template

| Módulo | Quando usar |
|---|---|
| `copy` | Arquivo **igual** em todos os hosts (um binário, um certificado, um script). |
| `template` | Arquivo que **varia** por host ou ambiente. |

## Exemplo: página inicial do nginx

`templates/index.html.j2`:

```jinja
<!DOCTYPE html>
<html>
<head><title>{{ inventory_hostname }}</title></head>
<body>
  <h1>{{ site_titulo | default('Curso Ansible') }}</h1>
  <p>Servidor: {{ ansible_facts['hostname'] }} ({{ ansible_facts['enp0s8']['ipv4']['address'] }})</p>
  <p>Sistema: {{ ansible_facts['distribution'] }} {{ ansible_facts['distribution_version'] }}</p>
</body>
</html>
```

```yaml
- name: Publica a página inicial
  ansible.builtin.template:
    src: templates/index.html.j2
    dest: /var/www/html/index.html
    owner: root
    group: root
    mode: "0644"
```

O caminho de `src` é relativo ao diretório `templates/` ao lado do playbook (ou da role).

## Cabeçalho de aviso

A variável `ansible_managed` gera um aviso padrão. Coloque-o no início dos arquivos para que
ninguém edite à mão:

```jinja
# {{ ansible_managed }}
# Alterações manuais serão sobrescritas.
```

## Laços e condicionais

`templates/hosts.j2` — um `/etc/hosts` gerado a partir do inventário:

```jinja
# {{ ansible_managed }}
127.0.0.1   localhost

{% for host in groups['all'] %}
{{ hostvars[host]['ansible_host'] | default(host) }}  {{ host }}.curso.lan {{ host }}
{% endfor %}
```

`templates/sshd_config.j2` — trechos opcionais:

```jinja
# {{ ansible_managed }}
Port {{ ssh_porta | default(22) }}
PermitRootLogin no
PasswordAuthentication {{ 'yes' if ssh_senha_permitida | default(false) else 'no' }}

{% if ssh_usuarios_permitidos is defined %}
AllowUsers {{ ssh_usuarios_permitidos | join(' ') }}
{% endif %}
```

## Controle de espaços

`{% ... %}` deixa linhas em branco no resultado. Use `-` para remover espaços e quebras
de linha antes ou depois da instrução:

```jinja
upstream app {
{% for host in groups['web'] -%}
    server {{ hostvars[host]['ansible_host'] }}:8080;
{% endfor -%}
}
```

O Ansible já usa `trim_blocks: true` por padrão (remove a quebra de linha logo após um
bloco), então na maioria dos casos o resultado sai limpo sem ajustes.

## Validando antes de aplicar

Um arquivo de configuração quebrado pode derrubar o serviço — ou trancar você para fora
do servidor. O parâmetro `validate` testa o arquivo gerado **antes** de substituir o atual
(`%s` é o arquivo temporário):

```yaml
- name: Configura o sshd
  ansible.builtin.template:
    src: sshd_config.j2
    dest: /etc/ssh/sshd_config
    mode: "0600"
    validate: /usr/sbin/sshd -t -f %s
    backup: true
```

Outros exemplos: `visudo -cf %s` (sudoers), `nginx -t -c %s` (só funciona com um
`nginx.conf` completo), `named-checkconf %s`.

`backup: true` mantém uma cópia do arquivo anterior com data e hora no nome.

## Exercícios

1. Gere o `/etc/hosts` de todos os servidores a partir do template acima.
2. Crie um template de `/etc/motd` que mostre o nome do host, o sistema, a memória em GB e
   a lista de grupos do inventário a que o host pertence.
3. Faça o template do `sshd_config` com `validate` e introduza um erro de propósito
   (`Port abc`). O que acontece?
