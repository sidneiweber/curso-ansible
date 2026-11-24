+++
title = "Planejamento e estrutura"
weight = 1
description = "O que vamos construir, a estrutura de diretórios e as variáveis do projeto."
+++

## O que vamos construir

```text
              navegador
                  │ http
        ┌─────────┴─────────┐
   ┌────┴────┐         ┌────┴────┐
   │  web01  │         │  web02  │   nginx + PHP-FPM + WordPress
   └────┬────┘         └────┬────┘
        └─────────┬─────────┘
                  │ 3306/tcp
             ┌────┴────┐
             │  db01   │            MariaDB
             └─────────┘
```

Começando dos servidores no snapshot **`base`** do laboratório, o comando
`ansible-playbook site.yml` deve:

1. Aplicar uma configuração **comum** a todos (pacotes, fuso horário, hosts, SSH, firewall).
2. Instalar e configurar o **MariaDB** em `db01`, criando banco e usuário da aplicação.
3. Instalar **nginx** e **PHP-FPM** em `web01` e `web02`, baixar o WordPress e gerar o
   `wp-config.php` apontando para o banco.

E, executado uma segunda vez, terminar com **`changed=0`**.

## Estrutura

```text
~/ansible/
├── ansible.cfg
├── inventory.ini
├── site.yml
├── comum.yml
├── db.yml
├── web.yml
├── group_vars/
│   ├── all.yml
│   ├── db.yml
│   └── web.yml
└── templates/
    ├── hosts.j2
    ├── 50-server.cnf.j2
    ├── nginx-wordpress.conf.j2
    └── wp-config.php.j2
```

## Inventário

```ini
# inventory.ini
[web]
web01 ansible_host=192.168.56.21
web02 ansible_host=192.168.56.22

[db]
db01 ansible_host=192.168.56.31
```

## Variáveis

```yaml
# group_vars/all.yml
timezone: America/Sao_Paulo
dominio: curso.lan
rede_lab: 192.168.56.0/24
pacotes_comuns:
  - vim
  - htop
  - curl
  - ufw
  - python3-pymysql          # necessário para os módulos community.mysql

wp_db_nome: wordpress
wp_db_usuario: wordpress
wp_db_senha: "TroqueEstaSenha!2026"   # será criptografada no capítulo de Vault
```

```yaml
# group_vars/db.yml
mariadb_bind_address: "{{ ansible_host }}"
mariadb_max_connections: 100
```

```yaml
# group_vars/web.yml
wp_versao: "6.8.3"
wp_dir: /var/www/wordpress
php_versao: "8.4"            # versão do PHP no Debian 13
wp_db_host: "{{ hostvars[groups['db'][0]]['ansible_host'] }}"
```

Repare em `wp_db_host`: em vez de repetir o IP do banco, ele é **calculado** a partir do
inventário — se o banco mudar de endereço, só o `inventory.ini` precisa ser alterado.

{{% notice warning "Senha em texto puro" %}}
A senha do banco em `group_vars/all.yml` é aceitável só no laboratório. No capítulo de
[Ansible Vault]({{% relref "/avancado/03-vault" %}}) ela será criptografada.
{{% /notice %}}

## O playbook principal

```yaml
# site.yml
- name: Configuração comum
  ansible.builtin.import_playbook: comum.yml

- name: Banco de dados
  ansible.builtin.import_playbook: db.yml

- name: Servidores web
  ansible.builtin.import_playbook: web.yml
```
