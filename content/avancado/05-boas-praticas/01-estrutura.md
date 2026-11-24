+++
title = "Estrutura do projeto e convenções"
weight = 1
description = "Inventários por ambiente, playbooks enxutos, nomes, idempotência e o checklist de um bom playbook."
+++

## Estrutura recomendada

```text
infra-ansible/
├── ansible.cfg
├── requirements.yml           # collections e roles externas, com versão
├── .ansible-lint
├── .gitignore
├── inventories/
│   ├── producao/
│   │   ├── hosts.ini
│   │   ├── group_vars/
│   │   │   ├── all/
│   │   │   │   ├── vars.yml
│   │   │   │   └── vault.yml
│   │   │   └── web.yml
│   │   └── host_vars/
│   └── homologacao/
│       ├── hosts.ini
│       └── group_vars/
├── playbooks/
│   ├── site.yml
│   ├── web.yml
│   ├── db.yml
│   └── manutencao/
│       ├── atualiza-sistema.yml
│       └── reinicia-app.yml
└── roles/
    ├── comum/
    ├── nginx/
    └── mariadb/
```

```bash
ansible-playbook -i inventories/homologacao playbooks/site.yml
ansible-playbook -i inventories/producao playbooks/site.yml --check --diff
```

Com um inventário por ambiente, **é impossível** atingir produção sem escrever
`producao` na linha de comando.

## Convenções

- **FQCN sempre:** `ansible.builtin.copy`, não `copy`.
- **Nomeie todas as plays e tarefas**, com letra maiúscula, descrevendo o estado desejado.
- **Variáveis de role prefixadas** com o nome da role (`nginx_port`).
- **`true`/`false`** para booleanos; permissões como string (`"0644"`).
- **Playbooks enxutos:** a lógica fica nas roles; o playbook diz *quais roles em quais hosts*.
- **Um arquivo de configuração = um template.** Evite dezenas de `lineinfile` no mesmo arquivo.
- **Módulos antes de `command`/`shell`.** Quando não houver jeito, use `creates`,
  `changed_when` e `failed_when`.
- **`state` explícito**, mesmo quando é o padrão: deixa a intenção clara.
- **Segredos no Vault e `no_log: true`** em tarefas que os manipulam.
- **Versões fixas** em `requirements.yml` e nos pacotes críticos.

## Checklist de um bom playbook

- [ ] Passa no `ansible-lint` sem avisos.
- [ ] A **segunda execução** termina com `changed=0`.
- [ ] Funciona com `--check` (ou as tarefas que não suportam estão marcadas com
  `check_mode: false` / `when: not ansible_check_mode`).
- [ ] Arquivos críticos usam `validate`.
- [ ] Serviços são reiniciados por **handlers**, não por tarefas.
- [ ] Nenhum segredo em texto puro no Git.
- [ ] Testado em homologação antes da produção.

## .gitignore

```text
*.retry
.vault_pass*
credenciais/
collections/
roles_externas/
.cache/
.venv/
```
