+++
title = "Pacotes, serviços e sistema"
weight = 2
description = "apt, repositórios, systemd, usuários e grupos, cron, firewall, montagens e git."
+++

## Pacotes

```yaml
- name: Instala pacotes
  ansible.builtin.apt:
    name: [nginx, git, curl]
    state: present
    update_cache: true
    cache_valid_time: 3600

- name: Versão específica
  ansible.builtin.apt:
    name: postgresql-17
    state: present

- name: Sempre a mais nova
  ansible.builtin.apt:
    name: openssl
    state: latest

- name: Atualiza tudo e limpa
  ansible.builtin.apt:
    upgrade: dist
    autoremove: true
    autoclean: true

- name: Instala um .deb baixado
  ansible.builtin.apt:
    deb: https://exemplo.com/pacote_1.0_amd64.deb
```

| `state` | Significado |
|---|---|
| `present` | Instala se não estiver instalado (não atualiza). **Use por padrão.** |
| `latest` | Instala ou atualiza para a versão mais nova — cuidado em produção. |
| `absent` | Remove. |

Para playbooks que rodam em várias distribuições, `ansible.builtin.package` escolhe o
gerenciador certo — mas os nomes dos pacotes podem variar (`apache2` × `httpd`).

## Repositórios

Repositórios no formato **deb822**, padrão do Debian 13:

```yaml
- name: Repositório oficial do PostgreSQL
  ansible.builtin.deb822_repository:
    name: pgdg
    types: deb
    uris: https://apt.postgresql.org/pub/repos/apt
    suites: "{{ ansible_facts['distribution_release'] }}-pgdg"
    components: main
    signed_by: https://www.postgresql.org/media/keys/ACCC4CF8.asc

- name: Atualiza o cache depois de adicionar o repositório
  ansible.builtin.apt:
    update_cache: true
```

{{% notice note %}}
O módulo antigo `apt_repository` e o `apt_key` (que usava o `apt-key`, removido do Debian)
estão obsoletos. Use `deb822_repository` com `signed_by`.
{{% /notice %}}

## Serviços

```yaml
- name: Garante o serviço rodando e habilitado
  ansible.builtin.systemd_service:
    name: nginx
    state: started
    enabled: true

- name: Recarrega o systemd após criar uma unit
  ansible.builtin.systemd_service:
    daemon_reload: true

- name: Mascara um serviço
  ansible.builtin.systemd_service:
    name: apache2
    masked: true
```

`ansible.builtin.service` é a versão genérica (funciona com outros sistemas de init).
Para coletar o estado de todos os serviços: `ansible.builtin.service_facts`.

## Usuários e grupos

```yaml
- name: Grupo
  ansible.builtin.group:
    name: deploy
    gid: 2000

- name: Usuário
  ansible.builtin.user:
    name: deploy
    uid: 2000
    group: deploy
    groups: [www-data]
    append: true
    shell: /bin/bash
    password: "{{ senha_deploy | password_hash('sha512') }}"
    update_password: on_create
    generate_ssh_key: true

- name: Chave SSH autorizada
  ansible.posix.authorized_key:
    user: deploy
    key: "{{ lookup('ansible.builtin.file', 'chaves/deploy.pub') }}"
    exclusive: false
```

{{% notice warning %}}
Sem `append: true`, `groups` **substitui** todos os grupos secundários do usuário — é fácil
remover alguém do grupo `sudo` por engano.
{{% /notice %}}

## sudoers

```yaml
- name: Permite o deploy reiniciar a aplicação
  community.general.sudoers:
    name: deploy-app
    user: deploy
    commands: /usr/bin/systemctl restart app
    nopassword: true
```

## Agendamentos

```yaml
- name: Backup diário
  ansible.builtin.cron:
    name: backup do banco           # identifica a entrada; não mude depois
    user: postgres
    minute: "30"
    hour: "2"
    job: /usr/local/bin/backup.sh > /dev/null 2>&1
```

## Firewall

```yaml
- name: Libera SSH e HTTP
  community.general.ufw:
    rule: allow
    port: "{{ item }}"
    proto: tcp
  loop: ["22", "80", "443"]

- name: Ativa o ufw com política de negar
  community.general.ufw:
    state: enabled
    policy: deny
```

{{% notice warning %}}
**Sempre** libere a porta SSH **antes** de ativar o firewall, ou o Ansible (e você) perderá
o acesso ao servidor.
{{% /notice %}}

## Montagens e kernel

```yaml
- name: Monta um compartilhamento NFS
  ansible.posix.mount:
    src: 192.168.56.31:/srv/nfs/dados
    path: /mnt/dados
    fstype: nfs
    opts: defaults,_netdev
    state: mounted               # monta e grava no fstab

- name: Habilita o roteamento
  ansible.posix.sysctl:
    name: net.ipv4.ip_forward
    value: "1"
    sysctl_file: /etc/sysctl.d/90-roteamento.conf
```

## Git

```yaml
- name: Clona a aplicação
  ansible.builtin.git:
    repo: https://github.com/exemplo/app.git
    dest: /srv/app
    version: v2.4.1           # tag, branch ou commit
    force: false
```

## Reinicialização e espera

```yaml
- name: Reinicia e aguarda voltar
  ansible.builtin.reboot:
    reboot_timeout: 600

- name: Aguarda a porta do banco
  ansible.builtin.wait_for:
    host: 192.168.56.31
    port: 5432
    timeout: 60
```
