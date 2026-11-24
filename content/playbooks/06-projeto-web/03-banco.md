+++
title = "Banco de dados"
weight = 3
description = "MariaDB com configuração por template, banco, usuário e regra de firewall."
+++

## db.yml

```yaml
- name: Servidor de banco de dados
  hosts: db
  become: true

  tasks:
    - name: Instala o MariaDB
      ansible.builtin.apt:
        name: mariadb-server
        state: present
      tags: pacotes

    - name: Configura o MariaDB
      ansible.builtin.template:
        src: templates/50-server.cnf.j2
        dest: /etc/mysql/mariadb.conf.d/50-server.cnf
        owner: root
        group: root
        mode: "0644"
      notify: Reinicia o MariaDB

    - name: Garante o MariaDB rodando
      ansible.builtin.systemd_service:
        name: mariadb
        state: started
        enabled: true

    - name: Cria o banco da aplicação
      community.mysql.mysql_db:
        name: "{{ wp_db_nome }}"
        state: present
        login_unix_socket: /run/mysqld/mysqld.sock

    - name: Cria o usuário da aplicação para cada servidor web
      community.mysql.mysql_user:
        name: "{{ wp_db_usuario }}"
        host: "{{ hostvars[item]['ansible_host'] }}"
        password: "{{ wp_db_senha }}"
        priv: "{{ wp_db_nome }}.*:ALL"
        state: present
        login_unix_socket: /run/mysqld/mysqld.sock
      loop: "{{ groups['web'] }}"
      no_log: true

    - name: Libera o MariaDB para os servidores web
      community.general.ufw:
        rule: allow
        port: "3306"
        proto: tcp
        from_ip: "{{ hostvars[item]['ansible_host'] }}"
      loop: "{{ groups['web'] }}"
      tags: firewall

  handlers:
    - name: Reinicia o MariaDB
      ansible.builtin.systemd_service:
        name: mariadb
        state: restarted
```

Destaques:

- **`login_unix_socket`**: no Debian, o root do MariaDB se autentica pelo *socket* Unix
  (como o root do sistema), então não precisamos de senha de root.
- **`loop: "{{ groups['web'] }}"`**: um usuário e uma regra de firewall para **cada**
  servidor web. Adicionou `web03` ao inventário? Ele ganha acesso na próxima execução.
- **`no_log: true`**: impede que a senha apareça na saída e nos logs.

## templates/50-server.cnf.j2

```jinja
# {{ ansible_managed }}
[server]

[mysqld]
user                    = mysql
pid-file                = /run/mysqld/mysqld.pid
basedir                 = /usr
datadir                 = /var/lib/mysql
tmpdir                  = /tmp
bind-address            = {{ mariadb_bind_address }}
max_connections         = {{ mariadb_max_connections }}
innodb_buffer_pool_size = {{ (ansible_facts['memtotal_mb'] * 0.5) | int }}M
character-set-server    = utf8mb4
collation-server        = utf8mb4_general_ci

[embedded]
[mariadb]
```

O `innodb_buffer_pool_size` é calculado como metade da memória do servidor — o mesmo
template serve para uma VM de 1 GB e para um servidor de 64 GB.

## Testando

```bash
ansible-playbook db.yml
ansible db01 -b -m ansible.builtin.command -a "mariadb -e 'SELECT user, host FROM mysql.user'"
```
