+++
title = "Servidores web"
weight = 4
description = "nginx, PHP-FPM e WordPress: download, configuração gerada por template e handlers."
+++

## web.yml

```yaml
- name: Servidores web
  hosts: web
  become: true

  tasks:
    - name: Instala nginx e PHP
      ansible.builtin.apt:
        name:
          - nginx
          - "php{{ php_versao }}-fpm"
          - "php{{ php_versao }}-mysql"
          - "php{{ php_versao }}-gd"
          - "php{{ php_versao }}-curl"
          - "php{{ php_versao }}-xml"
          - "php{{ php_versao }}-mbstring"
          - "php{{ php_versao }}-zip"
          - "php{{ php_versao }}-intl"
        state: present
      tags: pacotes

    - name: Baixa e extrai o WordPress
      ansible.builtin.unarchive:
        src: "https://wordpress.org/wordpress-{{ wp_versao }}.tar.gz"
        dest: "{{ wp_dir | dirname }}"
        remote_src: true
        owner: www-data
        group: www-data
        creates: "{{ wp_dir }}/wp-settings.php"
      tags: wordpress

    - name: Gera o wp-config.php
      ansible.builtin.template:
        src: templates/wp-config.php.j2
        dest: "{{ wp_dir }}/wp-config.php"
        owner: www-data
        group: www-data
        mode: "0640"
      tags: [wordpress, config]

    - name: Configura o site no nginx
      ansible.builtin.template:
        src: templates/nginx-wordpress.conf.j2
        dest: /etc/nginx/sites-available/wordpress
        mode: "0644"
      notify: Recarrega o nginx
      tags: config

    - name: Habilita o site
      ansible.builtin.file:
        src: /etc/nginx/sites-available/wordpress
        dest: /etc/nginx/sites-enabled/wordpress
        state: link
      notify: Recarrega o nginx

    - name: Remove o site padrão
      ansible.builtin.file:
        path: /etc/nginx/sites-enabled/default
        state: absent
      notify: Recarrega o nginx

    - name: Libera HTTP no firewall
      community.general.ufw:
        rule: allow
        name: Nginx HTTP
      tags: firewall

    - name: Garante os serviços rodando
      ansible.builtin.systemd_service:
        name: "{{ item }}"
        state: started
        enabled: true
      loop:
        - nginx
        - "php{{ php_versao }}-fpm"

    - name: Aplica a configuração antes de testar
      ansible.builtin.meta: flush_handlers

    - name: Testa o site
      ansible.builtin.uri:
        url: "http://{{ ansible_host }}/wp-admin/install.php"
        status_code: 200
      delegate_to: localhost
      become: false
      tags: teste

  handlers:
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

O teste final roda **no control node** (`delegate_to: localhost`), acessando o site como um
navegador faria. Se o PHP não conectar ao banco, a página de instalação retorna erro e o
playbook falha — a automação verifica o próprio trabalho.

## templates/nginx-wordpress.conf.j2

```jinja
# {{ ansible_managed }}
server {
    listen 80 default_server;
    server_name {{ inventory_hostname }}.{{ dominio }} {{ ansible_host }};

    root {{ wp_dir }};
    index index.php;

    client_max_body_size 64m;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php{{ php_versao }}-fpm.sock;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

## templates/wp-config.php.j2

```jinja
<?php
// {{ ansible_managed }}
define( 'DB_NAME',     '{{ wp_db_nome }}' );
define( 'DB_USER',     '{{ wp_db_usuario }}' );
define( 'DB_PASSWORD', '{{ wp_db_senha }}' );
define( 'DB_HOST',     '{{ wp_db_host }}' );
define( 'DB_CHARSET',  'utf8mb4' );
define( 'DB_COLLATE',  '' );

{% for chave in ['AUTH_KEY', 'SECURE_AUTH_KEY', 'LOGGED_IN_KEY', 'NONCE_KEY',
                 'AUTH_SALT', 'SECURE_AUTH_SALT', 'LOGGED_IN_SALT', 'NONCE_SALT'] %}
define( '{{ chave }}', '{{ lookup('ansible.builtin.password', 'credenciais/wp_' ~ chave | lower ~ ' length=64 chars=ascii_letters,digits') }}' );
{% endfor %}

$table_prefix = 'wp_';
define( 'WP_DEBUG', false );

if ( ! defined( 'ABSPATH' ) ) {
    define( 'ABSPATH', __DIR__ . '/' );
}
require_once ABSPATH . 'wp-settings.php';
```

As chaves de segurança do WordPress são geradas pelo lookup `password` **no control node** e
guardadas em `credenciais/`. Assim:

- `web01` e `web02` recebem **as mesmas chaves** (necessário para que uma sessão iniciada em
  um servidor valha no outro);
- a segunda execução reutiliza os arquivos e o `wp-config.php` não muda.

{{% notice warning %}}
Adicione `credenciais/` ao `.gitignore` — ou, melhor, criptografe essas chaves com o
[Vault]({{% relref "/avancado/03-vault" %}}).
{{% /notice %}}
