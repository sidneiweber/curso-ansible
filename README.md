# Curso Ansible

Material do curso de Ansible — da primeira linha de YAML a roles, Vault, Windows e deploys.

🌐 **https://cursoansible.sidneiweber.com.br**

## Estrutura

- `content/fundamentos` — o que é Ansible, laboratório, instalação, YAML, inventário, ad-hoc
- `content/playbooks` — playbooks, variáveis, facts, templates Jinja2, loops, condicionais, handlers, tags, blocks
- `content/avancado` — roles, Galaxy e collections, Vault, Windows, boas práticas, escala, deploys, módulos próprios

## Rodando localmente

Requer [Hugo](https://gohugo.io/) extended (≥ 0.149).

```bash
git clone --recurse-submodules git@github.com:sidneiweber/curso-ansible.git
cd curso-ansible
hugo server
```

Feito com o tema [Relearn](https://github.com/McShelby/hugo-theme-relearn).
