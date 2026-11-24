+++
title = "Roles e collections de terceiros"
weight = 1
description = "Ansible Galaxy, ansible-galaxy install, requirements.yml e como avaliar uma role pronta."
+++

O [Ansible Galaxy](https://galaxy.ansible.com/) é o repositório público de **roles** e
**collections**. Antes de escrever uma role do zero para algo comum (Docker, PostgreSQL,
node_exporter), vale procurar lá.

## Collections

Uma **collection** é o formato de distribuição atual: agrupa **módulos, plugins e roles**
sob um *namespace*.

```bash
ansible-galaxy collection install community.postgresql
ansible-galaxy collection install community.docker:>=4.0.0
ansible-galaxy collection list
ansible-galaxy collection install -U community.general       # atualizar
```

Collections são instaladas em `~/.ansible/collections` (ou no `collections_path` do
`ansible.cfg`). Use o conteúdo pelo FQCN:

```yaml
- name: Banco da aplicação
  community.postgresql.postgresql_db:
    name: loja

- name: Role dentro de uma collection
  ansible.builtin.include_role:
    name: geerlingguy.mac.homebrew
```

## Roles

```bash
ansible-galaxy role install geerlingguy.docker
ansible-galaxy role list
ansible-galaxy role remove geerlingguy.docker
```

```yaml
- name: Hosts de containers
  hosts: docker
  become: true
  roles:
    - role: geerlingguy.docker
      vars:
        docker_users: [deploy]
```

## requirements.yml

**Nunca** dependa do que está instalado na máquina de quem executa. Declare tudo — com
versões — em `requirements.yml`, versionado no projeto:

```yaml
# requirements.yml
collections:
  - name: community.general
    version: ">=10.0.0,<11.0.0"
  - name: community.mysql
    version: 3.14.0
  - name: ansible.posix
  - name: https://github.com/minha-org/colecao-interna.git
    type: git
    version: v1.2.0

roles:
  - name: geerlingguy.docker
    version: 7.4.7
  - name: minha_role
    src: git@github.com:minha-org/ansible-role-minha.git
    version: main
```

```bash
ansible-galaxy install -r requirements.yml
```

Para manter as dependências **dentro do projeto**, ajuste o `ansible.cfg`:

```ini
[defaults]
roles_path = ./roles:./roles_externas
collections_path = ./collections
```

e coloque `roles_externas/` e `collections/` no `.gitignore`.

## Avaliando uma role pronta

Antes de usar uma role de terceiros em produção, verifique:

- **Manutenção:** commits e releases recentes, issues respondidas.
- **Popularidade:** downloads e estrelas no GitHub.
- **Qualidade:** tem testes (Molecule, CI)? Passa no `ansible-lint`?
- **Suporte:** funciona na sua distribuição e versão?
- **Leia o código** — ela vai rodar como root nos seus servidores.

{{% notice tip %}}
Autores como **geerlingguy** (Jeff Geerling) mantêm dezenas de roles de alta qualidade que
são ótimas referências para aprender a escrever as suas.
{{% /notice %}}
