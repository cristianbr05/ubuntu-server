# 🐧 Ubuntu Server + Samba AD DC - ADSO Cv1

![Ubuntu](https://badgen.net/badge/Ubuntu/24.04%20LTS/E95420?icon=ubuntu)
![Windows](https://badgen.net/badge/OS/Windows/0078D6?icon=windows)
![Samba](https://badgen.net/badge/Samba/4.x/0052CC?icon=terminal)
![AWS](https://badgen.net/badge/AWS/EC2/FF9900?icon=aws)
![Status](https://badgen.net/badge/Status/Complete/green)

> **Guía técnica y manual paso a paso para la configuración de: Controlador de Dominio Active Directory bajo Linux, integración de clientes híbridos, gestión de GPOs, ACLs y auditoría de sistemas.**

| 👨‍💻 Autor | 👨‍🏫 Profesor | 🎓 Curso |
| :--- | :--- | :--- |
| **Cristian Bellmunt Redon** | Gregorio Mateu | 2º ASIX |

---

## Datos de configuración de la MV

| Parámetro | Valor |
|---|---|
| Hostname | ls02 |
| Dominio | lab02.lan |
| IP Bridge WAN (enp0s3) | 172.30.20.26/25 — Gateway: 172.30.20.1 |
| IP Interna LAN (enp0s8) | 192.168.11.2/24 |
| DNS Forwarders | 10.239.3.7, 10.239.3.8 |
| Usuario servidor Ubuntu | cristianbr / admin_21 |
| Usuario Windows cliente | user01 / admin_21 |
| Usuario Ubuntu cliente | user01 / admin_21 |

> **Si algo no funciona**, revisar estos tres archivos:
> - `/etc/netplan/00-installer-config.yaml` → IPs y DNS
> - `/etc/resolv.conf` → nameservers
> - `/etc/samba/smb.conf` → DNS forwarders del AD DC

---

## 🛡️ PRE-REQUISITO: Desactivación completa del Firewall (entorno controlado de pruebas)
> ⚠️ **Importante:** Deshabilitar UFW de forma robusta para evitar bloqueos en los puertos de Samba AD DC, DNS y conflictos con las reglas NAT de iptables.

```bash
# 1. Detener y deshabilitar UFW
sudo ufw disable
sudo systemctl stop ufw
sudo systemctl disable ufw

# 2. Vaciar reglas y eliminar cadenas de la tabla filter
sudo iptables -F
sudo iptables -X

# 3. Asegurar que las políticas por defecto permitan todo el tráfico (Opcional)
sudo iptables -P INPUT ACCEPT
sudo iptables -P FORWARD ACCEPT
sudo iptables -P OUTPUT ACCEPT
```

> 💡 **Nota técnica:** Se incluyen `sudo iptables -F` y `-X` para limpiar cualquier regla o cadena residual que haya quedado en memoria antes de empezar a configurar tus propias reglas de enrutamiento en la Hora 6 del Sprint 1.

---

## ÍNDICE

- [🧱 SPRINT 1 – Configuración Base del Servidor](#sprint-1)
- [🧱 SPRINT 2 – DHCP, Usuarios, Grupos y Carpetas Compartidas](#sprint-2)
- [🧱 SPRINT 3 – Integración de Cliente Windows al Dominio](#sprint-3)
- [🧱 SPRINT 4 – GPOs, Cliente Ubuntu y Gestión del Sistema](#sprint-4)
- [🧱 SPRINT 5 – Forest Trust entre dos servidores Samba AD DC](#sprint-5)

---

<a name="sprint-1"></a>
# 🧱 SPRINT 1 – Configuración Base del Servidor

---

## 🕒 HORA 1: Preparación del sistema y configuración de red

### 1.1 Cambiar hostname
→ Debe mostrar `Static hostname: ls02`
```bash
sudo hostnamectl set-hostname ls02
hostnamectl
```

### 1.2 Configurar Netplan (red estática)

> ⚠️ Si existe `/etc/netplan/50-cloud-init.yaml`, cloud-init puede sobrescribir la red. Desactivarlo primero:

```bash
sudo nano /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
# Contenido: network: {config: disabled}

sudo rm /etc/netplan/50-cloud-init.yaml
```

Editar el archivo de red y aplicar. → Debe mostrar `enp0s3: 172.30.20.26/25` y `enp0s8: 192.168.11.2/24`
```bash
sudo nano /etc/netplan/00-installer-config.yaml
sudo netplan apply
ip addr show
```

Contenido del archivo netplan:
```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 172.30.20.26/25
      routes:
        - to: default
          via: 172.30.20.1
      nameservers:
        addresses:
          - 10.239.3.7
          - 10.239.3.8
    enp0s8:
      dhcp4: false
      addresses:
        - 192.168.11.2/24
```

### 1.3 Actualizar el sistema
> ⚠️ Obligatorio antes de instalar Samba. Puede tardar varios minutos.
```bash
sudo apt update && sudo apt upgrade -y
```

### 1.4 Configurar /etc/hosts
→ `ping -c 2 ls02.lab02.lan` debe responder desde `192.168.11.2`
```bash
sudo nano /etc/hosts
# Añadir:
# 127.0.0.1   localhost
# 127.0.1.1   ls02
# 192.168.11.2   ls02.lab02.lan ls02

ping -c 2 ls02.lab02.lan
```

🛠 **Si falla el ping de comprobación:**
```bash
resolvectl status          # Verificar DNS correctos
ip route show              # Comprobar gateway
sudo netplan --debug apply # Reintentar aplicar netplan viendo errores detallados
```

---

## 🕒 HORA 2: Preparación de DNS y desactivación de systemd-resolved

### 2.1 – 2.2 Detener systemd-resolved y eliminar resolv.conf
> ⚠️ CRÍTICO: systemd-resolved SIEMPRE causa conflictos con Samba AD DC en el puerto 53.

→ Debe mostrar `inactive (dead)` y `disabled`
```bash
sudo systemctl disable --now systemd-resolved
sudo systemctl status systemd-resolved
sudo unlink /etc/resolv.conf
```

### 2.3 Crear resolv.conf temporal y hacerlo inmutable
→ `nslookup www.amazon.es` debe resolver correctamente
```bash
sudo nano /etc/resolv.conf
# Contenido:
# nameserver 10.239.3.7
# nameserver 10.239.3.8
# search lab02.lan

sudo chattr +i /etc/resolv.conf
nslookup www.amazon.es
```

> Para quitar el atributo inmutable después: `sudo chattr -i /etc/resolv.conf`

---

## 🕒 HORA 3: Instalación y preparación de Samba AD DC

### 3.1 Instalar paquetes necesarios
Durante la instalación de Kerberos introducir:
- Default realm: `LAB02.LAN` (mayúsculas)
- Kerberos servers: `ls02.lab02.lan`
- Administrative server: `ls02.lab02.lan`

> Si aparece ventana de configuración automática de Samba: seleccionar "No"
```bash
sudo apt install -y samba smbclient winbind krb5-user krb5-config
```

🛠 **Si la instalación falla por dependencias:**
```bash
sudo apt --fix-broken install
sudo apt install -y samba smbclient winbind krb5-user krb5-config
```

### 3.2 Detener servicios previos, hacer backup de smb.conf e instalar ldb-tools
→ `systemctl status smbd` debe mostrar `inactive (dead)`
```bash
sudo systemctl stop smbd nmbd winbind
sudo systemctl disable smbd nmbd winbind
sudo systemctl status smbd
sudo mv /etc/samba/smb.conf /etc/samba/smb.conf.backup
sudo apt install -y ldb-tools
```

---

## 🕒 HORA 4: Promoción a Controlador de Dominio

### 4.1 Provisionar el dominio
Respuestas al asistente interactivo:
- Realm: `LAB02.LAN`
- Domain: `LAB02`
- Server Role: `dc` (Enter)
- DNS backend: `SAMBA_INTERNAL` (Enter)
- DNS forwarder IP: `10.239.3.7`
- Administrator password: `admin_21`

> ⚠️ Contraseña: mínimo 7 caracteres, mayúsculas, minúsculas y números.
```bash
sudo samba-tool domain provision --use-rfc2307 --interactive
```

### 4.2 Forzar escucha en IPv4 (solución "Connection Refused")
Añadir en la sección `[global]` de smb.conf:
```ini
interfaces = lo enp0s8
bind interfaces only = yes
```
```bash
sudo nano /etc/samba/smb.conf
sudo systemctl restart samba-ad-dc
sudo samba-tool domain level show
```

🛠 Si falla con "DNS zone already exists":
```bash
sudo systemctl stop samba-ad-dc
sudo rm -rf /var/lib/samba/private/*
sudo rm -rf /var/lib/samba/*.tdb
sudo rm /etc/samba/smb.conf
sudo samba-tool domain provision --use-rfc2307 --interactive
```

### 4.3 Copiar Kerberos y actualizar resolv.conf
→ `cat /etc/krb5.conf | grep LAB02.LAN` debe mostrar el dominio. `resolv.conf` debe tener `127.0.0.1` como primero.
```bash
sudo cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
sudo chattr -i /etc/resolv.conf
sudo nano /etc/resolv.conf
# Contenido NUEVO:
# nameserver 127.0.0.1
# nameserver 10.239.3.7
# search lab02.lan

sudo chattr +i /etc/resolv.conf
cat /etc/resolv.conf
```

### 4.4 Iniciar y habilitar Samba AD DC
→ Debe mostrar `active (running)`
```bash
sudo systemctl unmask samba-ad-dc
sudo systemctl start samba-ad-dc
sudo systemctl enable samba-ad-dc
sudo systemctl status samba-ad-dc
```

🛠 Si el puerto 53 está ocupado:
```bash
sudo ss -tulpn | grep :53
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
sudo systemctl start samba-ad-dc
```

---

## 🕒 HORA 5: Configuración de DNS y verificación del dominio

### 5.1 Verificar que Samba DNS funciona
→ Esperar:
- `lab02.lan` → `192.168.11.2`
- `ls02.lab02.lan` → `192.168.11.2`
- `_ldap._tcp.lab02.lan` → registro SRV apuntando a `ls02.lab02.lan`
```bash
host -t A lab02.lan
host -t A ls02.lab02.lan
host -t SRV _ldap._tcp.lab02.lan
```

🛠 Si no resuelve:
```bash
sudo reboot now
# Tras reinicio:
cat /etc/resolv.conf
sudo journalctl -xeu samba-ad-dc | tail -50
sudo samba-tool dns query 127.0.0.1 lab02.lan @ ALL -U Administrator%admin_21
```

### 5.2 Configurar DNS forwarder en smb.conf
> ⚠️ Samba NO crea forwarders automáticamente aunque se especifique en provision.

Añadir en `[global]`: `dns forwarder = 10.239.3.7`

→ `nslookup www.amazon.es 127.0.0.1` debe resolver correctamente
```bash
sudo samba-tool dns serverinfo 127.0.0.1 -U Administrator%admin_21
sudo nano /etc/samba/smb.conf
sudo systemctl restart samba-ad-dc
nslookup www.amazon.es 127.0.0.1
```

### 5.3 Probar autenticación Kerberos
→ `klist` debe mostrar ticket válido para `Administrator@LAB02.LAN`
```bash
kinit Administrator@LAB02.LAN
# Contraseña: admin_21
klist
```

🛠 Si falla "Clock skew too great":
```bash
sudo timedatectl set-ntp true
timedatectl                      # Verificar zona horaria
kinit Administrator@LAB02.LAN
```

🛠 Si falla "Cannot find KDC for realm":
```bash
sudo cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
```

---

## 🕒 HORA 6: Configuración de enrutamiento NAT y políticas de contraseñas

### 6.1 – 6.2 Activar IP forwarding y configurar NAT
→ `ip_forward` debe devolver `1`. Regla `MASQUERADE` visible en `POSTROUTING`.
```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
cat /proc/sys/net/ipv4/ip_forward
sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
sudo iptables -A FORWARD -i enp0s8 -o enp0s3 -j ACCEPT
sudo iptables -A FORWARD -i enp0s3 -o enp0s8 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo apt install -y iptables-persistent
# Durante instalación: guardar IPv4 → Yes, IPv6 → Yes
sudo iptables -t nat -L -v
```

🛠 Si las reglas no persisten tras reinicio:
```bash
sudo iptables-save | sudo tee /etc/iptables/rules.v4
sudo systemctl enable netfilter-persistent
```

### 6.3 – 6.4 Ver y modificar políticas de contraseñas
```bash
sudo samba-tool domain passwordsettings show

# Longitud mínima de contraseña:
sudo samba-tool domain passwordsettings set --min-pwd-length=8

# Complejidad de contraseña (requiere mayúsculas, minúsculas, números):
sudo samba-tool domain passwordsettings set --complexity=on

# Historial de contraseñas (cuántas contraseñas anteriores se recuerdan):
sudo samba-tool domain passwordsettings set --history-length=12

# Edad máxima de contraseña (días antes de expirar):
sudo samba-tool domain passwordsettings set --max-pwd-age=60

# Edad mínima de contraseña (días antes de poder cambiar):
sudo samba-tool domain passwordsettings set --min-pwd-age=0

# Duración del bloqueo de cuenta (minutos):
sudo samba-tool domain passwordsettings set --account-lockout-duration=30

# Número de intentos incorrectos antes de bloqueo:
sudo samba-tool domain passwordsettings set --account-lockout-threshold=5

# Ventana de observación para intentos fallidos (minutos):
sudo samba-tool domain passwordsettings set --reset-account-lockout-after=15

sudo samba-tool domain passwordsettings show
```

### 6.5 Verificación integral del dominio
```bash
sudo samba-tool domain level show
sudo samba-tool user list
sudo samba-tool group list
sudo samba-tool dns query ls02.lab02.lan lab02.lan @ ALL -U Administrator%admin_21
```

---

## ✅ CHECKPOINT SPRINT 1

```bash
hostnamectl                                  # → ls02
ip addr show                                 # → enp0s3 172.30.20.26, enp0s8 192.168.11.2
ping -c 2 www.amazon.es                      # → funciona
sudo systemctl status systemd-resolved       # → inactive (dead) disabled
host lab02.lan                               # → 192.168.11.2
sudo systemctl status samba-ad-dc            # → active (running)
klist                                        # → ticket Administrator@LAB02.LAN
sudo iptables -t nat -L                      # → regla MASQUERADE
nslookup www.amazon.es 127.0.0.1             # → resuelve
host -t SRV _ldap._tcp.lab02.lan             # → registro SRV
```

## 🛠 PLAN DE RESCATE SPRINT 1

Si el dominio está completamente roto:
```bash
sudo systemctl stop samba-ad-dc
sudo rm -rf /var/lib/samba/private/*
sudo rm -rf /var/lib/samba/*.tdb
sudo rm /etc/samba/smb.conf
sudo samba-tool domain provision --use-rfc2307 --interactive
```

Si DNS no resuelve nada:
```bash
cat /etc/resolv.conf                   # → debe tener nameserver 127.0.0.1
sudo systemctl status samba-ad-dc
sudo ss -tulpn | grep :53              # → solo debe aparecer samba
dig @127.0.0.1 lab02.lan
```

---

## 🎯 FIN DEL SPRINT 1
- ✅ Red dual (puente + interna), ✅ systemd-resolved eliminado, ✅ Samba AD DC instalado
- ✅ Dominio lab02.lan creado, ✅ DNS + Kerberos, ✅ NAT, ✅ Políticas de contraseñas

---

<a name="sprint-2"></a>
# 🧱 SPRINT 2 – DHCP, Usuarios, Grupos y Carpetas Compartidas

> ⚠️ Usar siempre `Administrator` (mayúscula) — es el usuario por defecto de Samba.

---

## 🕒 HORA 1: Instalación y configuración del servidor DHCP

### 1.1 – 1.2 Instalar DHCP y configurar interfaz
Es normal que falle al iniciar (aún sin configurar). Editar `INTERFACESv4=""` → `INTERFACESv4="enp0s8"`
```bash
sudo apt install -y isc-dhcp-server
sudo nano /etc/default/isc-dhcp-server
```

### 1.3 Configurar rango DHCP
Añadir al final de `/etc/dhcp/dhcpd.conf`:
```bash
sudo nano /etc/dhcp/dhcpd.conf
```
```
subnet 192.168.11.0 netmask 255.255.255.0 {
    range 192.168.11.100 192.168.11.150;
    option domain-name "lab02.lan";
    option subnet-mask 255.255.255.0;
    option domain-name-servers 192.168.11.2;
    option routers 192.168.11.2;
    option broadcast-address 192.168.11.255;
    default-lease-time 600;
    max-lease-time 7200;
}
```

**Explicación rápida:**
- **Rango** → .100 a .150 (51 IPs disponibles)
- **DNS** → el propio servidor (192.168.11.2)
- **Gateway** → el propio servidor (192.168.11.2)
- **Lease** → 10 minutos por defecto, máximo 2 horas

### 1.4 – 1.6 Verificar sintaxis, iniciar y comprobar
→ Sintaxis sin errores. Servicio debe mostrar `active (running)`.
```bash
sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf
sudo systemctl restart isc-dhcp-server
sudo systemctl enable isc-dhcp-server
sudo systemctl status isc-dhcp-server
cat /var/lib/dhcp/dhcpd.leases
```

🛠 Si falla al iniciar:
```bash
sudo journalctl -xeu isc-dhcp-server
sudo touch /var/lib/dhcp/dhcpd.leases
sudo systemctl restart isc-dhcp-server
```

---

## 🕒 HORA 2: Creación de Unidades Organizativas (OUs)

### 2.1 – 2.2 Verificar dominio y crear OUs
→ `samba-tool ou list` debe mostrar las 4 OUs.
```bash
sudo samba-tool domain level show
sudo samba-tool ou create "OU=IT_Department,DC=lab02,DC=lan"
sudo samba-tool ou create "OU=HR_Department,DC=lab02,DC=lan"
sudo samba-tool ou create "OU=Students,DC=lab02,DC=lan"
sudo samba-tool ou create "OU=Groups,DC=lab02,DC=lan"
sudo samba-tool ou list
```

🛠 Para eliminar una OU (si hay error "Already exists"):
```bash
sudo samba-tool ou delete "OU=nombre,DC=lab02,DC=lan"
# Con objetos dentro (CUIDADO):
sudo samba-tool ou delete "OU=nombre,DC=lab02,DC=lan" --force-subtree-delete
```

---

## 🕒 HORA 3: Creación de usuarios en sus respectivas OUs

### 3.1 – 3.4 Crear todos los usuarios
Bob en IT (con cambio de contraseña obligatorio), Alice en HR, user01-03 en Students, techsupport en CN=Users (sin --userou).
```bash
sudo samba-tool user create bob admin_21 --userou="OU=IT_Department" --given-name="Bob" --surname="Smith" --must-change-at-next-login
sudo samba-tool user create alice admin_21 --userou="OU=HR_Department" --given-name="Alice" --surname="Johnson"
sudo samba-tool user create user01 admin_21 --userou="OU=Students" --given-name="User" --surname="One"
sudo samba-tool user create user02 admin_21 --userou="OU=Students" --given-name="User" --surname="Two"
sudo samba-tool user create user03 admin_21 --userou="OU=Students" --given-name="User" --surname="Three"
sudo samba-tool user create techsupport admin_21 --given-name="Tech" --surname="Support"
```

> ⚠️ **IMPORTANTE:** lee lo que hace `--must-change-at-next-login` si realmente lo quieres usar. A los estudiantes no se les pone para facilitar pruebas.

**Parámetros explicados:**
- `bob` → nombre de usuario
- `admin_21` → contraseña inicial
- `--userou` → especifica la OU donde se creará (sin esto, va al contenedor Users)
- `--must-change-at-next-login` → obliga a cambiar contraseña en el primer inicio

### 3.5 Verificar usuarios y OUs
→ Debe listar: Administrator, bob, alice, user01, user02, user03, techsupport
```bash
sudo samba-tool user list
sudo samba-tool user show bob
sudo ldbsearch -H /var/lib/samba/private/sam.ldb "(sAMAccountName=bob)" dn
```

🛠 **Si falla "Constraint violation" (contraseña no cumple política):**
```bash
sudo samba-tool domain passwordsettings set --complexity=off
sudo samba-tool user create bob admin_21 --userou="OU=IT_Department"
sudo samba-tool domain passwordsettings set --complexity=on
```

🛠 **Si un usuario se creó en la OU incorrecta:**
```bash
sudo samba-tool user delete bob
sudo samba-tool user create bob admin_21 --userou="OU=IT_Department"
```

---

## 🕒 HORA 4: Creación de grupos de seguridad y asignación de miembros

### 4.1 Crear grupos de seguridad
→ Debe listar: Finance, HR, IT Support
```bash
sudo samba-tool group add Finance --groupou="OU=Groups"
sudo samba-tool group add HR --groupou="OU=Groups"
sudo samba-tool group add "IT Support" --groupou="OU=Groups"
sudo samba-tool group list | grep -E "Finance|HR|IT Support"
```

🛠 **Si necesitas eliminar un grupo o verificar miembros antes:**
```bash
sudo samba-tool group listmembers Finance
sudo samba-tool group delete Finance
```

### 4.2 Añadir usuarios a grupos y verificar
Asignaciones: user01→Finance, alice→HR, bob→"IT Support", techsupport→"IT Support"
```bash
sudo samba-tool group addmembers Finance user01
sudo samba-tool group addmembers HR alice
sudo samba-tool group addmembers "IT Support" bob
sudo samba-tool group addmembers "IT Support" techsupport
sudo samba-tool group listmembers Finance
sudo samba-tool group listmembers HR
sudo samba-tool group listmembers "IT Support"
```

🛠 **Si necesitas verificar a qué grupos pertenece un usuario o eliminarlo de un grupo:**
```bash
sudo samba-tool user show bob | grep memberOf
sudo samba-tool group removemembers Finance user01
```

---

## 🕒 HORA 5: Creación de carpetas compartidas con ACLs

> Filosofía: Linux configura almacenamiento con permisos base amplios → Windows gestiona las ACLs visualmente.

### 5.1 – 5.2 Crear carpetas e instalar librerías winbind
→ `dpkg -l | grep winbind` debe mostrar `ii libnss-winbind`, `ii libpam-winbind`, `ii winbind`
```bash
sudo mkdir -p /srv/samba/FinanceDocs
sudo mkdir -p /srv/samba/HRDocs
sudo mkdir -p /srv/samba/Public
sudo apt-get install -y libnss-winbind libpam-winbind
sudo ldconfig
dpkg -l | grep winbind
```

### 5.3 Configurar winbind en smb.conf
Añadir en `[global]`:
```ini
winbind use default domain = yes
template shell = /bin/bash
template homedir = /home/%U
```
→ `sudo testparm -s | grep winbind` debe mostrar `winbind use default domain = yes`
```bash
sudo nano /etc/samba/smb.conf
sudo systemctl restart samba-ad-dc
sudo testparm -s | grep winbind
```

### 5.4 Configurar permisos base en Linux
→ `ls -la /srv/samba/` debe mostrar `drwxrwx--- root Domain Users` en cada carpeta
```bash
sudo chown root:"Domain Users" /srv/samba/FinanceDocs
sudo chown root:"Domain Users" /srv/samba/HRDocs
sudo chown root:"Domain Users" /srv/samba/Public
sudo chmod 770 /srv/samba/FinanceDocs
sudo chmod 770 /srv/samba/HRDocs
sudo chmod 770 /srv/samba/Public
ls -la /srv/samba/
```

🛠 Si falla "Domain Users: invalid group" (winbind no resuelve grupos aún):
```bash
sudo chown root:root /srv/samba/FinanceDocs /srv/samba/HRDocs /srv/samba/Public
sudo chmod 777 /srv/samba/FinanceDocs /srv/samba/HRDocs /srv/samba/Public
# Seguro: Samba controlará el acceso real mediante ACLs de Windows
```

### 5.5 – 5.6 Configurar recursos compartidos en smb.conf, verificar y reiniciar
Añadir al final del archivo (después de `[netlogon]` y `[sysvol]`):
```ini
[FinanceDocs]
    path = /srv/samba/FinanceDocs
    read only = no
    vfs objects = acl_xattr
    map acl inherit = yes

[HRDocs]
    path = /srv/samba/HRDocs
    read only = no
    vfs objects = acl_xattr
    map acl inherit = yes

[Public]
    path = /srv/samba/Public
    read only = no
    guest ok = yes
    vfs objects = acl_xattr
    map acl inherit = yes
```

**Explicación de parámetros:**
- `vfs objects = acl_xattr` → Habilita soporte completo de ACLs NTFS (vital para gestionar desde Windows).
- `map acl inherit = yes` → Permite heredar permisos estilo Windows.
- `guest ok = yes` → Solo en Public, permite acceso sin autenticación.

```bash
sudo nano /etc/samba/smb.conf
sudo testparm
sudo smbcontrol all reload-config
sudo smbclient -L localhost -U Administrator%admin_21
```
→ `testparm` debe decir `Loaded services file OK`  
→ `smbclient -L` debe listar FinanceDocs, HRDocs, Public.

### 5.7 Pruebas básicas de acceso desde Linux
> ⚠️ Todos los usuarios pueden acceder a todo aún — las restricciones se configuran desde Windows en Sprint 3.

**Prueba 1: "user01" a "FinanceDocs"**
```bash
sudo smbclient //localhost/FinanceDocs -U user01%admin_21
# Dentro del prompt smb: \>
# mkdir test_inicial
# ls
# exit
```

**Prueba 2: "alice" a "HRDocs"**
```bash
sudo smbclient //localhost/HRDocs -U alice%admin_21
# Dentro del prompt smb: \>
# mkdir test_hr_inicial
# ls
# exit
```

## 🛠 PLAN DE RESCATE SPRINT 2
```bash
# Si los recursos no aparecen:
sudo tail -100 /var/log/samba/log.smbd
sudo ufw status
```

---

## ✅ CHECKPOINT SPRINT 2

```bash
ls -la /srv/samba/                                                   # → 3 carpetas
sudo testparm -s | grep "winbind use default domain"                 # → yes
sudo testparm -s | grep "vfs objects"                                # → acl_xattr
sudo smbclient -L localhost -U Administrator%admin_21                # → recursos listados
sudo smbclient //localhost/FinanceDocs -U user01%admin_21 -c "ls"   # → sin error
```

---

## 🎯 FIN DEL SPRINT 2
- ✅ DHCP (192.168.11.100-150), ✅ 4 OUs, ✅ 7 usuarios, ✅ 3 grupos (Finance, HR, IT Support)
- ✅ 3 carpetas compartidas con ACLs habilitadas, ✅ Samba funcionando

**Siguiente:** SPRINT 3 → Integración de clientes Ubuntu y Windows al dominio

---

<a name="sprint-3"></a>
# 🧱 SPRINT 3 – Integración de Cliente Windows al Dominio

---

## 📋 PREPARACIÓN DEL CLIENTE WINDOWS

**Requisitos:** Windows 10 **Pro/Enterprise/Education** (Home NO puede unirse a dominio), 4GB RAM, 2 CPUs, 50GB disco. Nombre VM: `lc02`

**VirtualBox — Adaptador 1:**
- Conectado a: Red interna
- Nombre: `intnet`
- Modo promiscuo: Permitir todo

**Configuración TCP/IPv4 en Windows:**
- IP: `192.168.11.100` | Máscara: `255.255.255.0`
- Gateway: `192.168.11.2` | DNS preferido: `192.168.11.2`

### Estado inicial — verificar antes de empezar
→ Todo debe responder correctamente
```cmd
ipconfig /all
hostname
nslookup lab02.lan
ping 192.168.11.2
ping ls02.lab02.lan
ping www.amazon.es
```

---

## 🕒 HORA 1: Unir el cliente Windows al dominio

### 1.1 – 1.2 Unir al dominio
Pasos:
1. Clic derecho "Este equipo" → Propiedades → Configuración avanzada del sistema
2. Pestaña "Nombre de equipo" → Cambiar → Miembro de: **Dominio** → `lab02.lan`
3. Credenciales: Usuario `Administrator`, Contraseña `admin_21`
4. Mensaje "Bienvenido al dominio lab02.lan" → Reiniciar

→ Tras reinicio: pantalla de login debe mostrar "Iniciar sesión en: LAB02"

🛠 Errores frecuentes:
```cmd
REM "No se puede encontrar el dominio" → verificar DNS
nslookup lab02.lan
nslookup _ldap._tcp.lab02.lan
ipconfig /flushdns

REM "No se puede conectar al dominio" → verificar LDAP
Test-NetConnection -ComputerName ls02.lab02.lan -Port 389
```
```bash
# Desde servidor Ubuntu
sudo systemctl status samba-ad-dc
```

### 1.3 Verificar cuenta de equipo desde el servidor
→ Debe aparecer `LC02$`
```bash
sudo samba-tool computer list
sudo samba-tool computer show LC02$
```

---

## 🕒 HORA 2: Iniciar sesión con usuarios del dominio

### 2.1 – 2.2 Cerrar sesión e iniciar como user01
→ `whoami` debe devolver `lab02\user01`
```cmd
whoami
echo %USERDOMAIN%
echo %USERNAME%
```

### 2.3 Probar con otros usuarios
Cerrar sesión y probar:
- `bob` / `admin_21`
- `alice` / `admin_21`

🛠 Si falla inicio de sesión:
```bash
sudo samba-tool user list | grep user01
sudo samba-tool user show user01 | grep -i "disabled\|locked"
sudo samba-tool user enable user01
sudo samba-tool user setpassword user01
```

---

## 🕒 HORA 3: Acceso a recursos compartidos desde Windows

### 3.1 Acceder a recursos del servidor como user01
En la barra de direcciones del Explorador: `\\ls02.lab02.lan` o `\\192.168.11.2`
Debe mostrar: FinanceDocs, HRDocs, Public, NETLOGON, SYSVOL

Crear archivo de prueba en FinanceDocs → verificar desde servidor:
```bash
sudo ls -la /srv/samba/FinanceDocs/
# → debe aparecer test_user01.txt
```

### 3.2 Mapear FinanceDocs como unidad Z:
"Este equipo" → Conectar a unidad de red → Unidad `Z:` → `\\ls02.lab02.lan\FinanceDocs` → ✓ Reconectar al iniciar sesión

→ Debe mostrar `Z: \\ls02.lab02.lan\FinanceDocs`
```cmd
net use
```

---

## 🕒 HORA 4: Configurar permisos (ACLs) desde Windows

> Iniciar sesión como `Administrator` / `admin_21`

### 4.2 FinanceDocs
1. `\\ls02.lab02.lan\FinanceDocs` → clic derecho → Propiedades → Seguridad → Opciones avanzadas
2. Deshabilitar herencia → "Reemplazar todas las entradas..."
3. Eliminar todos excepto: Administradores del dominio, SYSTEM, CREATOR OWNER
4. Añadir grupo **Finance** → Control total → Permitir → Esta carpeta, subcarpetas y archivos
5. Añadir grupo **HR** → Control total → **Denegar**

### 4.3 HRDocs
Mismo proceso:
- Añadir **HR** → Control total → Permitir
- Añadir **Finance** → Control total → Denegar

### 4.4 Public
1. `\\ls02.lab02.lan\Public` → Propiedades → Seguridad → Opciones avanzadas
2. Deshabilitar herencia (NO marcar "Reemplazar...")
3. Resultado final:
   - Administrator → Control total
   - Domain Users → Lectura y ejecución
   - CREATOR OWNER → Control total (subcarpetas/archivos)

> **NOTA:** Si sale un aviso de Windows mediante una ventana que dice algo como:  

> *"Seguridad de Windows. La directiva de supervisión actual de este equipo no tiene activada la auditoría. Si este equipo obtiene la directiva de auditoría del dominio..."*

> Ignoralo, no hagas caso, básicamente es como si Windows te estuviera diciendo: "Oye, si quisieras registrar estos accesos en los logs, ahora mismo no lo estoy haciendo."

---

## 🕒 HORA 5: Verificar restricciones de acceso

### 5.1 Como user01 (grupo Finance)
- ✅ FinanceDocs → debe abrir, crear `test_acl_user01.txt`
- ❌ HRDocs → debe mostrar "No tiene permiso para acceder"
- ✅ Public → abre pero no puede crear archivos

### 5.2 Como alice (grupo HR)
- ✅ HRDocs → debe abrir, crear `test_acl_alice.txt`
- ❌ FinanceDocs → debe mostrar error de permisos

### 5.3 Verificar ACLs desde el servidor
```bash
sudo getfacl /srv/samba/FinanceDocs/
sudo getfacl /srv/samba/HRDocs/
```

---

## 🕒 HORA 6: Verificación final del SPRINT 3

```bash
# Desde servidor Ubuntu
sudo samba-tool computer list         # → LC02$
```
```cmd
REM Desde cliente Windows
systeminfo | findstr /B /C:"Dominio"  & REM → lab02.lan
net view \\ls02.lab02.lan             & REM → FinanceDocs, HRDocs, Public, NETLOGON, SYSVOL
gpresult /r
```

**ACLs esperadas:**

| Usuario | FinanceDocs | HRDocs | Public |
|---|---|---|---|
| user01 (Finance) | ✅ Accede | ❌ Denegado | ✅ Solo lectura |
| alice (HR) | ❌ Denegado | ✅ Accede | ✅ Solo lectura |

---

## 🛠 PLAN DE RESCATE SPRINT 3

```bash
# Cliente no se une al dominio
nslookup lab02.lan
sudo tail -100 /var/log/samba/log.samba

# Usuario no puede iniciar sesión
sudo samba-tool user setpassword user01

# ACLs no funcionan
sudo testparm -s | grep vfs
sudo systemctl restart samba-ad-dc
```

---

## 🎯 FIN DEL SPRINT 3
- ✅ Cliente Windows configurado y unido a lab02.lan
- ✅ Usuarios del dominio inician sesión, ✅ Recursos compartidos accesibles
- ✅ ACLs configuradas desde Windows, ✅ Restricciones de acceso por grupo verificadas

**Siguiente:** SPRINT 4 → GPOs, Cliente Ubuntu y Gestión del Sistema

---

<a name="sprint-4"></a>
# 🧱 SPRINT 4 – GPOs, Cliente Ubuntu y Gestión del Sistema

---

## 🕒 HORA 1: Configuración de GPOs (Comandos Ubuntu + RSAT Windows)

> **Estructura:**
> - **Parte A** — Crear GPOs vacías y vincularlas desde Ubuntu (solo estructura)
> - **Parte B** — Configurar políticas reales con RSAT desde Windows

---

## 📋 PARTE A: CREACIÓN DE GPOs DESDE UBUNTU

> ⚠️ Limitación: `samba-tool` solo crea GPOs vacías. El contenido real se configura con RSAT.

### A.1 – A.2 Autenticar con Kerberos y ver GPOs existentes
→ `klist` debe mostrar ticket de `Administrator@LAB02.LAN`
```bash
kdestroy
kinit Administrator
# Contraseña: admin_21
klist
sudo samba-tool gpo listall
```

### A.3 – A.4 Crear GPOs vacías
→ Anotar los GUIDs generados (necesarios para vincular)
```bash
sudo samba-tool gpo create "Restricciones_Usuarios" -U Administrator
sudo samba-tool gpo create "Configuracion_Escritorio" -U Administrator
sudo samba-tool gpo listall
```

### A.6 Vincular GPOs a OU=Students
Sustituir `{GUID_...}` por los GUIDs obtenidos en el paso anterior.

→ `gpo getlink` debe mostrar ambos GUIDs vinculados
```bash
sudo samba-tool gpo setlink "OU=Students,DC=lab02,DC=lan" "{GUID_DE_Restricciones_Usuarios}" -U Administrator
sudo samba-tool gpo setlink "OU=Students,DC=lab02,DC=lan" "{GUID_DE_Configuracion_Escritorio}" -U Administrator
sudo samba-tool gpo getlink "OU=Students,DC=lab02,DC=lan"
```

### A.7 Verificar políticas de contraseñas (ya configuradas en Sprint 1)
```bash
sudo samba-tool domain passwordsettings show
```

### A.8 – A.9 Verificar estructura física y checkpoint Parte A
```bash
ls -la /var/lib/samba/sysvol/lab02.lan/Policies/
sudo samba-tool gpo listall | grep -E "Restricciones|Configuracion"
sudo samba-tool gpo getlink "OU=Students,DC=lab02,DC=lan"
```

---

## 📋 PARTE B: CONFIGURACIÓN DE GPOs DESDE WINDOWS (RSAT)

### B.1 Instalar RSAT en Windows 10
Inicio → Configuración → Aplicaciones → Características opcionales → Agregar:
- `RSAT: Herramientas de administración de directivas de grupo`
- `RSAT: Herramientas de AD DS y AD LDS`

→ Buscar "Administración de directivas de grupo" en el menú inicio — debe aparecer.

🛠 Si no aparece en Características opcionales:
```powershell
Get-WindowsCapability -Name RSAT* -Online | Add-WindowsCapability -Online
```

### B.2 Abrir Consola de GPOs y verificar
Inicio → "Administración de directivas de grupo" → Bosque: lab02.lan → Dominios → lab02.lan → Objetos de directiva de grupo
→ Deben aparecer `Restricciones_Usuarios` y `Configuracion_Escritorio`

### B.3 Configurar GPO: Prohibir Panel de Control
Clic derecho `Restricciones_Usuarios` → Editar:
```
Configuración de usuario
  → Directivas → Plantillas administrativas
    → Panel de Control → Personalización
      → "Prohibit access to Control Panel and PC settings" → Habilitado
```

### B.4 Configurar GPO: Fondo de Escritorio
Clic derecho `Configuracion_Escritorio` → Editar:
```
Configuración de usuario → Directivas → Plantillas administrativas
  → Panel de Control → Personalización → "Impedir cambiar el fondo de pantalla"

Configuración de usuario → Directivas → Plantillas administrativas
  → Active Desktop → Active Desktop → "Tapiz del escritorio"
```

### B.6 – B.7 Forzar aplicación y verificar desde cliente (como user01)
→ Panel de Control debe estar bloqueado. Reporte HTML debe listar ambas GPOs.
```cmd
gpupdate /force
gpresult /r
gpresult /h C:\gpo_report.html
```

### B.8 GPOs adicionales útiles (configurar desde RSAT)

| GPO | Ruta | Configuración |
|---|---|---|
| Bloquear CMD | User Config → Admin Templates → System | Prevent access to the command prompt → Habilitado |
| Deshabilitar Task Manager | User Config → Admin Templates → System → Ctrl+Alt+Del Options | Remove Task Manager → Habilitado |
| Bloquear Registro | User Config → Admin Templates → System | Prevent access to registry editing tools → Habilitado |
| Ocultar unidades | User Config → Admin Templates → Windows Components → File Explorer | Hide these specified drives → Habilitado |

---

## 🎯 CHECKPOINT HORA 1

**Parte A:** GPOs creadas y vinculadas desde Ubuntu ✅  
**Parte B:** RSAT instalado, GPOs configuradas y aplicadas ✅

```bash
sudo samba-tool gpo getlink "OU=Students,DC=lab02,DC=lan"   # → GUIDs de ambas GPOs
```

## 🛠 Troubleshooting GPOs

```bash
# Error de permisos al editar GPO desde RSAT
sudo samba-tool ntacl sysvolreset
sudo systemctl restart samba-ad-dc

# GPO no se aplica en clientes → desde Windows
# gpupdate /force → eventvwr.msc → Windows Logs → System → filtrar "Group Policy"

# RSAT no muestra GPOs → verificar desde servidor
sudo samba-tool gpo listall
```

---

## 🕒 HORA 2: Preparación y unión del cliente Ubuntu al dominio

**Requisitos VM:** Ubuntu 22.04/24.04, nombre `lc02-ubu`, 2GB RAM, 20GB disco, Adaptador 1: Red Interna (intnet)

### 2.1 – 2.3 Configurar red, hostname y /etc/hosts
→ `ping lab02.lan` y `ping ls02.lab02.lan` deben responder desde `192.168.11.2`
```bash
sudo nano /etc/netplan/01-netcfg.yaml
sudo netplan apply
sudo hostnamectl set-hostname lc02-ubu
sudo nano /etc/hosts
# Añadir:
# 127.0.0.1    localhost
# 127.0.1.1    lc02-ubu
# 192.168.11.2 ls02.lab02.lan ls02
# 192.168.11.2 lab02.lan

ping lab02.lan
ping ls02.lab02.lan
```

Contenido netplan:
```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 192.168.11.101/24
      routes:
        - to: default
          via: 192.168.11.2
      nameservers:
        addresses:
          - 192.168.11.2
          - 10.239.3.7
```

### 2.4 – 2.5 Instalar paquetes y descubrir dominio
Durante Kerberos: realm `LAB02.LAN`, servers y admin `ls02.lab02.lan`
→ `realm discover` debe mostrar `configured: no` y `server-software: active-directory`
```bash
sudo apt update
sudo apt install -y realmd sssd sssd-tools adcli krb5-user samba-common-bin packagekit
sudo realm discover lab02.lan
```

### 2.6 Unir al dominio y verificar
→ `realm list` debe mostrar `configured: kerberos-member`. Desde servidor debe aparecer `lc02-ubu$`.
```bash
sudo realm join -U Administrator lab02.lan --verbose
# Contraseña: admin_21
sudo realm list
```
```bash
# Desde servidor Ubuntu
sudo samba-tool computer list
```

### 2.7 – 2.8 Configurar SSSD y home directories
En `/etc/sssd/sssd.conf`, sección `[domain/lab02.lan]`, añadir:
```ini
fallback_homedir = /home/%u@%d
default_shell = /bin/bash
```
```bash
sudo nano /etc/sssd/sssd.conf
sudo systemctl restart sssd
sudo pam-auth-update --enable mkhomedir
# Seleccionar con ESPACIO: [*] Create home directory on login → Tab → Ok
```

### 2.9 Iniciar sesión con usuario del dominio
→ `whoami` debe mostrar `bob@lab02.lan`, `pwd` debe mostrar `/home/bob@lab02.lan`
```bash
su - bob@lab02.lan
# Contraseña: admin_21
whoami
pwd
```

---

## 🕒 HORA 3: Montaje de recursos compartidos desde Linux

### 3.1 – 3.2 Instalar CIFS y verificar recursos
→ Debe listar FinanceDocs, HRDocs, Public, NETLOGON, SYSVOL
```bash
sudo apt update
sudo apt install -y cifs-utils smbclient
smbclient -L //ls02.lab02.lan -U bob
# Contraseña: admin_21
```

### 3.3 – 3.4 Probar acceso y montaje manual
→ `ls -la /mnt/financedocs` debe mostrar contenido
```bash
smbclient //ls02.lab02.lan/FinanceDocs -U user01
# Dentro del prompt: ls, mkdir test_from_ubuntu, ls, exit

sudo mkdir -p /mnt/financedocs
sudo mount -t cifs //ls02.lab02.lan/FinanceDocs /mnt/financedocs -o username=user01,password=admin_21,uid=1000,gid=1000
ls -la /mnt/financedocs
echo "Test from Ubuntu client" | sudo tee /mnt/financedocs/test_ubuntu.txt
sudo umount /mnt/financedocs
```

### 3.5 Montaje automático en /etc/fstab
Crear credenciales, protegerlas y añadir entrada en fstab.
→ `df -h | grep financedocs` debe mostrar el recurso montado (también tras reinicio).
```bash
sudo nano /root/.smbcredentials
# Contenido:
# username=user01
# password=admin_21
# domain=LAB02

sudo chmod 600 /root/.smbcredentials
sudo mkdir -p /mnt/financedocs
sudo nano /etc/fstab
# Añadir al final:
# //ls02.lab02.lan/FinanceDocs /mnt/financedocs cifs credentials=/root/.smbcredentials,uid=1000,gid=1000,iocharset=utf8 0 0

sudo mount -a
df -h | grep financedocs
sudo reboot
# Tras reinicio:
df -h | grep financedocs
```

---

## 🕒 HORA 4: Gestión de procesos (actividad práctica con SSH)

### 4.1 – 4.2 Instalar sl y conectar por SSH desde el servidor
```bash
# En el cliente Ubuntu:
sudo apt install -y openssh-server sl
sudo systemctl enable ssh && sudo systemctl start ssh

# Desde el servidor ls02:
ssh bob@192.168.11.101
# Contraseña: admin_21
```

> **NOTA:** Puede ayudar a copiar el PID directamente
```bash
# Opción 1 — Con xclip (X11)
ps aux | awk '$11=="sl"{print $2}' | xclip -selection clipboard

# Opción 2 — Con wl-copy (Wayland)
ps aux | awk '$11=="sl"{print $2}' | wl-copy

# Opción 3 — Más robusto (evita falsos positivos), ps aux es ruidoso. Mejor:
pgrep -x sl | xclip -selection clipboard

# o en Wayland:
pgrep -x sl | wl-copy
```

### 4.3 Ejecutar sl desde el cliente (Locomotora en la Terminal)
En otra terminal del cliente (como `bob@lab02.lan`):
```bash
# Tren normal
sl

# El tren es más largo
sl -l

# Todas las opciones del "sl"
sl -h
```

### 4.4 – 4.7 Identificar, pausar, reanudar y matar el proceso desde el servidor (via SSH)
Sustituir `12345` por el PID real obtenido.
```bash
# Identificar PID (2 opciones)
ps -aux | grep -w "sl"
ps aux | awk '$11=="sl"' | grep sl

# Pausar (tren se congela) - Signal 19 = SIGSTOP
kill -19 12345

# Reanudar (tren continúa) - Signal 18 = SIGCONT
kill -18 12345

# Matar - Signal 9 = SIGKILL (terminar inmediatamente, no se puede ignorar)
kill -9 12345
ps aux | grep sl
```

### 4.8 Monitoreo de procesos
```bash
top
# Filtrar por usuario: presionar 'u' → escribir 'bob' → Enter
# Salir: 'q'
```

---

## 🕒 HORA 5: Tareas programadas con CRON (Backup automático de Samba)

### 5.1 – 5.2 Crear script de backup y dar permisos
```bash
sudo nano /root/backup_samba.sh
sudo chmod +x /root/backup_samba.sh
```

Contenido del script:
```bash
#!/bin/bash

# --- CONFIGURACIÓN ---
DIR_DESTINO="/root/backups"
LOG_FILE="/var/log/samba_backup.log"
DIAS_A_GUARDAR=30

# --- COMANDOS (Rutas absolutas para CRON) ---
TAR=/bin/tar
DATE=/bin/date
ECHO=/bin/echo
FIND=/usr/bin/find
MKDIR=/bin/mkdir

# --- VARIABLES ---
FECHA=$($DATE +%F_%H-%M)
NOMBRE_ARCHIVO="backup_ad_$FECHA.tar.gz"
RUTA_COMPLETA="$DIR_DESTINO/$NOMBRE_ARCHIVO"

# --- CREAR DIRECTORIO SI NO EXISTE ---
if [ ! -d "$DIR_DESTINO" ]; then
    $MKDIR -p "$DIR_DESTINO"
fi

# --- 1. EJECUTAR BACKUP ---
$TAR -czf "$RUTA_COMPLETA" /var/lib/samba /etc/samba 2>/dev/null

# --- 2. VERIFICACIÓN Y LOG ---
if [ $? -eq 0 ]; then
    $ECHO "[$FECHA] OK: Backup creado exitosamente: $NOMBRE_ARCHIVO" >> $LOG_FILE
    # --- 3. LIMPIEZA (Solo si el backup salió bien) ---
    $FIND $DIR_DESTINO -name "backup_ad_*.tar.gz" -mtime +$DIAS_A_GUARDAR -delete
else
    $ECHO "[$FECHA] ERROR: Falló la creación del backup. Revisa espacio en disco." >> $LOG_FILE
fi
```

### 5.3 Probar el script manualmente
→ `cat /var/log/samba_backup.log` debe mostrar `OK: Backup creado exitosamente`
```bash
sudo /root/backup_samba.sh
ls -lh /root/backups/
cat /var/log/samba_backup.log
```

### 5.4 Programar con CRON
→ `sudo crontab -l` debe mostrar la línea añadida
```bash
sudo crontab -e
# Añadir al final:
# 0 2 * * * /root/backup_samba.sh

sudo crontab -l
```

### 5.5 Referencia rápida de formato CRON

```
# Cada 6 horas
0 */6 * * * /root/backup_samba.sh

# Cada domingo a las 3:00 AM
0 3 * * 0 /root/backup_samba.sh

# Cada día a las 23:30
30 23 * * * /root/backup_samba.sh
```

```
* * * * * comando
│ │ │ │ └─── Día semana (0-7, 0=Domingo)
│ │ │ └───── Mes (1-12)
│ │ └─────── Día mes (1-31)
│ └───────── Hora (0-23)
└─────────── Minuto (0-59)
```

### 5.6 – 5.7 Verificar CRON y restaurar si es necesario
```bash
sudo grep CRON /var/log/syslog | tail -20
tail -f /var/log/samba_backup.log

# Restaurar backup (PRECAUCIÓN: sobrescribe archivos actuales)
ls -lh /root/backups/
sudo tar -xzf /root/backups/backup_ad_FECHA.tar.gz -C /
```

---

## 🕒 HORA 6: Seguridad y Auditoría (Samba Audit)

### 6.1 Configurar auditoría en smb.conf
Modificar `[FinanceDocs]`, `[HRDocs]` y `[Public]` añadiendo las líneas de auditoría:
```ini
[FinanceDocs]
    path = /srv/samba/FinanceDocs
    read only = no
    vfs objects = acl_xattr full_audit
    map acl inherit = yes
    full_audit:prefix = %u|%I|%m|%S
    full_audit:success = mkdirat renameat unlinkat pwrite
    full_audit:failure = connect
    full_audit:facility = local7
    full_audit:priority = NOTICE
```
```bash
sudo nano /etc/samba/smb.conf
```

### 6.2 – 6.4 Configurar rsyslog, crear log y reiniciar servicios
→ Ambos servicios deben estar `active (running)`
```bash
sudo nano /etc/rsyslog.d/samba-audit.conf
# Contenido:
# local7.notice /var/log/samba_audit.log
# & stop

sudo touch /var/log/samba_audit.log
sudo chown syslog:adm /var/log/samba_audit.log
sudo chmod 640 /var/log/samba_audit.log
sudo systemctl restart rsyslog
sudo smbcontrol all reload-config
sudo systemctl status rsyslog
sudo systemctl status samba-ad-dc
```

### 6.5 Generar eventos de auditoría
Desde cliente Ubuntu:
```bash
smbclient //ls02.lab02.lan/FinanceDocs -U user01
# Dentro del prompt:
# mkdir audit_folder
# rmdir audit_folder
# exit
```

### 6.6 – 6.7 Verificar y filtrar logs
```bash
tail -f /var/log/samba_audit.log
grep "user01" /var/log/samba_audit.log
grep "mkdirat" /var/log/samba_audit.log
grep "fail" /var/log/samba_audit.log
grep "$(date +%Y-%m-%d)" /var/log/samba_audit.log
```

Formato del log: `Usuario | IP Cliente | Nombre Cliente | Recurso | Acción | Resultado | Archivo`

### 6.8 Configurar rotación de logs (recomendado)
```bash
sudo nano /etc/logrotate.d/samba-audit
```
```
/var/log/samba_audit.log {
    daily
    rotate 30
    compress
    delaycompress
    notifempty
    missingok
    create 0640 syslog adm
}
```
**Explicación:**
- `daily`: Rotar cada día
- `rotate 30`: Mantener 30 archivos
- `compress`: Comprimir archivos antiguos
- `create 0640 syslog adm`: Permisos del nuevo archivo

---

## ✅ CHECKPOINT FINAL DEL SPRINT 4

```bash
# Desde cliente Windows (como user01)
gpresult /r                                 # → Restricciones_Usuarios, Configuracion_Escritorio

# Desde cliente Ubuntu
sudo realm list                             # → configured: kerberos-member
df -h | grep financedocs                    # → recurso montado

# Desde servidor
ps aux | grep bob                           # → procesos de bob visibles
sudo crontab -l                             # → backup programado
ls -lh /root/backups/                       # → archivos .tar.gz
cat /var/log/samba_backup.log               # → OK: Backup creado
tail -20 /var/log/samba_audit.log           # → eventos de acceso
```

---

## 🛠 PLAN DE RESCATE SPRINT 4

```bash
# RSAT no se instala → PowerShell como admin en Windows:
# Get-WindowsCapability -Name RSAT* -Online | Add-WindowsCapability -Online

# Cliente Ubuntu no se une al dominio
nslookup lab02.lan
sudo journalctl -xe

# Recursos no montan
cat /root/.smbcredentials
sudo tail -50 /var/log/samba/log.smbd

# CRON no ejecuta
sudo grep CRON /var/log/syslog
sudo /root/backup_samba.sh               # probar manualmente

# Auditoría no registra
sudo testparm -s | grep full_audit
sudo systemctl status rsyslog
sudo systemctl restart samba-ad-dc
```

---

## 🎯 FIN DEL SPRINT 4
- ✅ GPOs creadas desde Ubuntu y configuradas con RSAT
- ✅ Cliente Ubuntu unido al dominio lab02.lan
- ✅ Recursos compartidos montados automáticamente vía fstab
- ✅ Gestión remota de procesos vía SSH
- ✅ Backup automático con CRON + limpieza automática
- ✅ Auditoría completa de accesos a Samba

**Estado final:**

| Componente | Estado |
|---|---|
| Servidor Ubuntu | DC completo con auditoría |
| Cliente Windows | Unido, GPOs aplicadas, ACLs configuradas |
| Cliente Ubuntu | Unido, recursos montados, gestión remota |
| Automatización | Backups diarios programados |
| Seguridad | Auditoría de todos los accesos |

**Siguiente:** SPRINT 5 → Forest Trust entre dominios

---

<a name="sprint-5"></a>
# 🧱 SPRINT 5 – Forest Trust entre dos servidores Samba AD DC

**Objetivo:** Establecer un Forest Trust bidireccional entre dos servidores Samba AD DC en VirtualBox.

---

## 📋 ARQUITECTURA

```
SERVIDOR 1 (ya existente)         SERVIDOR 2 (nuevo)
──────────────────────            ──────────────────────
IP:       192.168.11.2            IP:       192.168.11.3
Hostname: ls02                    Hostname: ls03
Dominio:  lab02.lan               Dominio:  lab03.lan
Realm:    LAB02.LAN               Realm:    LAB03.LAN
```

---

## 🌐 CONFIGURACIÓN DE RED EN VIRTUALBOX

### Opción A: Red Interna (RECOMENDADO)

Ambos servidores en red privada aislada con IPs estáticas 192.168.11.X. NAT en Adaptador 1 para Internet, Red Interna en Adaptador 2 para comunicación entre servidores.

**Servidor 1 (ls02):**
- Adaptador 1: NAT
- Adaptador 2: Red interna — nombre `intnet` — IP `192.168.11.2/24`

**Servidor 2 (ls03):**
- Adaptador 1: NAT
- Adaptador 2: Red interna — nombre `intnet` (MISMO que Servidor 1) — IP `192.168.11.3/24`

### Opción B: Adaptador Puente — ALTERNATIVA

Cada servidor obtiene IP del router físico. Más expuestos, menos aislados. IPs según DHCP o estáticas en el rango de la red física.

> ⚠️ Esta guía usa **Red Interna (Opción A)**.

---

## 🖥️ PARTE 1: Crear y configurar Servidor 2

### Crear nueva VM en VirtualBox
Parámetros:
- Nombre: `Servidor2-Samba` | Tipo: Linux | Versión: Ubuntu (64-bit)
- RAM: 2048 MB mínimo | Disco: VDI, 20 GB
- Instalar Ubuntu Server 24.04

### Configurar adaptadores de red (VM apagada)
En VirtualBox → clic derecho VM → Configuración → Red:
- **Adaptador 1:** NAT ✅
- **Adaptador 2:** Red interna ✅ — nombre `intnet`

### Paso 1: Configurar red estática en Servidor 2
→ Debe mostrar `inet 192.168.11.3/24`
```bash
sudo nano /etc/netplan/01-netcfg.yaml
sudo netplan apply
ip addr show enp0s8
```

Contenido del netplan:
```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: false
      addresses:
        - 192.168.11.3/24
      nameservers:
        addresses:
          - 127.0.0.1
          - 10.239.3.7
```

### Paso 2: Verificar conectividad entre servidores
→ Ambos pings deben funcionar
```bash
# Desde Servidor 2:
ping -c 2 192.168.11.2

# Desde Servidor 1:
ping -c 2 192.168.11.3
```

### Paso 3: Configurar hostname y /etc/hosts en Servidor 2
→ `hostnamectl` debe mostrar `ls03`
```bash
sudo hostnamectl set-hostname ls03
hostnamectl
sudo nano /etc/hosts
```

Contenido de /etc/hosts:
```
127.0.0.1       localhost
127.0.1.1       ls03.lab03.lan ls03
192.168.11.3    ls03.lab03.lan ls03
192.168.11.2    ls02.lab02.lan ls02
```

---

## 🌐 PARTE 2: Instalar Samba AD DC en Servidor 2

### Paso 4: Actualizar e instalar Samba
Durante Kerberos: realm `LAB03.LAN`, servers y admin `ls03.lab03.lan`
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y samba smbclient winbind krb5-user krb5-config
```

### Paso 5: Deshabilitar systemd-resolved y crear resolv.conf
```bash
sudo systemctl disable --now systemd-resolved
sudo unlink /etc/resolv.conf
sudo nano /etc/resolv.conf
# Contenido:
# nameserver 127.0.0.1
# nameserver 10.239.3.7
# search lab03.lan

sudo chattr +i /etc/resolv.conf
```

### Paso 6: Detener servicios Samba por defecto y hacer backup de smb.conf
```bash
sudo systemctl stop smbd nmbd winbind
sudo systemctl disable smbd nmbd winbind
sudo mv /etc/samba/smb.conf /etc/samba/smb.conf.bak 2>/dev/null || true
```

### Paso 7: Provisionar el dominio lab03.lan
Respuestas al asistente:
- Realm: `LAB03.LAN` | Domain: `LAB03` | Server Role: `dc`
- DNS backend: `SAMBA_INTERNAL`
- DNS forwarder: `10.239.3.7 192.168.11.2`
- Administrator password: `admin_21`

→ Debe mostrar `Provision OK for domain DN DC=lab03,DC=lan`
```bash
sudo samba-tool domain provision --use-rfc2307 --interactive
```

### Paso 8: Copiar Kerberos, configurar interfaces e iniciar Samba
Añadir en `[global]` de smb.conf: `interfaces = lo enp0s8` y `bind interfaces only = yes`

→ Debe mostrar `active (running)`
```bash
sudo cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
sudo nano /etc/samba/smb.conf
sudo systemctl unmask samba-ad-dc
sudo systemctl start samba-ad-dc
sudo systemctl enable samba-ad-dc
sudo systemctl status samba-ad-dc
```

### Paso 9: Verificar DNS y Kerberos en Servidor 2
→ `host` debe resolver `192.168.11.3`. `klist` debe mostrar ticket de `Administrator@LAB03.LAN`.
```bash
host lab03.lan
host ls03.lab03.lan
host -t SRV _ldap._tcp.lab03.lan
kinit Administrator
# Contraseña: admin_21
klist
```

---

## 🔗 PARTE 3: Configurar Servidor 1 para el Trust

### Paso 10: Añadir /etc/hosts y DNS forwarder en Servidor 1
En Servidor 1 (ls02), añadir en `/etc/hosts`: `192.168.11.3    ls03.lab03.lan ls03 lab03.lan`

En `[global]` de smb.conf modificar: `dns forwarder = 10.239.3.7 192.168.11.3`
```bash
# En Servidor 1:
sudo nano /etc/hosts
sudo nano /etc/samba/smb.conf
sudo systemctl restart samba-ad-dc
```

### Paso 11: Configurar DNS forwarder en Servidor 2
En `[global]` de smb.conf modificar: `dns forwarder = 10.239.3.7 192.168.11.2`
```bash
# En Servidor 2:
sudo nano /etc/samba/smb.conf
sudo systemctl restart samba-ad-dc
```

---

## 📋 PARTE 4: Verificar resolución DNS cruzada

### Paso 12: Servidor 1 resuelve Servidor 2
→ Cada comando debe devolver `192.168.11.3` (o registro SRV apuntando a `ls03.lab03.lan`)
```bash
# En Servidor 1:
host lab03.lan
host ls03.lab03.lan
host -t SRV _ldap._tcp.lab03.lan
```

### Paso 13: Servidor 2 resuelve Servidor 1
→ Cada comando debe devolver `192.168.11.2` (o registro SRV apuntando a `ls02.lab02.lan`)
```bash
# En Servidor 2:
host lab02.lan
host ls02.lab02.lan
host -t SRV _ldap._tcp.lab02.lan
```

> ✅ Si ambos servidores resuelven correctamente, continúa con el Trust.

---

## 🤝 PARTE 5: Crear Forest Trust bidireccional

### Paso 14: Crear Trust desde Servidor 1 → Servidor 2
Cuando pida contraseña del Administrator remoto: `admin_21`

→ Debe mostrar `Successfully created trust`
```bash
# En Servidor 1:
kdestroy
kinit Administrator@LAB02.LAN
sudo samba-tool domain trust create lab03.lan \
    --type=forest \
    --direction=both \
    -U Administrator%admin_21
```

### Paso 15: Crear Trust desde Servidor 2 → Servidor 1
Cuando pida contraseña del Administrator remoto: `admin_21`

→ Debe mostrar `Successfully created trust`
```bash
# En Servidor 2:
kdestroy
kinit Administrator@LAB03.LAN
sudo samba-tool domain trust create lab02.lan \
    --type=forest \
    --direction=both \
    -U Administrator%admin_21
```

---

## ✅ PARTE 6: Verificar Trust

### Paso 16: Verificar Trust en Servidor 1
→ Debe mostrar `lab03.lan | forest | both | yes`
```bash
# En Servidor 1:
sudo samba-tool domain trust list
sudo samba-tool domain trust show lab03.lan
```

### Paso 17: Verificar Trust en Servidor 2
→ Debe mostrar `lab02.lan | forest | both | yes`
```bash
# En Servidor 2:
sudo samba-tool domain trust list
sudo samba-tool domain trust show lab02.lan
```

---

## ✅ CHECKPOINT FINAL

```bash
# En Servidor 1:
sudo samba-tool domain trust list         # → Trust con lab03.lan activo
host lab03.lan                            # → 192.168.11.3
sudo systemctl status samba-ad-dc | grep Active
klist

# En Servidor 2:
sudo samba-tool domain trust list         # → Trust con lab02.lan activo
host lab02.lan                            # → 192.168.11.2
sudo systemctl status samba-ad-dc | grep Active
klist
```

---

## 🛠️ TROUBLESHOOTING

### Error: "Failed to find a writeable DC"
Causa: DNS no resuelve correctamente. Puede haber zonas DNS locales incorrectas.
```bash
# Verificar zonas DNS locales
sudo samba-tool dns zonelist 127.0.0.1 -U Administrator%admin_21

# Si aparece la zona del otro dominio (ej. lab03.lan en Servidor 1), eliminarla
sudo samba-tool dns zonedelete 127.0.0.1 lab03.lan -U Administrator%admin_21

sudo systemctl restart samba-ad-dc
host lab03.lan
```

### Error: "The object name is not found"
```bash
cat /etc/hosts | grep lab03
ping -c 2 192.168.11.3
# En Servidor 2, verificar que Samba esté corriendo:
sudo systemctl status samba-ad-dc
```

### Trust se crea pero no aparece en la lista
```bash
sudo samba-tool domain trust delete lab03.lan -U Administrator%admin_21
kdestroy
sudo systemctl restart samba-ad-dc
sleep 10
kinit Administrator
sudo samba-tool domain trust create lab03.lan --type=forest --direction=both -U Administrator%admin_21
```

### Diferencia de tiempo Kerberos (Clock skew)
```bash
sudo timedatectl set-ntp true
timedatectl status
```

---

## 📝 Red Interna vs Bridge — referencia rápida

| | Red Interna | Bridge |
|---|---|---|
| IPs | 192.168.11.X (fijas, definidas por ti) | Según router (192.168.1.X o similar) |
| Aislamiento | ✅ Aislados de la red física | ❌ Expuestos a la red |
| Internet | ❌ Necesitan NAT en Adaptador 1 | ✅ Directo sin NAT |
| Comunicación entre VMs | ✅ Directa sin problemas | ✅ Directa |

---

## 🎯 FIN DEL SPRINT 5

- ✅ VM Servidor 2 creada y configurada en VirtualBox
- ✅ Samba AD DC instalado en Servidor 2 (lab03.lan)
- ✅ Red interna entre ambos servidores
- ✅ DNS Forwarders bidireccionales configurados
- ✅ Resolución DNS cruzada verificada
- ✅ Forest Trust bidireccional establecido y verificado en ambos servidores

**Arquitectura final:**
```
Servidor 1 (ls02.lab02.lan — 192.168.11.2)
            ↕ Forest Trust bidireccional
Servidor 2 (ls03.lab03.lan — 192.168.11.3)
```
