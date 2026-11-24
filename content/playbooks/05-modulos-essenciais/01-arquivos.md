+++
title = "Arquivos e configurações"
weight = 1
description = "copy, file, lineinfile, blockinfile, replace, get_url, unarchive, synchronize e stat."
+++

## copy

```yaml
- name: Copia um arquivo (de files/ ao lado do playbook)
  ansible.builtin.copy:
    src: files/chrony.conf
    dest: /etc/chrony/chrony.conf
    owner: root
    group: root
    mode: "0644"
    backup: true

- name: Copia um diretório inteiro (com / no final: só o conteúdo)
  ansible.builtin.copy:
    src: files/site/
    dest: /var/www/html/

- name: Copia um arquivo que já está no host remoto
  ansible.builtin.copy:
    src: /etc/nginx/nginx.conf
    dest: /root/nginx.conf.orig
    remote_src: true
```

{{% notice tip "Permissões sempre entre aspas" %}}
Escreva `mode: "0644"` (string). Sem aspas e sem o zero, `644` vira o número decimal 644 e
gera permissões absurdas. Também é possível usar o formato simbólico: `mode: u=rw,g=r,o=r`.
{{% /notice %}}

## file

```yaml
- name: Diretório
  ansible.builtin.file:
    path: /srv/app/releases
    state: directory
    owner: deploy
    group: deploy
    mode: "0755"

- name: Link simbólico
  ansible.builtin.file:
    src: /srv/app/releases/2.4.1
    dest: /srv/app/current
    state: link

- name: Remove arquivo ou diretório
  ansible.builtin.file:
    path: /tmp/instalador
    state: absent

- name: Arquivo vazio (sem mudar a data se já existir)
  ansible.builtin.file:
    path: /var/log/app.log
    state: touch
    modification_time: preserve
    access_time: preserve
```

## lineinfile

Garante que **uma linha** exista (ou não) em um arquivo. Ideal para ajustes pontuais em
arquivos que você não quer gerenciar por inteiro:

```yaml
- name: Desabilita login de root via SSH
  ansible.builtin.lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^#?PermitRootLogin'
    line: PermitRootLogin no
    validate: /usr/sbin/sshd -t -f %s
  notify: Recarrega o sshd

- name: Remove uma linha
  ansible.builtin.lineinfile:
    path: /etc/hosts
    regexp: 'servidor-antigo'
    state: absent
```

`regexp` encontra a linha a substituir; se nada casar, `line` é adicionada no fim do arquivo
(ou onde indicarem `insertafter`/`insertbefore`).

## blockinfile

Gerencia um **bloco** de várias linhas, delimitado por marcadores:

```yaml
- name: Adiciona os servidores do laboratório ao /etc/hosts
  ansible.builtin.blockinfile:
    path: /etc/hosts
    block: |
      192.168.56.21 web01
      192.168.56.22 web02
      192.168.56.31 db01
```

```text
# BEGIN ANSIBLE MANAGED BLOCK
192.168.56.21 web01
...
# END ANSIBLE MANAGED BLOCK
```

Use `marker: "# {mark} bloco do curso"` para ter mais de um bloco no mesmo arquivo.

## replace

Substitui **todas** as ocorrências de uma expressão regular:

```yaml
- name: Troca o espelho do Debian
  ansible.builtin.replace:
    path: /etc/apt/sources.list.d/debian.sources
    regexp: 'deb\.debian\.org'
    replace: 'debian.c3sl.ufpr.br'
```

{{% notice tip "template > lineinfile" %}}
Se você está usando muitos `lineinfile` no mesmo arquivo, é sinal de que ele deveria ser
um **template**: o arquivo inteiro fica descrito em um lugar só e não há surpresas com
linhas que alguém adicionou à mão.
{{% /notice %}}

## get_url e unarchive

```yaml
- name: Baixa o node_exporter
  ansible.builtin.get_url:
    url: https://github.com/prometheus/node_exporter/releases/download/v1.9.1/node_exporter-1.9.1.linux-amd64.tar.gz
    dest: /tmp/node_exporter.tar.gz
    checksum: sha256:https://github.com/prometheus/node_exporter/releases/download/v1.9.1/sha256sums.txt
    mode: "0644"

- name: Extrai
  ansible.builtin.unarchive:
    src: /tmp/node_exporter.tar.gz
    dest: /opt/
    remote_src: true                # o arquivo já está no host remoto
    creates: /opt/node_exporter-1.9.1.linux-amd64
```

Sem `remote_src`, o `unarchive` copia o arquivo do control node antes de extrair.

## synchronize

Usa o **rsync** — muito mais rápido que o `copy` para diretórios grandes:

```yaml
- name: Sincroniza o site
  ansible.posix.synchronize:
    src: site/
    dest: /var/www/html/
    delete: true                   # remove no destino o que não existe na origem
    rsync_opts:
      - "--exclude=.git"
```

## stat e find

```yaml
- name: Informações do arquivo
  ansible.builtin.stat:
    path: /etc/nginx/nginx.conf
    checksum_algorithm: sha256
  register: conf

- name: Logs com mais de 30 dias
  ansible.builtin.find:
    paths: /var/log/app
    patterns: "*.log"
    age: 30d
  register: logs_antigos

- name: Remove
  ansible.builtin.file:
    path: "{{ item.path }}"
    state: absent
  loop: "{{ logs_antigos.files }}"
  loop_control:
    label: "{{ item.path }}"
```
