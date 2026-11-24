+++
title = "O arquivo ansible.cfg"
weight = 2
description = "Onde o Ansible procura a configuração, as opções mais usadas e o projeto do curso."
+++

## Onde o Ansible procura a configuração

O Ansible usa o **primeiro** arquivo que encontrar, nesta ordem:

1. A variável de ambiente `ANSIBLE_CONFIG` (caminho para um arquivo).
2. **`./ansible.cfg`** — no diretório atual.
3. `~/.ansible.cfg` — no diretório pessoal.
4. `/etc/ansible/ansible.cfg` — global.

{{% notice tip %}}
Prefira sempre o **`ansible.cfg` no diretório do projeto**, versionado no Git junto com
os playbooks. Assim quem clonar o repositório tem exatamente a mesma configuração.
{{% /notice %}}

{{% notice warning %}}
Por segurança, o Ansible **ignora** o `ansible.cfg` do diretório atual se ele tiver
permissão de escrita para todos (*world-writable*) — situação comum em pastas
compartilhadas do VirtualBox ou do WSL (`/mnt/c/...`).
{{% /notice %}}

## O projeto do curso

Crie o diretório que será usado em todos os capítulos:

```bash
mkdir -p ~/ansible && cd ~/ansible
git init
```

E o arquivo `ansible.cfg`:

```ini
# ~/ansible/ansible.cfg
[defaults]
inventory = inventory.ini
remote_user = ansible
forks = 10
interpreter_python = auto_silent
stdout_callback = default
result_format = yaml
retry_files_enabled = False

[privilege_escalation]
become_method = sudo

[ssh_connection]
pipelining = True
```

| Opção | Para que serve |
|---|---|
| `inventory` | Inventário padrão — dispensa o `-i` em todo comando. |
| `remote_user` | Usuário SSH nos managed nodes. |
| `forks` | Quantos hosts são atendidos em paralelo (padrão: 5). |
| `interpreter_python` | `auto_silent` descobre o Python do host sem mostrar aviso. |
| `result_format = yaml` | Mostra os resultados em YAML, bem mais legível que JSON. |
| `retry_files_enabled` | Não cria arquivos `.retry` após falhas. |
| `become_method` | Como escalar privilégio (`sudo`, `su`, `doas`...). |
| `pipelining` | Envia o módulo pela conexão SSH já aberta, sem copiar arquivos: bem mais rápido. Exige que o sudoers **não** tenha `requiretty` (o padrão no Debian). |

## Inspecionando a configuração

```bash
ansible --version | grep 'config file'     # qual arquivo está em uso
ansible-config dump --only-changed         # o que difere do padrão
ansible-config list | less                 # todas as opções, com descrição
ansible-config init --disabled > exemplo.cfg   # arquivo completo, tudo comentado
```

A saída de `ansible-config dump --only-changed` mostra também **de onde** cada valor veio:

```text
CONFIG_FILE() = /home/aluno/ansible/ansible.cfg
DEFAULT_FORKS(/home/aluno/ansible/ansible.cfg) = 10
DEFAULT_REMOTE_USER(/home/aluno/ansible/ansible.cfg) = ansible
...
```

## Variáveis de ambiente

Toda opção tem uma variável de ambiente equivalente, útil para mudar algo pontualmente:

```bash
ANSIBLE_FORKS=30 ansible-playbook site.yml
ANSIBLE_STDOUT_CALLBACK=json ansible-playbook site.yml > resultado.json
```
