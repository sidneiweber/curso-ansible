+++
title = "Módulos na prática"
weight = 2
description = "Pacotes, serviços, arquivos, usuários, chaves SSH, facts e reinicializações via ad-hoc."
+++

Os exemplos abaixo usam os módulos mais comuns. Rode cada um **duas vezes** e compare a
saída: na primeira, `CHANGED`; na segunda, `SUCCESS` — idempotência na prática.

## Pacotes

```bash
ansible debian -b -m ansible.builtin.apt -a "update_cache=true cache_valid_time=3600"
ansible web -b -m ansible.builtin.apt -a "name=nginx state=present"
ansible web -b -m ansible.builtin.apt -a "name=nginx state=absent purge=true"
ansible debian -b -m ansible.builtin.apt -a "upgrade=dist" -f 10
```

`ansible.builtin.package` é genérico e escolhe `apt`, `dnf` etc. conforme o sistema.

## Serviços

```bash
ansible web -b -m ansible.builtin.service -a "name=nginx state=started enabled=true"
ansible web -b -m ansible.builtin.service -a "name=nginx state=restarted"
```

## Arquivos

```bash
# copia um arquivo do control node
ansible all -b -m ansible.builtin.copy -a "src=/etc/hosts dest=/etc/hosts owner=root mode=0644"

# cria conteúdo direto
ansible all -b -m ansible.builtin.copy -a "content='Servidor gerenciado pelo Ansible\n' dest=/etc/motd"

# diretórios, permissões, links e remoção
ansible web -b -m ansible.builtin.file -a "path=/srv/app state=directory owner=www-data mode=0755"
ansible web -b -m ansible.builtin.file -a "path=/tmp/lixo state=absent"

# traz arquivos dos hosts para o control node (um diretório por host)
ansible all -b -m ansible.builtin.fetch -a "src=/etc/os-release dest=coleta/"
```

## Usuários e chaves SSH

```bash
ansible all -b -m ansible.builtin.user -a "name=maria groups=sudo append=true shell=/bin/bash"

ansible all -b -m ansible.posix.authorized_key \
  -a "user=maria key='{{ lookup(\"file\", \"~/.ssh/id_ed25519.pub\") }}'"

ansible all -b -m ansible.builtin.user -a "name=maria state=absent remove=true"
```

{{% notice tip "Inicializando servidores novos" %}}
Lembra da distribuição manual de chaves no [laboratório]({{% relref "/fundamentos/02-laboratorio/02-ssh" %}})?
Com um servidor novo que só tem acesso root por senha, dá para criar o usuário de automação
com um único comando: `ansible novos -u root -k -m ansible.builtin.user -a "name=ansible"`,
seguido do `authorized_key` acima. Melhor ainda: transforme isso em um playbook de *bootstrap*.
{{% /notice %}}

## Facts: informações dos hosts

O módulo `setup` coleta centenas de informações — sistema, memória, discos, interfaces:

```bash
ansible web01 -m ansible.builtin.setup | less
ansible all -m ansible.builtin.setup -a "filter=ansible_distribution*"
ansible all -m ansible.builtin.setup -a "filter=ansible_memtotal_mb" -o
ansible all -m ansible.builtin.setup -a "gather_subset=network"
```

Esses dados, os **facts**, ficam disponíveis como variáveis nos playbooks
(veja [Facts]({{% relref "/playbooks/02-variaveis/03-facts" %}})).

## Reiniciando

```bash
ansible homologacao -b -m ansible.builtin.reboot
```

O módulo `reboot` reinicia, **espera o host voltar** e confirma que está respondendo — muito
melhor que `-a "reboot"`, que derruba a conexão e reporta erro.

## Modo de verificação

Muitos módulos suportam `--check` (simula sem alterar) e `--diff` (mostra a diferença):

```bash
ansible all -b -m ansible.builtin.copy -a "content='novo\n' dest=/etc/motd" --check --diff
```

```diff
--- before: /etc/motd
+++ after: /etc/motd
@@ -1 +1 @@
-Servidor gerenciado pelo Ansible
+novo
```

## Quando ad-hoc, quando playbook?

| Ad-hoc | Playbook |
|---|---|
| Tarefas pontuais e únicas | Tudo que vai se repetir |
| Consultas ("qual a versão do kernel?") | Configuração de servidores |
| Emergências ("reinicie o nginx em todos") | Qualquer coisa com mais de um passo |
| Não fica registrado | Fica no Git, revisado e documentado |
