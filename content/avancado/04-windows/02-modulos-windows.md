+++
title = "Playbooks para Windows"
weight = 2
description = "Recursos do Windows, pacotes com Chocolatey/WinGet, serviços, usuários, registro e Windows Update."
+++

Os módulos para Windows estão nas collections `ansible.windows`, `community.windows` e
`chocolatey.chocolatey`. Os nomes começam com `win_`.

| Linux | Windows |
|---|---|
| `ansible.builtin.ping` | `ansible.windows.win_ping` |
| `ansible.builtin.copy` | `ansible.windows.win_copy` |
| `ansible.builtin.template` | `ansible.windows.win_template` |
| `ansible.builtin.file` | `ansible.windows.win_file` |
| `ansible.builtin.command` / `shell` | `ansible.windows.win_command` / `win_shell` |
| `ansible.builtin.service` | `ansible.windows.win_service` |
| `ansible.builtin.user` | `ansible.windows.win_user` |
| `ansible.builtin.apt` | `chocolatey.chocolatey.win_chocolatey` / `ansible.windows.win_package` |
| `ansible.builtin.reboot` | `ansible.windows.win_reboot` |

## Exemplo: servidor web IIS

```yaml
- name: Servidor IIS
  hosts: windows
  gather_facts: true

  tasks:
    - name: Instala o IIS
      ansible.windows.win_feature:
        name: Web-Server
        include_management_tools: true
        state: present
      register: iis

    - name: Reinicia se o recurso exigir
      ansible.windows.win_reboot:
      when: iis.reboot_required

    - name: Página inicial
      ansible.windows.win_template:
        src: templates/iis-index.html.j2
        dest: C:\inetpub\wwwroot\index.html

    - name: Garante o serviço do IIS
      ansible.windows.win_service:
        name: W3SVC
        state: started
        start_mode: auto

    - name: Libera HTTP no firewall
      community.windows.win_firewall_rule:
        name: HTTP
        localport: 80
        protocol: tcp
        direction: in
        action: allow
        state: present
        enabled: true
```

{{% notice note "Caminhos do Windows no YAML" %}}
Sem aspas, `C:\inetpub\wwwroot` funciona. **Com aspas duplas**, a barra invertida vira
escape (`"C:\temp"` contém um `\t` de tabulação!). Use aspas simples ou duplique as barras:
`'C:\temp'` ou `"C:\\temp"`.
{{% /notice %}}

## Pacotes

```yaml
- name: Instala programas via Chocolatey
  chocolatey.chocolatey.win_chocolatey:
    name: [7zip, notepadplusplus, git]
    state: present

- name: Instala um MSI
  ansible.windows.win_package:
    path: https://exemplo.com/agente-monitoramento.msi
    product_id: '{A1B2C3D4-...}'
    arguments: /quiet SERVER=monitor.curso.lan
```

## Usuários, registro e Windows Update

```yaml
- name: Usuário local
  ansible.windows.win_user:
    name: suporte
    password: "{{ vault_senha_suporte }}"
    groups: [Usuários da área de trabalho remota]
    password_never_expires: true

- name: Desabilita o SMBv1 pelo registro
  ansible.windows.win_regedit:
    path: HKLM:\SYSTEM\CurrentControlSet\Services\LanmanServer\Parameters
    name: SMB1
    data: 0
    type: dword

- name: Aplica atualizações de segurança e reinicia se preciso
  ansible.windows.win_updates:
    category_names:
      - SecurityUpdates
      - CriticalUpdates
    reboot: true
```

## PowerShell quando não há módulo

```yaml
- name: Lista os 5 processos que mais usam memória
  ansible.windows.win_shell: |
    Get-Process | Sort-Object WorkingSet -Descending |
      Select-Object -First 5 Name, @{n='MB';e={[int]($_.WorkingSet/1MB)}} |
      Format-Table -AutoSize | Out-String
  register: processos
  changed_when: false

- name: Mostra
  ansible.builtin.debug:
    var: processos.stdout_lines
```

Para lógica mais complexa com idempotência real, existe o módulo
`ansible.windows.win_powershell`, que permite ao script informar se mudou algo
(`$Ansible.Changed = $false`).

## Playbooks mistos

Um mesmo playbook pode configurar Linux e Windows em plays separadas, ou usar
`ansible_facts['os_family'] == 'Windows'` em condições.
