# 🐧 Ubuntu Server + Samba AD DC - ADSO Cv1

![Ubuntu](https://img.shields.io/badge/Ubuntu_24.04_LTS-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![Samba](https://img.shields.io/badge/Samba-AD_DC-blue?style=for-the-badge)
![Windows](https://img.shields.io/badge/Windows_10-0078D6?style=for-the-badge&logo=windows&logoColor=white)
![Bash](https://img.shields.io/badge/Bash-Scripting-4EAA25?style=for-the-badge&logo=gnu-bash&logoColor=white)

> **Technical guide and step-by-step manual for the configuration of: Active Directory Domain Controller under Linux, hybrid client integration, GPO management, ACLs and system auditing.**

| 👨‍💻 Author | 👨‍🏫 Teacher | 🎓 Course |
| :--- | :--- | :--- |
| **Cristian Bellmunt Redon** | Gregorio Mateu | 2º ASIX |

---

## VM Configuration Data

| Parameter | Value |
|---|---|
| Hostname | ls02 |
| Domain | lab02.lan |
| Bridge WAN IP (enp0s3) | 172.30.20.26/25 — Gateway: 172.30.20.1 |
| Internal LAN IP (enp0s8) | 192.168.11.2/24 |
| DNS Forwarders | 10.239.3.7, 10.239.3.8 |
| Ubuntu server user | cristianbr / admin_21 |
| Windows client user | user01 / admin_21 |
| Ubuntu client user | user01 / admin_21 |

> **If something doesn't work**, check these three files:
> - `/etc/netplan/00-installer-config.yaml` → IPs and DNS
> - `/etc/resolv.conf` → nameservers
> - `/etc/samba/smb.conf` → DNS forwarders of the AD DC

---

## 🛡️ PRE-REQUISITE: Complete Firewall Deactivation (controlled test environment)
> ⚠️ **Important:** Disable UFW robustly to avoid blockages on Samba AD DC, DNS ports and conflicts with iptables NAT rules.

```bash
# 1. Stop and disable UFW
sudo ufw disable
sudo systemctl stop ufw
sudo systemctl disable ufw

# 2. Flush rules and remove chains from the filter table
sudo iptables -F
sudo iptables -X

# 3. Ensure default policies allow all traffic (Optional)
sudo iptables -P INPUT ACCEPT
sudo iptables -P FORWARD ACCEPT
sudo iptables -P OUTPUT ACCEPT
```

> 💡 **Technical note:** `sudo iptables -F` and `-X` are included to clean any residual rules or chains that may remain in memory before starting to configure your own routing rules in Hour 6 of Sprint 1.

---

## TABLE OF CONTENTS

- [🧱 SPRINT 1 – Base Server Configuration](#sprint-1)
- [🧱 SPRINT 2 – DHCP, Users, Groups and Shared Folders](#sprint-2)
- [🧱 SPRINT 3 – Windows Client Integration into the Domain](#sprint-3)
- [🧱 SPRINT 4 – GPOs, Ubuntu Client and System Management](#sprint-4)
- [🧱 SPRINT 5 – Forest Trust between two Samba AD DC servers](#sprint-5)

---

<a name="sprint-1"></a>
# 🧱 SPRINT 1 – Base Server Configuration

---

## 🕒 HOUR 1: System preparation and network configuration

### 1.1 Change hostname
→ Should display `Static hostname: ls02`
```bash
sudo hostnamectl set-hostname ls02
hostnamectl
```

### 1.2 Configure Netplan (static network)

> ⚠️ If `/etc/netplan/50-cloud-init.yaml` exists, cloud-init may overwrite the network. Disable it first:

```bash
sudo nano /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
# Content: network: {config: disabled}

sudo rm /etc/netplan/50-cloud-init.yaml
```

Edit the network file and apply. → Should display `enp0s3: 172.30.20.26/25` and `enp0s8: 192.168.11.2/24`
```bash
sudo nano /etc/netplan/00-installer-config.yaml
sudo netplan apply
ip addr show
```

Netplan file content:
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

### 1.3 Update the system
> ⚠️ Mandatory before installing Samba. May take several minutes.
```bash
sudo apt update && sudo apt upgrade -y
```

### 1.4 Configure /etc/hosts
→ `ping -c 2 ls02.lab02.lan` should reply from `192.168.11.2`
```bash
sudo nano /etc/hosts
# Add:
# 127.0.0.1   localhost
# 127.0.1.1   ls02
# 192.168.11.2   ls02.lab02.lan ls02

ping -c 2 ls02.lab02.lan
```

🛠 **If the verification ping fails:**
```bash
resolvectl status          # Verify correct DNS
ip route show              # Check gateway
sudo netplan --debug apply # Retry applying netplan with detailed errors
```

---

## 🕒 HOUR 2: DNS preparation and disabling systemd-resolved

### 2.1 – 2.2 Stop systemd-resolved and remove resolv.conf
> ⚠️ CRITICAL: systemd-resolved ALWAYS causes conflicts with Samba AD DC on port 53.

→ Should display `inactive (dead)` and `disabled`
```bash
sudo systemctl disable --now systemd-resolved
sudo systemctl status systemd-resolved
sudo unlink /etc/resolv.conf
```

### 2.3 Create temporary resolv.conf and make it immutable
→ `nslookup www.amazon.es` should resolve correctly
```bash
sudo nano /etc/resolv.conf
# Content:
# nameserver 10.239.3.7
# nameserver 10.239.3.8
# search lab02.lan

sudo chattr +i /etc/resolv.conf
nslookup www.amazon.es
```

> To remove the immutable attribute afterwards: `sudo chattr -i /etc/resolv.conf`

---

## 🕒 HOUR 3: Installing and preparing Samba AD DC

### 3.1 Install required packages
During Kerberos installation enter:
- Default realm: `LAB02.LAN` (uppercase)
- Kerberos servers: `ls02.lab02.lan`
- Administrative server: `ls02.lab02.lan`

> If an automatic Samba configuration window appears: select "No"
```bash
sudo apt install -y samba smbclient winbind krb5-user krb5-config
```

🛠 **If installation fails due to dependencies:**
```bash
sudo apt --fix-broken install
sudo apt install -y samba smbclient winbind krb5-user krb5-config
```

### 3.2 Stop previous services, back up smb.conf and install ldb-tools
→ `systemctl status smbd` should show `inactive (dead)`
```bash
sudo systemctl stop smbd nmbd winbind
sudo systemctl disable smbd nmbd winbind
sudo systemctl status smbd
sudo mv /etc/samba/smb.conf /etc/samba/smb.conf.backup
sudo apt install -y ldb-tools
```

---

## 🕒 HOUR 4: Promotion to Domain Controller

### 4.1 Provision the domain
Answers to the interactive wizard:
- Realm: `LAB02.LAN`
- Domain: `LAB02`
- Server Role: `dc` (Enter)
- DNS backend: `SAMBA_INTERNAL` (Enter)
- DNS forwarder IP: `10.239.3.7`
- Administrator password: `admin_21`

> ⚠️ Password: minimum 7 characters, uppercase, lowercase and numbers.
```bash
sudo samba-tool domain provision --use-rfc2307 --interactive
```

### 4.2 Force listening on IPv4 (fix for "Connection Refused")
Add in the `[global]` section of smb.conf:
```ini
interfaces = lo enp0s8
bind interfaces only = yes
```
```bash
sudo nano /etc/samba/smb.conf
sudo systemctl restart samba-ad-dc
sudo samba-tool domain level show
```

🛠 If it fails with "DNS zone already exists":
```bash
sudo systemctl stop samba-ad-dc
sudo rm -rf /var/lib/samba/private/*
sudo rm -rf /var/lib/samba/*.tdb
sudo rm /etc/samba/smb.conf
sudo samba-tool domain provision --use-rfc2307 --interactive
```

### 4.3 Copy Kerberos and update resolv.conf
→ `cat /etc/krb5.conf | grep LAB02.LAN` should display the domain. `resolv.conf` should have `127.0.0.1` as the first entry.
```bash
sudo cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
sudo chattr -i /etc/resolv.conf
sudo nano /etc/resolv.conf
# NEW content:
# nameserver 127.0.0.1
# nameserver 10.239.3.7
# search lab02.lan

sudo chattr +i /etc/resolv.conf
cat /etc/resolv.conf
```

### 4.4 Start and enable Samba AD DC
→ Should display `active (running)`
```bash
sudo systemctl unmask samba-ad-dc
sudo systemctl start samba-ad-dc
sudo systemctl enable samba-ad-dc
sudo systemctl status samba-ad-dc
```

🛠 If port 53 is occupied:
```bash
sudo ss -tulpn | grep :53
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
sudo systemctl start samba-ad-dc
```

---

## 🕒 HOUR 5: DNS configuration and domain verification

### 5.1 Verify that Samba DNS works
→ Expected:
- `lab02.lan` → `192.168.11.2`
- `ls02.lab02.lan` → `192.168.11.2`
- `_ldap._tcp.lab02.lan` → SRV record pointing to `ls02.lab02.lan`
```bash
host -t A lab02.lan
host -t A ls02.lab02.lan
host -t SRV _ldap._tcp.lab02.lan
```

🛠 If it doesn't resolve:
```bash
sudo reboot now
# After reboot:
cat /etc/resolv.conf
sudo journalctl -xeu samba-ad-dc | tail -50
sudo samba-tool dns query 127.0.0.1 lab02.lan @ ALL -U Administrator%admin_21
```

### 5.2 Configure DNS forwarder in smb.conf
> ⚠️ Samba does NOT create forwarders automatically even if specified during provision.

Add in `[global]`: `dns forwarder = 10.239.3.7`

→ `nslookup www.amazon.es 127.0.0.1` should resolve correctly
```bash
sudo samba-tool dns serverinfo 127.0.0.1 -U Administrator%admin_21
sudo nano /etc/samba/smb.conf
sudo systemctl restart samba-ad-dc
nslookup www.amazon.es 127.0.0.1
```

### 5.3 Test Kerberos authentication
→ `klist` should display a valid ticket for `Administrator@LAB02.LAN`
```bash
kinit Administrator@LAB02.LAN
# Password: admin_21
klist
```

🛠 If "Clock skew too great" fails:
```bash
sudo timedatectl set-ntp true
timedatectl                      # Verify time zone
kinit Administrator@LAB02.LAN
```

🛠 If "Cannot find KDC for realm" fails:
```bash
sudo cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
```

---

## 🕒 HOUR 6: NAT routing configuration and password policies

### 6.1 – 6.2 Enable IP forwarding and configure NAT
→ `ip_forward` should return `1`. `MASQUERADE` rule visible in `POSTROUTING`.
```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
cat /proc/sys/net/ipv4/ip_forward
sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
sudo iptables -A FORWARD -i enp0s8 -o enp0s3 -j ACCEPT
sudo iptables -A FORWARD -i enp0s3 -o enp0s8 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo apt install -y iptables-persistent
# During installation: save IPv4 → Yes, IPv6 → Yes
sudo iptables -t nat -L -v
```

🛠 If rules don't persist after reboot:
```bash
sudo iptables-save | sudo tee /etc/iptables/rules.v4
sudo systemctl enable netfilter-persistent
```

### 6.3 – 6.4 View and modify password policies
```bash
sudo samba-tool domain passwordsettings show

# Minimum password length:
sudo samba-tool domain passwordsettings set --min-pwd-length=8

# Password complexity (requires uppercase, lowercase, numbers):
sudo samba-tool domain passwordsettings set --complexity=on

# Password history (how many previous passwords are remembered):
sudo samba-tool domain passwordsettings set --history-length=12

# Maximum password age (days before expiry):
sudo samba-tool domain passwordsettings set --max-pwd-age=60

# Minimum password age (days before it can be changed):
sudo samba-tool domain passwordsettings set --min-pwd-age=0

# Account lockout duration (minutes):
sudo samba-tool domain passwordsettings set --account-lockout-duration=30

# Number of incorrect attempts before lockout:
sudo samba-tool domain passwordsettings set --account-lockout-threshold=5

# Observation window for failed attempts (minutes):
sudo samba-tool domain passwordsettings set --reset-account-lockout-after=15

sudo samba-tool domain passwordsettings show
```

### 6.5 Comprehensive domain verification
```bash
sudo samba-tool domain level show
sudo samba-tool user list
sudo samba-tool group list
sudo samba-tool dns query ls02.lab02.lan lab02.lan @ ALL -U Administrator%admin_21
```

---

## ✅ SPRINT 1 CHECKPOINT

```bash
hostnamectl                                  # → ls02
ip addr show                                 # → enp0s3 172.30.20.26, enp0s8 192.168.11.2
ping -c 2 www.amazon.es                      # → works
sudo systemctl status systemd-resolved       # → inactive (dead) disabled
host lab02.lan                               # → 192.168.11.2
sudo systemctl status samba-ad-dc            # → active (running)
klist                                        # → ticket Administrator@LAB02.LAN
sudo iptables -t nat -L                      # → MASQUERADE rule
nslookup www.amazon.es 127.0.0.1             # → resolves
host -t SRV _ldap._tcp.lab02.lan             # → SRV record
```

## 🛠 SPRINT 1 RESCUE PLAN

If the domain is completely broken:
```bash
sudo systemctl stop samba-ad-dc
sudo rm -rf /var/lib/samba/private/*
sudo rm -rf /var/lib/samba/*.tdb
sudo rm /etc/samba/smb.conf
sudo samba-tool domain provision --use-rfc2307 --interactive
```

If DNS resolves nothing:
```bash
cat /etc/resolv.conf                   # → must have nameserver 127.0.0.1
sudo systemctl status samba-ad-dc
sudo ss -tulpn | grep :53              # → only samba should appear
dig @127.0.0.1 lab02.lan
```

---

## 🎯 END OF SPRINT 1
- ✅ Dual network (bridge + internal), ✅ systemd-resolved removed, ✅ Samba AD DC installed
- ✅ Domain lab02.lan created, ✅ DNS + Kerberos, ✅ NAT, ✅ Password policies

---

<a name="sprint-2"></a>
# 🧱 SPRINT 2 – DHCP, Users, Groups and Shared Folders

> ⚠️ Always use `Administrator` (capitalized) — it is the default Samba user.

---

## 🕒 HOUR 1: DHCP server installation and configuration

### 1.1 – 1.2 Install DHCP and configure interface
It is normal for it to fail on start (not yet configured). Edit `INTERFACESv4=""` → `INTERFACESv4="enp0s8"`
```bash
sudo apt install -y isc-dhcp-server
sudo nano /etc/default/isc-dhcp-server
```

### 1.3 Configure DHCP range
Add at the end of `/etc/dhcp/dhcpd.conf`:
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

**Quick explanation:**
- **Range** → .100 to .150 (51 available IPs)
- **DNS** → the server itself (192.168.11.2)
- **Gateway** → the server itself (192.168.11.2)
- **Lease** → 10 minutes by default, maximum 2 hours

### 1.4 – 1.6 Verify syntax, start and check
→ Syntax without errors. Service should show `active (running)`.
```bash
sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf
sudo systemctl restart isc-dhcp-server
sudo systemctl enable isc-dhcp-server
sudo systemctl status isc-dhcp-server
cat /var/lib/dhcp/dhcpd.leases
```

🛠 If it fails to start:
```bash
sudo journalctl -xeu isc-dhcp-server
sudo touch /var/lib/dhcp/dhcpd.leases
sudo systemctl restart isc-dhcp-server
```

---

## 🕒 HOUR 2: Creating Organizational Units (OUs)

### 2.1 – 2.2 Verify domain and create OUs
→ `samba-tool ou list` should show the 4 OUs.
```bash
sudo samba-tool domain level show
sudo samba-tool ou create "OU=IT_Department,DC=lab02,DC=lan"
sudo samba-tool ou create "OU=HR_Department,DC=lab02,DC=lan"
sudo samba-tool ou create "OU=Students,DC=lab02,DC=lan"
sudo samba-tool ou create "OU=Groups,DC=lab02,DC=lan"
sudo samba-tool ou list
```

🛠 To delete an OU (if "Already exists" error):
```bash
sudo samba-tool ou delete "OU=name,DC=lab02,DC=lan"
# With objects inside (CAUTION):
sudo samba-tool ou delete "OU=name,DC=lab02,DC=lan" --force-subtree-delete
```

---

## 🕒 HOUR 3: Creating users in their respective OUs

### 3.1 – 3.4 Create all users
Bob in IT (with mandatory password change), Alice in HR, user01-03 in Students, techsupport in CN=Users (without --userou).
```bash
sudo samba-tool user create bob admin_21 --userou="OU=IT_Department" --given-name="Bob" --surname="Smith" --must-change-at-next-login
sudo samba-tool user create alice admin_21 --userou="OU=HR_Department" --given-name="Alice" --surname="Johnson"
sudo samba-tool user create user01 admin_21 --userou="OU=Students" --given-name="User" --surname="One"
sudo samba-tool user create user02 admin_21 --userou="OU=Students" --given-name="User" --surname="Two"
sudo samba-tool user create user03 admin_21 --userou="OU=Students" --given-name="User" --surname="Three"
sudo samba-tool user create techsupport admin_21 --given-name="Tech" --surname="Support"
```

> ⚠️ **IMPORTANT:** read what `--must-change-at-next-login` does if you really want to use it. It is not applied to students to ease testing.

**Parameters explained:**
- `bob` → username
- `admin_21` → initial password
- `--userou` → specifies the OU where it will be created (without this, it goes to the Users container)
- `--must-change-at-next-login` → forces password change on first login

### 3.5 Verify users and OUs
→ Should list: Administrator, bob, alice, user01, user02, user03, techsupport
```bash
sudo samba-tool user list
sudo samba-tool user show bob
sudo ldbsearch -H /var/lib/samba/private/sam.ldb "(sAMAccountName=bob)" dn
```

🛠 **If "Constraint violation" fails (password doesn't meet policy):**
```bash
sudo samba-tool domain passwordsettings set --complexity=off
sudo samba-tool user create bob admin_21 --userou="OU=IT_Department"
sudo samba-tool domain passwordsettings set --complexity=on
```

🛠 **If a user was created in the wrong OU:**
```bash
sudo samba-tool user delete bob
sudo samba-tool user create bob admin_21 --userou="OU=IT_Department"
```

---

## 🕒 HOUR 4: Creating security groups and assigning members

### 4.1 Create security groups
→ Should list: Finance, HR, IT Support
```bash
sudo samba-tool group add Finance --groupou="OU=Groups"
sudo samba-tool group add HR --groupou="OU=Groups"
sudo samba-tool group add "IT Support" --groupou="OU=Groups"
sudo samba-tool group list | grep -E "Finance|HR|IT Support"
```

🛠 **If you need to delete a group or check members first:**
```bash
sudo samba-tool group listmembers Finance
sudo samba-tool group delete Finance
```

### 4.2 Add users to groups and verify
Assignments: user01→Finance, alice→HR, bob→"IT Support", techsupport→"IT Support"
```bash
sudo samba-tool group addmembers Finance user01
sudo samba-tool group addmembers HR alice
sudo samba-tool group addmembers "IT Support" bob
sudo samba-tool group addmembers "IT Support" techsupport
sudo samba-tool group listmembers Finance
sudo samba-tool group listmembers HR
sudo samba-tool group listmembers "IT Support"
```

🛠 **If you need to check which groups a user belongs to or remove them from a group:**
```bash
sudo samba-tool user show bob | grep memberOf
sudo samba-tool group removemembers Finance user01
```

---

## 🕒 HOUR 5: Creating shared folders with ACLs

> Philosophy: Linux configures storage with broad base permissions → Windows manages ACLs visually.

### 5.1 – 5.2 Create folders and install winbind libraries
→ `dpkg -l | grep winbind` should show `ii libnss-winbind`, `ii libpam-winbind`, `ii winbind`
```bash
sudo mkdir -p /srv/samba/FinanceDocs
sudo mkdir -p /srv/samba/HRDocs
sudo mkdir -p /srv/samba/Public
sudo apt-get install -y libnss-winbind libpam-winbind
sudo ldconfig
dpkg -l | grep winbind
```

### 5.3 Configure winbind in smb.conf
Add in `[global]`:
```ini
winbind use default domain = yes
template shell = /bin/bash
template homedir = /home/%U
```
→ `sudo testparm -s | grep winbind` should show `winbind use default domain = yes`
```bash
sudo nano /etc/samba/smb.conf
sudo systemctl restart samba-ad-dc
sudo testparm -s | grep winbind
```

### 5.4 Configure base permissions in Linux
→ `ls -la /srv/samba/` should show `drwxrwx--- root Domain Users` on each folder
```bash
sudo chown root:"Domain Users" /srv/samba/FinanceDocs
sudo chown root:"Domain Users" /srv/samba/HRDocs
sudo chown root:"Domain Users" /srv/samba/Public
sudo chmod 770 /srv/samba/FinanceDocs
sudo chmod 770 /srv/samba/HRDocs
sudo chmod 770 /srv/samba/Public
ls -la /srv/samba/
```

🛠 If "Domain Users: invalid group" fails (winbind doesn't resolve groups yet):
```bash
sudo chown root:root /srv/samba/FinanceDocs /srv/samba/HRDocs /srv/samba/Public
sudo chmod 777 /srv/samba/FinanceDocs /srv/samba/HRDocs /srv/samba/Public
# Safe: Samba will control actual access via Windows ACLs
```

### 5.5 – 5.6 Configure shared resources in smb.conf, verify and restart
Add at the end of the file (after `[netlogon]` and `[sysvol]`):
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

**Parameter explanation:**
- `vfs objects = acl_xattr` → Enables full NTFS ACL support (vital for managing from Windows).
- `map acl inherit = yes` → Allows inheriting Windows-style permissions.
- `guest ok = yes` → Only for Public, allows unauthenticated access.

```bash
sudo nano /etc/samba/smb.conf
sudo testparm
sudo smbcontrol all reload-config
sudo smbclient -L localhost -U Administrator%admin_21
```
→ `testparm` should say `Loaded services file OK`  
→ `smbclient -L` should list FinanceDocs, HRDocs, Public.

### 5.7 Basic access tests from Linux
> ⚠️ All users can still access everything — restrictions are configured from Windows in Sprint 3.

**Test 1: "user01" to "FinanceDocs"**
```bash
sudo smbclient //localhost/FinanceDocs -U user01%admin_21
# Inside the smb prompt: \>
# mkdir test_inicial
# ls
# exit
```

**Test 2: "alice" to "HRDocs"**
```bash
sudo smbclient //localhost/HRDocs -U alice%admin_21
# Inside the smb prompt: \>
# mkdir test_hr_inicial
# ls
# exit
```

## 🛠 SPRINT 2 RESCUE PLAN
```bash
# If resources don't appear:
sudo tail -100 /var/log/samba/log.smbd
sudo ufw status
```

---

## ✅ SPRINT 2 CHECKPOINT

```bash
ls -la /srv/samba/                                                   # → 3 folders
sudo testparm -s | grep "winbind use default domain"                 # → yes
sudo testparm -s | grep "vfs objects"                                # → acl_xattr
sudo smbclient -L localhost -U Administrator%admin_21                # → resources listed
sudo smbclient //localhost/FinanceDocs -U user01%admin_21 -c "ls"   # → no error
```

---

## 🎯 END OF SPRINT 2
- ✅ DHCP (192.168.11.100-150), ✅ 4 OUs, ✅ 7 users, ✅ 3 groups (Finance, HR, IT Support)
- ✅ 3 shared folders with ACLs enabled, ✅ Samba working

**Next:** SPRINT 3 → Integration of Ubuntu and Windows clients into the domain

---

<a name="sprint-3"></a>
# 🧱 SPRINT 3 – Windows Client Integration into the Domain

---

## 📋 WINDOWS CLIENT PREPARATION

**Requirements:** Windows 10 **Pro/Enterprise/Education** (Home CANNOT join a domain), 4GB RAM, 2 CPUs, 50GB disk. VM Name: `lc02`

**VirtualBox — Adapter 1:**
- Connected to: Internal Network
- Name: `intnet`
- Promiscuous mode: Allow all

**TCP/IPv4 configuration in Windows:**
- IP: `192.168.11.100` | Mask: `255.255.255.0`
- Gateway: `192.168.11.2` | Preferred DNS: `192.168.11.2`

### Initial state — verify before starting
→ Everything should respond correctly
```cmd
ipconfig /all
hostname
nslookup lab02.lan
ping 192.168.11.2
ping ls02.lab02.lan
ping www.amazon.es
```

---

## 🕒 HOUR 1: Join the Windows client to the domain

### 1.1 – 1.2 Join the domain
Steps:
1. Right-click "This PC" → Properties → Advanced system settings
2. "Computer Name" tab → Change → Member of: **Domain** → `lab02.lan`
3. Credentials: User `Administrator`, Password `admin_21`
4. Message "Welcome to the domain lab02.lan" → Restart

→ After restart: login screen should show "Sign in to: LAB02"

🛠 Frequent errors:
```cmd
REM "Cannot find the domain" → verify DNS
nslookup lab02.lan
nslookup _ldap._tcp.lab02.lan
ipconfig /flushdns

REM "Cannot connect to the domain" → verify LDAP
Test-NetConnection -ComputerName ls02.lab02.lan -Port 389
```
```bash
# From Ubuntu server
sudo systemctl status samba-ad-dc
```

### 1.3 Verify machine account from the server
→ `LC02$` should appear
```bash
sudo samba-tool computer list
sudo samba-tool computer show LC02$
```

---

## 🕒 HOUR 2: Log in with domain users

### 2.1 – 2.2 Log out and log in as user01
→ `whoami` should return `lab02\user01`
```cmd
whoami
echo %USERDOMAIN%
echo %USERNAME%
```

### 2.3 Test with other users
Log out and test:
- `bob` / `admin_21`
- `alice` / `admin_21`

🛠 If login fails:
```bash
sudo samba-tool user list | grep user01
sudo samba-tool user show user01 | grep -i "disabled\|locked"
sudo samba-tool user enable user01
sudo samba-tool user setpassword user01
```

---

## 🕒 HOUR 3: Accessing shared resources from Windows

### 3.1 Access server resources as user01
In the Explorer address bar: `\\ls02.lab02.lan` or `\\192.168.11.2`
Should show: FinanceDocs, HRDocs, Public, NETLOGON, SYSVOL

Create a test file in FinanceDocs → verify from server:
```bash
sudo ls -la /srv/samba/FinanceDocs/
# → test_user01.txt should appear
```

### 3.2 Map FinanceDocs as drive Z:
"This PC" → Map network drive → Drive `Z:` → `\\ls02.lab02.lan\FinanceDocs` → ✓ Reconnect at sign-in

→ Should display `Z: \\ls02.lab02.lan\FinanceDocs`
```cmd
net use
```

---

## 🕒 HOUR 4: Configure permissions (ACLs) from Windows

> Log in as `Administrator` / `admin_21`

### 4.2 FinanceDocs
1. `\\ls02.lab02.lan\FinanceDocs` → right-click → Properties → Security → Advanced
2. Disable inheritance → "Replace all entries..."
3. Remove everyone except: Domain Administrators, SYSTEM, CREATOR OWNER
4. Add group **Finance** → Full control → Allow → This folder, subfolders and files
5. Add group **HR** → Full control → **Deny**

### 4.3 HRDocs
Same process:
- Add **HR** → Full control → Allow
- Add **Finance** → Full control → Deny

### 4.4 Public
1. `\\ls02.lab02.lan\Public` → Properties → Security → Advanced
2. Disable inheritance (do NOT check "Replace...")
3. Final result:
   - Administrator → Full control
   - Domain Users → Read and execute
   - CREATOR OWNER → Full control (subfolders/files)

> **NOTE:** If a Windows warning appears in a window saying something like:  

> *"Windows Security. The current audit policy on this computer does not have auditing enabled. If this computer obtains the audit policy from the domain..."*

> Ignore it, don't pay attention to it, basically it's like Windows telling you: "Hey, if you wanted to log these accesses in the logs, I'm not currently doing that."

---

## 🕒 HOUR 5: Verify access restrictions

### 5.1 As user01 (Finance group)
- ✅ FinanceDocs → should open, create `test_acl_user01.txt`
- ❌ HRDocs → should show "You don't have permission to access"
- ✅ Public → opens but cannot create files

### 5.2 As alice (HR group)
- ✅ HRDocs → should open, create `test_acl_alice.txt`
- ❌ FinanceDocs → should show permissions error

### 5.3 Verify ACLs from the server
```bash
sudo getfacl /srv/samba/FinanceDocs/
sudo getfacl /srv/samba/HRDocs/
```

---

## 🕒 HOUR 6: Final verification of SPRINT 3

```bash
# From Ubuntu server
sudo samba-tool computer list         # → LC02$
```
```cmd
REM From Windows client
systeminfo | findstr /B /C:"Domain"  & REM → lab02.lan
net view \\ls02.lab02.lan             & REM → FinanceDocs, HRDocs, Public, NETLOGON, SYSVOL
gpresult /r
```

**Expected ACLs:**

| User | FinanceDocs | HRDocs | Public |
|---|---|---|---|
| user01 (Finance) | ✅ Access | ❌ Denied | ✅ Read only |
| alice (HR) | ❌ Denied | ✅ Access | ✅ Read only |

---

## 🛠 SPRINT 3 RESCUE PLAN

```bash
# Client doesn't join the domain
nslookup lab02.lan
sudo tail -100 /var/log/samba/log.samba

# User cannot log in
sudo samba-tool user setpassword user01

# ACLs not working
sudo testparm -s | grep vfs
sudo systemctl restart samba-ad-dc
```

---

## 🎯 END OF SPRINT 3
- ✅ Windows client configured and joined to lab02.lan
- ✅ Domain users log in, ✅ Shared resources accessible
- ✅ ACLs configured from Windows, ✅ Group-based access restrictions verified

**Next:** SPRINT 4 → GPOs, Ubuntu Client and System Management

---

<a name="sprint-4"></a>
# 🧱 SPRINT 4 – GPOs, Ubuntu Client and System Management

---

## 🕒 HOUR 1: GPO configuration (Ubuntu commands + Windows RSAT)

> **Structure:**
> - **Part A** — Create empty GPOs and link them from Ubuntu (structure only)
> - **Part B** — Configure real policies with RSAT from Windows

---

## 📋 PART A: CREATING GPOs FROM UBUNTU

> ⚠️ Limitation: `samba-tool` only creates empty GPOs. Real content is configured with RSAT.

### A.1 – A.2 Authenticate with Kerberos and view existing GPOs
→ `klist` should show ticket for `Administrator@LAB02.LAN`
```bash
kdestroy
kinit Administrator
# Password: admin_21
klist
sudo samba-tool gpo listall
```

### A.3 – A.4 Create empty GPOs
→ Note the generated GUIDs (needed for linking)
```bash
sudo samba-tool gpo create "Restricciones_Usuarios" -U Administrator
sudo samba-tool gpo create "Configuracion_Escritorio" -U Administrator
sudo samba-tool gpo listall
```

### A.6 Link GPOs to OU=Students
Replace `{GUID_...}` with the GUIDs obtained in the previous step.

→ `gpo getlink` should show both linked GUIDs
```bash
sudo samba-tool gpo setlink "OU=Students,DC=lab02,DC=lan" "{GUID_DE_Restricciones_Usuarios}" -U Administrator
sudo samba-tool gpo setlink "OU=Students,DC=lab02,DC=lan" "{GUID_DE_Configuracion_Escritorio}" -U Administrator
sudo samba-tool gpo getlink "OU=Students,DC=lab02,DC=lan"
```

### A.7 Verify password policies (already configured in Sprint 1)
```bash
sudo samba-tool domain passwordsettings show
```

### A.8 – A.9 Verify physical structure and Part A checkpoint
```bash
ls -la /var/lib/samba/sysvol/lab02.lan/Policies/
sudo samba-tool gpo listall | grep -E "Restricciones|Configuracion"
sudo samba-tool gpo getlink "OU=Students,DC=lab02,DC=lan"
```

---

## 📋 PART B: GPO CONFIGURATION FROM WINDOWS (RSAT)

### B.1 Install RSAT on Windows 10
Start → Settings → Apps → Optional features → Add:
- `RSAT: Group Policy Management Tools`
- `RSAT: AD DS and AD LDS Tools`

→ Search for "Group Policy Management" in the start menu — it should appear.

🛠 If it doesn't appear in Optional features:
```powershell
Get-WindowsCapability -Name RSAT* -Online | Add-WindowsCapability -Online
```

### B.2 Open GPO Console and verify
Start → "Group Policy Management" → Forest: lab02.lan → Domains → lab02.lan → Group Policy Objects
→ `Restricciones_Usuarios` and `Configuracion_Escritorio` should appear

### B.3 Configure GPO: Prohibit Control Panel
Right-click `Restricciones_Usuarios` → Edit:
```
User Configuration
  → Policies → Administrative Templates
    → Control Panel → Personalization
      → "Prohibit access to Control Panel and PC settings" → Enabled
```

### B.4 Configure GPO: Desktop Wallpaper
Right-click `Configuracion_Escritorio` → Edit:
```
User Configuration → Policies → Administrative Templates
  → Control Panel → Personalization → "Prevent changing desktop background"

User Configuration → Policies → Administrative Templates
  → Active Desktop → Active Desktop → "Desktop Wallpaper"
```

### B.6 – B.7 Force application and verify from client (as user01)
→ Control Panel should be locked. HTML report should list both GPOs.
```cmd
gpupdate /force
gpresult /r
gpresult /h C:\gpo_report.html
```

### B.8 Additional useful GPOs (configure from RSAT)

| GPO | Path | Configuration |
|---|---|---|
| Block CMD | User Config → Admin Templates → System | Prevent access to the command prompt → Enabled |
| Disable Task Manager | User Config → Admin Templates → System → Ctrl+Alt+Del Options | Remove Task Manager → Enabled |
| Block Registry | User Config → Admin Templates → System | Prevent access to registry editing tools → Enabled |
| Hide drives | User Config → Admin Templates → Windows Components → File Explorer | Hide these specified drives → Enabled |

---

## 🎯 HOUR 1 CHECKPOINT

**Part A:** GPOs created and linked from Ubuntu ✅  
**Part B:** RSAT installed, GPOs configured and applied ✅

```bash
sudo samba-tool gpo getlink "OU=Students,DC=lab02,DC=lan"   # → GUIDs of both GPOs
```

## 🛠 GPO Troubleshooting

```bash
# Permission error when editing GPO from RSAT
sudo samba-tool ntacl sysvolreset
sudo systemctl restart samba-ad-dc

# GPO not applying on clients → from Windows
# gpupdate /force → eventvwr.msc → Windows Logs → System → filter "Group Policy"

# RSAT doesn't show GPOs → verify from server
sudo samba-tool gpo listall
```

---

## 🕒 HOUR 2: Ubuntu client preparation and domain join

**VM Requirements:** Ubuntu 22.04/24.04, name `lc02-ubu`, 2GB RAM, 20GB disk, Adapter 1: Internal Network (intnet)

### 2.1 – 2.3 Configure network, hostname and /etc/hosts
→ `ping lab02.lan` and `ping ls02.lab02.lan` should reply from `192.168.11.2`
```bash
sudo nano /etc/netplan/01-netcfg.yaml
sudo netplan apply
sudo hostnamectl set-hostname lc02-ubu
sudo nano /etc/hosts
# Add:
# 127.0.0.1    localhost
# 127.0.1.1    lc02-ubu
# 192.168.11.2 ls02.lab02.lan ls02
# 192.168.11.2 lab02.lan

ping lab02.lan
ping ls02.lab02.lan
```

Netplan content:
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

### 2.4 – 2.5 Install packages and discover domain
During Kerberos: realm `LAB02.LAN`, servers and admin `ls02.lab02.lan`
→ `realm discover` should show `configured: no` and `server-software: active-directory`
```bash
sudo apt update
sudo apt install -y realmd sssd sssd-tools adcli krb5-user samba-common-bin packagekit
sudo realm discover lab02.lan
```

### 2.6 Join the domain and verify
→ `realm list` should show `configured: kerberos-member`. From server `lc02-ubu$` should appear.
```bash
sudo realm join -U Administrator lab02.lan --verbose
# Password: admin_21
sudo realm list
```
```bash
# From Ubuntu server
sudo samba-tool computer list
```

### 2.7 – 2.8 Configure SSSD and home directories
In `/etc/sssd/sssd.conf`, section `[domain/lab02.lan]`, add:
```ini
fallback_homedir = /home/%u@%d
default_shell = /bin/bash
```
```bash
sudo nano /etc/sssd/sssd.conf
sudo systemctl restart sssd
sudo pam-auth-update --enable mkhomedir
# Select with SPACE: [*] Create home directory on login → Tab → Ok
```

### 2.9 Log in with a domain user
→ `whoami` should show `bob@lab02.lan`, `pwd` should show `/home/bob@lab02.lan`
```bash
su - bob@lab02.lan
# Password: admin_21
whoami
pwd
```

---

## 🕒 HOUR 3: Mounting shared resources from Linux

### 3.1 – 3.2 Install CIFS and verify resources
→ Should list FinanceDocs, HRDocs, Public, NETLOGON, SYSVOL
```bash
sudo apt update
sudo apt install -y cifs-utils smbclient
smbclient -L //ls02.lab02.lan -U bob
# Password: admin_21
```

### 3.3 – 3.4 Test access and manual mount
→ `ls -la /mnt/financedocs` should show content
```bash
smbclient //ls02.lab02.lan/FinanceDocs -U user01
# Inside the prompt: ls, mkdir test_from_ubuntu, ls, exit

sudo mkdir -p /mnt/financedocs
sudo mount -t cifs //ls02.lab02.lan/FinanceDocs /mnt/financedocs -o username=user01,password=admin_21,uid=1000,gid=1000
ls -la /mnt/financedocs
echo "Test from Ubuntu client" | sudo tee /mnt/financedocs/test_ubuntu.txt
sudo umount /mnt/financedocs
```

### 3.5 Automatic mount in /etc/fstab
Create credentials, protect them and add entry to fstab.
→ `df -h | grep financedocs` should show the mounted resource (also after reboot).
```bash
sudo nano /root/.smbcredentials
# Content:
# username=user01
# password=admin_21
# domain=LAB02

sudo chmod 600 /root/.smbcredentials
sudo mkdir -p /mnt/financedocs
sudo nano /etc/fstab
# Add at the end:
# //ls02.lab02.lan/FinanceDocs /mnt/financedocs cifs credentials=/root/.smbcredentials,uid=1000,gid=1000,iocharset=utf8 0 0

sudo mount -a
df -h | grep financedocs
sudo reboot
# After reboot:
df -h | grep financedocs
```

---

## 🕒 HOUR 4: Process management (practical activity with SSH)

### 4.1 – 4.2 Install sl and connect via SSH from the server
```bash
# On the Ubuntu client:
sudo apt install -y openssh-server sl
sudo systemctl enable ssh && sudo systemctl start ssh

# From server ls02:
ssh bob@192.168.11.101
# Password: admin_21
```

> **NOTE:** This can help to copy the PID directly
```bash
# Option 1 — With xclip (X11)
ps aux | awk '$11=="sl"{print $2}' | xclip -selection clipboard

# Option 2 — With wl-copy (Wayland)
ps aux | awk '$11=="sl"{print $2}' | wl-copy

# Option 3 — More robust (avoids false positives), ps aux is noisy. Better:
pgrep -x sl | xclip -selection clipboard

# or in Wayland:
pgrep -x sl | wl-copy
```

### 4.3 Run sl from the client (Locomotive in the Terminal)
In another terminal of the client (as `bob@lab02.lan`):
```bash
# Normal train
sl

# The train is longer
sl -l

# All "sl" options
sl -h
```

### 4.4 – 4.7 Identify, pause, resume and kill the process from the server (via SSH)
Replace `12345` with the actual PID obtained.
```bash
# Identify PID (2 options)
ps -aux | grep -w "sl"
ps aux | awk '$11=="sl"' | grep sl

# Pause (train freezes) - Signal 19 = SIGSTOP
kill -19 12345

# Resume (train continues) - Signal 18 = SIGCONT
kill -18 12345

# Kill - Signal 9 = SIGKILL (terminate immediately, cannot be ignored)
kill -9 12345
ps aux | grep sl
```

### 4.8 Process monitoring
```bash
top
# Filter by user: press 'u' → type 'bob' → Enter
# Exit: 'q'
```

---

## 🕒 HOUR 5: Scheduled tasks with CRON (Automatic Samba Backup)

### 5.1 – 5.2 Create backup script and set permissions
```bash
sudo nano /root/backup_samba.sh
sudo chmod +x /root/backup_samba.sh
```

Script content:
```bash
#!/bin/bash

# --- CONFIGURATION ---
DIR_DESTINO="/root/backups"
LOG_FILE="/var/log/samba_backup.log"
DIAS_A_GUARDAR=30

# --- COMMANDS (Absolute paths for CRON) ---
TAR=/bin/tar
DATE=/bin/date
ECHO=/bin/echo
FIND=/usr/bin/find
MKDIR=/bin/mkdir

# --- VARIABLES ---
FECHA=$($DATE +%F_%H-%M)
NOMBRE_ARCHIVO="backup_ad_$FECHA.tar.gz"
RUTA_COMPLETA="$DIR_DESTINO/$NOMBRE_ARCHIVO"

# --- CREATE DIRECTORY IF IT DOESN'T EXIST ---
if [ ! -d "$DIR_DESTINO" ]; then
    $MKDIR -p "$DIR_DESTINO"
fi

# --- 1. RUN BACKUP ---
$TAR -czf "$RUTA_COMPLETA" /var/lib/samba /etc/samba 2>/dev/null

# --- 2. VERIFICATION AND LOG ---
if [ $? -eq 0 ]; then
    $ECHO "[$FECHA] OK: Backup created successfully: $NOMBRE_ARCHIVO" >> $LOG_FILE
    # --- 3. CLEANUP (Only if backup succeeded) ---
    $FIND $DIR_DESTINO -name "backup_ad_*.tar.gz" -mtime +$DIAS_A_GUARDAR -delete
else
    $ECHO "[$FECHA] ERROR: Backup creation failed. Check disk space." >> $LOG_FILE
fi
```

### 5.3 Test the script manually
→ `cat /var/log/samba_backup.log` should show `OK: Backup created successfully`
```bash
sudo /root/backup_samba.sh
ls -lh /root/backups/
cat /var/log/samba_backup.log
```

### 5.4 Schedule with CRON
→ `sudo crontab -l` should show the added line
```bash
sudo crontab -e
# Add at the end:
# 0 2 * * * /root/backup_samba.sh

sudo crontab -l
```

### 5.5 Quick CRON format reference

```
# Every 6 hours
0 */6 * * * /root/backup_samba.sh

# Every Sunday at 3:00 AM
0 3 * * 0 /root/backup_samba.sh

# Every day at 23:30
30 23 * * * /root/backup_samba.sh
```

```
* * * * * command
│ │ │ │ └─── Day of week (0-7, 0=Sunday)
│ │ │ └───── Month (1-12)
│ │ └─────── Day of month (1-31)
│ └───────── Hour (0-23)
└─────────── Minute (0-59)
```

### 5.6 – 5.7 Verify CRON and restore if necessary
```bash
sudo grep CRON /var/log/syslog | tail -20
tail -f /var/log/samba_backup.log

# Restore backup (CAUTION: overwrites current files)
ls -lh /root/backups/
sudo tar -xzf /root/backups/backup_ad_DATE.tar.gz -C /
```

---

## 🕒 HOUR 6: Security and Auditing (Samba Audit)

### 6.1 Configure auditing in smb.conf
Modify `[FinanceDocs]`, `[HRDocs]` and `[Public]` adding the audit lines:
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

### 6.2 – 6.4 Configure rsyslog, create log and restart services
→ Both services should be `active (running)`
```bash
sudo nano /etc/rsyslog.d/samba-audit.conf
# Content:
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

### 6.5 Generate audit events
From Ubuntu client:
```bash
smbclient //ls02.lab02.lan/FinanceDocs -U user01
# Inside the prompt:
# mkdir audit_folder
# rmdir audit_folder
# exit
```

### 6.6 – 6.7 Verify and filter logs
```bash
tail -f /var/log/samba_audit.log
grep "user01" /var/log/samba_audit.log
grep "mkdirat" /var/log/samba_audit.log
grep "fail" /var/log/samba_audit.log
grep "$(date +%Y-%m-%d)" /var/log/samba_audit.log
```

Log format: `User | Client IP | Client Name | Resource | Action | Result | File`

### 6.8 Configure log rotation (recommended)
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
**Explanation:**
- `daily`: Rotate every day
- `rotate 30`: Keep 30 files
- `compress`: Compress old files
- `create 0640 syslog adm`: Permissions for the new file

---

## ✅ FINAL CHECKPOINT OF SPRINT 4

```bash
# From Windows client (as user01)
gpresult /r                                 # → Restricciones_Usuarios, Configuracion_Escritorio

# From Ubuntu client
sudo realm list                             # → configured: kerberos-member
df -h | grep financedocs                    # → resource mounted

# From server
ps aux | grep bob                           # → bob's processes visible
sudo crontab -l                             # → backup scheduled
ls -lh /root/backups/                       # → .tar.gz files
cat /var/log/samba_backup.log               # → OK: Backup created
tail -20 /var/log/samba_audit.log           # → access events
```

---

## 🛠 SPRINT 4 RESCUE PLAN

```bash
# RSAT won't install → PowerShell as admin on Windows:
# Get-WindowsCapability -Name RSAT* -Online | Add-WindowsCapability -Online

# Ubuntu client won't join the domain
nslookup lab02.lan
sudo journalctl -xe

# Resources won't mount
cat /root/.smbcredentials
sudo tail -50 /var/log/samba/log.smbd

# CRON doesn't execute
sudo grep CRON /var/log/syslog
sudo /root/backup_samba.sh               # test manually

# Auditing doesn't log
sudo testparm -s | grep full_audit
sudo systemctl status rsyslog
sudo systemctl restart samba-ad-dc
```

---

## 🎯 END OF SPRINT 4
- ✅ GPOs created from Ubuntu and configured with RSAT
- ✅ Ubuntu client joined to domain lab02.lan
- ✅ Shared resources automatically mounted via fstab
- ✅ Remote process management via SSH
- ✅ Automatic backup with CRON + automatic cleanup
- ✅ Complete auditing of Samba accesses

**Final state:**

| Component | Status |
|---|---|
| Ubuntu Server | Full DC with auditing |
| Windows Client | Joined, GPOs applied, ACLs configured |
| Ubuntu Client | Joined, resources mounted, remote management |
| Automation | Daily scheduled backups |
| Security | Auditing of all accesses |

**Next:** SPRINT 5 → Forest Trust between domains

---

<a name="sprint-5"></a>
# 🧱 SPRINT 5 – Forest Trust between two Samba AD DC servers

**Objective:** Establish a bidirectional Forest Trust between two Samba AD DC servers in VirtualBox.

---

## 📋 ARCHITECTURE

```
SERVER 1 (already existing)         SERVER 2 (new)
──────────────────────            ──────────────────────
IP:       192.168.11.2            IP:       192.168.11.3
Hostname: ls02                    Hostname: ls03
Domain:   lab02.lan               Domain:   lab03.lan
Realm:    LAB02.LAN               Realm:    LAB03.LAN
```

---

## 🌐 NETWORK CONFIGURATION IN VIRTUALBOX

### Option A: Internal Network (RECOMMENDED)

Both servers on an isolated private network with static IPs 192.168.11.X. NAT on Adapter 1 for Internet, Internal Network on Adapter 2 for communication between servers.

**Server 1 (ls02):**
- Adapter 1: NAT
- Adapter 2: Internal network — name `intnet` — IP `192.168.11.2/24`

**Server 2 (ls03):**
- Adapter 1: NAT
- Adapter 2: Internal network — name `intnet` (SAME as Server 1) — IP `192.168.11.3/24`

### Option B: Bridged Adapter — ALTERNATIVE

Each server gets an IP from the physical router. More exposed, less isolated. IPs according to DHCP or static on the physical network range.

> ⚠️ This guide uses **Internal Network (Option A)**.

---

## 🖥️ PART 1: Create and configure Server 2

### Create new VM in VirtualBox
Parameters:
- Name: `Servidor2-Samba` | Type: Linux | Version: Ubuntu (64-bit)
- RAM: 2048 MB minimum | Disk: VDI, 20 GB
- Install Ubuntu Server 24.04

### Configure network adapters (VM powered off)
In VirtualBox → right-click VM → Settings → Network:
- **Adapter 1:** NAT ✅
- **Adapter 2:** Internal network ✅ — name `intnet`

### Step 1: Configure static network on Server 2
→ Should display `inet 192.168.11.3/24`
```bash
sudo nano /etc/netplan/01-netcfg.yaml
sudo netplan apply
ip addr show enp0s8
```

Netplan content:
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

### Step 2: Verify connectivity between servers
→ Both pings should work
```bash
# From Server 2:
ping -c 2 192.168.11.2

# From Server 1:
ping -c 2 192.168.11.3
```

### Step 3: Configure hostname and /etc/hosts on Server 2
→ `hostnamectl` should show `ls03`
```bash
sudo hostnamectl set-hostname ls03
hostnamectl
sudo nano /etc/hosts
```

Content of /etc/hosts:
```
127.0.0.1       localhost
127.0.1.1       ls03.lab03.lan ls03
192.168.11.3    ls03.lab03.lan ls03
192.168.11.2    ls02.lab02.lan ls02
```

---

## 🌐 PART 2: Install Samba AD DC on Server 2

### Step 4: Update and install Samba
During Kerberos: realm `LAB03.LAN`, servers and admin `ls03.lab03.lan`
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y samba smbclient winbind krb5-user krb5-config
```

### Step 5: Disable systemd-resolved and create resolv.conf
```bash
sudo systemctl disable --now systemd-resolved
sudo unlink /etc/resolv.conf
sudo nano /etc/resolv.conf
# Content:
# nameserver 127.0.0.1
# nameserver 10.239.3.7
# search lab03.lan

sudo chattr +i /etc/resolv.conf
```

### Step 6: Stop default Samba services and back up smb.conf
```bash
sudo systemctl stop smbd nmbd winbind
sudo systemctl disable smbd nmbd winbind
sudo mv /etc/samba/smb.conf /etc/samba/smb.conf.bak 2>/dev/null || true
```

### Step 7: Provision the lab03.lan domain
Wizard answers:
- Realm: `LAB03.LAN` | Domain: `LAB03` | Server Role: `dc`
- DNS backend: `SAMBA_INTERNAL`
- DNS forwarder: `10.239.3.7 192.168.11.2`
- Administrator password: `admin_21`

→ Should display `Provision OK for domain DN DC=lab03,DC=lan`
```bash
sudo samba-tool domain provision --use-rfc2307 --interactive
```

### Step 8: Copy Kerberos, configure interfaces and start Samba
Add in `[global]` of smb.conf: `interfaces = lo enp0s8` and `bind interfaces only = yes`

→ Should display `active (running)`
```bash
sudo cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
sudo nano /etc/samba/smb.conf
sudo systemctl unmask samba-ad-dc
sudo systemctl start samba-ad-dc
sudo systemctl enable samba-ad-dc
sudo systemctl status samba-ad-dc
```

### Step 9: Verify DNS and Kerberos on Server 2
→ `host` should resolve `192.168.11.3`. `klist` should show ticket for `Administrator@LAB03.LAN`.
```bash
host lab03.lan
host ls03.lab03.lan
host -t SRV _ldap._tcp.lab03.lan
kinit Administrator
# Password: admin_21
klist
```

---

## 🔗 PART 3: Configure Server 1 for the Trust

### Step 10: Add /etc/hosts and DNS forwarder on Server 1
On Server 1 (ls02), add to `/etc/hosts`: `192.168.11.3    ls03.lab03.lan ls03 lab03.lan`

In `[global]` of smb.conf modify: `dns forwarder = 10.239.3.7 192.168.11.3`
```bash
# On Server 1:
sudo nano /etc/hosts
sudo nano /etc/samba/smb.conf
sudo systemctl restart samba-ad-dc
```

### Step 11: Configure DNS forwarder on Server 2
In `[global]` of smb.conf modify: `dns forwarder = 10.239.3.7 192.168.11.2`
```bash
# On Server 2:
sudo nano /etc/samba/smb.conf
sudo systemctl restart samba-ad-dc
```

---

## 📋 PART 4: Verify cross DNS resolution

### Step 12: Server 1 resolves Server 2
→ Each command should return `192.168.11.3` (or SRV record pointing to `ls03.lab03.lan`)
```bash
# On Server 1:
host lab03.lan
host ls03.lab03.lan
host -t SRV _ldap._tcp.lab03.lan
```

### Step 13: Server 2 resolves Server 1
→ Each command should return `192.168.11.2` (or SRV record pointing to `ls02.lab02.lan`)
```bash
# On Server 2:
host lab02.lan
host ls02.lab02.lan
host -t SRV _ldap._tcp.lab02.lan
```

> ✅ If both servers resolve correctly, proceed with the Trust.

---

## 🤝 PART 5: Create bidirectional Forest Trust

### Step 14: Create Trust from Server 1 → Server 2
When prompted for the remote Administrator password: `admin_21`

→ Should display `Successfully created trust`
```bash
# On Server 1:
kdestroy
kinit Administrator@LAB02.LAN
sudo samba-tool domain trust create lab03.lan \
    --type=forest \
    --direction=both \
    -U Administrator%admin_21
```

### Step 15: Create Trust from Server 2 → Server 1
When prompted for the remote Administrator password: `admin_21`

→ Should display `Successfully created trust`
```bash
# On Server 2:
kdestroy
kinit Administrator@LAB03.LAN
sudo samba-tool domain trust create lab02.lan \
    --type=forest \
    --direction=both \
    -U Administrator%admin_21
```

---

## ✅ PART 6: Verify Trust

### Step 16: Verify Trust on Server 1
→ Should display `lab03.lan | forest | both | yes`
```bash
# On Server 1:
sudo samba-tool domain trust list
sudo samba-tool domain trust show lab03.lan
```

### Step 17: Verify Trust on Server 2
→ Should display `lab02.lan | forest | both | yes`
```bash
# On Server 2:
sudo samba-tool domain trust list
sudo samba-tool domain trust show lab02.lan
```

---

## ✅ FINAL CHECKPOINT

```bash
# On Server 1:
sudo samba-tool domain trust list         # → Trust with lab03.lan active
host lab03.lan                            # → 192.168.11.3
sudo systemctl status samba-ad-dc | grep Active
klist

# On Server 2:
sudo samba-tool domain trust list         # → Trust with lab02.lan active
host lab02.lan                            # → 192.168.11.2
sudo systemctl status samba-ad-dc | grep Active
klist
```

---

## 🛠️ TROUBLESHOOTING

### Error: "Failed to find a writeable DC"
Cause: DNS doesn't resolve correctly. There may be incorrect local DNS zones.
```bash
# Verify local DNS zones
sudo samba-tool dns zonelist 127.0.0.1 -U Administrator%admin_21

# If the other domain's zone appears (e.g. lab03.lan on Server 1), delete it
sudo samba-tool dns zonedelete 127.0.0.1 lab03.lan -U Administrator%admin_21

sudo systemctl restart samba-ad-dc
host lab03.lan
```

### Error: "The object name is not found"
```bash
cat /etc/hosts | grep lab03
ping -c 2 192.168.11.3
# On Server 2, verify that Samba is running:
sudo systemctl status samba-ad-dc
```

### Trust is created but doesn't appear in the list
```bash
sudo samba-tool domain trust delete lab03.lan -U Administrator%admin_21
kdestroy
sudo systemctl restart samba-ad-dc
sleep 10
kinit Administrator
sudo samba-tool domain trust create lab03.lan --type=forest --direction=both -U Administrator%admin_21
```

### Kerberos time difference (Clock skew)
```bash
sudo timedatectl set-ntp true
timedatectl status
```

---

## 📝 Internal Network vs Bridge — quick reference

| | Internal Network | Bridge |
|---|---|---|
| IPs | 192.168.11.X (fixed, defined by you) | According to router (192.168.1.X or similar) |
| Isolation | ✅ Isolated from the physical network | ❌ Exposed to the network |
| Internet | ❌ Needs NAT on Adapter 1 | ✅ Direct without NAT |
| Communication between VMs | ✅ Direct without issues | ✅ Direct |

---

## 🎯 END OF SPRINT 5

- ✅ Server 2 VM created and configured in VirtualBox
- ✅ Samba AD DC installed on Server 2 (lab03.lan)
- ✅ Internal network between both servers
- ✅ Bidirectional DNS Forwarders configured
- ✅ Cross DNS resolution verified
- ✅ Bidirectional Forest Trust established and verified on both servers

**Final architecture:**
```
Server 1 (ls02.lab02.lan — 192.168.11.2)
            ↕ Bidirectional Forest Trust
Server 2 (ls03.lab03.lan — 192.168.11.3)
```
