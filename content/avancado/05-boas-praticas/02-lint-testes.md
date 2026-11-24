+++
title = "ansible-lint, Molecule e CI"
weight = 2
description = "Análise estática com ansible-lint, testes de roles com Molecule e execução em pipelines."
+++

## ansible-lint

Verifica sintaxe, boas práticas, nomes, FQCN, idempotência suspeita, permissões sem aspas e
muito mais:

```bash
pipx install ansible-lint      # ou pip install ansible-lint no venv do projeto
ansible-lint                   # analisa o projeto inteiro
ansible-lint playbooks/web.yml
ansible-lint --fix             # corrige automaticamente o que for possível
```

```text
name[casing]: All names should start with an uppercase letter.
playbooks/web.yml:12 Task/Handler: instala nginx

fqcn[action-core]: Use FQCN for builtin module actions (copy).
roles/nginx/tasks/main.yml:20 Use `ansible.builtin.copy` or `ansible.legacy.copy` instead.

no-changed-when: Commands should not change things if nothing needs doing.
roles/app/tasks/main.yml:8 Task/Handler: Executa migração
```

Configuração em `.ansible-lint`:

```yaml
profile: production        # min, basic, moderate, safety, shared, production
exclude_paths:
  - collections/
  - roles_externas/
skip_list:
  - yaml[line-length]
```

### pre-commit

Para o lint rodar **antes de cada commit**:

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/ansible/ansible-lint
    rev: v25.9.2
    hooks:
      - id: ansible-lint
```

```bash
pipx install pre-commit && pre-commit install
```

## Molecule: testando roles

O [Molecule](https://ansible.readthedocs.io/projects/molecule/) cria um ambiente
descartável (containers ou VMs), aplica a role, verifica a idempotência e destrói tudo:

```bash
pipx install molecule
pipx inject molecule molecule-plugins[podman]
cd roles/nginx
molecule init scenario --driver-name podman
```

```yaml
# roles/nginx/molecule/default/molecule.yml
driver:
  name: podman
platforms:
  - name: debian13
    image: docker.io/geerlingguy/docker-debian13-ansible:latest
    command: /lib/systemd/systemd
    privileged: true
    pre_build_image: true
provisioner:
  name: ansible
verifier:
  name: ansible
```

```yaml
# roles/nginx/molecule/default/verify.yml
- name: Verifica
  hosts: all
  gather_facts: false
  tasks:
    - name: Testa a resposta HTTP
      ansible.builtin.uri:
        url: http://localhost/
        status_code: [200, 403]
```

```bash
molecule test        # create → converge → idempotence → verify → destroy
molecule converge    # só aplica (para desenvolver)
molecule login       # entra no container
```

A etapa **idempotence** aplica a role duas vezes e **falha** se a segunda execução tiver
qualquer `changed` — o teste mais valioso de todos.

## Integração contínua

```yaml
# .github/workflows/ansible.yml
name: Ansible
on: [push, pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.13"
      - run: pip install ansible ansible-lint
      - run: ansible-galaxy install -r requirements.yml
      - run: ansible-lint
      - run: ansible-playbook playbooks/site.yml --syntax-check -i inventories/homologacao
```

Um fluxo maduro: **lint e Molecule** em todo *pull request*; após o *merge*, o pipeline
aplica em **homologação**; produção com aprovação manual — ou via AWX/AAP
(veja [escala]({{% relref "/avancado/06-escala" %}})).
