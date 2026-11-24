+++
title = "Instalando o Ansible"
weight = 1
description = "ansible × ansible-core, instalação com apt, pipx e venv, e verificação."
+++

O Ansible é instalado **apenas no control node**. Os managed nodes não recebem nada.

## ansible × ansible-core

| Pacote | Conteúdo |
|---|---|
| **`ansible-core`** | O motor: os comandos (`ansible`, `ansible-playbook`...) e a collection `ansible.builtin` (módulos como `apt`, `copy`, `service`, `template`). |
| **`ansible`** | O `ansible-core` **mais** um pacote de dezenas de collections da comunidade (`community.general`, `ansible.posix`, `community.mysql`, `ansible.windows`...). |

Para o curso, instale o pacote **`ansible`** completo: todos os módulos usados já estarão
disponíveis. Em projetos reais, muita gente prefere o `ansible-core` e declara apenas as
collections necessárias em um `requirements.yml` (veja [Galaxy e collections]({{% relref "/avancado/02-galaxy" %}})).

{{% notice info "Versões" %}}
Os números de versão são diferentes: o pacote `ansible` 12 traz o `ansible-core` 2.19, por
exemplo. Quando a documentação fala em "Ansible 2.19", normalmente se refere ao **core**.
{{% /notice %}}

## Opção 1: pacote da distribuição

O mais simples. A versão é a que a distribuição congelou no lançamento, o que costuma ser
suficiente:

```bash
sudo apt update
sudo apt install ansible          # Debian/Ubuntu
sudo dnf install ansible          # Fedora/RHEL (EPEL no RHEL)
```

## Opção 2: pipx (recomendado para ter a versão mais nova)

O [pipx](https://pipx.pypa.io/) instala aplicações Python em ambientes isolados, sem
interferir no Python do sistema:

```bash
sudo apt install pipx
pipx ensurepath                   # adiciona ~/.local/bin ao PATH (abra outro terminal)
pipx install --include-deps ansible
pipx inject ansible pywinrm       # dependências extras (ex.: Windows) vão no mesmo ambiente
```

Para atualizar: `pipx upgrade --include-injected ansible`.

## Opção 3: ambiente virtual por projeto

Quando projetos diferentes precisam de versões diferentes do Ansible, use um `venv` em
cada um e fixe a versão em um `requirements.txt`:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install 'ansible==12.*' ansible-lint
```

{{% notice warning %}}
Não use `sudo pip install`. No Debian 12+ isso é bloqueado de propósito
(*externally-managed-environment*), e misturar pacotes do pip com os do apt quebra o sistema.
{{% /notice %}}

## Verificando

```bash
ansible --version
```

```text
ansible [core 2.19.x]
  config file = None
  configured module search path = ['/home/aluno/.ansible/plugins/modules', ...]
  ansible python module location = /home/aluno/.local/share/pipx/venvs/ansible/lib/python3.13/site-packages/ansible
  ansible collection location = /home/aluno/.ansible/collections:/usr/share/ansible/collections
  executable location = /home/aluno/.local/bin/ansible
  python version = 3.13.x
  jinja version = 3.1.x
```

Repare em `config file = None`: ainda não há um `ansible.cfg`. É o próximo passo.

```bash
ansible-galaxy collection list | head -20    # collections instaladas
```

## O editor

Qualquer editor serve, mas um com suporte a YAML poupa muitos erros de indentação:

- **VS Code** com a extensão oficial **Ansible** (Red Hat): autocompletar de módulos,
  documentação ao passar o mouse e `ansible-lint` integrado.
- **Vim/Neovim**: `set expandtab shiftwidth=2` e um *language server* YAML/Ansible.
