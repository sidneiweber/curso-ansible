+++
title = "AWX, Semaphore e execution environments"
weight = 4
description = "Executando Ansible como serviço: interface web, agendamentos, permissões e ambientes de execução padronizados."
+++

Rodar playbooks do notebook de alguém funciona até a equipe crescer. Aí surgem as perguntas:
**quem** executou o quê, **quando**, com qual versão do código? Quem pode rodar em produção?
Onde ficam as credenciais?

## AWX e Ansible Automation Platform

O **AWX** é o projeto de código aberto por trás do **Red Hat Ansible Automation Platform**
(AAP). Oferece:

- interface web e API REST para executar playbooks;
- **projetos** ligados a repositórios Git (sempre a versão do *commit* certo);
- **credenciais** guardadas de forma segura — quem executa não vê as senhas;
- **permissões** por equipe, inventário e *job template*;
- **agendamentos** e **workflows** (playbook A, se der certo B, senão C);
- histórico completo e logs de cada execução;
- formulários (*surveys*) para quem não usa terminal.

O AWX é instalado em Kubernetes pelo **AWX Operator** (um k3s em uma VM resolve para laboratório).

## Alternativas mais leves

- **[Semaphore UI](https://semaphoreui.com/)**: interface web simples, um binário ou container.
- **CI/CD** (GitHub Actions, GitLab CI, Jenkins): para muitas equipes, o pipeline já é a
  "plataforma de automação" — com histórico, aprovações e segredos.
- **`ansible-pull` + cron/systemd timer**: cada host baixa o repositório e se configura
  sozinho, invertendo o modelo para *pull*.

```bash
ansible-pull -U https://github.com/minha-org/config-estacoes.git local.yml
```

## Execution environments

Diferenças de versão do Ansible, das collections e das bibliotecas Python entre máquinas
causam o clássico "na minha máquina funciona". Um **execution environment** (EE) é uma
**imagem de container** com tudo isso fixado:

```yaml
# execution-environment.yml
version: 3
images:
  base_image:
    name: quay.io/fedora/fedora:42
dependencies:
  ansible_core:
    package_pip: ansible-core==2.19.*
  ansible_runner:
    package_pip: ansible-runner
  galaxy: requirements.yml
  python:
    - pywinrm
    - pymysql
```

```bash
pipx install ansible-builder ansible-navigator
ansible-builder build -t registry.curso.lan/ee-infra:1.0
ansible-navigator run playbooks/site.yml --eei registry.curso.lan/ee-infra:1.0 -m stdout
```

O mesmo EE roda no notebook (via `ansible-navigator`), no CI e no AWX — exatamente o
mesmo ambiente em todo lugar.
