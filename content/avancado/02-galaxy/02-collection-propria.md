+++
title = "Criando uma collection"
weight = 2
description = "Empacotando roles, módulos e plugins da equipe em uma collection versionada."
+++

Quando uma equipe tem várias roles e plugins próprios, empacotá-los em uma **collection**
facilita versionar e distribuir entre projetos.

## Estrutura

```bash
ansible-galaxy collection init sidnei.infra
```

```text
sidnei/infra/
├── galaxy.yml          # nome, versão, autor, dependências
├── README.md
├── meta/
│   └── runtime.yml     # versão mínima do ansible-core, redirecionamentos
├── plugins/
│   ├── modules/        # módulos próprios
│   ├── filter/         # filtros Jinja2
│   └── inventory/      # plugins de inventário
├── roles/
│   ├── nginx/
│   └── mariadb/
└── playbooks/          # playbooks distribuídos com a collection
```

```yaml
# galaxy.yml (trecho)
namespace: sidnei
name: infra
version: 1.0.0
authors:
  - Sidnei Weber
dependencies:
  community.general: ">=10.0.0"
  ansible.posix: ">=1.5.0"
```

## Construindo e instalando

```bash
cd sidnei/infra
ansible-galaxy collection build            # gera sidnei-infra-1.0.0.tar.gz
ansible-galaxy collection install sidnei-infra-1.0.0.tar.gz
```

Ou direto do Git, sem publicar em lugar nenhum:

```yaml
# requirements.yml de outro projeto
collections:
  - name: git+https://github.com/sidneiweber/ansible-collection-infra.git
    version: v1.0.0
```

## Usando

```yaml
- name: Servidores web
  hosts: web
  roles:
    - sidnei.infra.nginx
```

Para publicar no Galaxy público: `ansible-galaxy collection publish arquivo.tar.gz --api-key ...`.
Empresas costumam usar um **Automation Hub** privado ou simplesmente repositórios Git.
