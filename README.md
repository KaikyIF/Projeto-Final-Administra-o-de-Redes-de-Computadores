# Projeto-Final-Administra-o-de-Redes-de-Computadores
# Projeto de Rede Residencial 

**Autores:** Kaiky Malaquias,Heitor Messias Gomes

**Resumo:**
Este projeto descreve a concepção de uma rede residencial organizada, com topologia física e lógica, segmentação em sub-redes, e os requisitos para implementação dos seguintes serviços: DHCP, DNS, servidor web (Apache/Nginx), FTP (vsftpd), e NFS. O objetivo é fornecer um roteiro técnico claro para implantação em um laboratório ou ambiente real, incluindo endereçamento IP, pools DHCP, registros DNS, requisitos de hardware/software e medidas básicas de segurança.

---

## 1. Objetivos do projeto

* Fornecer atribuição automática de endereços IP (DHCP).
* Resolver nomes internamente (DNS) com domínio `kaiky-home.local`.
* Hospedar sites internos (Apache ou Nginx).
* Permitir transferência de arquivos via FTP seguro (vsftpd).
* Compartilhar diretórios entre máquinas via NFS.
* Segmentar a rede para isolar dispositivos (IoT / guests / servidores / usuários domésticos).

---

## 2. Topologia física e lógica

**Componentes principais:**

* Roteador/Firewall (com suporte a VLANs)
* Switch gerenciável (4–8 portas com suporte a VLANs)
* Servidor doméstico (p. ex. Raspberry Pi 4, Intel NUC ou PC pequeno) — hospedará DHCP, DNS, web, FTP e NFS
* Dispositivos clientes (laptops, smartphones, SmartTVs, dispositivos IoT)

**Topologia (descrição):**

* WAN -> Roteador (gateway) -> Switch -> Dispositivos
* Servidor conectado por cabo ao switch (interface física `eth0`).
* VLANs configuradas no roteador+switch para separar tráfego.

**Diagrama ASCII simplificado:**

```
[WAN/ISP]
   |
[Roteador/Gateway (VLAN-aware)]
   |
[Switch gerenciável]
  |    |     |      
Srv   PC   SmartTV  AP (Wi‑Fi)
(eth) (eth) (eth)   (trunks p/ VLANs)
```

---

## 3. Plano de endereçamento e segmentação de sub-redes

Escolhemos a faixa privada `192.168.100.0/24` e a dividimos em 4 sub-redes /26 (cada uma com 62 hosts úteis). Valores pensados para uma residência com isolamento de IoT e guest.

* **VLAN 10 — Management / Servidores**: `192.168.100.0/26` (hosts: `192.168.100.1`–`192.168.100.62`)

  * Gateway: `192.168.100.1`
  * Servidor (DHCP/DNS/Web/FTP/NFS): `192.168.100.10` (hostname: `kaiky-server`)

* **VLAN 20 — Usuários (familiares)**: `192.168.100.64/26` (hosts `192.168.100.65`–`192.168.100.126`)

  * Gateway: `192.168.100.65`
  * DHCP range sugerido: `192.168.100.80`–`192.168.100.110`

* **VLAN 30 — IoT (câmeras, automação)**: `192.168.100.128/26` (hosts `192.168.100.129`–`192.168.100.190`)

  * Gateway: `192.168.100.129`
  * DHCP range sugerido: `192.168.100.140`–`192.168.100.170`

* **VLAN 40 — Guest Wi‑Fi**: `192.168.100.192/26` (hosts `192.168.100.193`–`192.168.100.254`)

  * Gateway: `192.168.100.193`
  * DHCP range sugerido: `192.168.100.200`–`192.168.100.240`

---

## 4. Requisitos e configurações dos serviços

A seguir estão os requisitos por serviço, exemplos de configuração mínima e portas usadas.

### 4.1 DHCP (isc-dhcp-server)

**Função:** distribuir endereços IP e opções (gateway, DNS).

**Requisitos:**

* Servidor na VLAN 10 com IP fixo `192.168.100.10`.
* Escopos DHCP para cada VLAN (ou delegação ao roteador para VLANs).

**Exemplo (escopo VLAN 20):**

```
subnet 192.168.100.64 netmask 255.255.255.192 {
  range 192.168.100.80 192.168.100.110;
  option routers 192.168.100.65;
  option domain-name-servers 192.168.100.10;
  option domain-name "kaiky-home.local";
}
```

**Portas:** UDP 67/68.

---

### 4.2 DNS interno (BIND9)

**Função:** resolver nomes locais do domínio `kaiky-home.local` e fazer forward para DNS público.

**Requisitos:**

* Zona direta `kaiky-home.local` e zona reversa para `192.168.100.0/24`.
* Entrada A para `kaiky-server.kaiky-home.local -> 192.168.100.10`.
* Registrar hosts importantes (pc, nas, camera1, etc.).

**Registros sugeridos:**

```
ns1 IN A 192.168.100.10
kaiky-server IN A 192.168.100.10
nas IN A 192.168.100.11
web IN A 192.168.100.10
```

**Portas:** TCP/UDP 53.

---

### 4.3 Servidor Web (Apache / Nginx)

**Função:** hospedar página interna (ex.: painel do NAS, intranet familiar).

**Requisitos:**

* Document root: `/var/www/kaiky-home/` com `index.html`.
* VirtualHost para `intranet.kaiky-home.local` apontando para o servidor.
* Certificado TLS autoassinada ou Let's Encrypt (se for expor externamente).

**Portas:** TCP 80 (HTTP) e TCP 443 (HTTPS).

---

### 4.4 FTP (vsftpd)

**Função:** permitir transferência de arquivos dentro da rede.

**Requisitos:**

* Evitar FTP anônimo em redes residenciais (ou desabilitar). Preferir FTPS (FTP sobre TLS) ou SFTP (via SSH) para segurança.
* Usuários locais com diretório chrooted (`/srv/ftp/usuário`).

**Configurações importantes:**

```
local_enable=YES
write_enable=YES
chroot_local_user=YES
ssl_enable=YES   # para FTPS, se configurado
```

**Portas:** TCP 21 (controle) + portas de data/passive range configurada.

---

### 4.5 NFS (nfs‑kernel‑server)

**Função:** compartilhar diretórios entre Linux na rede (ex.: `/compartilhado`).

**Requisitos:**

* Diretório com permissões adequadas e export definido em `/etc/exports`.
* Restrição de acesso por subnet (ex.: `192.168.100.0/26` para servidores/management).

**Exemplo `/etc/exports`:**

```
/compartilhado 192.168.100.0/26(rw,sync,no_subtree_check)
```

**Portas:** NFS usa várias portas (2049 para NFSv4). Configurar firewall para permitir apenas entre sub‑redes autorizadas.

---

## 5. Requisitos de hardware e software

* **Servidor:** Raspberry Pi 4 (4GB), SSD/SD para sistema, 1 NIC — ou PC pequeno (Intel NUC).
* **Sistema operacional:** Ubuntu Server LTS / Debian.
* **Switch:** Gerenciável (suporta VLANs, 8 portas).
* **Roteador:** Suporte a VLANs/Firewall/NAT.
* **Backup:** HD externo ou NAS para backups regulares.

---

## 6. Segurança e boas práticas

* Manter sistema e pacotes atualizados (`sudo apt update && sudo apt upgrade`).
* Usar senhas fortes e, onde possível, autenticação por chave (SSH).
* Restringir acesso FTP/NFS a sub‑redes específicas e preferir SFTP/SMB com TLS quando possível.
* Configurar regras de firewall (ufw/iptables) no servidor: permitir só as portas necessárias.
* Fazer logs e rotacionamento de logs, monitorar serviços.

---

## 7. Testes e verificação

* **DHCP:** desconectar um cliente e verificar se recebe IP no escopo; checar `tail /var/log/syslog` ou `journalctl -u isc-dhcp-server`.
* **DNS:** `dig kaiky-server.kaiky-home.local` e `nslookup` a partir de clientes.
* **Web:** abrir `http://intranet.kaiky-home.local` no navegador de um cliente.
* **FTP:** conectar com `ftp` ou cliente GUI; testar upload/download.
* **NFS:** montar `sudo mount 192.168.100.10:/compartilhado /mnt` em cliente e testar leitura/escrita.

---
## RESUMO DOS COMANDOS

---

## 0) Atualizar sistema

```bash
sudo apt update && sudo apt upgrade -y
```

---

## 1) Configurar IP fixo (NetworkManager — recomendado no Mint)

Substitua `"Wired connection 1"` pelo nome da sua conexão (`nmcli con show` lista as conexões).

```bash
# Ver o nome da conexão
nmcli con show

# Configurar IP estático /26
sudo nmcli con modify "Wired connection 1" ipv4.addresses 192.168.100.10/26
sudo nmcli con modify "Wired connection 1" ipv4.gateway 192.168.100.1
sudo nmcli con modify "Wired connection 1" ipv4.dns "192.168.100.10 1.1.1.1"
sudo nmcli con modify "Wired connection 1" ipv4.method manual

# Reiniciar a conexão
sudo nmcli con down "Wired connection 1"
sudo nmcli con up "Wired connection 1"

# Conferir IP
ip addr show eth0
ip route
```


---

## 2) Instalar utilitários básicos

```bash
sudo apt install -y vim curl nano net-tools dnsutils
```

---

## 3) DHCP — `isc-dhcp-server`

### Instalar

```bash
sudo apt install -y isc-dhcp-server
```

### Definir interface do serviço

Edite (ou escreva) o arquivo de defaults:

```bash
sudo sed -i 's/^INTERFACESv4=.*/INTERFACESv4="eth0"/' /etc/default/isc-dhcp-server || echo 'INTERFACESv4="eth0"' | sudo tee /etc/default/isc-dhcp-server
```

### Criar arquivo de configuração (exemplo para VLAN 20)

```bash
sudo tee /etc/dhcp/dhcpd.conf > /dev/null <<'EOF'
authoritative;
default-lease-time 600;
max-lease-time 7200;

option domain-name "kaiky-home.local";
option domain-name-servers 192.168.100.10;

subnet 192.168.100.64 netmask 255.255.255.192 {
  range 192.168.100.80 192.168.100.110;
  option routers 192.168.100.65;
}
EOF
```

### Reiniciar e checar status

```bash
sudo systemctl restart isc-dhcp-server
sudo systemctl enable isc-dhcp-server
sudo systemctl status isc-dhcp-server --no-pager
```

### Teste rápido

Conecte um cliente à rede e veja se ele recebe IP. No servidor:

```bash
sudo tail -n 50 /var/log/syslog | grep dhcp
```

---

## 4) DNS interno — `bind9`

### Instalar

```bash
sudo apt install -y bind9 bind9utils bind9-doc
```

### Declarar a zona (named.conf.local)

```bash
sudo tee /etc/bind/named.conf.local > /dev/null <<'EOF'
zone "kaiky-home.local" {
    type master;
    file "/etc/bind/db.kaiky-home.local";
};
zone "100.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.100";
};
EOF
```


### Criar arquivo de zona direta

```bash
sudo tee /etc/bind/db.kaiky-home.local > /dev/null <<'EOF'
$TTL 604800
@   IN  SOA ns1.kaiky-home.local. admin.kaiky-home.local. (
        2025113001 ; Serial (YYYYMMDDnn)
        604800
        86400
        2419200
        604800 )
;
@       IN      NS      ns1
ns1     IN      A       192.168.100.10
kaiky-server IN A       192.168.100.10
web     IN      A       192.168.100.10
nas     IN      A       192.168.100.11
EOF
```

### Criar arquivo de zona reversa (exemplo /24)

```bash
sudo tee /etc/bind/db.192.168.100 > /dev/null <<'EOF'
$TTL 604800
@   IN  SOA ns1.kaiky-home.local. admin.kaiky-home.local. (
        2025113001 ;
        604800
        86400
        2419200
        604800 )
;
@       IN      NS      ns1.
10      IN      PTR     kaiky-server.kaiky-home.local.
11      IN      PTR     nas.kaiky-home.local.
EOF
```

### Reiniciar e checar

```bash
sudo systemctl restart bind9
sudo systemctl enable bind9
sudo systemctl status bind9 --no-pager

# Testes:
dig @192.168.100.10 kaiky-server.kaiky-home.local A
dig @192.168.100.10 -x 192.168.100.10
```

---

## 5) Servidor Web — Apache

### Instalar

```bash
sudo apt install -y apache2
```

### Criar documento e VirtualHost

```bash
sudo mkdir -p /var/www/intranet
echo "<h1>Intranet Kaiky-Home</h1><p>Servidor: kaiky-server</p>" | sudo tee /var/www/intranet/index.html

sudo tee /etc/apache2/sites-available/intranet.conf > /dev/null <<'EOF'
<VirtualHost *:80>
    ServerName intranet.kaiky-home.local
    DocumentRoot /var/www/intranet
    <Directory /var/www/intranet>
        Require all granted
    </Directory>
</VirtualHost>
EOF
```

### Ativar site e reiniciar

```bash
sudo a2ensite intranet.conf
sudo systemctl reload apache2
sudo systemctl enable apache2
sudo systemctl status apache2 --no-pager
```

### Teste

No servidor ou cliente:

```bash
curl -I http://intranet.kaiky-home.local
# ou abrir no navegador: http://intranet.kaiky-home.local
```

> Se DNS não estiver propagado no cliente, teste usando `/etc/hosts` temporário apontando `192.168.100.10 intranet.kaiky-home.local`.

---

## 6) FTP — `vsftpd` (configuração simples)

### Instalar

```bash
sudo apt install -y vsftpd
```

### Backup do config atual e criar novo config mínimo seguro

```bash
sudo cp /etc/vsftpd.conf /etc/vsftpd.conf.bak

sudo tee /etc/vsftpd.conf > /dev/null <<'EOF'
listen=YES
listen_ipv6=NO
anonymous_enable=NO
local_enable=YES
write_enable=YES
chroot_local_user=YES
user_sub_token=\$USER
local_root=/home/\$USER/ftp
pasv_enable=YES
pasv_min_port=30000
pasv_max_port=30010
ssl_enable=NO
EOF
```

### Criar usuário FTP e diretório chroot

```bash
sudo adduser ftpuser
# Siga as instruções para senha

sudo mkdir -p /home/ftpuser/ftp/upload
sudo chown -R ftpuser:ftpuser /home/ftpuser/ftp
sudo chmod a-w /home/ftpuser/ftp
sudo chmod 755 /home/ftpuser/ftp/upload
```

### Reiniciar e checar

```bash
sudo systemctl restart vsftpd
sudo systemctl enable vsftpd
sudo systemctl status vsftpd --no-pager
```

### Teste

No cliente (ou no servidor):

```bash
ftp 192.168.100.10
# login com ftpuser
# testar put/get
```

> Para produção, habilite `ssl_enable=YES` e configure certificados (ou prefira SFTP via SSH).

---

## 7) NFS — `nfs-kernel-server`

### Instalar

```bash
sudo apt install -y nfs-kernel-server
```

### Criar diretório compartilhado e permissões

```bash
sudo mkdir -p /compartilhado
sudo chown nobody:nogroup /compartilhado
sudo chmod 777 /compartilhado  # só para testes; ajuste permissões depois
```

### Editar `/etc/exports`

```bash
sudo tee -a /etc/exports > /dev/null <<'EOF'
/compartilhado 192.168.100.0/26(rw,sync,no_subtree_check)
EOF
```

### Aplicar e reiniciar

```bash
sudo exportfs -ra
sudo systemctl restart nfs-kernel-server
sudo systemctl enable nfs-kernel-server
sudo systemctl status nfs-kernel-server --no-pager
```

### Teste de cliente (instalar nfs-common no cliente)

No cliente Linux:

```bash
sudo apt install -y nfs-common
sudo mount 192.168.100.10:/compartilhado /mnt
ls -la /mnt
sudo umount /mnt
```

---

## 8) Firewall básico (ufw) — permitir apenas o necessário

```bash
sudo apt install -y ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Abrir portas essenciais (ajuste conforme precisar)
sudo ufw allow 22/tcp     # SSH (se quiser)
sudo ufw allow 53         # DNS (TCP/UDP) - ufw separa TCP/UDP: abrir ambos
sudo ufw allow 53/udp
sudo ufw allow 80/tcp     # HTTP
sudo ufw allow 443/tcp    # HTTPS
sudo ufw allow 21/tcp     # FTP controle (se for usar FTP)
sudo ufw allow 30000:30010/tcp  # passive FTP range
sudo ufw allow 2049/tcp   # NFS v4

sudo ufw enable
sudo ufw status numbered
```


---

## 9) Comandos úteis de verificação

```bash
# Ver serviços ativos
sudo systemctl status isc-dhcp-server bind9 apache2 vsftpd nfs-kernel-server --no-pager

# Logs
sudo journalctl -u isc-dhcp-server -n 50
sudo journalctl -u bind9 -n 50
sudo journalctl -u apache2 -n 50

# DNS
dig @192.168.100.10 kaiky-server.kaiky-home.local
dig @192.168.100.10 intranet.kaiky-home.local

# DHCP leases
cat /var/lib/dhcp/dhcpd.leases

# Teste web
curl -v http://192.168.100.10/
```

---

