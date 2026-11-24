+++
title = "Deploy com releases e rollback"
weight = 1
description = "A estrutura releases/current, troca atômica, limpeza de versões antigas e rollback."
+++

## A estrutura

```text
/srv/app/
├── releases/
│   ├── 20261120-1530_v2.3.0/
│   ├── 20261122-1012_v2.4.0/
│   └── 20261123-0945_v2.4.1/     ← nova versão
├── shared/                       ← dados que sobrevivem entre versões
│   ├── .env
│   ├── uploads/
│   └── logs/
└── current -> releases/20261123-0945_v2.4.1
```

- Cada versão é extraída em um diretório **novo**, sem tocar na que está no ar.
- A troca é feita mudando o link `current` — uma operação **atômica**.
- **Rollback** é apontar `current` de volta para a release anterior.

## A aplicação de exemplo

Uma aplicação Python (Flask, FastAPI, Django — tanto faz) executada por um serviço systemd
que roda `/srv/app/current/venv/bin/gunicorn` como usuário `app`.

## A role deploy_app

```yaml
# roles/deploy_app/defaults/main.yml
app_nome: loja
app_dir: /srv/app
app_usuario: app
app_repo: https://github.com/exemplo/loja.git
app_versao: "{{ undef(hint='Informe a versão com -e app_versao=vX.Y.Z') }}"
app_releases_manter: 5
app_url_saude: http://localhost:8000/saude
```

```yaml
# roles/deploy_app/tasks/main.yml
- name: Define o nome da release
  ansible.builtin.set_fact:
    app_release: "{{ app_dir }}/releases/{{ now(fmt='%Y%m%d-%H%M') }}_{{ app_versao }}"
  run_once: true          # o mesmo nome em todos os servidores

- name: Cria os diretórios
  ansible.builtin.file:
    path: "{{ item }}"
    state: directory
    owner: "{{ app_usuario }}"
    group: "{{ app_usuario }}"
    mode: "0755"
  loop:
    - "{{ app_dir }}/releases"
    - "{{ app_dir }}/shared/uploads"
    - "{{ app_dir }}/shared/logs"

- name: Baixa o código da versão
  ansible.builtin.git:
    repo: "{{ app_repo }}"
    dest: "{{ app_release }}"
    version: "{{ app_versao }}"
    depth: 1
  become_user: "{{ app_usuario }}"

- name: Instala as dependências
  ansible.builtin.pip:
    requirements: "{{ app_release }}/requirements.txt"
    virtualenv: "{{ app_release }}/venv"
    virtualenv_command: python3 -m venv
  become_user: "{{ app_usuario }}"

- name: Liga os arquivos compartilhados
  ansible.builtin.file:
    src: "{{ app_dir }}/shared/{{ item }}"
    dest: "{{ app_release }}/{{ item }}"
    state: link
    force: true
  loop: [.env, uploads, logs]

- name: Aplica as migrações do banco (uma vez)
  ansible.builtin.command:
    cmd: "{{ app_release }}/venv/bin/python manage.py migrate --noinput"
    chdir: "{{ app_release }}"
  become_user: "{{ app_usuario }}"
  run_once: true
  register: migracao
  changed_when: "'No migrations to apply' not in migracao.stdout"

- name: Guarda a release atual (para rollback)
  ansible.builtin.stat:
    path: "{{ app_dir }}/current"
  register: release_anterior

- name: Ativa a nova versão
  ansible.builtin.file:
    src: "{{ app_release }}"
    dest: "{{ app_dir }}/current"
    state: link
  notify: Reinicia a aplicação

- name: Reinicia agora
  ansible.builtin.meta: flush_handlers

- name: Verifica a saúde da aplicação
  block:
    - name: Aguarda o health check
      ansible.builtin.uri:
        url: "{{ app_url_saude }}"
      register: saude
      until: saude.status == 200
      retries: 15
      delay: 2
  rescue:
    - name: Volta para a release anterior
      ansible.builtin.file:
        src: "{{ release_anterior.stat.lnk_source }}"
        dest: "{{ app_dir }}/current"
        state: link
      when: release_anterior.stat.exists

    - name: Reinicia com a versão anterior
      ansible.builtin.systemd_service:
        name: "{{ app_nome }}"
        state: restarted

    - name: Falha o deploy
      ansible.builtin.fail:
        msg: "Versão {{ app_versao }} não passou no health check — rollback feito."

- name: Lista as releases
  ansible.builtin.find:
    paths: "{{ app_dir }}/releases"
    file_type: directory
  register: releases

- name: Remove releases antigas
  ansible.builtin.file:
    path: "{{ item.path }}"
    state: absent
  loop: "{{ (releases.files | sort(attribute='path'))[:-app_releases_manter] }}"
  loop_control:
    label: "{{ item.path | basename }}"
```

```yaml
# roles/deploy_app/handlers/main.yml
- name: Reinicia a aplicação
  ansible.builtin.systemd_service:
    name: "{{ app_nome }}"
    state: restarted
```

## O playbook

```yaml
# playbooks/deploy.yml
- name: Deploy da aplicação
  hosts: app
  become: true
  serial: 1                    # um servidor por vez: sem downtime
  max_fail_percentage: 0       # se um falhar, os outros não são atualizados
  roles:
    - deploy_app
```

```bash
ansible-playbook playbooks/deploy.yml -e app_versao=v2.4.1
```

## Rollback manual

```yaml
# playbooks/rollback.yml
- name: Rollback para a release anterior
  hosts: app
  become: true
  serial: 1
  tasks:
    - name: Lista as releases
      ansible.builtin.find:
        paths: /srv/app/releases
        file_type: directory
      register: releases

    - name: Aponta current para a penúltima
      ansible.builtin.file:
        src: "{{ (releases.files | sort(attribute='path'))[-2].path }}"
        dest: /srv/app/current
        state: link

    - name: Reinicia
      ansible.builtin.systemd_service:
        name: loja
        state: restarted
```

{{% notice tip %}}
Em vez de clonar o Git em cada servidor, pipelines maduros geram um **artefato** (um
`.tar.gz` ou uma imagem de container) no CI e o Ansible apenas o distribui. Assim o
código que vai para produção é exatamente o que foi testado.
{{% /notice %}}
