+++
title = "Por que usar Ansible"
weight = 2
description = "Vantagens, limitações e comparação com Puppet, Chef, Salt e Terraform."
+++

## Vantagens

- **Sem agente (*agentless*):** os servidores gerenciados não precisam de nenhum software
  extra além de **SSH** e **Python** — que praticamente toda distribuição Linux já tem.
  Nada para instalar, atualizar ou monitorar nas máquinas.
- **Simples de aprender:** playbooks em YAML, legíveis por quem não programa.
- **Push:** você decide **quando** as mudanças são aplicadas, executando o playbook a partir
  do *control node*. Não há servidor central obrigatório.
- **Milhares de módulos:** pacotes, arquivos, serviços, bancos de dados, nuvem, rede, Windows,
  containers — organizados em *collections*.
- **Comunidade enorme:** roles prontas no [Ansible Galaxy](https://galaxy.ansible.com/),
  documentação extensa, suporte comercial (Red Hat Ansible Automation Platform) para quem precisa.

## Limitações

- **Desempenho em escala muito grande:** por usar SSH, milhares de hosts exigem ajustes
  (paralelismo, *pipelining*, *mitogen*, AWX). Para algumas centenas, funciona muito bem.
- **Sem detecção contínua de desvio:** o Ansible só corrige o que mudou quando é executado.
  Para garantir o estado o tempo todo, agende as execuções (cron, AWX, CI).
- **YAML tem limites:** lógica complexa em playbooks fica difícil de ler. Quando isso
  acontece, o caminho é um *filter plugin* ou um módulo em Python.

## Comparação com outras ferramentas

| Ferramenta | Linguagem | Agente | Modelo | Foco |
|---|---|---|---|---|
| **Ansible** | YAML | Não (SSH/WinRM) | Push | Configuração, deploy, orquestração |
| **Puppet** | DSL própria | Sim | Pull | Configuração contínua |
| **Chef** | Ruby | Sim | Pull | Configuração contínua |
| **Salt** | YAML | Sim (ou SSH) | Push e pull | Configuração, execução remota rápida |
| **Terraform/OpenTofu** | HCL | Não | APIs | **Provisionamento** de infraestrutura |

{{% notice tip "Ansible e Terraform juntos" %}}
Não é um ou outro. Um padrão muito comum é o **Terraform criar** as máquinas, redes e
balanceadores na nuvem e o **Ansible configurar** o que roda dentro delas.
{{% /notice %}}

## Casos de uso comuns

- Padronizar a instalação de servidores novos (*baseline*: usuários, SSH, NTP, monitoramento).
- Aplicar atualizações de segurança em todos os servidores, um grupo por vez.
- Distribuir arquivos de configuração gerados a partir de templates.
- Deploy de aplicações web com *rolling update*.
- Criar ambientes de desenvolvimento idênticos à produção.
- Auditorias: coletar informações de todas as máquinas em segundos.
