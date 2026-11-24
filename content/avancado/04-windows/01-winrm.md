+++
title = "Conectando ao Windows"
weight = 1
description = "WinRM ou SSH, preparando o host Windows, pywinrm e variáveis de inventário."
+++

O Windows não pode ser *control node*, mas pode ser **gerenciado**. Os módulos para Windows
são escritos em **PowerShell** — o Python não é necessário no host.

## WinRM ou SSH?

| | WinRM | OpenSSH |
|---|---|---|
| Disponibilidade | Nativo em todo Windows Server | Recurso opcional (nativo no Server 2025) |
| Configuração | Listener, autenticação, certificados | Igual ao Linux: chaves SSH |
| Autenticação de domínio | Kerberos, NTLM, CredSSP | Senha ou chave |
| Situação | O mais usado | Suportado oficialmente; tende a ser o futuro |

O curso usa **WinRM**, que é o que você encontrará na maioria dos ambientes.

## Preparando o host Windows (laboratório)

No `win01`, em um PowerShell **como administrador**:

```powershell
# rede como "Privada" (o WinRM não aceita conexões em redes "Públicas")
Get-NetConnectionProfile | Set-NetConnectionProfile -NetworkCategory Private

# habilita o WinRM com um listener HTTP e libera o firewall
winrm quickconfig -quiet
Enable-PSRemoting -Force

# usuário para o Ansible
$senha = Read-Host -AsSecureString "Senha"
New-LocalUser -Name ansible -Password $senha -PasswordNeverExpires
Add-LocalGroupMember -Group Administradores -Member ansible   # "Administrators" em inglês
```

Confira: `winrm enumerate winrm/config/Listener` deve mostrar um listener na porta **5985**.

{{% notice warning "HTTP só em laboratório" %}}
Com NTLM a **mensagem** é criptografada mesmo sobre HTTP, mas em produção use um listener
**HTTPS** (porta 5986) com um certificado válido, ou Kerberos em um domínio Active Directory.
{{% /notice %}}

## Preparando o control node

```bash
pipx inject ansible pywinrm        # instalação com pipx
# ou: sudo apt install python3-winrm
```

## Inventário

```ini
[windows]
win01 ansible_host=192.168.56.41

[windows:vars]
ansible_connection=winrm
ansible_port=5985
ansible_winrm_scheme=http
ansible_winrm_transport=ntlm
ansible_user=ansible
ansible_password="{{ vault_win_senha }}"
```

(A senha vai para o [Vault]({{% relref "/avancado/03-vault" %}}), em `group_vars/windows/vault.yml`.)

Em um domínio AD, use `ansible_user=ansible@CURSO.LAN` e `ansible_winrm_transport=kerberos`.

## Testando

```bash
ansible windows -m ansible.windows.win_ping
```

```text
win01 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

{{% notice tip "Com SSH" %}}
Com o OpenSSH Server instalado no Windows, a configuração fica:
`ansible_connection=ssh`, `ansible_shell_type=powershell` — e o resto é igual ao Linux.
{{% /notice %}}
