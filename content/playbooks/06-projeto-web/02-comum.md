+++
title = "Configuração comum"
weight = 2
description = "O baseline aplicado a todos os servidores: pacotes, hosts, SSH e firewall."
+++

## comum.yml

```yaml
- name: Configuração comum a todos os servidores
  hosts: all
  become: true

  tasks:
    - name: Instala os pacotes comuns
      ansible.builtin.apt:
        name: "{{ pacotes_comuns }}"
        state: present
        update_cache: true
        cache_valid_time: 3600
      tags: pacotes

    - name: Define o fuso horário
      community.general.timezone:
        name: "{{ timezone }}"

    - name: Gera o /etc/hosts a partir do inventário
      ansible.builtin.template:
        src: templates/hosts.j2
        dest: /etc/hosts
        owner: root
        group: root
        mode: "0644"

    - name: Desabilita login de root e por senha no SSH
      ansible.builtin.copy:
        dest: /etc/ssh/sshd_config.d/10-curso.conf
        content: |
          PermitRootLogin no
          PasswordAuthentication no
        mode: "0644"
        validate: /usr/sbin/sshd -t -f %s
      notify: Recarrega o sshd
      tags: ssh

    - name: Libera o SSH no firewall
      community.general.ufw:
        rule: allow
        name: OpenSSH
      tags: firewall

    - name: Ativa o firewall
      community.general.ufw:
        state: enabled
        policy: deny
      tags: firewall

  handlers:
    - name: Recarrega o sshd
      ansible.builtin.systemd_service:
        name: ssh
        state: reloaded
```

## templates/hosts.j2

```jinja
# {{ ansible_managed }}
127.0.0.1   localhost
::1         localhost ip6-localhost ip6-loopback

{% for host in groups['all'] | sort %}
{{ hostvars[host]['ansible_host'] }}  {{ host }}.{{ dominio }} {{ host }}
{% endfor %}
```

{{% notice note "validate em um arquivo parcial" %}}
O `sshd -t -f %s` valida o arquivo **sozinho**, como se fosse a configuração completa. Para
um *drop-in* simples como este funciona bem; para trechos que dependem do arquivo principal,
valide depois com um handler que execute `sshd -t`.
{{% /notice %}}

## Testando só esta parte

```bash
ansible-playbook comum.yml --check --diff
ansible-playbook comum.yml
ansible-playbook comum.yml          # segunda vez: changed=0
```
