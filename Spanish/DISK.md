# GESTIÓN DE DISCOS EN LINUX — Guía Completa para Examen

**Herramientas principales:** `fdisk` · `parted` · `mkfs` · `mount` · `lsblk` · `df` · `du`

---

## 📋 ÍNDICE

1. [Ver información de discos](#parte-1)
2. [Añadir un segundo disco (VirtualBox/AWS)](#parte-2)
3. [Particionar con fdisk](#parte-3)
4. [Formatear particiones](#parte-4)
5. [Montar particiones](#parte-5)
6. [Montaje permanente (/etc/fstab)](#parte-6)
7. [Redimensionar particiones](#parte-7)
8. [Cambiar tipo de partición](#parte-8)
9. [Eliminar particiones](#parte-9)
10. [Particiones SWAP](#parte-10)
11. [Permisos y propietarios](#parte-11)
12. [Troubleshooting](#parte-12)
13. [Cheatsheet rápido](#cheatsheet)
14. [Escenarios típicos de examen](#escenarios)

---

<a name="parte-1"></a>
## 📊 PARTE 1: Ver información de discos

### lsblk — ver todos los discos y particiones

```bash
lsblk
```

Salida típica:
```
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   20G  0 disk
├─sda1   8:1    0   19G  0 part /
├─sda2   8:2    0    1K  0 part
└─sda5   8:5    0  975M  0 part [SWAP]
sdb      8:16   0   10G  0 disk
```

- `sda` = Primer disco (principal) · `sdb` = Segundo disco (sin particionar)
- `sda1` = Primera partición de sda · `MOUNTPOINT` = Dónde está montado

Ver con información de filesystems:
```bash
lsblk -f
```

### fdisk -l — lista detallada de particiones

```bash
sudo fdisk -l
sudo fdisk -l /dev/sda
sudo fdisk -l /dev/sdb
```

### df — espacio usado y disponible en particiones montadas

→ `-h` = Human readable · `-T` = mostrar tipo de filesystem
```bash
df -h
df -hT
```

### du — espacio ocupado por carpetas

```bash
du -sh /srv/samba          # Total de la carpeta
du -h /srv/samba           # Desglose por subcarpetas
du -h /srv | sort -h       # Ordenado por tamaño
```

### blkid — ver UUIDs de todas las particiones

```bash
sudo blkid
```

Salida:
```
/dev/sda1: UUID="a1b2c3d4-e5f6-7890-abcd-ef1234567890" TYPE="ext4"
/dev/sda5: UUID="12345678-90ab-cdef-1234-567890abcdef" TYPE="swap"
/dev/sdb1: UUID="11111111-2222-3333-4444-555555555555" TYPE="ext4"
```

---

<a name="parte-2"></a>
## 💾 PARTE 2: Añadir un segundo disco

### Opción A: VirtualBox (VM apagada)

1. Clic derecho en la VM → Configuración → Almacenamiento
2. Controlador SATA → icono "+" → Crear → VDI → Reservado dinámicamente
3. Tamaño: 10 GB · Nombre: `segundo_disco.vdi` → Crear → Aceptar

→ Debe aparecer `/dev/sdb` (nuevo disco de 10 GB)
```bash
lsblk
```

### Opción B: AWS EC2 (instancia en ejecución)

1. EC2 → Volumes → Create Volume → Size: 10 GiB
2. Availability Zone: **la misma que tu instancia** → Create Volume
3. Seleccionar volumen → Actions → Attach Volume → seleccionar instancia
4. Device name: `/dev/sdf` (AWS lo cambia a `/dev/xvdf` automáticamente)

→ Debe aparecer `/dev/xvdf` o `/dev/nvme1n1` (instancias modernas)
```bash
lsblk
```

---

<a name="parte-3"></a>
## 🔧 PARTE 3: Particionar con fdisk

### Referencia de comandos dentro de fdisk

| Tecla | Acción |
|---|---|
| `n` | Nueva partición |
| `d` | Eliminar partición |
| `t` | Cambiar tipo |
| `p` | Ver tabla de particiones actual |
| `l` | Listar tipos disponibles |
| `w` | Guardar y salir |
| `q` | Salir SIN guardar |

> `p` = Primaria (máx. 4) · En sectores, **Enter = valor por defecto**

---

### Escenario 1: Una partición ocupando todo el disco

→ Debe aparecer `/dev/sdb1`
```bash
sudo fdisk /dev/sdb
# n → p → 1 → [Enter] → [Enter] → w

lsblk
```

### Escenario 2: Dos particiones (5 GB cada una)

Primera partición:
```bash
sudo fdisk /dev/sdb
# n → p → 1 → [Enter] → +5G → w

lsblk
```

Segunda partición (resto del espacio):
```bash
sudo fdisk /dev/sdb
# n → p → 2 → [Enter] → [Enter] → w

lsblk
# → /dev/sdb1 (5G) y /dev/sdb2 (5G)
```

### Escenario 3: Partición de tamaño específico

Referencia de tamaños: `+2G` · `+500M` · `+1T` · `+2048M`

Crear partición de 3 GB exactos:
```bash
sudo fdisk /dev/sdb
# n → p → 1 → [Enter] → +3G → w
```

---

<a name="parte-4"></a>
## 💿 PARTE 4: Formatear particiones

### ext4 (más común en Linux)
```bash
sudo mkfs.ext4 /dev/sdb1
```

### xfs
```bash
sudo mkfs.xfs /dev/sdb1
```

### vfat — FAT32, compatible Windows
```bash
sudo mkfs.vfat /dev/sdb1
```

### ntfs — compatible Windows
```bash
sudo apt install -y ntfs-3g
sudo mkfs.ntfs /dev/sdb1
```

### Con etiqueta (label)
→ `lsblk -f` debe mostrar `LABEL: DATOS`
```bash
sudo mkfs.ext4 -L DATOS /dev/sdb1
lsblk -f
```

---

<a name="parte-5"></a>
## 🗂️ PARTE 5: Montar particiones

### Montaje manual temporal
→ `df -h` debe mostrar `/dev/sdb1` montado en `/mnt/datos`
```bash
sudo mkdir -p /mnt/datos
sudo mount /dev/sdb1 /mnt/datos
df -h | grep datos
ls -la /mnt/datos
```

### Montar con opciones específicas
```bash
sudo mount -o rw,uid=1000,gid=1000 /dev/sdb1 /mnt/datos
```

Opciones: `rw` = lectura/escritura · `ro` = solo lectura · `uid=1000` = propietario · `gid=1000` = grupo

### Desmontar
```bash
sudo umount /mnt/datos
# O por dispositivo:
sudo umount /dev/sdb1
```

🛠 Si dice "target is busy":
```bash
sudo lsof /mnt/datos          # Ver qué proceso lo usa
sudo kill -9 [PID]
sudo umount -l /mnt/datos     # Forzar (con precaución)
```

---

<a name="parte-6"></a>
## 🔄 PARTE 6: Montaje permanente (/etc/fstab)

### Formato de una entrada en fstab
```
<dispositivo>  <punto_montaje>  <filesystem>  <opciones>  <dump>  <fsck>
```
- `UUID=...` → identificador único · `defaults` → opciones estándar (rw, suid, dev, exec, auto, nouser, async)
- `dump`: 0 = no backup · `fsck`: 0 = no chequear, 1 = primero, 2 = después

### Añadir montaje automático al arranque

Obtener UUID, editar fstab, probar y reiniciar:
→ `sudo mount -a` sin errores = correcto. Tras reinicio `df -h | grep datos` debe mostrar el disco montado.
```bash
sudo blkid /dev/sdb1
# Copiar el UUID

sudo nano /etc/fstab
# Añadir al final:
# UUID=11111111-2222-3333-4444-555555555555  /mnt/datos  ext4  defaults  0  2

sudo mount -a
df -h | grep datos
sudo reboot
# Tras reinicio:
df -h | grep datos
```

### Opciones avanzadas de montaje en fstab

```
# Solo lectura
UUID=...  /mnt/datos  ext4  ro  0  2

# Con permisos específicos
UUID=...  /mnt/datos  ext4  defaults,uid=1000,gid=1000  0  2

# Sin ejecución de binarios (seguridad)
UUID=...  /mnt/datos  ext4  defaults,noexec  0  2

# No montar automáticamente
UUID=...  /mnt/datos  ext4  noauto  0  0
```

---

<a name="parte-7"></a>
## 📏 PARTE 7: Redimensionar particiones

### Método 1: Eliminar y recrear — PIERDE DATOS

> ⚠️ Esto BORRA todos los datos de la partición.

Escenario: cambiar `/dev/sdb1` de 5 GB a 8 GB.
```bash
sudo umount /dev/sdb1
sudo fdisk /dev/sdb
# d → 1
# n → p → 1 → [Enter] → +8G → w

sudo mkfs.ext4 /dev/sdb1
sudo mount /dev/sdb1 /mnt/datos
```

### Método 2: Redimensionar sin perder datos — solo ext4

> ⚠️ La partición debe estar **desmontada**.

> ⚠️ Al recrear la partición con fdisk, asegurarse de empezar en el **mismo sector inicial**. Cuando fdisk pregunte si eliminar la firma ext4, responder **N**.

```bash
sudo umount /dev/sdb1
sudo fdisk /dev/sdb
# d → 1
# n → p → 1 → [Enter] → [Enter]
# N (NO eliminar firma ext4)
# w

sudo e2fsck -f /dev/sdb1
sudo resize2fs /dev/sdb1
sudo mount /dev/sdb1 /mnt/datos
df -h /mnt/datos
```

---

<a name="parte-8"></a>
## 🔄 PARTE 8: Cambiar tipo de partición

### Tipos más usados

| Código | Tipo |
|---|---|
| `83` | Linux (por defecto) |
| `82` | Linux swap |
| `8e` | Linux LVM |

Ver lista completa dentro de fdisk: tecla `l`

### Cambiar tipo — ejemplo a Linux LVM (8e)

→ `sudo fdisk -l /dev/sdb` debe mostrar `Type: Linux LVM`
```bash
sudo fdisk /dev/sdb
# t → 1 → 8e → w

sudo fdisk -l /dev/sdb
```

---

<a name="parte-9"></a>
## 🗑️ PARTE 9: Eliminar particiones

### Eliminar una partición
```bash
sudo fdisk /dev/sdb
# d → 1 → w
```

### Eliminar todas las particiones
```bash
sudo fdisk /dev/sdb
# d → 1 → d → 2 → d → 3 → ... → w
```

### Limpiar completamente un disco

> ⚠️ Esto BORRA PERMANENTEMENTE todo el disco. Ctrl+C para cancelar.

```bash
# Todo el disco:
sudo dd if=/dev/zero of=/dev/sdb bs=1M status=progress

# Solo los primeros 10 MB (tabla de particiones):
sudo dd if=/dev/zero of=/dev/sdb bs=1M count=10
```

---

<a name="parte-10"></a>
## 🔄 PARTE 10: Particiones SWAP

### Crear, activar y hacer permanente la SWAP

```bash
# 1. Crear partición swap con fdisk
sudo fdisk /dev/sdb
# n → p → 2 → [Enter] → +2G → w

# 2. Cambiar tipo a swap (82)
sudo fdisk /dev/sdb
# t → 2 → 82 → w

# 3. Formatear, activar y verificar
sudo mkswap /dev/sdb2
sudo swapon /dev/sdb2
swapon --show
free -h

# 4. Hacer permanente en fstab
sudo blkid /dev/sdb2
sudo nano /etc/fstab
# Añadir: UUID=... none swap sw 0 0
```

### Desactivar swap
```bash
sudo swapoff /dev/sdb2
```

---

<a name="parte-11"></a>
## 🔐 PARTE 11: Permisos y propietarios

### Cambiar propietario y permisos del disco montado

→ `ls -ld /mnt/datos` debe mostrar el propietario y permisos correctos
```bash
sudo chown ubuntu:ubuntu /mnt/datos         # Cambiar propietario
sudo chown -R ubuntu:ubuntu /mnt/datos      # Recursivo
sudo chmod 755 /mnt/datos                   # rwxr-xr-x
sudo chmod 777 /mnt/datos                   # Todos los permisos
sudo chmod 700 /mnt/datos                   # Solo propietario
ls -ld /mnt/datos
```

### Leer la salida de ls -ld

```
drwxr-xr-x 3 ubuntu ubuntu 4096 Feb 22 10:00 /mnt/datos
│└──┘└──┘└──┘
│  │   │   └── Otros: r-x
│  │   └────── Grupo: r-x
│  └────────── Propietario: rwx
└───────────── d = directorio
```

---

<a name="parte-12"></a>
## 🛠️ PARTE 12: Troubleshooting

### Error: "No space left on device"
```bash
df -h                                  # Ver espacio en disco
df -i                                  # Ver inodos (puede estar lleno aunque haya espacio)
sudo du -h / | sort -h | tail -20      # Buscar archivos grandes
```

### Error: "mount: wrong fs type, bad option, bad superblock"
Causas: filesystem no formateado o tipo incorrecto en fstab.
```bash
sudo blkid /dev/sdb1          # Ver tipo real
sudo mkfs.ext4 /dev/sdb1      # Formatear si está vacío
cat /etc/fstab                # Verificar fstab
```

### Error: "target is busy" al desmontar
```bash
sudo lsof /mnt/datos          # Ver qué proceso lo usa
sudo kill -9 [PID]
sudo umount -l /mnt/datos     # Forzar si es necesario
```

### Error en /etc/fstab que impide arrancar
1. En GRUB → "Advanced options" → "recovery mode" → "root"
```bash
nano /etc/fstab
# Comentar la línea problemática con #
reboot
```

### Disco no aparece en lsblk — VirtualBox
Verificar en VirtualBox → Configuración → Almacenamiento que el disco aparece.
```bash
echo "- - -" | sudo tee /sys/class/scsi_host/host*/scan
ls -la /dev/sd*
```

### Disco no aparece en lsblk — AWS
Verificar en EC2 → Volumes que el estado sea "in-use".
```bash
ls -la /dev/nvme*
ls -la /dev/xvd*
```

---

<a name="cheatsheet"></a>
## 📝 CHEATSHEET RÁPIDO

```bash
# VER DISCOS
lsblk                                      # Todos los discos y particiones
lsblk -f                                   # Con filesystems y UUIDs
sudo fdisk -l                              # Lista detallada
df -h                                      # Espacio usado/disponible
df -i                                      # Inodos
du -sh /carpeta                            # Tamaño de carpeta
sudo blkid                                 # UUIDs de todas las particiones

# PARTICIONAR (dentro de fdisk)
sudo fdisk /dev/sdb
# n = nueva | d = eliminar | t = cambiar tipo | p = ver | w = guardar | q = salir sin guardar

# FORMATEAR
sudo mkfs.ext4 /dev/sdb1                   # ext4
sudo mkfs.ext4 -L DATOS /dev/sdb1          # ext4 con etiqueta
sudo mkfs.xfs /dev/sdb1                    # xfs
sudo mkfs.vfat /dev/sdb1                   # FAT32
sudo mkswap /dev/sdb2                      # swap

# MONTAR / DESMONTAR
sudo mkdir /mnt/datos
sudo mount /dev/sdb1 /mnt/datos
sudo umount /mnt/datos
sudo mount -a                              # Montar todo de fstab

# SWAP
sudo mkswap /dev/sdb2
sudo swapon /dev/sdb2
sudo swapoff /dev/sdb2
swapon --show

# REDIMENSIONAR (ext4, sin datos)
sudo e2fsck -f /dev/sdb1
sudo resize2fs /dev/sdb1

# PERMISOS
sudo chown ubuntu:ubuntu /mnt/datos
sudo chown -R ubuntu:ubuntu /mnt/datos
sudo chmod 755 /mnt/datos
```

---

<a name="escenarios"></a>
## 🎯 ESCENARIOS TÍPICOS DE EXAMEN

### Escenario 1: Añadir disco de 10 GB para datos

```bash
lsblk

sudo fdisk /dev/sdb
# n → p → 1 → [Enter] → [Enter] → w

sudo mkfs.ext4 -L DATOS /dev/sdb1
sudo mkdir /mnt/datos
sudo mount /dev/sdb1 /mnt/datos

sudo blkid /dev/sdb1
# Copiar UUID
sudo nano /etc/fstab
# Añadir: UUID=... /mnt/datos ext4 defaults 0 2

sudo mount -a
df -h | grep datos
```

### Escenario 2: Crear partición SWAP de 2 GB

```bash
sudo fdisk /dev/sdb
# n → p → 2 → [Enter] → +2G → w

sudo fdisk /dev/sdb
# t → 2 → 82 → w

sudo mkswap /dev/sdb2
sudo swapon /dev/sdb2
swapon --show

sudo blkid /dev/sdb2
# Copiar UUID
sudo nano /etc/fstab
# Añadir: UUID=... none swap sw 0 0
```

### Escenario 3: Dividir disco en 2 particiones iguales (disco de 10 GB → 5 GB + 5 GB)

```bash
sudo fdisk /dev/sdb
# n → p → 1 → [Enter] → +5G → w

sudo fdisk /dev/sdb
# n → p → 2 → [Enter] → [Enter] → w

sudo mkfs.ext4 /dev/sdb1
sudo mkfs.ext4 /dev/sdb2
sudo mkdir /mnt/datos1 /mnt/datos2
sudo mount /dev/sdb1 /mnt/datos1
sudo mount /dev/sdb2 /mnt/datos2
df -h | grep datos
```

---

## 🎯 FIN DE LA GUÍA

- ✅ Ver información de discos — lsblk, fdisk, df, du, blkid
- ✅ Añadir discos en VirtualBox y AWS
- ✅ Particionar con fdisk — una, varias, tamaño específico
- ✅ Formatear — ext4, xfs, vfat, ntfs, swap
- ✅ Montar manual y automático · /etc/fstab completo
- ✅ Redimensionar particiones con y sin pérdida de datos
- ✅ Cambiar tipos de partición · Eliminar particiones
- ✅ Crear y gestionar SWAP · Permisos y propietarios
- ✅ Troubleshooting completo

> Para el examen: practica los 3 escenarios · memoriza el cheatsheet · verifica siempre con `lsblk` después de cada paso · usa `sudo mount -a` para probar fstab antes de reiniciar.