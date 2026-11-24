+++
title = "Acesso SSH e sudo"
weight = 2
description = "Usuário de automação, chaves SSH, sudo sem senha e o primeiro teste de conexão."
+++

O Ansible acessa os managed nodes por **SSH**, com um usuário que pode usar **sudo**.
Vamos preparar isso uma vez em cada servidor.

## O usuário de automação

Em produção, crie um usuário dedicado (`ansible`, `deploy`, `automacao`). Assim fica claro
nos logs o que foi feito pela automação e o acesso pode ser revogado sem afetar pessoas.

Em **cada managed node** (`web01`, `web02`, `db01`), como root:

```bash
adduser --disabled-password --gecos "" ansible
echo 'ansible ALL=(ALL) NOPASSWD: ALL' > /etc/sudoers.d/ansible
chmod 440 /etc/sudoers.d/ansible
visudo -cf /etc/sudoers.d/ansible     # valida a sintaxe
apt install -y python3                # normalmente já vem instalado
```

{{% notice note "Sudo sem senha?" %}}
É o mais prático em laboratório e muito comum em produção, já que o acesso é protegido pela
chave SSH. Se a política da empresa exigir senha no sudo, use `--ask-become-pass` (`-K`)
na hora de executar, ou guarde a senha no [Vault]({{% relref "/avancado/03-vault" %}}).
{{% /notice %}}

## A chave SSH

No **control node** (`ansible01`), com o seu usuário normal:

```bash
ssh-keygen -t ed25519 -C "ansible@curso.lan"
```

Copie a chave pública para o usuário `ansible` de cada servidor. Como o usuário não tem
senha, faça isso como root no managed node, ou use um usuário que já tenha acesso:

```bash
# no managed node, como root
install -d -m 700 -o ansible -g ansible /home/ansible/.ssh
echo "ssh-ed25519 AAAA... ansible@curso.lan" >> /home/ansible/.ssh/authorized_keys
chown ansible: /home/ansible/.ssh/authorized_keys
chmod 600 /home/ansible/.ssh/authorized_keys
```

{{% notice tip %}}
Repetir isso em cada servidor à mão é exatamente o tipo de trabalho que o Ansible elimina.
No capítulo de [comandos ad-hoc]({{% relref "/fundamentos/06-ad-hoc" %}}) você vai
distribuir chaves com o módulo `ansible.posix.authorized_key`; aqui fazemos manualmente só
para dar o primeiro passo.
{{% /notice %}}

## Testando

Do control node, **conecte-se uma vez em cada host** — isso também grava a chave de host em
`~/.ssh/known_hosts`:

```bash
for h in web01 web02 db01; do ssh ansible@$h 'hostname; sudo -n true && echo sudo OK'; done
```

A saída esperada:

```text
web01
sudo OK
web02
sudo OK
db01
sudo OK
```

{{% notice warning "Host key checking" %}}
Na primeira conexão o SSH pergunta se você confia na chave do servidor. O Ansible respeita
essa verificação e **falha** se a chave for desconhecida. Desligá-la
(`host_key_checking = False`) é tentador, mas abre espaço para ataques *man-in-the-middle*;
prefira popular o `known_hosts` com `ssh-keyscan` ou conectando uma vez, como acima.
{{% /notice %}}

## Snapshot

Com tudo funcionando, tire um snapshot de todas as VMs chamado **`base`**. Sempre que um
exercício bagunçar o ambiente, volte para ele.
