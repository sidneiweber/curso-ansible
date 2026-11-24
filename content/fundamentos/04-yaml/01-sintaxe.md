+++
title = "Sintaxe do YAML"
weight = 1
description = "Chaves e valores, listas, dicionários, strings, multilinha e as armadilhas mais comuns."
+++

**YAML** (*YAML Ain't Markup Language*) representa dados — os mesmos de um JSON — de forma
legível. Um arquivo YAML do Ansible normalmente começa com `---`.

## Regras de ouro

- A estrutura é definida pela **indentação**, com **espaços**. **Tabs são proibidos.**
- Use **2 espaços** por nível e seja consistente.
- `chave: valor` — sempre com **um espaço depois dos dois-pontos**.
- `#` inicia um comentário.

## Escalares: chave e valor

```yaml
nome: nginx
porta: 80
habilitado: true
versao: "1.24"
vazio: null       # ou ~, ou simplesmente nada
```

## Listas

Cada item começa com `- ` no mesmo nível de indentação:

```yaml
pacotes:
  - nginx
  - curl
  - git

# forma compacta (fluxo), igual ao JSON
pacotes: [nginx, curl, git]
```

## Dicionários

```yaml
usuario:
  nome: maria
  shell: /bin/bash
  grupos:
    - sudo
    - www-data

# forma compacta
usuario: {nome: maria, shell: /bin/bash}
```

## Combinando: lista de dicionários

É a estrutura mais comum no Ansible — um playbook, inclusive, é uma lista de plays, e cada
play tem uma lista de tarefas:

```yaml
usuarios:
  - nome: maria
    uid: 1001
    grupos: [sudo]
  - nome: joao
    uid: 1002
    grupos: []
```

O equivalente em JSON:

```json
{"usuarios": [
  {"nome": "maria", "uid": 1001, "grupos": ["sudo"]},
  {"nome": "joao",  "uid": 1002, "grupos": []}
]}
```

## Strings multilinha

```yaml
# | (literal) preserva as quebras de linha — ideal para scripts e arquivos
motd: |
  Bem-vindo ao servidor.
  Acesso monitorado.

# > (dobrado) junta as linhas com espaços — ideal para textos longos
descricao: >
  Esta frase longa foi quebrada em várias linhas
  apenas para ficar legível no arquivo.
```

Use `|-` ou `>-` para remover a quebra de linha final.

## Armadilhas

### Booleanos

YAML aceita muitas formas de verdadeiro e falso. No Ansible, padronize em **`true`/`false`**
(é o que o `ansible-lint` exige):

```yaml
enabled: true      # bom
enabled: yes       # funciona, mas evite
```

### Aspas

Valores que **parecem** outro tipo precisam de aspas para serem strings:

```yaml
versao: 1.10        # vira o número 1.1 !
versao: "1.10"      # string "1.10"
modo: 0644          # octal → número 420; no Ansible use sempre "0644"
cep: "01310-100"
pais: "NO"          # sem aspas, alguns parsers YAML antigos leem como false
```

### Dois-pontos e chaves de Jinja

Um valor que **começa** com `{{` precisa de aspas, senão o YAML acha que é um dicionário:

```yaml
arquivo: {{ caminho }}/app.conf      # ERRO de sintaxe
arquivo: "{{ caminho }}/app.conf"    # correto
```

E texto com `: ` no meio também:

```yaml
mensagem: "Atenção: reinício agendado"
```

## Validando

```bash
python3 -c 'import sys, yaml; yaml.safe_load(open(sys.argv[1]))' arquivo.yml
yamllint arquivo.yml                    # apt install yamllint
ansible-playbook --syntax-check site.yml
```

{{% notice tip %}}
Configure o editor para mostrar espaços em branco e converter tabs em espaços. Metade dos
erros "misteriosos" do Ansible é indentação.
{{% /notice %}}
