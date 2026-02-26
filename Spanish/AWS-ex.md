# EJERCICIO AWS ACADEMY - Examen (Guía Paso a Paso Completa)

**Objetivo:** Crear dominio `cloud02.city` en AWS con carpeta compartida `/city/trap` donde solo `lando` tiene acceso y `boba` está denegado.

**Configuración:**
- Dominio: `cloud02.city`
- NetBIOS Name: `Bespin02`
- Usuarios: `lando` (acceso), `boba` (denegado)
- Carpeta compartida: `/city/trap`
- Crear directorio con iniciales: `CB` (Cristian BR)

---

## 📋 PARTE 1: INICIAR AWS ACADEMY Y DESCARGAR CLAVES

### Paso 1: Entrar a AWS Academy

```
1. Abrir navegador (Chrome/Firefox)
2. Ir a: https://awsacademy.instructure.com/
3. Iniciar sesión con tu email de estudiante
4. Clic en tu curso "AWS Academy Learner Lab"
5. Menú izquierdo → "Módulos" → "Learner Lab"
6. Clic en "Iniciar laboratorio" (botón verde)
7. Espera 1-3 minutos hasta que el círculo esté 🟢 verde
8. Cuando esté verde, clic en "AWS"
```

Se abre la consola de AWS.

---

### Paso 2: Descargar par de claves SSH

```
1. En la página del Learner Lab, clic en "Detalles de AWS" (arriba)
2. Clic en "Descargar PEM" (al lado de "Clave SSH")
3. Se descarga "labsuser.pem" en tu carpeta Descargas
```

**Preparar la clave:**

```bash
# Abrir terminal (Ctrl+Alt+T)
cd ~/Descargas

# Mover a ~/.ssh/
mv labsuser.pem ~/.ssh/

# Dar permisos correctos (OBLIGATORIO)
chmod 400 ~/.ssh/labsuser.pem

# Verificar
ls -la ~/.ssh/labsuser.pem
```

Debe mostrar: `-r-------- 1 tu_usuario tu_usuario ...`

---

## 🔐 PARTE 2: CREAR GRUPO DE SEGURIDAD

### Paso 3: Verificar rango de tu VPC (OPCIONAL)

⚠️ **Puedes saltarte este paso** y usar `0.0.0.0/0` en todas las reglas (más fácil).

**Si quieres ser más específico:**
```
1. En la consola de AWS, buscar: "VPC"
2. Clic en "VPC"
3. Menú izquierdo → "Tus VPCs"
4. Seleccionar "VPC de laboratorio"
5. Ver columna "CIDR IPv4"
6. Anotar el rango (ejemplo: 172.31.0.0/16)
```

📝 **Anotar:**
```
Rango VPC: ___.___.___.___/___ (ejemplo: 172.31.0.0/16)
```

---

### Paso 4: Configurar grupo de seguridad

**Detalles básicos:**
```
Nombre del grupo de seguridad: Cloud02-SG
Descripción: Grupo de seguridad para dominio cloud02.city
VPC: Seleccionar la VPC del laboratorio (VPC de laboratorio o vpc-XXXXXXX)
```

---

### Paso 5: Añadir reglas de entrada

**Clic en "Agregar regla" para cada una de estas:**

| Tipo | Puerto | Protocolo | Origen | Descripción |
|------|--------|-----------|--------|-------------|
| SSH | 22 | TCP | 0.0.0.0/0 | SSH |
| RDP | 3389 | TCP | 0.0.0.0/0 | RDP |
| TCP personalizado | 53 | TCP | 0.0.0.0/0 | DNS TCP |
| UDP personalizado | 53 | UDP | 0.0.0.0/0 | DNS UDP |
| TCP personalizado | 88 | TCP | 0.0.0.0/0 | Kerberos |
| UDP personalizado | 88 | UDP | 0.0.0.0/0 | Kerberos UDP |
| TCP personalizado | 389 | TCP | 0.0.0.0/0 | LDAP |
| TCP personalizado | 445 | TCP | 0.0.0.0/0 | SMB |
| TCP personalizado | 636 | TCP | 0.0.0.0/0 | LDAPS |
| TCP personalizado | 464 | TCP | 0.0.0.0/0 | Contraseña Kerberos |
| UDP personalizado | 464 | UDP | 0.0.0.0/0 | Contraseña Kerberos UDP |
| Todo el tráfico | Todo | Todo | 0.0.0.0/0 | VPC interna |

**⚠️ IMPORTANTE:** 
- `10.0.0.0/16` es el rango interno de la VPC
- Si tu VPC usa otro rango, ajústalo

**Clic en "Crear grupo de seguridad"**

---

## 🖥️ PARTE 3: CREAR INSTANCIA UBUNTU SERVER

### Paso 6: Lanzar instancia Ubuntu

```
1. Buscar: "EC2"
2. Clic en "EC2"
3. Menú izquierdo → "Instancias"
4. Clic en "Lanzar instancias" (botón naranja)
```

---

### Paso 7: Configurar instancia Ubuntu

**Nombre y etiquetas:**
```
Nombre: Ubuntu-DC-Cloud02
```

**Imágenes de aplicaciones y SO:**
```
AMI: Ubuntu Server 24.04 LTS
Arquitectura: 64 bits (x86)
```

**Tipo de instancia:**
```
Tipo de instancia: t3.small (o t2.medium si no hay t3.small)
```

**Par de claves:**
```
Par de claves: vockey (ya existe)
```

---

**Configuración de red → Clic en "Editar":**

```
VPC: VPC de laboratorio (la que tiene el laboratorio)
Subred: Cualquier subred pública (Subred pública 1)
Asignar IP pública automáticamente: Habilitar ✅
Firewall (grupos de seguridad): Seleccionar grupo de seguridad existente
  → Seleccionar: Cloud02-SG
```

---

**Configurar almacenamiento:**
```
Tamaño: 20 GiB
Tipo de volumen: gp3
Eliminar al terminar: ✅
```

---

**Detalles avanzados:**
```
Desplazarse hasta "Perfil de instancia de IAM"
Seleccionar: LabInstanceProfile
```

---

**Clic en "Lanzar instancia"**

---

### Paso 8: Esperar y anotar IPs del Ubuntu

```
1. Clic en "Ver todas las instancias"
2. Espera 2-3 minutos
3. Estado debe ser: 🟢 En ejecución
4. Comprobaciones de estado: ✅ 2/2 comprobaciones aprobadas
5. Seleccionar la instancia "Ubuntu-DC-Cloud02"
6. Panel inferior "Detalles" → Anotar:
```

📝 **Anotar en papel:**
```
Servidor Ubuntu:
  IP pública: ___.___.___.___ (ejemplo: 54.173.102.89)
  IP privada: 10.0.___.___ (ejemplo: 10.0.1.229)
```

---

### Paso 9: Asignar IP elástica al Ubuntu

**¿Por qué?** Para que la IP pública no cambie al reiniciar.

```
1. EC2 → Menú izquierdo → "IPs elásticas"
2. Clic en "Asignar dirección IP elástica"
3. Clic en "Asignar"
4. Seleccionar la IP elástica recién creada (casilla de verificación)
5. Acciones → Asociar dirección IP elástica
6. Instancia: Seleccionar "Ubuntu-DC-Cloud02"
7. IP privada: Dejar la que aparece
8. Clic en "Asociar"
```

📝 **Actualizar nota:**
```
Servidor Ubuntu:
  IP elástica: ___.___.___.___ (nueva IP pública)
  IP privada: 10.0.___.___
```

---

## 💻 PARTE 4: CREAR INSTANCIA WINDOWS SERVER

### Paso 10: Lanzar instancia Windows

```
1. EC2 → Instancias → Lanzar instancias
```

---

### Paso 11: Configurar instancia Windows

**Nombre:**
```
Nombre: Windows-Client-Cloud02
```

**Imágenes de aplicaciones y SO:**
```
AMI: Microsoft Windows Server 2022 Base
Arquitectura: 64 bits (x86)
```

**Tipo de instancia:**
```
Tipo de instancia: t3.small
```

**Par de claves:**
```
Par de claves: vockey (el mismo)
```

---

**Configuración de red → Editar:**

```
VPC: MISMA VPC que Ubuntu (VPC de laboratorio)
Subred: MISMA subred o cualquier pública
Asignar IP pública automáticamente: Habilitar ✅
Grupo de seguridad: Seleccionar existente
  → Seleccionar: Cloud02-SG (el mismo que Ubuntu)
```

---

**Almacenamiento:**
```
Tamaño: 30 GiB
Tipo: gp3
```

**Detalles avanzados:**
```
Perfil de instancia de IAM: LabInstanceProfile
```

**Lanzar instancia**

---

### Paso 12: Esperar y anotar IPs de Windows

```
1. Ver todas las instancias
2. Espera 5-7 minutos (Windows tarda más)
3. Estado: 🟢 En ejecución
4. Comprobaciones de estado: ✅ 2/2 comprobaciones aprobadas
5. Seleccionar "Windows-Client-Cloud02"
6. Panel inferior → Anotar:
```

📝 **Anotar:**
```
Servidor Windows:
  IP pública: ___.___.___.___ (ejemplo: 54.221.100.222)
  IP privada: 10.0.___.___ (ejemplo: 10.0.14.107)
```

---

### Paso 13: Asignar IP elástica a Windows

```
1. EC2 → IPs elásticas → Asignar dirección IP elástica
2. Asignar
3. Seleccionar la nueva IP elástica
4. Acciones → Asociar dirección IP elástica
5. Instancia: Seleccionar "Windows-Client-Cloud02"
6. Asociar
```

📝 **Actualizar:**
```
Servidor Windows:
  IP elástica: ___.___.___.___
  IP privada: 10.0.___.___
```

---

### Paso 14: Obtener contraseña de Windows

⚠️ **ESPERAR 5-7 minutos después de lanzar antes de hacer esto.**

```
1. EC2 → Instancias → Seleccionar "Windows-Client-Cloud02"
2. Botón "Conectar" (arriba)
3. Pestaña "Cliente RDP"
4. Clic en "Obtener contraseña"
5. Clic en "Cargar archivo de clave privada"
6. Navegar a: ~/.ssh/labsuser.pem
7. Seleccionar y abrir
8. Clic en "Descifrar contraseña"
9. COPIAR la contraseña que aparece
```

📝 **Anotar:**
```
Windows Administrator:
  Usuario: Administrator
  Contraseña: _____________________ (ejemplo: xY9!mK2@pL5#qR8)
```

---

## 🔧 PARTE 5: CONFIGURAR UBUNTU SERVER CON SAMBA

### Paso 15: Conectar por SSH al Ubuntu

**Desde tu terminal:**

```bash
ssh -i ~/.ssh/labsuser.pem ubuntu@54.173.102.89
```

**Cambiar `54.173.102.89` por tu Elastic IP de Ubuntu.**

**Si pregunta "Are you sure...?"**
```
Escribir: yes
Presionar Enter
```

Debe aparecer:
```
ubuntu@ip-10-0-X-X:~$
```

✅ Estás dentro del servidor Ubuntu.

---

### Paso 16: Actualizar sistema

```bash
sudo apt update
sudo apt upgrade -y
```

Espera 2-5 minutos.

---

### Paso 17: Configurar hostname

```bash
sudo hostnamectl set-hostname bespin02
```

Verificar:
```bash
hostnamectl
```

Debe mostrar: `Static hostname: bespin02`

---

### Paso 18: Configurar /etc/hosts

```bash
sudo nano /etc/hosts
```

**BORRAR TODO y escribir:**
```
127.0.0.1       localhost
127.0.1.1       bespin02.cloud02.city bespin02

# IP privada de esta instancia (CAMBIAR por la tuya)
10.0.1.229      bespin02.cloud02.city bespin02
```

**⚠️ CAMBIAR `10.0.1.229` por TU IP privada de Ubuntu.**

**Guardar:**
```
Ctrl + O
Enter
Ctrl + X
```

---

### Paso 19: Deshabilitar systemd-resolved

```bash
sudo systemctl disable --now systemd-resolved
sudo unlink /etc/resolv.conf
```

---

### Paso 20: Crear /etc/resolv.conf manual

```bash
sudo nano /etc/resolv.conf
```

**Escribir:**
```
nameserver 127.0.0.1
nameserver 8.8.8.8
search cloud02.city
```

**Guardar:** Ctrl+O, Enter, Ctrl+X

**Hacer inmutable:**
```bash
sudo chattr +i /etc/resolv.conf
```

---

### Paso 21: Instalar Samba y dependencias

```bash
sudo apt install -y samba smbclient winbind krb5-user krb5-config
```

**Durante instalación, aparecen ventanas de Kerberos:**

**Pantalla 1: Default Kerberos realm**
```
Escribir: CLOUD02.CITY
Tab → Ok → Enter
```

**Pantalla 2: Kerberos servers**
```
Escribir: bespin02.cloud02.city
Tab → Ok → Enter
```

**Pantalla 3: Administrative server**
```
Escribir: bespin02.cloud02.city
Tab → Ok → Enter
```

---

### Paso 22: Detener servicios Samba por defecto

```bash
sudo systemctl stop smbd nmbd winbind
sudo systemctl disable smbd nmbd winbind
```

**Respaldar smb.conf (si existe):**
```bash
sudo mv /etc/samba/smb.conf /etc/samba/smb.conf.bak 2>/dev/null || true
```

---

### Paso 23: PROVISION DEL DOMINIO (MUY IMPORTANTE)

```bash
sudo samba-tool domain provision --use-rfc2307 --interactive
```

**Responder EXACTAMENTE así:**

```
Realm [CLOUD02.CITY]:
→ Presionar Enter (acepta CLOUD02.CITY)

Domain [CLOUD02]:
→ Escribir: BESPIN02
→ Presionar Enter

Server Role (dc, member, standalone) [dc]:
→ Presionar Enter (acepta dc)

DNS backend (SAMBA_INTERNAL, BIND9_FLATFILE, BIND9_DLZ, NONE) [SAMBA_INTERNAL]:
→ Presionar Enter (acepta SAMBA_INTERNAL)

DNS forwarder IP address (write 'none' to disable forwarding) [127.0.0.53]:
→ Escribir: 8.8.8.8
→ Presionar Enter

Administrator password:
→ Escribir: admin_21
→ Presionar Enter (NO SE VE mientras escribes)

Retype password:
→ Escribir: admin_21 (de nuevo)
→ Presionar Enter
```

**Espera 10-30 segundos.**

**Debe decir:**
```
Provision OK for domain DN DC=cloud02,DC=city
```

✅ **Si ves "Provision OK", está bien.**

---

### Paso 24: Copiar krb5.conf e iniciar Samba

```bash
# Copiar configuración Kerberos
sudo cp /var/lib/samba/private/krb5.conf /etc/krb5.conf

# Iniciar Samba AD DC
sudo systemctl unmask samba-ad-dc
sudo systemctl start samba-ad-dc
sudo systemctl enable samba-ad-dc
```

**Verificar estado:**
```bash
sudo systemctl status samba-ad-dc
```

Debe mostrar: `Active: active (running)` en verde.

**Presionar `q` para salir.**

---

### Paso 25: Verificar DNS y Kerberos

```bash
# Verificar DNS
host cloud02.city
```
**Debe responder:** `cloud02.city has address 10.0.1.229`

```bash
host bespin02.cloud02.city
```
**Debe responder:** `bespin02.cloud02.city has address 10.0.1.229`

```bash
host -t SRV _ldap._tcp.cloud02.city
```
**Debe responder:** `_ldap._tcp.cloud02.city has SRV record 0 100 389 bespin02.cloud02.city.`

---

```bash
# Verificar Kerberos
kinit Administrator
```

**Pide contraseña:**
```
Password for Administrator@CLOUD02.CITY:
→ Escribir: admin_21
→ Presionar Enter
```

**No muestra nada si salió bien.**

```bash
klist
```

**Debe mostrar:**
```
Ticket cache: FILE:/tmp/krb5cc_1000
Default principal: Administrator@CLOUD02.CITY
```

✅ **Si todo esto funciona, el dominio está OK.**

---

## 👥 PARTE 6: CREAR USUARIOS Y CARPETAS

### Paso 26: Crear usuarios lando y boba

```bash
# Crear usuario lando
sudo samba-tool user create lando admin_21 --given-name="Lando" --surname="Calrissian"

# Crear usuario boba
sudo samba-tool user create boba admin_21 --given-name="Boba" --surname="Fett"
```

**Verificar:**
```bash
sudo samba-tool user list
```

Debe mostrar:
```
Administrator
krbtgt
lando
boba
```

---

### Paso 27: Crear estructura de carpetas

```bash
# Crear carpetas
sudo mkdir -p /city/trap

# Permisos base (temporales, Samba controlará el acceso real)
sudo chmod 777 /city/trap
```

---

### Paso 28: Configurar recurso compartido en smb.conf

```bash
sudo nano /etc/samba/smb.conf
```

**Ir al final del archivo (Ctrl+V varias veces) y añadir:**

```ini
[trap]
    path = /city/trap
    read only = no
    valid users = lando
    vfs objects = acl_xattr
    map acl inherit = yes
```

**⚠️ IMPORTANTE:** `valid users = lando` significa que SOLO lando puede acceder.

**Guardar:** Ctrl+O, Enter, Ctrl+X

---

**Recargar configuración:**
```bash
sudo smbcontrol all reload-config
```

**Verificar sintaxis:**
```bash
sudo testparm
```

Debe decir: `Loaded services file OK.`

---

### Paso 29: Verificar recurso compartido

```bash
sudo smbclient -L localhost -U Administrator%admin_21
```

**Debe mostrar:**
```
Sharename       Type      Comment
---------       ----      -------
trap            Disk
netlogon        Disk
sysvol          Disk
```

✅ El recurso `trap` está configurado.

---

## 🔗 PARTE 7: CONECTAR POR RDP AL WINDOWS

### Paso 30: Instalar FreeRDP (si no lo tienes)

**En tu terminal LOCAL (no en el SSH):**

```bash
# Abrir nueva terminal (Ctrl+Alt+T)
sudo apt update
sudo apt install -y freerdp2-x11
```

---

### Paso 31: Conectar por RDP

```bash
xfreerdp /v:54.221.100.222 \
         /u:Administrator \
         /p:'xY9!mK2@pL5#qR8' \
         /cert:ignore \
         /dynamic-resolution \
         /clipboard
```

**CAMBIAR:**
- `54.221.100.222` → Tu Elastic IP de Windows
- `xY9!mK2@pL5#qR8` → Tu contraseña de Windows

**Se abre ventana de escritorio de Windows.**

---

### Paso 32: Configurar Windows (primera vez)

**Abrir PowerShell como Administrador:**
```
Clic derecho en Inicio → Windows PowerShell (Administrador)
```

**Cambiar contraseña a algo simple:**
```powershell
net user Administrator admin_21
```

**Configurar teclado español:**
```powershell
Set-WinUserLanguageList -LanguageList es-ES -Force
```

**Permitir ping:**
```powershell
netsh advfirewall firewall add rule name="ICMP Allow" protocol=icmpv4:8,any dir=in action=allow
```

**Cerrar sesión RDP:**
```
Inicio → Icono usuario → Cerrar sesión
```

---

### Paso 33: Reconectar con nueva contraseña

```bash
xfreerdp /v:54.221.100.222 \
         /u:Administrator \
         /p:'admin_21' \
         /cert:ignore \
         /dynamic-resolution \
         /clipboard
```

---

## 🌐 PARTE 8: CONFIGURAR WINDOWS PARA EL DOMINIO

### Paso 34: Añadir servidor Ubuntu al archivo hosts de Windows

**En Windows (RDP), abrir Bloc de notas como Administrador:**

```
Inicio → Buscar: notepad
Clic derecho en Bloc de notas → Ejecutar como administrador
Archivo → Abrir
Navegar a: C:\Windows\System32\drivers\etc\
Cambiar filtro de "Documentos de texto" a "Todos los archivos"
Abrir: hosts
```

**Añadir al final:**
```
10.0.1.229      bespin02.cloud02.city bespin02 cloud02.city
```

**⚠️ CAMBIAR `10.0.1.229` por TU IP privada de Ubuntu.**

**Guardar:** Archivo → Guardar

**Cerrar Bloc de notas.**

---

### Paso 35: Configurar DNS en Windows

**Abrir PowerShell como Administrator:**

```powershell
# Ver adaptadores de red
Get-NetAdapter
```

Debe mostrar algo como:
```
Name                      InterfaceDescription
----                      --------------------
Ethernet                  AWS PV Network Device
```

**Configurar DNS:**
```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses ("10.0.1.229","8.8.8.8")
```

**⚠️ CAMBIAR `10.0.1.229` por TU IP privada de Ubuntu.**

**Verificar:**
```powershell
Get-DnsClientServerAddress -InterfaceAlias "Ethernet" -AddressFamily IPv4
```

Debe mostrar tu IP privada de Ubuntu como DNS primario.

---

### Paso 36: Verificar conectividad y DNS

```powershell
# Ping al servidor Ubuntu
ping 10.0.1.229

# Debe responder

# Resolver DNS
nslookup cloud02.city

# Debe resolver a la IP privada de Ubuntu
```

✅ Si ambos funcionan, continúa.

---

### Paso 37: Unir Windows al dominio

**Opción 1: Desde PowerShell (más rápido):**

```powershell
Add-Computer -DomainName cloud02.city -Credential BESPIN02\Administrator -Restart
```

Alternativas>
```powershell
# Diferente inicio de sesión v2
Add-Computer -DomainName cloud02.city -Credential cloud02\Administrator -Restart
# Diferente inicio de sesión v3
Add-Computer -DomainName cloud02.city -Credential Administrator@cloud02.city -Restart
```

⚠️ CUIDADO, revisa tener el teclado en el idioma correcto [ESP (Español)] y no poner carácteres que no son (quizá por eso te puede llegar a fallar al poner la contraseña incorrectamente)

**Pide contraseña:**
```
Password for BESPIN02\Administrator:
→ Escribir: admin_21
```

El Windows se reiniciará automáticamente.

---

**Opción 2: Desde GUI:**

```
Inicio → Buscar: "This PC"
Clic derecho → Properties
"Rename this PC (advanced)"
Clic en "Change..."
Seleccionar "Domain"
Escribir: cloud02.city
OK
Usuario: Administrator
Contraseña: admin_21
OK → Restart Now
```

---

**Espera 2-3 minutos a que reinicie.**

---

### Paso 38: Reconectar y verificar

**Reconectar por RDP:**
```bash
xfreerdp /v:54.221.100.222 /u:Administrator /p:'admin_21' /cert:ignore /dynamic-resolution /clipboard
```

**Verificar en PowerShell:**
```powershell
systeminfo | findstr /B /C:"Domain"
```

Debe mostrar:
```
Domain: cloud02.city
```

✅ Windows unido correctamente al dominio.

---

## 📁 PARTE 9: PROBAR ACCESO A LA CARPETA TRAP

### Paso 39: Cerrar sesión de Administrator

```
En Windows (RDP):
Inicio → Icono usuario → Cerrar sesión
```
O tambien está esta opción:
**Desde Windows (conectado como Administrator):**

**Abrir Explorador de archivos (Windows + E)**

**En la barra de direcciones, escribir:**
```
\\bespin02.cloud02.city\trap
```

**Aparece ventana de credenciales:**
```
Usuario: BESPIN02\lando
Contraseña: admin_21
```

✅ **DEBE ABRIR** la carpeta trap (lando tiene acceso)

Si fuera con las credenciales de "boba" no debería de dejar:
```
\\bespin02.cloud02.city\trap
Usuario: BESPIN02\boba
Contraseña: admin_21
```

❌ **NO DEBE ABRIR** la carpeta trap (boba NO tiene acceso)

---

### Paso 40: Crear directorio con iniciales CB

**Dentro de la carpeta trap que se abrió:**
```
1. Clic derecho → Nuevo → Carpeta
   Nombre: CB
   Enter

2. Doble clic en la carpeta CB para entrar

3. Crear archivo de prueba:
   Clic derecho → Nuevo → Documento de texto
   Nombre: prueba.txt
   
4. Abrir prueba.txt y escribir:
   "Acceso OK - Lando - Cristian BR"
   
5. Guardar (Ctrl+S) y cerrar
```

✅ lando puede crear carpetas y archivos.

---

### Paso 41: Cerrar sesión y probar con boba

**Cerrar la ventana del Explorador de archivos**

**Cerrar sesión del Administrator:**
```
Inicio → Icono usuario → Cerrar sesión
```

**Reconectar por RDP:**
```bash
xfreerdp /v:174.129.224.105 /u:Administrator /p:'admin_21' /cert:ignore /dynamic-resolution /clipboard
```

---

### Paso 42: Intentar acceder a trap como boba

**Abrir Explorador de archivos (Windows + E)**

**Escribir en barra de direcciones:**
```
\\bespin02.cloud02.city\trap
```

**Aparece ventana de credenciales:**
```
Nombre de usuario: BESPIN02\boba
Contraseña: admin_21
```

**Clic en Aceptar**

❌ **DEBE DENEGAR ACCESO**

**Debe mostrar:**
```
Windows no puede acceder a \\bespin02.cloud02.city\trap
No tienes permiso para acceder...
```

✅ **Correcto - boba NO tiene acceso.**

---

**💡 Nota:** Si necesitas probar de nuevo con diferentes usuarios, cierra sesión y reconecta por RDP para limpiar las credenciales guardadas.

---

### Paso 43: Cerrar sesión de lando y probar con boba

```
Inicio → Icono usuario → Cerrar sesión
```

**Iniciar sesión como boba:**
```
Otro usuario
Usuario: BESPIN02\boba
Contraseña: admin_21
```

---

### Paso 44: Intentar acceder a trap como boba

**Explorador de archivos (Windows + E)**

**Escribir en barra de direcciones:**
```
\\bespin02.cloud02.city\trap
```

❌ **DEBE DENEGAR ACCESO**

**Debe mostrar:**
```
Windows no puede acceder a \\bespin02.cloud02.city\trap
No tienes permiso para acceder...
```

✅ **Correcto - boba NO tiene acceso.**

---

## ✅ VERIFICACIÓN FINAL COMPLETA

### En el servidor Ubuntu (SSH):

```bash
# 1. Samba funcionando
sudo systemctl status samba-ad-dc | grep Active

# 2. Usuarios creados
sudo samba-tool user list

# 3. Carpeta existe
ls -la /city/trap/

# 4. Carpeta CB creada por lando
ls -la /city/trap/CB/

# 5. DNS funciona
host cloud02.city

# 6. Recurso compartido configurado
sudo testparm -s | grep -A 5 "\[trap\]"
```

---

### En Windows (como lando):

```
1. Iniciar sesión: BESPIN02\lando / admin_21
2. Acceder a: \\bespin02.cloud02.city\trap → ✅ Abre
3. Ver carpeta CB → ✅ Existe
4. Ver archivo prueba.txt → ✅ Existe
```

---

### En Windows (como boba):

```
1. Iniciar sesión: BESPIN02\boba / admin_21
2. Intentar acceder: \\bespin02.cloud02.city\trap → ❌ Denegado
```

---

## 📊 RESUMEN DE CONFIGURACIÓN

| Elemento | Valor |
|----------|-------|
| **Dominio** | cloud02.city |
| **NetBIOS Name** | BESPIN02 |
| **DC Hostname** | bespin02.cloud02.city |
| **IP Privada Ubuntu** | 10.0.X.X (depende de AWS) |
| **Contraseña Administrator** | admin_21 |
| **Usuario con acceso** | lando |
| **Usuario denegado** | boba |
| **Carpeta compartida** | /city/trap |
| **Recurso SMB** | \\bespin02.cloud02.city\trap |
| **Carpeta creada** | CB (iniciales Cristian BR) |

---

## 🛠️ TROUBLESHOOTING

### Windows no puede unirse al dominio

**Verificar DNS en Windows:**
```powershell
ipconfig /all
```

El DNS primario debe ser la IP privada de Ubuntu.

**Probar resolución:**
```powershell
nslookup cloud02.city
```

Debe resolver a la IP privada de Ubuntu.

---

### No puedo acceder a \\bespin02.cloud02.city\trap

**Verificar en Ubuntu:**
```bash
# Samba corriendo
sudo systemctl status samba-ad-dc

# Recurso configurado
sudo testparm -s | grep trap

# Usuario lando existe
sudo samba-tool user show lando
```

**Verificar en Windows:**
```powershell
# Ping al servidor
ping 10.0.1.229

# Puerto SMB abierto
Test-NetConnection -ComputerName 10.0.1.229 -Port 445

# Listar recursos
net view \\10.0.1.229
```

---

### RDP no conecta

**Verificar:**
1. Elastic IP correcta
2. Security group tiene puerto 3389 abierto
3. Contraseña sin espacios extra
4. Instancia Windows está Running

**Probar:**
```bash
nc -zv ELASTIC_IP 3389
```

Debe decir: `Connection to ... 3389 port [tcp/ms-wbt-server] succeeded!`

---

### La IP privada cambió

**Si paras y arrancas las instancias:**
- Elastic IPs NO cambian ✅
- IPs privadas SÍ cambian ❌

**Solución:**
1. Ver nueva IP privada en EC2 → Instances → Details
2. Actualizar /etc/hosts en Ubuntu
3. Actualizar DNS en Windows
4. Actualizar hosts en Windows

---

## 🎯 CHECKLIST FINAL PARA EL EXAMEN

**Antes del examen:**
- [ ] Sé cómo entrar a AWS Academy
- [ ] Sé descargar labsuser.pem
- [ ] Sé crear grupo de seguridad con todos los puertos
- [ ] Sé lanzar instancia Ubuntu
- [ ] Sé lanzar instancia Windows
- [ ] Sé asignar IPs elásticas
- [ ] Sé obtener contraseña de Windows
- [ ] Sé conectar por SSH
- [ ] Sé conectar por RDP con xfreerdp

**Durante el examen:**
- [ ] Anotar todas las IPs (pública, privada, Elastic)
- [ ] Anotar contraseña de Windows
- [ ] Seguir pasos en orden
- [ ] Verificar cada paso antes de continuar
- [ ] Provision con dominio CORRECTO (cloudXX.city)
- [ ] NetBIOS Name correcto (BespinXX)
- [ ] Crear usuarios lando y boba
- [ ] Configurar recurso trap solo para lando
- [ ] Crear carpeta con iniciales
- [ ] Probar acceso con ambos usuarios

**Verificación final:**
- [ ] lando puede acceder a trap
- [ ] boba NO puede acceder a trap
- [ ] Carpeta CB existe dentro de trap
- [ ] Archivo de prueba existe

---

## ⏱️ ESTIMACIÓN DE TIEMPO

| Tarea | Tiempo estimado |
|-------|-----------------|
| Iniciar AWS y descargar claves | 2 min |
| Crear Security Group | 5 min |
| Crear instancia Ubuntu | 3 min |
| Crear instancia Windows | 3 min |
| Asignar Elastic IPs | 3 min |
| Obtener contraseña Windows | 2 min |
| Configurar Ubuntu (provision) | 15 min |
| Crear usuarios y carpetas | 3 min |
| Configurar smb.conf | 3 min |
| Conectar por RDP | 2 min |
| Configurar Windows | 5 min |
| Unir Windows al dominio | 5 min |
| Probar accesos | 5 min |
| **TOTAL** | **~56 min** |

**⏰ Tiempo de margen:** Tienes tiempo de sobra si sigues los pasos.

---

## 🎯 COMANDOS CLAVE RÁPIDOS

**SSH al Ubuntu:**
```bash
ssh -i ~/.ssh/labsuser.pem ubuntu@ELASTIC_IP
```

**Provision del dominio:**
```bash
sudo samba-tool domain provision --use-rfc2307 --interactive
# Realm: CLOUD02.CITY
# Domain: BESPIN02
# DNS forwarder: 8.8.8.8
# Password: admin_21
```

**Crear usuarios:**
```bash
sudo samba-tool user create lando admin_21
sudo samba-tool user create boba admin_21
```

**Configurar recurso:**
```bash
sudo nano /etc/samba/smb.conf
# Añadir al final:
[trap]
    path = /city/trap
    read only = no
    valid users = lando
    vfs objects = acl_xattr
    map acl inherit = yes
```

**RDP desde Linux:**
```bash
xfreerdp /v:ELASTIC_IP /u:Administrator /p:'admin_21' /cert:ignore /dynamic-resolution
```

**DNS en Windows:**
```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses ("IP_PRIVADA_UBUNTU","8.8.8.8")
```

**Unir dominio:**
```powershell
Add-Computer -DomainName cloud02.city -Credential BESPIN02\Administrator -Restart
```

---

## 🎓 ÉXITO GARANTIZADO

Si sigues esta guía PASO A PASO sin saltarte nada, el ejercicio funcionará correctamente.

**Puntos críticos a NO olvidar:**
1. ✅ Security Group con TODOS los puertos
2. ✅ Elastic IPs asignadas
3. ✅ Provision con dominio `cloud02.city` y NetBIOS `BESPIN02`
4. ✅ `valid users = lando` en smb.conf
5. ✅ DNS configurado en Windows (IP privada de Ubuntu)
6. ✅ Crear carpeta CB dentro de trap

**¡Mucha suerte! 🚀**

---

## 🎓 EXTRA: CONFIGURACIONES DE PERMISOS (Para Variaciones del Examen)

Esta sección cubre **todas las posibles variaciones** que el profesor puede pedir en el examen sobre permisos de carpetas compartidas.

---

### ESCENARIO 1: Un usuario con acceso, otro sin acceso (EXAMEN ACTUAL)

**Requisito:** Solo `lando` puede acceder a `trap`, `boba` NO puede.

**Configuración en smb.conf:**
```ini
[trap]
    path = /city/trap
    read only = no
    valid users = lando
    vfs objects = acl_xattr
    map acl inherit = yes
```

**Comandos:**
```bash
# Crear usuarios
sudo samba-tool user create lando admin_21
sudo samba-tool user create boba admin_21

# Crear carpeta
sudo mkdir -p /city/trap
sudo chmod 777 /city/trap

# Editar smb.conf
sudo nano /etc/samba/smb.conf
# (añadir la configuración de arriba)

# Recargar
sudo smbcontrol all reload-config
```

**Verificación:**
- ✅ lando accede: `\\bespin02.cloud02.city\trap`
- ❌ boba denegado

---

### ESCENARIO 2: Varios usuarios con acceso, uno sin acceso

**Requisito:** `lando` y `han` pueden acceder, `boba` NO puede.

**Configuración:**
```ini
[trap]
    path = /city/trap
    read only = no
    valid users = lando han
    vfs objects = acl_xattr
    map acl inherit = yes
```

**Comandos:**
```bash
# Crear usuarios
sudo samba-tool user create lando admin_21
sudo samba-tool user create han admin_21
sudo samba-tool user create boba admin_21

# Editar smb.conf
sudo nano /etc/samba/smb.conf
# valid users = lando han

# Recargar
sudo smbcontrol all reload-config
```

**Verificación:**
- ✅ lando accede
- ✅ han accede
- ❌ boba denegado

---

### ESCENARIO 3: Solo lectura para unos, lectura/escritura para otros

**Requisito:** `lando` puede leer/escribir, `boba` solo puede leer.

**Configuración:**
```ini
[trap]
    path = /city/trap
    read only = no
    valid users = lando boba
    write list = lando
    vfs objects = acl_xattr
    map acl inherit = yes
```

**Comandos:**
```bash
# Crear usuarios
sudo samba-tool user create lando admin_21
sudo samba-tool user create boba admin_21

# Editar smb.conf
sudo nano /etc/samba/smb.conf
# valid users = lando boba
# write list = lando

# Recargar
sudo smbcontrol all reload-config
```

**Verificación:**
- ✅ lando puede crear carpetas/archivos
- ✅ boba puede ver pero NO crear/modificar

---

### ESCENARIO 4: Acceso por grupos

**Requisito:** Solo el grupo `Rebeldes` puede acceder, `boba` (que NO está en el grupo) no puede.

**Configuración:**
```ini
[trap]
    path = /city/trap
    read only = no
    valid users = @Rebeldes
    vfs objects = acl_xattr
    map acl inherit = yes
```

**Comandos:**
```bash
# Crear usuarios
sudo samba-tool user create lando admin_21
sudo samba-tool user create han admin_21
sudo samba-tool user create boba admin_21

# Crear grupo
sudo samba-tool group add Rebeldes

# Añadir usuarios al grupo
sudo samba-tool group addmembers Rebeldes lando,han

# Verificar miembros
sudo samba-tool group listmembers Rebeldes

# Editar smb.conf
sudo nano /etc/samba/smb.conf
# valid users = @Rebeldes

# Recargar
sudo smbcontrol all reload-config
```

**Verificación:**
- ✅ lando accede (está en Rebeldes)
- ✅ han accede (está en Rebeldes)
- ❌ boba denegado (NO está en Rebeldes)

---

### ESCENARIO 5: Denegar acceso explícito a usuarios específicos

**Requisito:** Todos pueden acceder EXCEPTO `boba`.

**Configuración:**
```ini
[trap]
    path = /city/trap
    read only = no
    invalid users = boba
    vfs objects = acl_xattr
    map acl inherit = yes
```

**Comandos:**
```bash
# Crear usuarios
sudo samba-tool user create lando admin_21
sudo samba-tool user create han admin_21
sudo samba-tool user create boba admin_21

# Editar smb.conf
sudo nano /etc/samba/smb.conf
# invalid users = boba

# Recargar
sudo smbcontrol all reload-config
```

**Verificación:**
- ✅ lando accede
- ✅ han accede
- ✅ cualquier otro usuario accede
- ❌ boba denegado explícitamente

---

### ESCENARIO 6: Carpeta pública (todos pueden acceder)

**Requisito:** Cualquier usuario puede acceder sin autenticación.

**Configuración:**
```ini
[trap]
    path = /city/trap
    read only = no
    guest ok = yes
    vfs objects = acl_xattr
    map acl inherit = yes
```

**Comandos:**
```bash
# Crear carpeta
sudo mkdir -p /city/trap
sudo chmod 777 /city/trap

# Editar smb.conf
sudo nano /etc/samba/smb.conf
# guest ok = yes

# Recargar
sudo smbcontrol all reload-config
```

**Verificación:**
- ✅ Cualquier usuario accede (incluso sin credenciales)

---

### ESCENARIO 7: Múltiples carpetas con diferentes permisos

**Requisito:** 
- `/city/trap` → solo `lando`
- `/city/cloud` → solo `boba`
- `/city/public` → todos

**Configuración:**
```ini
[trap]
    path = /city/trap
    read only = no
    valid users = lando
    vfs objects = acl_xattr
    map acl inherit = yes

[cloud]
    path = /city/cloud
    read only = no
    valid users = boba
    vfs objects = acl_xattr
    map acl inherit = yes

[public]
    path = /city/public
    read only = no
    guest ok = yes
    vfs objects = acl_xattr
    map acl inherit = yes
```

**Comandos:**
```bash
# Crear carpetas
sudo mkdir -p /city/trap
sudo mkdir -p /city/cloud
sudo mkdir -p /city/public
sudo chmod 777 /city/trap
sudo chmod 777 /city/cloud
sudo chmod 755 /city/public

# Crear usuarios
sudo samba-tool user create lando admin_21
sudo samba-tool user create boba admin_21

# Editar smb.conf (añadir las 3 secciones de arriba)
sudo nano /etc/samba/smb.conf

# Recargar
sudo smbcontrol all reload-config
```

**Verificación:**
- ✅ lando accede a `trap`, NO a `cloud`
- ✅ boba accede a `cloud`, NO a `trap`
- ✅ Ambos acceden a `public`

---

### ESCENARIO 8: Grupo con acceso + usuario individual extra

**Requisito:** Grupo `Rebeldes` puede acceder + `chewie` (que NO está en el grupo) también puede.

**Configuración:**
```ini
[trap]
    path = /city/trap
    read only = no
    valid users = @Rebeldes chewie
    vfs objects = acl_xattr
    map acl inherit = yes
```

**Comandos:**
```bash
# Crear usuarios
sudo samba-tool user create lando admin_21
sudo samba-tool user create han admin_21
sudo samba-tool user create chewie admin_21
sudo samba-tool user create boba admin_21

# Crear grupo y añadir miembros
sudo samba-tool group add Rebeldes
sudo samba-tool group addmembers Rebeldes lando,han

# Editar smb.conf
sudo nano /etc/samba/smb.conf
# valid users = @Rebeldes chewie

# Recargar
sudo smbcontrol all reload-config
```

**Verificación:**
- ✅ lando accede (está en Rebeldes)
- ✅ han accede (está en Rebeldes)
- ✅ chewie accede (listado individual)
- ❌ boba denegado

---

### ESCENARIO 9: Solo lectura para todos excepto uno

**Requisito:** Todos pueden leer, solo `lando` puede escribir.

**Configuración:**
```ini
[trap]
    path = /city/trap
    read only = yes
    write list = lando
    vfs objects = acl_xattr
    map acl inherit = yes
```

**Comandos:**
```bash
# Crear usuarios
sudo samba-tool user create lando admin_21
sudo samba-tool user create boba admin_21
sudo samba-tool user create han admin_21

# Editar smb.conf
sudo nano /etc/samba/smb.conf
# read only = yes
# write list = lando

# Recargar
sudo smbcontrol all reload-config
```

**Verificación:**
- ✅ lando puede crear/modificar archivos
- ✅ boba solo puede ver (no crear/modificar)
- ✅ han solo puede ver (no crear/modificar)

---

## 📋 CHEATSHEET DE PERMISOS

### Parámetros principales de smb.conf

| Parámetro | Descripción | Ejemplo |
|-----------|-------------|---------|
| `valid users` | Solo estos usuarios/grupos pueden acceder | `valid users = lando @Rebeldes` |
| `invalid users` | Estos usuarios NO pueden acceder | `invalid users = boba` |
| `read only` | Si es `yes`, nadie puede escribir (solo lectura) | `read only = yes` |
| `write list` | Usuarios que SÍ pueden escribir (anula `read only`) | `write list = lando` |
| `guest ok` | Permitir acceso sin autenticación | `guest ok = yes` |
| `@NombreGrupo` | Referencia a un grupo (usar arroba @) | `valid users = @Rebeldes` |

---

### Comandos rápidos

**Crear usuario:**
```bash
sudo samba-tool user create NOMBRE CONTRASEÑA
```

**Crear grupo:**
```bash
sudo samba-tool group add NOMBRE_GRUPO
```

**Añadir usuario a grupo:**
```bash
sudo samba-tool group addmembers GRUPO usuario1,usuario2
```

**Listar miembros de grupo:**
```bash
sudo samba-tool group listmembers GRUPO
```

**Verificar configuración:**
```bash
sudo testparm
```

**Recargar Samba:**
```bash
sudo smbcontrol all reload-config
```

**Listar recursos compartidos:**
```bash
sudo smbclient -L localhost -U Administrator%admin_21
```

---

## 🎯 ESTRATEGIA PARA EL EXAMEN

### Si te piden algo diferente:

1. **Anotar exactamente qué te piden:**
   - ¿Qué usuarios pueden acceder?
   - ¿Qué usuarios NO pueden?
   - ¿Hay grupos involucrados?
   - ¿Solo lectura o lectura/escritura?

2. **Crear usuarios necesarios:**
```bash
   sudo samba-tool user create USUARIO admin_21
```

3. **Crear grupos si es necesario:**
```bash
   sudo samba-tool group add GRUPO
   sudo samba-tool group addmembers GRUPO usuario1,usuario2
```

4. **Editar smb.conf con la configuración adecuada:**
```bash
   sudo nano /etc/samba/smb.conf
```

5. **Recargar:**
```bash
   sudo smbcontrol all reload-config
```

6. **Probar desde Windows:**
```
   \\bespin02.cloud02.city\RECURSO
```

---

### Combinaciones más comunes en exámenes:

| Requisito | Configuración |
|-----------|---------------|
| Solo USER1 accede | `valid users = USER1` |
| Solo USER1 y USER2 acceden | `valid users = USER1 USER2` |
| Solo grupo GRUPO accede | `valid users = @GRUPO` |
| Todos EXCEPTO USER1 | `invalid users = USER1` |
| USER1 escribe, otros solo leen | `read only = yes` + `write list = USER1` |
| Todos pueden acceder | `guest ok = yes` |
| Grupo + usuario extra | `valid users = @GRUPO USER1` |

---

**💡 Consejo final:** Si tienes dudas durante el examen, usa `valid users` - es lo más directo y funciona siempre.


