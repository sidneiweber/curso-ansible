+++
title = "Variáveis de inventário"
weight = 2
description = "Variáveis de conexão, variáveis de host e de grupo, e os diretórios host_vars e group_vars."
+++

## Variáveis de conexão

Variáveis que começam com `ansible_` mudam **como** o Ansible se conecta ao host:

| Variável | Para que serve |
|---|---|
| `ansible_host` | Endereço real (IP ou nome) quando difere do nome no inventário. |
| `ansible_port` | Porta SSH (padrão 22). |
| `ansible_user` | Usuário da conexão. |
| `ansible_ssh_private_key_file` | Chave privada a usar. |
| `ansible_connection` | Tipo de conexão: `ssh` (padrão), `local`, `winrm`, `psrp`, `community.docker.docker`... |
| `ansible_python_interpreter` | Caminho do Python no managed node. |
| `ansible_become` | Usar sudo por padrão neste host. |

```ini
[web]
web01 ansible_host=192.168.56.21
web02 ansible_host=192.168.56.22 ansible_port=2222

[local]
localhost ansible_connection=local
```

## Variáveis de grupo no próprio inventário

```ini
[web:vars]
http_port=80
ambiente=producao
```

Isso funciona, mas o inventário fica bagunçado rapidamente. O formato INI também trata
tudo como string. A forma recomendada são os diretórios abaixo.

## host_vars e group_vars

O Ansible carrega automaticamente arquivos YAML de dois diretórios **ao lado do inventário**
(ou ao lado do playbook):

```text
~/ansible/
├── ansible.cfg
├── inventory.ini
├── group_vars/
│   ├── all.yml          # vale para todos os hosts
│   ├── web.yml          # vale para o grupo web
│   └── db.yml
└── host_vars/
    ├── web01.yml        # só para web01
    └── db01/            # também pode ser um diretório com vários arquivos
        ├── main.yml
        └── vault.yml
```

```yaml
# group_vars/all.yml
ntp_servers:
  - a.ntp.br
  - b.ntp.br
timezone: America/Sao_Paulo
```

```yaml
# group_vars/web.yml
http_port: 80
pacotes_extras:
  - nginx
  - certbot
```

```yaml
# host_vars/web01.yml
http_port: 8080      # sobrescreve o valor do grupo só para web01
```

## Quem ganha?

Quando a mesma variável aparece em vários lugares do inventário, vale a **mais específica**:

```text
group_vars/all  <  group_vars/<grupo>  <  host_vars/<host>
  (mais fraca)                              (mais forte)
```

Entre dois grupos do **mesmo nível**, vence o que vier por último em ordem alfabética
(é possível mudar isso com `ansible_group_priority`). Grupos filhos vencem os grupos pais.

```bash
ansible-inventory --host web01 --yaml
```

```yaml
ansible_host: 192.168.56.21
http_port: 8080
ntp_servers:
- a.ntp.br
- b.ntp.br
pacotes_extras:
- nginx
- certbot
timezone: America/Sao_Paulo
```

A ordem completa de precedência — inventário, playbook, linha de comando — está em
[Variáveis]({{% relref "/playbooks/02-variaveis" %}}).

{{% notice tip %}}
Mantenha o inventário só com **hosts e grupos**, e todas as variáveis em `group_vars` e
`host_vars`. Fica fácil de ler, de revisar no Git e de criptografar segredos com o Vault.
{{% /notice %}}
