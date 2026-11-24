+++
title = "Grupos dinâmicos: group_by e add_host"
weight = 5
description = "Criando grupos durante a execução a partir de facts e adicionando hosts recém-criados."
+++

O inventário nem sempre sabe tudo de antemão. Dois módulos criam grupos e hosts **durante
a execução** do playbook.

## group_by: grupos a partir de facts

Cria grupos com base em qualquer variável — muito usado para separar por sistema operacional:

```yaml
- name: Classifica os hosts
  hosts: all
  tasks:
    - name: Agrupa por distribuição e versão
      ansible.builtin.group_by:
        key: "{{ ansible_facts['distribution'] | lower }}_{{ ansible_facts['distribution_major_version'] }}"

- name: Só no Debian 13
  hosts: debian_13
  tasks:
    - name: Mostra
      ansible.builtin.debug:
        msg: "{{ inventory_hostname }} é Debian 13"

- name: Só no Ubuntu 24
  hosts: ubuntu_24
  tasks:
    - name: Mostra
      ansible.builtin.debug:
        msg: "{{ inventory_hostname }} é Ubuntu 24"
```

Em vez de encher as tarefas de `when: ansible_facts['distribution'] == ...`, cada play
mira diretamente o grupo certo.

## add_host: hosts que acabaram de nascer

Ao **criar** uma máquina no meio do playbook (na nuvem, no Proxmox, um container), ela não
está no inventário. O `add_host` a coloca no inventário **em memória**, para que as plays
seguintes possam configurá-la:

```yaml
- name: Cria a instância
  hosts: localhost
  gather_facts: false
  tasks:
    - name: Cria a VM no provedor
      # módulo do seu provedor: amazon.aws.ec2_instance, community.general.proxmox_kvm,
      # google.cloud.gcp_compute_instance, azure.azcollection.azure_rm_virtualmachine...
      amazon.aws.ec2_instance:
        name: web03
        instance_type: t3.micro
        image_id: ami-xxxxxxxx
        key_name: curso
        wait: true
      register: vm

    - name: Adiciona a nova VM ao grupo web
      ansible.builtin.add_host:
        name: web03
        ansible_host: "{{ vm.instances[0].public_ip_address }}"
        groups: [web, novos]

- name: Aguarda o SSH das máquinas novas
  hosts: novos
  gather_facts: false
  tasks:
    - name: Espera a conexão
      ansible.builtin.wait_for_connection:
        timeout: 300

- name: Configura como servidor web
  ansible.builtin.import_playbook: web.yml
```

O fluxo é sempre o mesmo: **criar → `add_host` → `wait_for_connection` → configurar**.

{{% notice tip "Inventário dinâmico" %}}
Para infraestruturas em nuvem, em vez de `add_host`, o normal é usar um **plugin de
inventário dinâmico**, que consulta a API do provedor e monta os grupos a cada execução.
Veja [escala]({{% relref "/avancado/06-escala" %}}).
{{% /notice %}}

## Exercícios do projeto

1. Execute `ansible-playbook site.yml` a partir do snapshot `base` e acesse
   `http://192.168.56.21` no navegador. Conclua a instalação do WordPress.
2. Acesse `http://192.168.56.22`: o site já está instalado? Por quê?
3. Execute `site.yml` de novo e confirme `changed=0`.
4. Altere `mariadb_max_connections` e aplique só a configuração do banco usando `--limit` e
   confira que o handler reiniciou o MariaDB.
5. Crie `host_vars/web02.yml` mudando `client_max_body_size` para `128m` (dica: transforme o
   valor em variável com `default`).
6. Adicione uma play com `group_by` por `ansible_facts['os_family']` e use-a para instalar
   `apache2-utils` só nos hosts da família Debian.
