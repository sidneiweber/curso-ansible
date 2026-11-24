+++
title = "Documentação com ansible-doc"
weight = 3
description = "Encontrando módulos e lendo parâmetros, exemplos e valores de retorno offline."
+++

O `ansible-doc` traz a documentação de **todos os módulos e plugins instalados**, offline e
exatamente na versão que você está usando.

```bash
ansible-doc -l | wc -l                      # quantos módulos estão disponíveis
ansible-doc -l | grep -i firewall           # procurar
ansible-doc -l ansible.builtin              # módulos de uma collection
ansible-doc ansible.builtin.user            # documentação completa
ansible-doc -s ansible.builtin.user         # só o resumo dos parâmetros (snippet)
ansible-doc -t filter ansible.builtin.to_yaml   # outros tipos de plugin
ansible-doc -t lookup -l
```

## Lendo a página de um módulo

| Seção | O que procurar |
|---|---|
| **OPTIONS** | Parâmetros. Os marcados com `=` são **obrigatórios**; veja `default` e `choices`. |
| **ATTRIBUTES** | Se suporta `check_mode` e `diff_mode`, e em quais plataformas. |
| **NOTES** | Requisitos e comportamentos importantes. |
| **SEE ALSO** | Módulos relacionados — às vezes existe um mais adequado. |
| **EXAMPLES** | Exemplos de tarefas prontos para copiar. **Comece por aqui.** |
| **RETURN VALUES** | O que o módulo devolve — útil com `register`. |

{{% notice tip %}}
Na web, a documentação fica em [docs.ansible.com](https://docs.ansible.com/), em
*Collection Index*. Confira se a versão da página corresponde à sua (`ansible --version`).
{{% /notice %}}

## Exercícios

1. Liste os módulos da collection `ansible.posix`.
2. Descubra qual parâmetro do módulo `ansible.builtin.apt` remove pacotes que não são mais
   necessários (equivalente a `apt autoremove`).
3. Usando ad-hoc, instale o `htop` em todos os servidores e confirme a versão instalada
   com o módulo `command`.
4. Crie o arquivo `/etc/motd` em todos os hosts com o texto `Gerenciado por Ansible` e
   execute novamente com `--check --diff` alterando o texto.
5. Colete com o módulo `setup` a quantidade de memória de cada host em uma linha por host.
