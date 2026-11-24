+++
title = "Escrevendo um módulo"
weight = 1
description = "Um módulo Python idempotente com AnsibleModule, check mode, diff e documentação."
+++

Um módulo é um programa que recebe parâmetros em JSON, faz o trabalho no managed node e
devolve JSON. O Ansible fornece a classe `AnsibleModule`, que cuida da validação dos
parâmetros, do check mode e da saída.

## Quando escrever um módulo

- A mesma combinação de `command` + `register` + `changed_when` se repete em vários lugares.
- É preciso consultar uma API ou um formato de arquivo que nenhum módulo suporta.
- A lógica em YAML/Jinja ficou ilegível.

## O exemplo: módulo `motd_linha`

Garante que uma mensagem exista (ou não) no `/etc/motd`, com prioridade de ordenação.
Crie `library/motd_linha.py` ao lado do playbook (ou `plugins/modules/` em uma collection):

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-

DOCUMENTATION = r"""
module: motd_linha
short_description: Gerencia mensagens no /etc/motd
description:
  - Garante que uma mensagem esteja presente ou ausente no arquivo de motd.
options:
  mensagem:
    description: Texto da mensagem.
    type: str
    required: true
  state:
    description: Se a mensagem deve existir.
    type: str
    choices: [present, absent]
    default: present
  path:
    description: Arquivo a gerenciar.
    type: path
    default: /etc/motd
author: Sidnei Weber
"""

EXAMPLES = r"""
- name: Aviso de manutenção
  motd_linha:
    mensagem: "Manutenção programada: sábado, 22h"
"""

RETURN = r"""
linhas:
  description: Conteúdo final do arquivo, linha a linha.
  type: list
  returned: always
"""

import os

from ansible.module_utils.basic import AnsibleModule


def ler_linhas(path):
    if not os.path.exists(path):
        return []
    with open(path, encoding="utf-8") as f:
        return f.read().splitlines()


def main():
    module = AnsibleModule(
        argument_spec=dict(
            mensagem=dict(type="str", required=True),
            state=dict(type="str", choices=["present", "absent"], default="present"),
            path=dict(type="path", default="/etc/motd"),
        ),
        supports_check_mode=True,
    )

    mensagem = module.params["mensagem"]
    state = module.params["state"]
    path = module.params["path"]

    antes = ler_linhas(path)
    depois = list(antes)

    if state == "present" and mensagem not in depois:
        depois.append(mensagem)
    elif state == "absent" and mensagem in depois:
        depois = [linha for linha in depois if linha != mensagem]

    changed = antes != depois
    resultado = dict(changed=changed, linhas=depois)

    if module._diff:
        resultado["diff"] = dict(
            before="\n".join(antes) + "\n",
            after="\n".join(depois) + "\n",
            before_header=path,
            after_header=path,
        )

    if changed and not module.check_mode:
        try:
            with open(path, "w", encoding="utf-8") as f:
                f.write("\n".join(depois) + "\n")
        except OSError as e:
            module.fail_json(msg=f"Erro ao gravar {path}: {e}", **resultado)

    module.exit_json(**resultado)


if __name__ == "__main__":
    main()
```

Os pontos que tornam o módulo um **bom** módulo:

| Prática | No código |
|---|---|
| **Idempotência** | Lê o estado atual e só escreve se `antes != depois`. |
| **`changed` correto** | Calculado a partir da comparação, não fixo. |
| **Check mode** | `supports_check_mode=True` e nada é gravado quando `module.check_mode`. |
| **Diff** | Preenche `diff` quando executado com `--diff`. |
| **Erros claros** | `fail_json(msg=...)` em vez de deixar a exceção estourar. |
| **Documentação** | `DOCUMENTATION`, `EXAMPLES` e `RETURN` — lidos pelo `ansible-doc`. |

## Usando

```yaml
- name: Mensagens do sistema
  hosts: all
  become: true
  tasks:
    - name: Aviso de manutenção
      motd_linha:
        mensagem: "Manutenção programada: sábado, 22h"

    - name: Remove aviso antigo
      motd_linha:
        mensagem: "Servidor em migração"
        state: absent
```

```bash
ansible-doc -M library motd_linha
ansible-playbook motd.yml --check --diff
```

## Testando rapidamente

O módulo pode ser executado direto no control node, passando os argumentos em JSON:

```bash
echo '{"ANSIBLE_MODULE_ARGS": {"mensagem": "teste", "path": "/tmp/motd"}}' > args.json
python3 library/motd_linha.py args.json
```

```json
{"changed": true, "linhas": ["teste"], "invocation": {"module_args": {...}}}
```

Execute de novo: `"changed": false`.

{{% notice tip %}}
Em uma collection, os testes vão em `tests/unit/` (pytest) e `tests/integration/`, e são
executados com `ansible-test units` e `ansible-test integration`.
{{% /notice %}}
