+++
title = "Primeiros comandos"
weight = 1
description = "ansible ping, a anatomia de um comando ad-hoc, command × shell × raw e become."
+++

## O primeiro comando

Dentro de `~/ansible` (onde estão o `ansible.cfg` e o `inventory.ini`):

```bash
ansible all -m ansible.builtin.ping
```

```text
web01 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
web02 | SUCCESS => ...
db01 | SUCCESS => ...
```

{{% notice info %}}
O módulo `ping` **não** é um ping ICMP: ele conecta via SSH, executa um pequeno programa
Python e devolve `pong`. Se funcionou, o Ansible consegue gerenciar o host.
{{% /notice %}}

## Anatomia de um comando ad-hoc

```text
ansible  web  -m ansible.builtin.apt  -a "name=nginx state=present"  -b
   │      │        │                        │                         │
   │   padrão    módulo               argumentos do módulo        become (sudo)
 comando
```

| Opção | Significado |
|---|---|
| `-m` | Módulo (padrão: `command`) |
| `-a` | Argumentos do módulo |
| `-b` / `--become` | Executa com sudo |
| `-K` | Pede a senha do sudo |
| `-u` | Usuário SSH |
| `-k` | Pede a senha do SSH (precisa do pacote `sshpass`) |
| `-i` | Inventário |
| `-f` | Forks (paralelismo) |
| `-o` | Saída em uma linha por host |
| `-v`, `-vvv` | Mais detalhes — `-vvv` mostra a conexão SSH; `-vvvv`, ainda mais |

## Nomes de módulos (FQCN)

Desde o Ansible 2.10 os módulos vivem em **collections**, e o nome completo
(*Fully Qualified Collection Name*) é `coleção.módulo`:

```text
ansible.builtin.copy       ← vem com o ansible-core
ansible.posix.firewalld    ← collection ansible.posix
community.general.ufw      ← collection community.general
```

Nos comandos ad-hoc o nome curto (`-m ping`) funciona para os módulos do `builtin`, mas
**nos playbooks use sempre o FQCN**: é inequívoco e é o que o `ansible-lint` exige.

## command, shell e raw

```bash
ansible all -a "uptime"                       # command é o módulo padrão
ansible all -a "df -h /" -o
ansible db -m ansible.builtin.shell -a "ps aux | grep -c postgres"
```

| Módulo | Diferença |
|---|---|
| `command` | Executa o programa **diretamente**, sem shell. Não entende `|`, `>`, `&&`, `$VAR`, `*`. Mais seguro. |
| `shell` | Executa via `/bin/sh`: aceita pipes, redirecionamentos e variáveis. |
| `raw` | Executa pelo SSH **sem Python** no destino. Serve para instalar o Python em máquinas que não o têm (ou equipamentos de rede). |
| `script` | Copia um script local e o executa no host remoto. |

```bash
# máquina sem Python: o raw resolve
ansible novo01 -m ansible.builtin.raw -a "apt-get install -y python3" -b
```

{{% notice warning "command e shell não são idempotentes" %}}
O Ansible não sabe o que o seu comando faz, então **sempre** reporta `changed`. Antes de
usar `command`/`shell`, procure um módulo específico: `apt`, `user`, `copy`, `lineinfile`,
`service`... Quase sempre existe um.
{{% /notice %}}

## Become: executando como root

```bash
ansible all -a "cat /etc/shadow"            # falha: Permission denied
ansible all -a "head -1 /etc/shadow" -b     # funciona: sudo
ansible all -a "whoami" -b --become-user postgres
```

## Saída e cores

| Cor | Status | Significado |
|---|---|---|
| verde | `SUCCESS` / `ok` | Tudo certo, nada mudou |
| amarelo | `CHANGED` | Algo foi alterado |
| vermelho | `FAILED` / `UNREACHABLE` | Erro na tarefa / não conseguiu conectar |
| azul/ciano | `skipping` | Tarefa pulada (condição falsa) |
| roxo | aviso | *Deprecations* e alertas |
