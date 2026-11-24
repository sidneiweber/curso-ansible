+++
title = "O que é Ansible"
weight = 1
description = "Infraestrutura como código, estado desejado e idempotência."
+++

## O problema

Configurar **um** servidor à mão é fácil: `apt install`, editar dois arquivos, reiniciar o
serviço. Configurar **cinquenta** do mesmo jeito — e garantir que continuem iguais seis meses
depois — não é. Os problemas aparecem rápido:

- passos esquecidos ou feitos em ordem diferente em cada máquina;
- servidores "floco de neve": únicos, ninguém sabe exatamente como foram montados;
- documentação desatualizada no instante em que alguém faz um ajuste manual;
- medo de reconstruir uma máquina, porque ninguém tem certeza de como ela chegou ali.

## Infraestrutura como código

**Infraestrutura como código** (*Infrastructure as Code*, IaC) é descrever servidores,
redes e aplicações em **arquivos de texto** que uma ferramenta lê e aplica. Esses arquivos:

- ficam no **Git**: têm histórico, revisão por pares (*pull requests*) e podem ser revertidos;
- **são** a documentação — e nunca ficam desatualizados, porque são o que de fato roda;
- podem ser **testados** e executados quantas vezes for preciso;
- permitem **recriar** um ambiente inteiro a partir do zero.

## Ansible

O **Ansible** é uma ferramenta de automação de código aberto, mantida pela Red Hat, que
executa **configuração de sistemas, instalação de software, deploy de aplicações e
orquestração** a partir de arquivos YAML chamados **playbooks**:

```yaml
- name: Servidores web
  hosts: web
  become: true
  tasks:
    - name: Instala o nginx
      ansible.builtin.apt:
        name: nginx
        state: present

    - name: Garante o nginx rodando e habilitado no boot
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true
```

Mesmo sem conhecer Ansible, dá para entender o que esse playbook faz. Essa legibilidade é
proposital e é uma das maiores qualidades da ferramenta.

## Estado desejado e idempotência

Repare que o playbook acima **não** diz "instale o nginx"; ele diz "o nginx deve estar
**presente**" (`state: present`). Essa é a ideia de **estado desejado**: você descreve como o
sistema deve ficar, e o Ansible decide o que precisa ser feito para chegar lá.

Daí vem a propriedade mais importante de uma boa automação, a **idempotência**: executar a
mesma coisa uma ou cem vezes produz o mesmo resultado.

| Execução | O que o Ansible faz | Resultado |
|---|---|---|
| 1ª | nginx não está instalado → instala | `changed` |
| 2ª | nginx já está instalado → não faz nada | `ok` |
| 3ª (alguém parou o serviço) | serviço parado → inicia | `changed` |

{{% notice info "Script × Ansible" %}}
Um script shell com `apt install nginx && echo "conf" >> /etc/nginx/nginx.conf` **não** é
idempotente: na segunda execução a linha é adicionada de novo. Os módulos do Ansible
verificam o estado atual antes de agir — esse é o trabalho que você deixa de fazer.
{{% /notice %}}

## O que dá para automatizar

- **Configuração de servidores:** pacotes, usuários, arquivos de configuração, serviços, firewall.
- **Deploy de aplicações:** baixar uma versão, migrar o banco, trocar o link, reiniciar sem downtime.
- **Orquestração:** tirar um servidor do balanceador, atualizar, devolver, passar para o próximo.
- **Rede:** switches e roteadores Cisco, Juniper, MikroTik e outros.
- **Nuvem:** criar instâncias na AWS, Azure, GCP, Proxmox, VMware.
- **Windows:** via WinRM ou SSH, com os mesmos playbooks.
- **Tarefas pontuais:** "reinicie todos os servidores de homologação", "qual versão do OpenSSL
  está instalada em cada máquina?".
