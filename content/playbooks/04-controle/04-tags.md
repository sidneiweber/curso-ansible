+++
title = "Tags"
weight = 4
description = "Executando ou pulando partes do playbook com --tags e --skip-tags."
+++

Tags permitem executar **só uma parte** de um playbook grande — por exemplo, só atualizar a
configuração do nginx sem passar por toda a instalação.

```yaml
- name: Servidor web
  hosts: web
  become: true
  tasks:
    - name: Instala o nginx
      ansible.builtin.apt:
        name: nginx
      tags: [nginx, pacotes]

    - name: Configura o nginx
      ansible.builtin.template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: Recarrega o nginx
      tags: [nginx, config]

    - name: Publica o site
      ansible.builtin.copy:
        src: site/
        dest: /var/www/html/
      tags: deploy
```

```bash
ansible-playbook web.yml --list-tags          # tags disponíveis
ansible-playbook web.yml --tags config        # só as tarefas com a tag config
ansible-playbook web.yml --tags "nginx,deploy"
ansible-playbook web.yml --skip-tags pacotes  # tudo, menos pacotes
```

## Tags em plays, blocos e roles

Tags aplicadas a uma play, bloco ou role são **herdadas** por todas as tarefas de dentro:

```yaml
- name: Banco de dados
  hosts: db
  tags: db
  roles:
    - role: postgresql
      tags: postgres
```

## Tags especiais

| Tag | Efeito |
|---|---|
| `always` | Roda sempre, a menos que seja pulada explicitamente (`--skip-tags always`). |
| `never` | Nunca roda, a menos que seja pedida explicitamente. |
| `tagged` / `untagged` | Em `--tags`: todas as tarefas com tag / sem tag. |
| `all` | Padrão: todas as tarefas. |

`never` é ótimo para tarefas perigosas ou raras dentro de um playbook comum:

```yaml
- name: Apaga o banco e recria do zero
  community.postgresql.postgresql_db:
    name: loja
    state: absent
  tags: [never, reset-banco]
```

```bash
ansible-playbook db.yml --tags reset-banco
```

{{% notice warning %}}
Tarefas que **registram variáveis** usadas depois (`register`, `set_fact`) também precisam
estar na tag — ou usar `always` —, senão a tarefa seguinte falha com variável indefinida
quando você filtra por tags. O *Gathering Facts* sempre roda.
{{% /notice %}}
