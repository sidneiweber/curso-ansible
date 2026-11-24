+++
title = "Executando e lendo a saída"
weight = 2
description = "ansible-playbook, o PLAY RECAP, verbosidade, check mode, diff e execução passo a passo."
+++

## Executando

```bash
ansible-playbook basico.yml
```

```text
PLAY [Configuração básica dos servidores] **************************************

TASK [Gathering Facts] *********************************************************
ok: [web01]
ok: [db01]
ok: [web02]

TASK [Atualiza o cache do apt] *************************************************
changed: [web01]
changed: [web02]
changed: [db01]

TASK [Instala pacotes básicos] *************************************************
changed: [web02]
changed: [web01]
changed: [db01]

TASK [Define o fuso horário] ***************************************************
ok: [web01]
ok: [web02]
ok: [db01]

TASK [Cria a mensagem de boas-vindas] ******************************************
changed: [web01]
changed: [db01]
changed: [web02]

PLAY RECAP *********************************************************************
db01   : ok=5  changed=3  unreachable=0  failed=0  skipped=0  rescued=0  ignored=0
web01  : ok=5  changed=3  unreachable=0  failed=0  skipped=0  rescued=0  ignored=0
web02  : ok=5  changed=3  unreachable=0  failed=0  skipped=0  rescued=0  ignored=0
```

Execute **de novo**. Agora o `changed` deve ser `0` (o cache do apt tem validade de uma hora).
Esse é o teste de idempotência: **um bom playbook não muda nada na segunda execução**.

## O PLAY RECAP

| Campo | Significado |
|---|---|
| `ok` | Tarefas executadas com sucesso (inclui as `changed`). |
| `changed` | Tarefas que alteraram algo. |
| `unreachable` | Não conseguiu conectar — o host sai da execução. |
| `failed` | Tarefas com erro — o host sai da execução. |
| `skipped` | Tarefas puladas por um `when` falso. |
| `rescued` | Erros tratados por um bloco `rescue`. |
| `ignored` | Erros ignorados com `ignore_errors`. |

{{% notice note %}}
Quando uma tarefa falha em um host, **só aquele host** para; os outros continuam. O código de
saída do `ansible-playbook` será diferente de zero (útil em CI).
{{% /notice %}}

## Antes de executar

```bash
ansible-playbook basico.yml --syntax-check    # só valida a sintaxe
ansible-playbook basico.yml --list-hosts      # em quais hosts vai rodar
ansible-playbook basico.yml --list-tasks      # quais tarefas
ansible-lint basico.yml                       # boas práticas (pipx install ansible-lint)
```

## Check mode e diff

```bash
ansible-playbook basico.yml --check           # simula: mostra o que MUDARIA
ansible-playbook basico.yml --check --diff    # e mostra as diferenças nos arquivos
ansible-playbook basico.yml --diff            # aplica e mostra as diferenças
```

{{% notice warning "Limites do --check" %}}
Módulos sem suporte a check mode são pulados, e tarefas que dependem do resultado de
outras (um pacote que seria instalado, um `register` de um `command`) podem falhar na
simulação. O `--check` é um ótimo indicativo, não uma garantia.
{{% /notice %}}

## Controlando a execução

```bash
ansible-playbook basico.yml -l web01                  # só em web01
ansible-playbook basico.yml --start-at-task "Define o fuso horário"
ansible-playbook basico.yml --step                    # pergunta antes de cada tarefa
ansible-playbook basico.yml -v                        # resultado de cada tarefa
ansible-playbook basico.yml -vvv                      # detalhes da conexão SSH
ansible-playbook basico.yml -e "variavel=valor"       # variável extra
ansible-playbook basico.yml -K                        # pede a senha do sudo
```

## Exercícios

1. Execute o `basico.yml` duas vezes e confirme `changed=0` na segunda.
2. Altere o texto do `/etc/motd` e rode com `--check --diff`. Depois aplique.
3. Entre em `web01` e apague o `/etc/motd` manualmente. Rode o playbook e veja o Ansible
   corrigir o desvio.
4. Crie um segundo play no mesmo arquivo que instala o `nginx` apenas no grupo `web`.
