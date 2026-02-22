# AWS Academy: Servidor Ubuntu Samba + Cliente Windows (Guía Completa)

**Objetivo:** Desplegar servidor Ubuntu con Samba AD DC y cliente Windows Server en AWS EC2 con conectividad completa y acceso RDP desde Linux.

---

## 📋 ARQUITECTURA FINAL

```
┌─────────────────────────────────────────────────────┐
│              AWS EC2 - LAB VPC                      │
│                                                     │
│  ┌─────────────────────┐  ┌─────────────────────┐ │
│  │  Ubuntu Server      │  │  Windows Server     │ │
│  │  Samba AD DC        │  │  Cliente unido      │ │
│  │                     │  │                     │ │
│  │  IP Pública: X.X.X  │  │  IP Pública: Y.Y.Y  │ │
│  │  IP Privada: 10.0.X │  │  IP Privada: 10.0.Y │ │
│  │  Dominio: awslab.lan│  │                     │ │
│  └─────────────────────┘  └─────────────────────┘ │
│           ↕                         ↕              │
│     Security Group LAB-SG                          │
└─────────────────────────────────────────────────────┘
           ↕                         ↕
    SSH desde Linux          RDP desde Linux
      (puerto 22)             (puerto 3389)
```

---

## 🚀 PARTE 1: Preparar AWS Learner Lab

### Paso 1: Iniciar laboratorio

```
1. Entra a https://awsacademy.instructure.com/
2. Inicia sesión con tu email de estudiante
3. Clic en tu curso (AWS Academy Learner Lab)
4. Menú izquierdo → "Modules" → "Learner Lab"
5. Clic en "Start Lab" (botón verde)
6. Espera 1-3 minutos hasta que el círculo esté 🟢 verde
7. Clic en "AWS" (abre consola de AWS)
```

---

### Paso 2: Descargar par de claves SSH

```
1. En la página del Learner Lab, clic en "AWS Details" (arriba)
2. Clic en "Download PEM" (al lado de SSH Key)
3. Se descarga "labsuser.pem" en Descargas
4. Guardar en lugar seguro
```

**Preparar la clave en Linux:**
```bash
# Mover a ~/.ssh/
mv ~/Descargas/labsuser.pem ~/.ssh/

# Dar permisos correctos
chmod 400 ~/.ssh/labsuser.pem
```

---

## 🔐 PARTE 2: Configurar Security Group

### Paso 3: Crear Security Group compartido

**En la consola de AWS:**

```
1. Buscar: "VPC"
2. Clic en "VPC" (servicio)
3. Menú izquierdo → "Security Groups"
4. Clic en "Create security group"
```

**Configuración:**
```
Name: LAB-SG
Description: Security group for Samba AD DC and Windows client
VPC: Seleccionar la VPC del lab (ejemplo: Lab VPC o vpc-XXXXXXX)
```

---

### Paso 4: Añadir reglas de entrada (Inbound rules)

**Clic en "Add rule" para cada una:**

| Tipo | Puerto | Protocolo | Origen | Descripción |
|------|--------|-----------|--------|-------------|
| SSH | 22 | TCP | 0.0.0.0/0 | SSH desde cualquier lugar |
| RDP | 3389 | TCP | 0.0.0.0/0 | RDP desde cualquier lugar |
| Custom TCP | 53 | TCP | 10.0.0.0/20 | DNS (TCP) interno |
| Custom UDP | 53 | UDP | 10.0.0.0/20 | DNS (UDP) interno |
| Custom TCP | 88 | TCP | 10.0.0.0/20 | Kerberos interno |
| Custom UDP | 88 | UDP | 10.0.0.0/20 | Kerberos UDP interno |
| Custom TCP | 389 | TCP | 10.0.0.0/20 | LDAP interno |
| Custom TCP | 445 | TCP | 10.0.0.0/20 | SMB/CIFS interno |
| Custom TCP | 636 | TCP | 10.0.0.0/20 | LDAPS interno |
| Custom TCP | 3268 | TCP | 10.0.0.0/20 | Global Catalog interno |
| Custom TCP | 3269 | TCP | 10.0.0.0/20 | Global Catalog SSL |
| All traffic | All | All | 10.0.0.0/20 | Comunicación interna VPC |

⚠️ **IMPORTANTE:** 
- `10.0.0.0/20` es el rango de la VPC interna (ajustar según tu VPC)
- SSH y RDP desde `0.0.0.0/0` para acceso desde tu casa
- Puertos AD solo accesibles internamente (seguridad)

**Clic en "Create security group"**

---

## 🖥️ PARTE 3: Crear instancia Ubuntu Server

### Paso 5: Lanzar instancia Ubuntu

**En la consola de AWS:**

```
1. Buscar: "EC2"
2. Clic en "EC2"
3. Menú izquierdo → "Instances"
4. Clic en "Launch instances"
```

---

### Paso 6: Configurar instancia Ubuntu

| Campo | Valor |
|-------|-------|
| **Nombre** | `ubuntu-samba-server` |
| **AMI** | Ubuntu Server 24.04 LTS |
| **Tipo de instancia** | `t3.micro` (2 vCPU, 2 GiB RAM) |
| **Par de claves** | `vockey` (o el que descargaste) |

---

**Configuración de red:**

```
1. En "Network settings", clic en "Edit"

2. VPC: Seleccionar la VPC del lab (Lab VPC)

3. Subnet: Cualquier subnet pública (ejemplo: subnet-public1)

4. Auto-assign public IP: ✅ Enable

5. Security group: Select existing security group
   - Seleccionar: LAB-SG (el que creamos antes)
```

---

**Storage:**
```
Size: 20 GiB
Type: gp3
```

**Advanced details:**
```
IAM instance profile: LabInstanceProfile
```

**Clic en "Launch instance"**

---

### Paso 7: Asignar Elastic IP al servidor Ubuntu

⚠️ **¿Por qué Elastic IP?** Las IPs públicas normales cambian al reiniciar la instancia. Las Elastic IPs son fijas.

```
1. EC2 → Menú izquierdo → "Elastic IPs"
2. Clic en "Allocate Elastic IP address"
3. Clic en "Allocate"
4. Seleccionar la Elastic IP recién creada
5. Actions → Associate Elastic IP address
6. Instance: Seleccionar "ubuntu-samba-server"
7. Private IP: Dejar la que aparece
8. Clic en "Associate"
```

📝 **Anotar:**
```
Elastic IP del servidor Ubuntu: ___.___.___.___ (ejemplo: 54.173.102.89)
```

---

### Paso 8: Obtener IP privada del servidor

```
1. EC2 → Instances
2. Seleccionar "ubuntu-samba-server"
3. Panel inferior "Details" → Anotar:
   - Private IPv4 addresses (ejemplo: 10.0.1.226)
```

📝 **Anotar:**
```
IP privada del servidor Ubuntu: 10.0.___.___
```

---

## 💻 PARTE 4: Crear instancia Windows Server

### Paso 9: Lanzar instancia Windows

```
1. EC2 → Instances → Launch instances
2. Nombre: windows-client
```

---

### Paso 10: Configurar instancia Windows

| Campo | Valor |
|-------|-------|
| **Nombre** | `windows-client` |
| **AMI** | Microsoft Windows Server 2022 Base |
| **Tipo de instancia** | `t3.micro` (2 vCPU, 2 GiB RAM) |
| **Par de claves** | `vockey` (el mismo que Ubuntu) |

---

**Configuración de red:**

```
1. En "Network settings", clic en "Edit"

2. VPC: MISMA VPC que el servidor Ubuntu

3. Subnet: MISMA subnet que el servidor Ubuntu (o cualquier pública de la VPC)

4. Auto-assign public IP: ✅ Enable

5. Security group: Select existing security group
   - Seleccionar: LAB-SG (el mismo que Ubuntu)
```

---

**Storage:**
```
Size: 30 GiB
Type: gp3
```

**Advanced details:**
```
IAM instance profile: LabInstanceProfile
```

**Clic en "Launch instance"**

---

### Paso 11: Asignar Elastic IP a Windows

```
1. EC2 → Elastic IPs → Allocate Elastic IP address
2. Allocate
3. Seleccionar la nueva Elastic IP
4. Actions → Associate Elastic IP address
5. Instance: Seleccionar "windows-client"
6. Clic en "Associate"
```

📝 **Anotar:**
```
Elastic IP del Windows: ___.___.___.___ (ejemplo: 54.221.100.222)
```

---

### Paso 12: Obtener IP privada de Windows

```
1. EC2 → Instances → Seleccionar "windows-client"
2. Panel inferior → Anotar:
   - Private IPv4 addresses (ejemplo: 10.0.14.107)
```

📝 **Anotar:**
```
IP privada del Windows: 10.0.___.___
```

---

### Paso 13: Obtener contraseña de Administrator de Windows

⚠️ **IMPORTANTE:** Espera 5-7 minutos después de lanzar la instancia antes de hacer esto.

```
1. EC2 → Instances → Seleccionar "windows-client"
2. Botón "Connect" (arriba)
3. Pestaña "RDP client"
4. Clic en "Get password"
5. Clic en "Upload private key file"
6. Seleccionar: labsuser.pem (de ~/.ssh/)
7. Clic en "Decrypt password"
8. Copiar la contraseña que aparece
```

📝 **Anotar:**
```
Usuario Windows: Administrator
Contraseña Windows: _________________ (ejemplo: xY9!mK2@pL5#qR8)
```

---

## 🔗 PARTE 5: Conectar por RDP desde Linux

### Paso 14: Instalar FreeRDP en tu máquina Linux

**Desde tu Linux Mint local:**

```bash
# Instalar FreeRDP
sudo apt update
sudo apt install -y freerdp2-x11

# Verificar instalación
xfreerdp --version
```

---

### Paso 15: Conectar por RDP al Windows Server

**Conectar con xfreerdp:**

```bash
xfreerdp /v:54.221.100.222 \
         /u:Administrator \
         /p:'xY9!mK2@pL5#qR8' \
         /cert:ignore \
         /dynamic-resolution \
         /clipboard
```

**Cambiar:**
- `54.221.100.222` → Tu Elastic IP de Windows
- `xY9!mK2@pL5#qR8` → Tu contraseña de Windows

**Explicación de parámetros:**
```
/v:          → IP del servidor Windows
/u:          → Usuario (Administrator)
/p:          → Contraseña (entre comillas simples)
/cert:ignore → Ignorar certificado SSL
/dynamic-resolution → Ajustar resolución automáticamente
/clipboard   → Compartir portapapeles
```

---

### Paso 16: Primera configuración en Windows

**Una vez dentro del RDP:**

**1. Esperar configuración inicial (1-2 minutos)**

**2. Cambiar contraseña a algo más simple:**

Abrir PowerShell (como Administrator):
```
Clic derecho en Inicio → Windows PowerShell (Admin)
```

Ejecutar:
```powershell
# Cambiar contraseña a admin_21
net user Administrator admin_21
```

**3. Configurar teclado español:**

```powershell
# Configurar teclado español
Set-WinUserLanguageList -LanguageList es-ES -Force
```

**4. Permitir ICMP (ping) en el firewall:**

```powershell
# Permitir ping
netsh advfirewall firewall add rule name="ICMP Allow" protocol=icmpv4:8,any dir=in action=allow
```

**Reiniciar la sesión RDP:**
```
Inicio → Reiniciar
```

---

### Paso 17: Reconectar por RDP con nueva contraseña

```bash
xfreerdp /v:54.221.100.222 \
         /u:Administrator \
         /p:'admin_21' \
         /cert:ignore \
         /dynamic-resolution \
         /clipboard
```

✅ Ahora la contraseña es más simple: `admin_21`

---

## 🔧 PARTE 6: Configurar servidor Ubuntu con Samba

### Paso 18: Conectar por SSH al servidor Ubuntu

**Desde tu Linux local:**

```bash
ssh -i ~/.ssh/labsuser.pem ubuntu@54.173.102.89
```

**Cambiar `54.173.102.89` por tu Elastic IP de Ubuntu.**

---

### Paso 19: Actualizar sistema

```bash
sudo apt update
sudo apt upgrade -y
```

---

### Paso 20: Configurar hostname

```bash
sudo hostnamectl set-hostname samba-server
```

---

### Paso 21: Configurar /etc/hosts

```bash
sudo nano /etc/hosts
```

**Contenido (cambiar 10.0.1.226 por tu IP privada):**
```
127.0.0.1       localhost
127.0.1.1       samba-server.awslab.lan samba-server

# IP privada de esta instancia
10.0.1.226      samba-server.awslab.lan samba-server
```

**Guardar:** Ctrl+O, Enter, Ctrl+X

---

### Paso 22: Deshabilitar systemd-resolved

⚠️ **IMPORTANTE:** AWS usa DHCP y systemd-resolved interfiere con el DNS de Samba.

```bash
# Deshabilitar systemd-resolved
sudo systemctl disable --now systemd-resolved

# Eliminar enlace simbólico
sudo unlink /etc/resolv.conf
```

---

### Paso 23: Crear /etc/resolv.conf manual

```bash
sudo nano /etc/resolv.conf
```

**Contenido:**
```
nameserver 127.0.0.1
nameserver 8.8.8.8
search awslab.lan
```

**Guardar y hacer inmutable:**
```bash
sudo chattr +i /etc/resolv.conf
```

---

### Paso 24: Instalar Samba y dependencias

```bash
sudo apt install -y acl attr samba samba-dsdb-modules samba-vfs-modules \
  winbind libpam-winbind libnss-winbind libpam-krb5 krb5-config krb5-user \
  dnsutils ldap-utils
```

**Configuración Kerberos:**
```
Default realm: AWSLAB.LAN
Kerberos servers: samba-server.awslab.lan
Administrative server: samba-server.awslab.lan
```

---

### Paso 25: Detener servicios Samba por defecto

```bash
sudo systemctl stop smbd nmbd winbind
sudo systemctl disable smbd nmbd winbind
```

**Respaldar smb.conf:**
```bash
sudo mv /etc/samba/smb.conf /etc/samba/smb.conf.bak 2>/dev/null || true
```

---

### Paso 26: Provision del dominio

```bash
sudo samba-tool domain provision --use-rfc2307 --interactive
```

**Respuestas:**
```
Realm: AWSLAB.LAN (presionar Enter)
Domain: AWSLAB (presionar Enter)
Server Role: dc (presionar Enter)
DNS backend: SAMBA_INTERNAL (presionar Enter)
DNS forwarder: 8.8.8.8
Administrator password: Admin_21
Retype password: Admin_21
```

**Debe decir:**
```
Provision OK for domain DN DC=awslab,DC=lan
```

---

### Paso 27: Copiar krb5.conf e iniciar Samba

```bash
# Copiar configuración Kerberos
sudo cp /var/lib/samba/private/krb5.conf /etc/krb5.conf

# Iniciar Samba AD DC
sudo systemctl unmask samba-ad-dc
sudo systemctl start samba-ad-dc
sudo systemctl enable samba-ad-dc

# Verificar estado
sudo systemctl status samba-ad-dc
```

Debe estar `active (running)`.

---

### Paso 28: Verificar DNS y Kerberos

```bash
# DNS
host awslab.lan
host samba-server.awslab.lan
host -t SRV _ldap._tcp.awslab.lan

# Kerberos
kinit Administrator
# Contraseña: Admin_21
klist
```

✅ Todo debe funcionar correctamente.

---

### Paso 29: Crear carpetas compartidas

```bash
# Crear carpetas
sudo mkdir -p /srv/samba/FinanceDocs
sudo mkdir -p /srv/samba/HRDocs
sudo mkdir -p /srv/samba/Public

# Permisos
sudo chmod 777 /srv/samba/FinanceDocs
sudo chmod 777 /srv/samba/HRDocs
sudo chmod 755 /srv/samba/Public
```

---

### Paso 30: Configurar recursos compartidos en smb.conf

```bash
sudo nano /etc/samba/smb.conf
```

**Añadir al final:**
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

**Guardar y recargar:**
```bash
sudo smbcontrol all reload-config
```

---

## 🌐 PARTE 7: Verificar conectividad

### Paso 31: Desde Ubuntu, hacer ping a Windows

```bash
# Ping usando IP privada de Windows
ping -c 4 10.0.14.107
```

**Cambiar `10.0.14.107` por tu IP privada de Windows.**

✅ Debe responder correctamente.

---

### Paso 32: Desde Windows, hacer ping a Ubuntu

**Abrir PowerShell en Windows (RDP):**

```powershell
# Ping usando IP privada de Ubuntu
ping 10.0.1.226
```

**Cambiar `10.0.1.226` por tu IP privada de Ubuntu.**

✅ Debe responder.

---

### Paso 33: Probar puertos desde Windows

```powershell
# Probar SSH (puerto 22)
Test-NetConnection -ComputerName 10.0.1.226 -Port 22

# Probar LDAP (puerto 389)
Test-NetConnection -ComputerName 10.0.1.226 -Port 389

# Probar SMB (puerto 445)
Test-NetConnection -ComputerName 10.0.1.226 -Port 445
```

✅ Todos deben mostrar `TcpTestSucceeded: True`

---

## 🔐 PARTE 8: Unir Windows al dominio

### Paso 34: Configurar DNS en Windows

**En Windows (RDP), abrir PowerShell como Administrator:**

```powershell
# Obtener nombre de la interfaz de red
Get-NetAdapter
```

**Debe mostrar algo como:**
```
Name                      InterfaceDescription
----                      --------------------
Ethernet                  AWS PV Network Device
```

**Configurar DNS:**
```powershell
# Cambiar DNS a la IP privada del servidor Ubuntu
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses ("10.0.1.226","8.8.8.8")

# Verificar
Get-DnsClientServerAddress -InterfaceAlias "Ethernet" -AddressFamily IPv4
```

**Cambiar `10.0.1.226` por tu IP privada de Ubuntu.**

---

### Paso 35: Verificar resolución DNS desde Windows

```powershell
# Resolver el dominio
nslookup awslab.lan

# Resolver el servidor
nslookup samba-server.awslab.lan
```

✅ Debe resolver correctamente.

---

### Paso 36: Unir Windows al dominio

**Método 1: Desde PowerShell (más rápido):**

```powershell
# Unir al dominio
Add-Computer -DomainName awslab.lan -Credential AWSLAB\Administrator -Restart
```

**Introducir contraseña:** `Admin_21`

---

**Método 2: Desde GUI:**

```
1. Inicio → Buscar: "This PC"
2. Clic derecho → Properties
3. Clic en "Rename this PC (advanced)"
4. Clic en "Change..."
5. Seleccionar "Domain"
6. Escribir: awslab.lan
7. OK
8. Usuario: Administrator
9. Contraseña: Admin_21
10. OK → Reiniciar
```

**El Windows se reiniciará.**

---

### Paso 37: Reconectar y verificar unión al dominio

**Reconectar por RDP:**

```bash
xfreerdp /v:54.221.100.222 \
         /u:Administrator \
         /p:'admin_21' \
         /cert:ignore \
         /dynamic-resolution \
         /clipboard
```

**Verificar en PowerShell:**

```powershell
# Ver información del equipo
systeminfo | findstr /B /C:"Domain"
```

**Debe mostrar:**
```
Domain: awslab.lan
```

✅ Windows unido correctamente al dominio.

---

## 📁 PARTE 9: Crear usuarios y probar acceso

### Paso 38: Crear usuarios en el servidor Ubuntu

**Volver al SSH del servidor Ubuntu:**

```bash
# Crear usuarios
sudo samba-tool user create alice Admin_21 --given-name="Alice" --surname="Finance"
sudo samba-tool user create bob Admin_21 --given-name="Bob" --surname="HR"

# Crear grupos
sudo samba-tool group add Finance
sudo samba-tool group add HR

# Añadir usuarios a grupos
sudo samba-tool group addmembers Finance alice
sudo samba-tool group addmembers HR bob

# Verificar
sudo samba-tool group listmembers Finance
sudo samba-tool group listmembers HR
```

---

### Paso 39: Configurar permisos en smb.conf

```bash
sudo nano /etc/samba/smb.conf
```

**Modificar las secciones:**
```ini
[FinanceDocs]
    path = /srv/samba/FinanceDocs
    read only = no
    valid users = @Finance
    vfs objects = acl_xattr
    map acl inherit = yes

[HRDocs]
    path = /srv/samba/HRDocs
    read only = no
    valid users = @HR
    vfs objects = acl_xattr
    map acl inherit = yes

[Public]
    path = /srv/samba/Public
    read only = no
    guest ok = yes
    vfs objects = acl_xattr
    map acl inherit = yes
```

**Guardar y recargar:**
```bash
sudo smbcontrol all reload-config
```

---

### Paso 40: Iniciar sesión como usuario del dominio en Windows

**En Windows (RDP):**

```
1. Cerrar sesión (Inicio → Icono usuario → Sign out)
2. En la pantalla de login:
   - Clic en "Other user"
   - Usuario: AWSLAB\alice
   - Contraseña: Admin_21
3. Iniciar sesión
```

**Primera vez tardará 1-2 minutos (crea perfil).**

---

### Paso 41: Acceder a recursos compartidos

**Abrir Explorador de archivos (Windows + E):**

**En la barra de direcciones, escribir:**
```
\\samba-server.awslab.lan
```

**O usando IP privada:**
```
\\10.0.1.226
```

**Debe mostrar:**
```
FinanceDocs
Public
```

⚠️ **HRDocs NO aparece** porque alice no está en el grupo HR.

---

### Paso 42: Probar accesos

**Abrir FinanceDocs:**
```
1. Doble clic en FinanceDocs
2. DEBE abrir correctamente
3. Clic derecho → New → Text Document
4. Nombrar: test_alice.txt
5. Abrir y escribir algo
6. Guardar
```

✅ alice puede acceder a FinanceDocs.

---

**Intentar acceder a HRDocs:**
```
Desde el Explorador, escribir en la barra:
\\samba-server.awslab.lan\HRDocs
```

❌ **DEBE denegar acceso** (alice no está en HR).

---

**Abrir Public:**
```
\\samba-server.awslab.lan\Public
```

✅ DEBE abrir (Public es para todos).

---

### Paso 43: Mapear unidad de red

```
1. Explorador de archivos → This PC
2. Menú superior → Computer → Map network drive
3. Drive: Z:
4. Folder: \\samba-server.awslab.lan\FinanceDocs
5. ✅ Reconnect at sign-in
6. Finish
```

**La unidad Z: aparece en This PC.**

---

## ✅ CHECKPOINT FINAL

### Verificaciones en servidor Ubuntu:

```bash
# 1. Samba corriendo
sudo systemctl status samba-ad-dc | grep Active

# 2. Usuarios creados
sudo samba-tool user list

# 3. Grupos creados
sudo samba-tool group list

# 4. Miembros de Finance
sudo samba-tool group listmembers Finance

# 5. DNS funciona
host awslab.lan

# 6. Kerberos funciona
klist
```

---

### Verificaciones en Windows:

```powershell
# 1. Unido al dominio
systeminfo | findstr /B /C:"Domain"

# 2. Usuario actual
whoami

# 3. Información del usuario
net user alice /domain

# 4. Resolución DNS
nslookup samba-server.awslab.lan

# 5. Conectividad
ping 10.0.1.226
Test-NetConnection -ComputerName 10.0.1.226 -Port 445
```

---

### Verificaciones de acceso:

| Usuario | FinanceDocs | HRDocs | Public |
|---------|-------------|--------|--------|
| alice | ✅ Acceso | ❌ Denegado | ✅ Acceso |
| bob | ❌ Denegado | ✅ Acceso | ✅ Acceso |
| Administrator | ✅ Acceso | ✅ Acceso | ✅ Acceso |

---

## 🛠️ TROUBLESHOOTING

### No puedo conectar por RDP

**Verificar:**
```bash
# Security group tiene puerto 3389 abierto
# Elastic IP es correcta
# Contraseña copiada sin espacios extra
# Instancia está "Running"
```

**Probar conexión:**
```bash
# Probar si el puerto está abierto
telnet 54.221.100.222 3389

# Si no tienes telnet:
nc -zv 54.221.100.222 3389
```

---

### Windows no puede unirse al dominio

**En Windows, verificar DNS:**
```powershell
Get-DnsClientServerAddress -InterfaceAlias "Ethernet" -AddressFamily IPv4
```

Debe mostrar la IP privada de Ubuntu como DNS primario.

**Verificar resolución:**
```powershell
nslookup awslab.lan
```

Debe resolver a la IP privada de Ubuntu.

**En Ubuntu, verificar Samba:**
```bash
sudo systemctl status samba-ad-dc
host -t SRV _ldap._tcp.awslab.lan
```

---

### No puedo acceder a carpetas compartidas

**Verificar en Windows:**
```powershell
# Ping al servidor
ping 10.0.1.226

# Puerto SMB abierto
Test-NetConnection -ComputerName 10.0.1.226 -Port 445

# Listar recursos
net view \\10.0.1.226
```

**Verificar en Ubuntu:**
```bash
# Recursos compartidos configurados
sudo smbclient -L localhost -U Administrator%Admin_21

# Permisos de carpetas
ls -la /srv/samba/

# Configuración smb.conf
sudo grep -A 5 "\[FinanceDocs\]" /etc/samba/smb.conf
```

---

### FreeRDP no conecta

**Instalar versión actualizada:**
```bash
sudo apt update
sudo apt install -y freerdp2-x11 freerdp2-shadow-x11
```

**Probar con parámetros mínimos:**
```bash
xfreerdp /v:54.221.100.222 /u:Administrator /p:'admin_21' /cert:ignore
```

**Si sigue fallando, usar Remmina:**
```bash
sudo apt install -y remmina remmina-plugin-rdp
remmina
```

---

### Las IPs cambiaron al reiniciar

**Elastic IPs NO cambian.** Si cambiaron, no son Elastic IPs.

**Verificar:**
```
EC2 → Elastic IPs → Debe haber 2 IPs asociadas
```

**Las IPs privadas (10.0.X.X) SÍ persisten aunque pares/inicies la instancia.**

---

## 💰 COSTOS Y LÍMITES

**Créditos disponibles:** $50-100

**Consumo aproximado:**
```
t3.micro Ubuntu:  ~$0.02/hora
t3.micro Windows: ~$0.02/hora
Elastic IPs:      $0 (mientras estén asociadas a instancias running)

Total: ~$0.04/hora = ~$0.96 por 24 horas
```

**Consejos:**
```
1. PARAR (Stop) las instancias cuando no las uses
2. NO terminar (Terminate) si quieres conservarlas
3. Elastic IPs asociadas a instancias paradas SÍ cuestan dinero
4. El lab se apaga automáticamente (varía según curso)
```

---

## 🎯 RESUMEN FINAL

**Has completado:**
- ✅ Security Group configurado con todos los puertos AD
- ✅ Servidor Ubuntu con Samba AD DC en AWS
- ✅ Windows Server en AWS
- ✅ Elastic IPs persistentes en ambas instancias
- ✅ Conexión RDP desde Linux con FreeRDP
- ✅ Teclado español y contraseña simple en Windows
- ✅ Conectividad bidireccional verificada (ping, puertos)
- ✅ Windows unido al dominio awslab.lan
- ✅ Usuarios y grupos creados
- ✅ Carpetas compartidas con permisos por grupos
- ✅ Acceso verificado (alice → FinanceDocs ✅, HRDocs ❌)
- ✅ Unidades de red mapeadas

**Arquitectura final:**
```
Internet
    ↓
┌───────────────────────────────────────┐
│          AWS Lab VPC                  │
│   Security Group: LAB-SG              │
│                                       │
│  Ubuntu Server          Windows       │
│  ┌─────────────────┐   ┌──────────┐  │
│  │ Samba AD DC     │   │ Cliente  │  │
│  │ awslab.lan      │←──│ RDP      │  │
│  │ 10.0.1.226      │   │ 10.0.14. │  │
│  └─────────────────┘   └──────────┘  │
│    Elastic IP            Elastic IP   │
└───────────────────────────────────────┘
      ↑                      ↑
    SSH (22)             RDP (3389)
   desde Linux         desde Linux
                      (FreeRDP)
```
