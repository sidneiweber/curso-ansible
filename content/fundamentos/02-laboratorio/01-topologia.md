+++
title = "Topologia e máquinas"
weight = 1
description = "As VMs, a rede do laboratório e os endereços usados nos exemplos."
+++

## Topologia

```text
                           Internet
                              │
                      (NAT do VirtualBox)
                              │ enp0s3 em todas as VMs
   ┌─────────────┬────────────┼─────────────┬─────────────┐
   │  ansible01  │    web01   │    web02    │    db01     │   win01 (opcional)
   │    .10      │    .21     │    .22      │    .31      │     .41
   └──────┬──────┴─────┬──────┴──────┬──────┴──────┬──────┴──────┬──────
          └────────────┴─────────────┴─────────────┴─────────────┘
                 enp0s8 — rede host-only 192.168.56.0/24
```

| Máquina | Sistema | Endereço | Papel |
|---|---|---|---|
| `ansible01` | Debian 13 | `192.168.56.10` | **Control node** — onde o Ansible é instalado |
| `web01` | Debian 13 | `192.168.56.21` | Servidor web |
| `web02` | Debian 13 | `192.168.56.22` | Servidor web |
| `db01` | Debian 13 | `192.168.56.31` | Banco de dados |
| `win01` | Windows Server 2022/2025 (opcional) | `192.168.56.41` | Capítulo de Windows |

- **Domínio:** `curso.lan`
- **Rede:** `192.168.56.0/24`, *host-only*: a sua máquina também enxerga as VMs.
- **Recursos:** 1 vCPU e 1 GB de RAM bastam para cada Debian sem interface gráfica.

{{% notice tip "Posso usar a minha máquina como control node?" %}}
Pode, se ela for Linux, macOS ou Windows com **WSL**. Nesse caso, dispense a `ansible01`
e execute tudo a partir do seu terminal. O curso usa uma VM dedicada para que todos
tenham exatamente o mesmo ambiente.
{{% /notice %}}

## Opção 1: criando as VMs manualmente

1. Crie uma VM Debian 13 com dois adaptadores de rede:
   - Adaptador 1: **NAT** (Internet).
   - Adaptador 2: **Placa de rede exclusiva de hospedeiro** (*host-only*), `vboxnet0`.
2. Na instalação, marque apenas **servidor SSH** e **utilitários padrão do sistema**.
3. Configure o IP fixo na segunda interface em `/etc/network/interfaces.d/enp0s8`:

   ```text
   auto enp0s8
   iface enp0s8 inet static
       address 192.168.56.21/24
   ```

4. Desligue e **clone** a VM para as demais, ajustando em cada uma o IP e o nome
   (`hostnamectl set-hostname web02`).

{{% notice warning %}}
Ao clonar, peça ao VirtualBox para **gerar novos endereços MAC**. E regenere as chaves de
host SSH em cada clone, ou o SSH vai reclamar de chaves repetidas:
`rm /etc/ssh/ssh_host_* && dpkg-reconfigure openssh-server`.
{{% /notice %}}

## Opção 2: Vagrant

O [Vagrant](https://developer.hashicorp.com/vagrant) cria e configura as VMs a partir de um
arquivo — infraestrutura como código antes mesmo do Ansible. Crie um diretório `lab/` com
o arquivo `Vagrantfile`:

```ruby
# lab/Vagrantfile
MAQUINAS = {
  "ansible01" => "192.168.56.10",
  "web01"     => "192.168.56.21",
  "web02"     => "192.168.56.22",
  "db01"      => "192.168.56.31",
}

Vagrant.configure("2") do |config|
  config.vm.box = "bento/debian-13"

  config.vm.provider "virtualbox" do |vb|
    vb.memory = 1024
    vb.cpus = 1
  end

  MAQUINAS.each do |nome, ip|
    config.vm.define nome do |m|
      m.vm.hostname = nome
      m.vm.network "private_network", ip: ip
    end
  end
end
```

```bash
cd lab
vagrant up                # cria e liga todas as VMs
vagrant ssh ansible01     # entra no control node
vagrant snapshot save limpo   # snapshot de todas as VMs
vagrant destroy -f        # apaga tudo
```

## Resolução de nomes

Para usar nomes em vez de IPs, adicione ao `/etc/hosts` da `ansible01` (e da sua máquina,
se quiser):

```text
192.168.56.10  ansible01.curso.lan ansible01
192.168.56.21  web01.curso.lan     web01
192.168.56.22  web02.curso.lan     web02
192.168.56.31  db01.curso.lan      db01
192.168.56.41  win01.curso.lan     win01
```

{{% notice info "Ansible também roda em containers?" %}}
Sim — e para testar roles é ótimo (veja *Molecule* em [boas práticas]({{% relref "/avancado/05-boas-praticas" %}})).
Mas containers não têm systemd, kernel próprio nem SSH por padrão; para aprender, VMs se
comportam como servidores de verdade e evitam surpresas.
{{% /notice %}}
