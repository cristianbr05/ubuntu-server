# LIMPIEZA Y REINICIO DE CONFIGURACIÓN AWS

---

## 🧹 PARTE A: LIMPIAR UBUNTU SERVER (Volver al estado ANTES de Samba)

### Opción 1: Limpieza completa (RECOMENDADO para examen)

**Conectar por SSH al Ubuntu:**
```bash
ssh -i ~/.ssh/labsuser.pem ubuntu@ELASTIC_IP_UBUNTU
```

---

**Paso 1: Desinstalar Samba completamente**
```bash
# Detener servicios
sudo systemctl stop samba-ad-dc
sudo systemctl disable samba-ad-dc

# Desinstalar Samba y dependencias
sudo apt purge -y samba samba-common samba-common-bin samba-libs \
    smbclient winbind libpam-winbind libnss-winbind \
    krb5-user krb5-config samba-dsdb-modules samba-vfs-modules

# Limpiar paquetes huérfanos
sudo apt autoremove -y
sudo apt autoclean
```

---

**Paso 2: Eliminar configuraciones y datos de Samba**
```bash
# Eliminar configuraciones
sudo rm -rf /etc/samba/
sudo rm -rf /var/lib/samba/
sudo rm -rf /var/cache/samba/
sudo rm -rf /var/log/samba/
sudo rm -rf /run/samba/

# Eliminar carpetas compartidas
sudo rm -rf /city/
sudo rm -rf /srv/samba/
```

---

**Paso 3: Limpiar configuraciones de red/DNS**
```bash
# Hacer resolv.conf editable de nuevo
sudo chattr -i /etc/resolv.conf

# Restaurar resolv.conf por defecto de AWS
sudo rm /etc/resolv.conf
sudo ln -s /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf

# Reactivar systemd-resolved
sudo systemctl enable systemd-resolved
sudo systemctl start systemd-resolved
```

---

**Paso 4: Restaurar /etc/hosts original**
```bash
sudo nano /etc/hosts
```

**Dejar solo esto:**
```
127.0.0.1       localhost
127.0.1.1       ip-172-31-0-164.ec2.internal

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

**Cambiar `ip-172-31-0-164` por tu IP privada en formato `ip-172-31-X-X`**

**Guardar:** Ctrl+O, Enter, Ctrl+X

---

**Paso 5: Restaurar hostname original de AWS**
```bash
# Obtener hostname original (formato: ip-172-31-X-X)
# Usar tu IP privada (ejemplo: 172.31.0.164 → ip-172-31-0-164)

sudo hostnamectl set-hostname ip-172-31-0-164
```

**Cambiar `ip-172-31-0-164` por tu IP privada en formato correcto.**

---

**Paso 6: Limpiar configuraciones de Kerberos**
```bash
sudo rm -rf /etc/krb5.conf
sudo rm -rf /var/lib/krb5kdc/
sudo rm -rf /etc/krb5kdc/
```

---

**Paso 7: Reiniciar Ubuntu**
```bash
sudo reboot
```

**Espera 1-2 minutos.**

---

**Paso 8: Verificar que está limpio**

**Reconectar por SSH:**
```bash
ssh -i ~/.ssh/labsuser.pem ubuntu@ELASTIC_IP_UBUNTU
```

**Verificar:**
```bash
# No debe existir Samba
which samba-tool
# Debe decir: samba-tool not found

# Hostname correcto
hostname
# Debe decir: ip-172-31-X-X

# DNS funciona
ping -c 2 google.com
# Debe responder

# Sistema actualizado
sudo apt update
```

✅ **Ubuntu limpio y listo para empezar de nuevo desde el Paso 16 del documento.**

---

## 🔄 PARTE B: CAMBIAR NOMBRES SIN RECREAR MÁQUINAS

### Escenario: El profesor te pide cloud03.city en lugar de cloud02.city

---

### B1: Cambiar dominio en Ubuntu (ANTES de provision)

**Si AÚN NO hiciste provision:**

Simplemente cuando hagas `sudo samba-tool domain provision --use-rfc2307 --interactive`, usa el nuevo nombre:
```
Realm: CLOUD03.CITY
Domain: BESPIN03
```

---

**Si YA hiciste provision:**

**No hay forma fácil de cambiar el dominio.** Debes limpiar y volver a hacer provision:
```bash
# Detener Samba
sudo systemctl stop samba-ad-dc

# Eliminar provision anterior
sudo rm -rf /var/lib/samba/
sudo rm -rf /etc/samba/smb.conf

# Hacer nueva provision
sudo samba-tool domain provision --use-rfc2307 --interactive
# Realm: CLOUD03.CITY
# Domain: BESPIN03

# Copiar krb5.conf
sudo cp /var/lib/samba/private/krb5.conf /etc/krb5.conf

# Reiniciar Samba
sudo systemctl restart samba-ad-dc
```

---

### B2: Cambiar hostname del servidor
```bash
# Nuevo hostname
sudo hostnamectl set-hostname bespin03

# Actualizar /etc/hosts
sudo nano /etc/hosts
```

**Cambiar todas las referencias:**
```
127.0.0.1       localhost
127.0.1.1       bespin03.cloud03.city bespin03
172.31.0.164    bespin03.cloud03.city bespin03
```

**Actualizar /etc/resolv.conf:**
```bash
sudo chattr -i /etc/resolv.conf
sudo nano /etc/resolv.conf
```
```
nameserver 127.0.0.1
nameserver 8.8.8.8
search cloud03.city
```
```bash
sudo chattr +i /etc/resolv.conf
```

**Reiniciar:**
```bash
sudo reboot
```

---

### B3: Cambiar nombre de usuario (renombrar usuario existente)

**No se puede renombrar directamente en Samba AD.** Debes:

**Eliminar usuario antiguo:**
```bash
sudo samba-tool user delete lando
```

**Crear usuario nuevo:**
```bash
sudo samba-tool user create luke admin_21
```

---

### B4: Cambiar nombre del recurso compartido

**Editar smb.conf:**
```bash
sudo nano /etc/samba/smb.conf
```

**Cambiar:**
```ini
# Antes:
[trap]
    path = /city/trap
    ...

# Después:
[carbonite]
    path = /city/carbonite
    ...
```

**Crear nueva carpeta:**
```bash
sudo mkdir -p /city/carbonite
sudo chmod 777 /city/carbonite
```

**Eliminar carpeta antigua (opcional):**
```bash
sudo rm -rf /city/trap
```

**Recargar:**
```bash
sudo smbcontrol all reload-config
```

---

### B5: Cambiar NetBIOS Name (DESPUÉS de provision - NO RECOMENDADO)

**Editar smb.conf:**
```bash
sudo nano /etc/samba/smb.conf
```

**En [global], cambiar:**
```ini
netbios name = BESPIN03
```

**Reiniciar Samba:**
```bash
sudo systemctl restart samba-ad-dc
```

⚠️ **IMPORTANTE:** Esto puede causar problemas. Es mejor hacer provision de nuevo.

---

## 🔄 PARTE C: LIMPIAR WINDOWS (Salir del dominio)

### Si Windows ya está unido y quieres empezar de nuevo:

**Conectar por RDP como Administrator:**
```bash
xfreerdp /v:ELASTIC_IP_WINDOWS /u:Administrator /p:'admin_21' /cert:ignore /dynamic-resolution /clipboard
```

---

**Paso 1: Salir del dominio**

**PowerShell como Administrador:**
```powershell
# Salir del dominio y volver a grupo de trabajo
Remove-Computer -UnjoinDomainCredential BESPIN02\Administrator -WorkgroupName WORKGROUP -Restart
```

**Pide contraseña:**
```
Password: admin_21
```

**Windows se reinicia.**

---

**Paso 2: Limpiar caché DNS (después del reinicio)**

**Reconectar por RDP:**
```bash
xfreerdp /v:ELASTIC_IP_WINDOWS /u:Administrator /p:'admin_21' /cert:ignore /dynamic-resolution /clipboard
```

**PowerShell:**
```powershell
# Limpiar caché DNS
ipconfig /flushdns

# Restaurar DNS a automático
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ResetServerAddresses

# Limpiar credenciales guardadas
cmdkey /list
# Si aparece alguna de \\bespin02, ejecutar:
cmdkey /delete:bespin02.cloud02.city
```

---

**Paso 3: Limpiar archivo hosts de Windows**

**Abrir Bloc de notas como Administrador:**
```
Inicio → notepad → Clic derecho → Ejecutar como administrador
Archivo → Abrir → C:\Windows\System32\drivers\etc\hosts
```

**Eliminar líneas añadidas:**
```
# Eliminar esta línea:
172.31.0.164    bespin02.cloud02.city bespin02 cloud02.city
```

**Guardar y cerrar.**

---

**Paso 4: Reiniciar Windows**
```powershell
shutdown /r /t 0
```

✅ **Windows limpio, listo para unirse a un nuevo dominio.**

---

## 📋 RESUMEN DE CAMBIOS RÁPIDOS

### Cambiar solo el número del dominio (cloud02 → cloud03)

**En Ubuntu:**
```bash
# 1. Limpiar Samba
sudo systemctl stop samba-ad-dc
sudo rm -rf /var/lib/samba/
sudo rm -rf /etc/samba/smb.conf

# 2. Actualizar hostname
sudo hostnamectl set-hostname bespin03

# 3. Actualizar /etc/hosts
sudo nano /etc/hosts
# Cambiar todas las referencias a cloud03.city

# 4. Actualizar /etc/resolv.conf
sudo chattr -i /etc/resolv.conf
sudo nano /etc/resolv.conf
# search cloud03.city
sudo chattr +i /etc/resolv.conf

# 5. Nueva provision
sudo samba-tool domain provision --use-rfc2307 --interactive
# Realm: CLOUD03.CITY
# Domain: BESPIN03

# 6. Copiar krb5.conf y reiniciar
sudo cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
sudo systemctl restart samba-ad-dc
```

**En Windows:**
```powershell
# Actualizar DNS
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses ("IP_PRIVADA_UBUNTU","8.8.8.8")

# Actualizar hosts
# notepad como admin → C:\Windows\System32\drivers\etc\hosts
# Cambiar cloud02 por cloud03

# Unir nuevo dominio
Add-Computer -DomainName cloud03.city -Credential BESPIN03\Administrator -Restart
```

---

## ⏱️ TIEMPO DE LIMPIEZA

| Tarea | Tiempo |
|-------|--------|
| Desinstalar Samba completo | 3 min |
| Limpiar configuraciones | 2 min |
| Restaurar red/DNS | 2 min |
| Reiniciar y verificar | 2 min |
| **TOTAL Ubuntu** | **~9 min** |
| | |
| Salir dominio Windows | 2 min |
| Limpiar DNS/hosts | 2 min |
| Reiniciar | 2 min |
| **TOTAL Windows** | **~6 min** |

---

## 🎯 RECOMENDACIÓN PARA EL EXAMEN

**Si tienes tiempo antes del examen:**

1. **Practica la limpieza** (hoy)
2. **Mañana al empezar el examen:**
   - Usar las MISMAS máquinas (ya tienes IPs elásticas)
   - Limpiar Ubuntu (9 min)
   - Limpiar Windows (6 min)
   - Empezar desde Paso 16 (instalación Samba)

**Ventajas:**
- ✅ IPs elásticas ya asignadas
- ✅ Security group ya configurado
- ✅ Claves SSH ya descargadas
- ✅ Más rápido que crear todo de nuevo

---

**Si NO tienes tiempo o algo falla:**

Crear instancias completamente nuevas desde Paso 1.

---

## 🔍 VERIFICACIÓN POST-LIMPIEZA

**Ubuntu limpio:**
```bash
which samba-tool              # → not found
hostname                      # → ip-172-31-X-X
ping google.com               # → funciona
ls /city/                     # → no existe
ls /etc/samba/                # → no existe
```

**Windows limpio:**
```powershell
systeminfo | findstr Domain   # → WORKGROUP
nslookup cloud02.city         # → falla (correcto)
```

✅ **Listo para empezar de nuevo.**
