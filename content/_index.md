+++
title = "Curso Ansible"
type = "home"
description = "Curso de Ansible em trilhas: fundamentos, playbooks e automação avançada."
+++

Material de estudo de **Ansible** organizado em trilhas: da primeira linha de YAML até roles,
segredos com Vault, máquinas Windows e deploys de aplicações. Ao final você será capaz de:

- **automatizar a instalação** de uma máquina do zero;
- **provisionar tarefas** em dezenas de servidores com um único comando;
- aplicar **boas práticas** com variáveis e templates;
- **reutilizar** código com roles e collections;
- montar **ambientes complexos** com um simples playbook;
- fazer **deploys** sem downtime.

{{% notice tip "Como usar este material" %}}
Monte o [laboratório]({{% relref "/fundamentos/02-laboratorio" %}}) e **execute todos os
exemplos**. Ansible se aprende rodando playbooks, lendo a saída e rodando de novo — e um
playbook só está bom quando a segunda execução não muda nada.
{{% /notice %}}

## Trilhas

| Trilha | Para quem | Conteúdo |
|---|---|---|
| [Fundamentos]({{% relref "/fundamentos" %}}) | Quem nunca usou Ansible | Infraestrutura como código, arquitetura, laboratório, instalação, YAML, inventário, comandos ad-hoc |
| [Playbooks]({{% relref "/playbooks" %}}) | Quem já roda comandos ad-hoc | Playbooks, módulos essenciais, variáveis, facts, templates Jinja2, loops, condicionais, handlers, tags, blocks |
| [Avançado]({{% relref "/avancado" %}}) | Quem já escreve playbooks | Roles, Galaxy e collections, Vault, Windows, boas práticas, escala, deploys, módulos próprios |

**Pré-requisitos:** uso do terminal Linux, SSH e edição de arquivos de texto. Se precisar
revisar, veja o [Curso Linux](https://cursolinux.sidneiweber.com.br/).
