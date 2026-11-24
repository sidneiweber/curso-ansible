+++
title = "Inventários dinâmicos"
weight = 3
description = "Plugins de inventário para nuvem e virtualização, constructed e keyed_groups."
+++

Em nuvem e virtualização, máquinas nascem e morrem o tempo todo. Manter um `hosts.ini` à mão
é inviável. **Plugins de inventário** consultam a API do provedor a cada execução.

## Exemplo: AWS

```yaml
# inventories/aws/aws_ec2.yml  (o nome precisa terminar em aws_ec2.yml)
plugin: amazon.aws.aws_ec2
regions:
  - sa-east-1
filters:
  instance-state-name: running
  tag:Projeto: loja
keyed_groups:
  - key: tags.Funcao          # grupo por tag: funcao_web, funcao_db
    prefix: funcao
  - key: placement.availability_zone
    prefix: az
hostnames:
  - tag:Name
compose:
  ansible_host: private_ip_address
```

```bash
ansible-inventory -i inventories/aws --graph
ansible-playbook -i inventories/aws playbooks/site.yml
```

## Exemplo: Proxmox

```yaml
# inventories/lab/lab.proxmox.yml
plugin: community.general.proxmox
url: https://pve.curso.lan:8006
user: ansible@pve
token_id: inventario
token_secret: "{{ lookup('ansible.builtin.env', 'PROXMOX_TOKEN') }}"
want_facts: true
keyed_groups:
  - key: proxmox_tags_parsed
    separator: ""
    prefix: ""
compose:
  ansible_host: proxmox_ipconfig0.ip | default(proxmox_net0.ip) | ansible.utils.ipaddr('address')
```

## constructed: grupos a partir de variáveis

O plugin `constructed` cria grupos a partir de variáveis de **qualquer** inventário — inclusive
estático. Basta colocá-lo no mesmo diretório:

```yaml
# inventories/producao/zz-grupos.yml
plugin: ansible.builtin.constructed
strict: false
groups:
  pouca_memoria: ansible_facts['memtotal_mb'] < 2048
  debian: ansible_facts['os_family'] == 'Debian'
keyed_groups:
  - key: ambiente
    prefix: amb
```

Um **diretório** como inventário combina todos os arquivos dentro dele (em ordem
alfabética — daí o `zz-`).

## Outros plugins

`azure.azcollection.azure_rm`, `google.cloud.gcp_compute`, `community.vmware.vmware_vm_inventory`,
`community.docker.docker_containers`, `community.general.proxmox`, `netbox.netbox.nb_inventory`
(o NetBox como fonte da verdade da infraestrutura).

```bash
ansible-doc -t inventory -l
```
