+++
title = "Criando uma role"
weight = 2
description = "Refatorando o nginx do projeto em uma role parametrizável, com defaults e argument_specs."
+++

Vamos transformar a parte do nginx do [projeto web]({{% relref "/playbooks/06-projeto-web" %}})
em uma role reutilizável.

## defaults: a interface da role

Tudo que quem usa a role pode querer mudar vai em `defaults/main.yml`, **prefixado com o
nome da role**:

```yaml
# roles/nginx/defaults/main.yml
nginx_sites: []
nginx_remover_site_padrao: true
nginx_client_max_body_size: 16m
nginx_worker_connections: 1024
nginx_firewall: true
```

## tasks

```yaml
# roles/nginx/tasks/main.yml
- name: Instala o nginx
  ansible.builtin.apt:
    name: nginx
    state: present
  tags: pacotes

- name: Configuração principal
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    mode: "0644"
  notify: Recarrega o nginx

- name: Configura os sites
  ansible.builtin.template:
    src: site.conf.j2
    dest: "/etc/nginx/sites-available/{{ item.nome }}"
    mode: "0644"
  loop: "{{ nginx_sites }}"
  loop_control:
    label: "{{ item.nome }}"
  notify: Recarrega o nginx

- name: Habilita os sites
  ansible.builtin.file:
    src: "/etc/nginx/sites-available/{{ item.nome }}"
    dest: "/etc/nginx/sites-enabled/{{ item.nome }}"
    state: link
  loop: "{{ nginx_sites }}"
  loop_control:
    label: "{{ item.nome }}"
  notify: Recarrega o nginx

- name: Remove o site padrão
  ansible.builtin.file:
    path: /etc/nginx/sites-enabled/default
    state: absent
  when: nginx_remover_site_padrao
  notify: Recarrega o nginx

- name: Libera HTTP no firewall
  community.general.ufw:
    rule: allow
    name: Nginx Full
  when: nginx_firewall

- name: Garante o nginx rodando
  ansible.builtin.systemd_service:
    name: nginx
    state: started
    enabled: true
```

## handlers

```yaml
# roles/nginx/handlers/main.yml
- name: Valida a configuração do nginx
  ansible.builtin.command: nginx -t
  changed_when: false
  listen: Recarrega o nginx

- name: Recarrega o serviço nginx
  ansible.builtin.systemd_service:
    name: nginx
    state: reloaded
  listen: Recarrega o nginx
```

## templates

```jinja
{# roles/nginx/templates/site.conf.j2 #}
# {{ ansible_managed }}
server {
    listen {{ item.porta | default(80) }};
    server_name {{ item.server_name | default('_') }};
    root {{ item.root }};
    index {{ item.index | default('index.html') }};
    client_max_body_size {{ nginx_client_max_body_size }};

{% if item.php_socket is defined %}
    location / {
        try_files $uri $uri/ /index.php?$args;
    }
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:{{ item.php_socket }};
    }
{% else %}
    location / {
        try_files $uri $uri/ =404;
    }
{% endif %}
}
```

## Usando a role

```yaml
# web.yml
- name: Servidores web
  hosts: web
  become: true
  roles:
    - role: nginx
      vars:
        nginx_client_max_body_size: 64m
        nginx_sites:
          - nome: wordpress
            root: /var/www/wordpress
            index: index.php
            php_socket: /run/php/php8.4-fpm.sock
```

A mesma role agora serve para um site estático, um proxy para uma aplicação ou o WordPress.

## Validando as entradas: argument_specs

`meta/argument_specs.yml` documenta e **valida** as variáveis da role antes de qualquer tarefa
rodar — erros de digitação viram mensagens claras:

```yaml
# roles/nginx/meta/argument_specs.yml
argument_specs:
  main:
    short_description: Instala e configura o nginx
    options:
      nginx_sites:
        type: list
        elements: dict
        default: []
        description: Sites a configurar.
        options:
          nome: {type: str, required: true}
          root: {type: str, required: true}
          porta: {type: int, default: 80}
          php_socket: {type: str}
      nginx_client_max_body_size:
        type: str
        default: 16m
      nginx_firewall:
        type: bool
        default: true
```

```bash
ansible-doc -t role -r roles nginx      # a documentação gerada a partir do argument_specs
```

## Exercício

Refatore o restante do projeto em roles: `comum`, `mariadb`, `php_fpm` e `wordpress`. O
`site.yml` final deve ficar assim:

```yaml
- name: Todos os servidores
  hosts: all
  become: true
  roles: [comum]

- name: Banco de dados
  hosts: db
  become: true
  roles: [mariadb]

- name: Servidores web
  hosts: web
  become: true
  roles: [php_fpm, nginx, wordpress]
```
