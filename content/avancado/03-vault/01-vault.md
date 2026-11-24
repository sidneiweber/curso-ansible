+++
title = "Criptografando segredos"
weight = 1
description = "ansible-vault create, edit, encrypt e encrypt_string; arquivos de senha e vault IDs."
+++

## Arquivos criptografados

```bash
ansible-vault create group_vars/all/vault.yml     # cria e abre no $EDITOR
ansible-vault edit group_vars/all/vault.yml       # edita
ansible-vault view group_vars/all/vault.yml       # só visualiza
ansible-vault encrypt credenciais.yml             # criptografa um arquivo existente
ansible-vault decrypt credenciais.yml             # volta para texto puro
ansible-vault rekey group_vars/all/vault.yml      # troca a senha
```

O arquivo fica assim no disco — seguro para ir ao Git:

```text
$ANSIBLE_VAULT;1.1;AES256
62313365396662343061393464336163383764373764613633653634306231386433626436623361
6134333665353966363534333632666535333761666131620a663537646436643839616531643561
...
```

## O padrão vars + vault

Separe as variáveis em dois arquivos, transformando `group_vars/all.yml` em diretório:

```text
group_vars/
└── all/
    ├── vars.yml      # texto puro, legível
    └── vault.yml     # criptografado
```

```yaml
# group_vars/all/vault.yml (antes de criptografar)
vault_wp_db_senha: "S3nh@-Muito-Forte-2026"
```

```yaml
# group_vars/all/vars.yml
wp_db_nome: wordpress
wp_db_usuario: wordpress
wp_db_senha: "{{ vault_wp_db_senha }}"
```

Assim, um `grep wp_db_senha` ainda encontra onde a variável é definida, e fica claro que o
valor real está no vault — sem precisar descriptografar para entender o projeto.

## Executando

```bash
ansible-playbook site.yml --ask-vault-pass            # pergunta a senha (-J)
ansible-playbook site.yml --vault-password-file ~/.vault_pass
```

Para não digitar sempre, aponte o arquivo de senha no `ansible.cfg`:

```ini
[defaults]
vault_password_file = ~/.vault_pass
```

```bash
openssl rand -base64 32 > ~/.vault_pass
chmod 600 ~/.vault_pass
```

{{% notice warning %}}
O arquivo de senha fica **fora** do repositório. Se ele for executável, o Ansible roda o
script e usa a saída como senha — o que permite buscar a senha em um gerenciador
(`pass`, Bitwarden, 1Password, HashiCorp Vault) em vez de deixá-la em um arquivo.
{{% /notice %}}

## Variáveis criptografadas individualmente

Para criptografar **só um valor** dentro de um arquivo comum:

```bash
ansible-vault encrypt_string 'S3nh@-Muito-Forte-2026' --name 'vault_wp_db_senha'
```

```yaml
vault_wp_db_senha: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  3365396662343061393464336163383764373764613633653634306231386433...
```

É prático, mas arquivos inteiros criptografados são mais fáceis de manter (`rekey`, `edit`).

## Vault IDs: senhas diferentes por ambiente

```bash
ansible-vault create --vault-id prod@prompt inventories/producao/group_vars/all/vault.yml
ansible-vault create --vault-id hml@~/.vault_hml inventories/homologacao/group_vars/all/vault.yml

ansible-playbook site.yml --vault-id hml@~/.vault_hml --vault-id prod@prompt
```

Assim quem tem acesso à homologação não consegue ler os segredos de produção.

## Não vazando segredos na saída

Criptografar não adianta se a senha aparece no terminal ou no log do CI:

```yaml
- name: Cria o usuário do banco
  community.mysql.mysql_user:
    name: app
    password: "{{ wp_db_senha }}"
  no_log: true
```

{{% notice tip "git diff legível" %}}
Configure o Git para mostrar o conteúdo descriptografado nos diffs:
`echo '*/vault.yml diff=ansible-vault' >> .gitattributes` e
`git config diff.ansible-vault.textconv 'ansible-vault view'`.
{{% /notice %}}

## Exercício

1. Mova a senha do banco do projeto WordPress para `group_vars/all/vault.yml`.
2. Criptografe também as chaves geradas em `credenciais/` (dica: em vez do lookup
   `password`, defina-as como variáveis no vault).
3. Execute o `site.yml` com `--ask-vault-pass` e confirme `changed=0`.
