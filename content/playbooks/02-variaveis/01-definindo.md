+++
title = "Definindo e usando variáveis"
weight = 1
description = "vars, vars_files, group_vars, -e, tipos de dados e a sintaxe {{ }}."
+++

## Usando: `{{ variavel }}`

Variáveis são usadas com a sintaxe do **Jinja2**, entre chaves duplas:

```yaml
- name: Servidor web
  hosts: web
  become: true
  vars:
    pacote_web: nginx
    http_port: 80

  tasks:
    - name: "Instala o {{ pacote_web }}"
      ansible.builtin.apt:
        name: "{{ pacote_web }}"
        state: present

    - name: Mostra a porta
      ansible.builtin.debug:
        msg: "O {{ pacote_web }} vai escutar na porta {{ http_port }}"
```

{{% notice warning "Aspas" %}}
Quando o valor **começa** com `{{`, ele precisa estar entre aspas — senão o YAML interpreta
as chaves como um dicionário e o playbook nem carrega.
{{% /notice %}}

## Nomes

Letras, números e `_`, começando por letra. Nada de `-`, `.` ou espaços.
Prefixe as variáveis de uma role com o nome dela (`nginx_port`, `postgres_version`) para
evitar colisões.

## Onde definir

### Na play: `vars`

```yaml
vars:
  usuarios:
    - maria
    - joao
```

### Em arquivos: `vars_files`

```yaml
- name: Aplicação
  hosts: web
  vars_files:
    - vars/app.yml
```

### No inventário: `group_vars` e `host_vars`

O lugar certo para o que **muda por host ou ambiente** (portas, IPs, tamanhos de memória).
Veja [variáveis de inventário]({{% relref "/fundamentos/05-inventario/02-variaveis-de-inventario" %}}).

### Na linha de comando: `-e` (extra vars)

```bash
ansible-playbook app.yml -e "versao=2.4.1"
ansible-playbook app.yml -e '{"usuarios": ["maria", "joao"]}'
ansible-playbook app.yml -e @vars/release.yml
```

As *extra vars* vencem **qualquer** outra definição.

### Perguntando: `vars_prompt`

```yaml
vars_prompt:
  - name: versao
    prompt: Qual versão deseja instalar?
    private: false
```

## Listas e dicionários

```yaml
vars:
  app:
    nome: loja
    porta: 8080
    workers: 4
  dns:
    - 1.1.1.1
    - 8.8.8.8

tasks:
  - name: Acessando dicionário e lista
    ansible.builtin.debug:
      msg: "{{ app.nome }} na porta {{ app['porta'] }}, DNS primário {{ dns[0] }}"
```

Prefira a notação de colchetes (`app['porta']`) quando a chave puder conflitar com métodos
do Python (`items`, `keys`, `values`) ou tiver caracteres especiais.

## Variáveis especiais

| Variável | Conteúdo |
|---|---|
| `inventory_hostname` | Nome do host no inventário |
| `ansible_host` | Endereço usado para conectar |
| `group_names` | Lista de grupos do host atual |
| `groups` | Todos os grupos e seus hosts (`groups['web']`) |
| `hostvars` | Variáveis de **todos** os hosts (`hostvars['db01']['ansible_host']`) |
| `play_hosts` / `ansible_play_hosts` | Hosts ativos na play |
| `playbook_dir` | Diretório do playbook |
| `ansible_check_mode` | `true` quando executado com `--check` |
| `ansible_facts` | Facts coletados (próxima página) |

## Valores padrão e variáveis indefinidas

Usar uma variável que não existe é **erro**. Para tornar opcional, use o filtro `default`:

```yaml
msg: "Workers: {{ app_workers | default(2) }}"
```

E para exigir que ela exista com uma mensagem clara:

```yaml
- name: Valida variáveis obrigatórias
  ansible.builtin.assert:
    that:
      - app_versao is defined
      - app_versao is match('^\d+\.\d+\.\d+$')
    fail_msg: "Informe a versão com -e app_versao=X.Y.Z"
```
