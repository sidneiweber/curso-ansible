+++
title = "Deploy pelo pipeline"
weight = 2
description = "Disparando o playbook de deploy a partir do GitHub Actions ao criar uma tag."
+++

O último passo é ninguém precisar rodar o deploy do próprio computador: criar uma **tag** no
repositório dispara o playbook.

```yaml
# .github/workflows/deploy.yml (no repositório de infraestrutura)
name: Deploy
on:
  workflow_dispatch:
    inputs:
      versao:
        description: Versão (tag) a implantar
        required: true
      ambiente:
        type: choice
        options: [homologacao, producao]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.ambiente }}      # exige aprovação em produção
    steps:
      - uses: actions/checkout@v4

      - name: Instala o Ansible
        run: pip install ansible

      - name: Instala as dependências
        run: ansible-galaxy install -r requirements.yml

      - name: Configura a chave SSH
        run: |
          install -m 700 -d ~/.ssh
          echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/id_ed25519
          chmod 600 ~/.ssh/id_ed25519
          echo "${{ secrets.SSH_KNOWN_HOSTS }}" > ~/.ssh/known_hosts

      - name: Deploy
        env:
          ANSIBLE_VAULT_PASSWORD: ${{ secrets.VAULT_PASSWORD }}
        run: |
          echo "$ANSIBLE_VAULT_PASSWORD" > .vault_pass
          ansible-playbook -i inventories/${{ inputs.ambiente }} playbooks/deploy.yml \
            --vault-password-file .vault_pass \
            -e app_versao=${{ inputs.versao }}
          rm -f .vault_pass
```

Pontos importantes:

- **Segredos** (chave SSH, senha do vault, `known_hosts`) ficam nos *secrets* do GitHub.
- O **environment** `producao` pode exigir aprovação de outra pessoa antes de rodar.
- O runner precisa **alcançar** os servidores: runners públicos só funcionam com servidores
  expostos na Internet. Para redes internas, use um *self-hosted runner* ou uma VPN.

{{% notice tip %}}
Fixe a versão do Ansible no pipeline (`pip install 'ansible==12.*'`) ou, melhor, use o mesmo
[execution environment]({{% relref "/avancado/06-escala/04-awx" %}}) usado pela equipe.
{{% /notice %}}
