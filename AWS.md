# AWS Academy: Ubuntu Samba Server + Windows Client (Complete Guide)

**Objective:** Deploy Ubuntu server with Samba AD DC and Windows Server client on AWS EC2 with full connectivity and RDP access from Linux.

---

## 📋 FINAL ARCHITECTURE

```
┌─────────────────────────────────────────────────────┐
│              AWS EC2 - LAB VPC                      │
│                                                     │
│  ┌─────────────────────┐  ┌─────────────────────┐   │
│  │  Ubuntu Server      │  │  Windows Server     │   │
│  │  Samba AD DC        │  │  Joined client      │   │
│  │                     │  │                     │   │
│  │  Public IP: X.X.X   │  │  Public IP: Y.Y.Y   │   │
│  │  Private IP: 10.0.X │  │  Private IP: 10.0.Y │   │
│  │  Domain: awslab.lan │  │                     │   │
│  └─────────────────────┘  └─────────────────────┘   │
│           ↕                         ↕               │
│     Security Group LAB-SG                           │
└─────────────────────────────────────────────────────┘
           ↕                         ↕
    SSH from Linux          RDP from Linux
      (port 22)              (port 3389)
```

---

## 🚀 PART 1: Set up AWS Learner Lab

### Step 1: Start the lab

```
1. Go to https://awsacademy.instructure.com/
2. Log in with your student email
3. Click on your course (AWS Academy Learner Lab)
4. Left menu → "Modules" → "Learner Lab"
5. Click "Start Lab" (green button)
6. Wait 1-3 minutes until the circle is 🟢 green
7. Click "AWS" (opens the AWS console)
```

---

### Step 2: Download SSH key pair

```
1. On the Learner Lab page, click "AWS Details" (above)
2. Click "Download PEM" (next to SSH Key)
3. "labsuser.pem" is downloaded to Downloads
4. Save in a safe place
```

**Prepare the key in Linux:**
```bash
# Move to ~/.ssh/
mv ~/Downloads/labsuser.pem ~/.ssh/

# Set correct permissions
chmod 400 ~/.ssh/labsuser.pem
```

---

## 🔐 PART 2: Configure Security Group

### Step 3: Create shared Security Group

**In the AWS console:**

```
1. Search: "VPC"
2. Click on "VPC" (service)
3. Left menu → "Security Groups"
4. Click on "Create security group"
```

**Configuration:**
```
Name: LAB-SG
Description: Security group for Samba AD DC and Windows client
VPC: Select the lab VPC (example: Lab VPC or vpc-XXXXXXX)
```

---

### Step 4: Add inbound rules

**Click "Add rule" for each one:**

| Type | Port | Protocol | Source | Description |
|------|------|----------|--------|-------------|
| SSH | 22 | TCP | 0.0.0.0/0 | SSH from anywhere |
| RDP | 3389 | TCP | 0.0.0.0/0 | RDP from anywhere |
| Custom TCP | 53 | TCP | 10.0.0.0/20 | Internal DNS (TCP) |
| Custom UDP | 53 | UDP | 10.0.0.0/20 | Internal DNS (UDP) |
| Custom TCP | 88 | TCP | 10.0.0.0/20 | Internal Kerberos |
| Custom UDP | 88 | UDP | 10.0.0.0/20 | Internal Kerberos UDP |
| Custom TCP | 389 | TCP | 10.0.0.0/20 | Internal LDAP |
| Custom TCP | 445 | TCP | 10.0.0.0/20 | Internal SMB/CIFS |
| Custom TCP | 636 | TCP | 10.0.0.0/20 | Internal LDAPS |
| Custom TCP | 3268 | TCP | 10.0.0.0/20 | Internal Global Catalog |
| Custom TCP | 3269 | TCP | 10.0.0.0/20 | Global Catalog SSL |
| All traffic | All | All | 10.0.0.0/20 | Internal VPC communication |

⚠️ **IMPORTANT:** 
- `10.0.0.0/20` is the internal VPC range (adjust according to your VPC)
- SSH and RDP from `0.0.0.0/0` for access from home
- AD ports only accessible internally (security)

**Click "Create security group"**

---

## 🖥️ PART 3: Create Ubuntu Server instance

### Step 5: Launch Ubuntu instance

**In the AWS console:**

```
1. Search: "EC2"
2. Click on "EC2"
3. Left menu → "Instances"
4. Click on "Launch instances"
```

---

### Step 6: Configure Ubuntu instance

| Field | Value |
|-------|-------|
| **Name** | `ubuntu-samba-server` |
| **AMI** | Ubuntu Server 24.04 LTS |
| **Instance type** | `t3.micro` (2 vCPU, 2 GiB RAM) |
| **Key pair** | `vockey` (or the one you downloaded) |

---

**Network configuration:**

```
1. In "Network settings", click "Edit"

2. VPC: Select the lab VPC (Lab VPC)

3. Subnet: Any public subnet (example: subnet-public1)

4. Auto-assign public IP: ✅ Enable

5. Security group: Select existing security group
   - Select: LAB-SG (the one we created earlier)
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

**Click "Launch instance"**

---

### Step 7: Assign Elastic IP to Ubuntu server

⚠️ **Why Elastic IP?** Normal public IPs change when the instance is restarted. Elastic IPs are fixed.

```
1. EC2 → Left menu → "Elastic IPs"
2. Click "Allocate Elastic IP address"
3. Click "Allocate"
4. Select the newly created Elastic IP
5. Actions → Associate Elastic IP address
6. Instance: Select "ubuntu-samba-server"
7. Private IP: Leave the one that appears
8. Click "Associate"
```

📝 **Note down:**
```
Elastic IP of Ubuntu server: ___.___.___.___ (example: 54.173.102.89)
```

---

### Step 8: Get the server's private IP

```
1. EC2 → Instances
2. Select "ubuntu-samba-server"
3. Bottom panel "Details" → Note down:
   - Private IPv4 addresses (example: 10.0.1.226)
```

📝 **Note down:**
```
Private IP of Ubuntu server: 10.0.___.___
```

---

## 💻 PART 4: Create Windows Server instance

### Step 9: Launch Windows instance

```
1. EC2 → Instances → Launch instances
2. Name: windows-client
```

---

### Step 10: Configure Windows instance

| Field | Value |
|-------|-------|
| **Name** | `windows-client` |
| **AMI** | Microsoft Windows Server 2022 Base |
| **Instance type** | `t3.micro` (2 vCPU, 2 GiB RAM) |
| **Key pair** | `vockey` (same as Ubuntu) |

---

**Network configuration:**

```
1. In "Network settings", click "Edit"

2. VPC: SAME VPC as the Ubuntu server

3. Subnet: SAME subnet as the Ubuntu server (or any public one in the VPC)

4. Auto-assign public IP: ✅ Enable

5. Security group: Select existing security group
   - Select: LAB-SG (the same as Ubuntu)
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

**Click "Launch instance"**

---

### Step 11: Assign Elastic IP to Windows

```
1. EC2 → Elastic IPs → Allocate Elastic IP address
2. Allocate
3. Select the new Elastic IP
4. Actions → Associate Elastic IP address
5. Instance: Select "windows-client"
6. Click "Associate"
```

📝 **Note down:**
```
Elastic IP of Windows: ___.___.___.___ (example: 54.221.100.222)
```

---

### Step 12: Get Windows private IP

```
1. EC2 → Instances → Select "windows-client"
2. Bottom panel → Note down:
   - Private IPv4 addresses (example: 10.0.14.107)
```

📝 **Note down:**
```
Private IP of Windows: 10.0.___.___
```

---

### Step 13: Get Windows Administrator password

⚠️ **IMPORTANT:** Wait 5-7 minutes after launching the instance before doing this.

```
1. EC2 → Instances → Select "windows-client"
2. "Connect" button (above)
3. "RDP client" tab
4. Click "Get password"
5. Click "Upload private key file"
6. Select: labsuser.pem (from ~/.ssh/)
7. Click "Decrypt password"
8. Copy the password that appears
```

📝 **Note down:**
```
Windows user: Administrator
Windows password: _________________ (example: xY9!mK2@pL5#qR8)
```

---

## 🔗 PART 5: Connect via RDP from Linux

### Step 14: Install FreeRDP on your Linux machine

**From your local Linux Mint:**

```bash
# Install FreeRDP
sudo apt update
sudo apt install -y freerdp2-x11

# Verify installation
xfreerdp --version
```

---

### Step 15: Connect via RDP to Windows Server

**Connect with xfreerdp:**

```bash
xfreerdp /v:54.221.100.222 \
         /u:Administrator \
         /p:'xY9!mK2@pL5#qR8' \
         /cert:ignore \
         /dynamic-resolution \
         /clipboard
```

**Change:**
- `54.221.100.222` → Your Windows Elastic IP
- `xY9!mK2@pL5#qR8` → Your Windows password

**Parameter explanation:**
```
/v:          → IP of the Windows server
/u:          → User (Administrator)
/p:          → Password (in single quotes)
/cert:ignore → Ignore SSL certificate
/dynamic-resolution → Automatically adjust resolution
/clipboard   → Share clipboard
```

---

### Step 16: Initial configuration in Windows

**Once inside RDP:**

**1. Wait for initial configuration (1-2 minutes)**

**2. Change password to something simpler:**

Open PowerShell (as Administrator):
```
Right-click on Start → Windows PowerShell (Admin)
```

Run:
```powershell
# Change password to admin_21
net user Administrator admin_21
```

**3. Configure Spanish keyboard:**

```powershell
# Configure Spanish keyboard
Set-WinUserLanguageList -LanguageList es-ES -Force
```

**4. Allow ICMP (ping) in the firewall:**

```powershell
# Allow ping
netsh advfirewall firewall add rule name="ICMP Allow" protocol=icmpv4:8,any dir=in action=allow
```

**Restart the RDP session:**
```
Start → Restart
```

---

### Step 17: Reconnect via RDP with new password

```bash
xfreerdp /v:54.221.100.222 \
         /u:Administrator \
         /p:'admin_21' \
         /cert:ignore \
         /dynamic-resolution \
         /clipboard
```

✅ Now the password is simpler: `admin_21`

---

## 🔧 PART 6: Configure Ubuntu server with Samba

### Step 18: Connect via SSH to Ubuntu server

**From your local Linux:**

```bash
ssh -i ~/.ssh/labsuser.pem ubuntu@54.173.102.89
```

**Change `54.173.102.89` to your Ubuntu Elastic IP.**

---

### Step 19: Update system

```bash
sudo apt update
sudo apt upgrade -y
```

---

### Step 20: Configure hostname

```bash
sudo hostnamectl set-hostname samba-server
```

---

### Step 21: Configure /etc/hosts

```bash
sudo nano /etc/hosts
```

**Content (change 10.0.1.226 to your private IP):**
```
127.0.0.1       localhost
127.0.1.1       samba-server.awslab.lan samba-server

# Private IP of this instance
10.0.1.226      samba-server.awslab.lan samba-server
```

**Save:** Ctrl+O, Enter, Ctrl+X

---

### Step 22: Disable systemd-resolved

⚠️ **IMPORTANT:** AWS uses DHCP and systemd-resolved interferes with Samba's DNS.

```bash
# Disable systemd-resolved
sudo systemctl disable --now systemd-resolved

# Remove symbolic link
sudo unlink /etc/resolv.conf
```

---

### Step 23: Create manual /etc/resolv.conf

```bash
sudo nano /etc/resolv.conf
```

**Content:**
```
nameserver 127.0.0.1
nameserver 8.8.8.8
search awslab.lan
```

**Save and make immutable:**
```bash
sudo chattr +i /etc/resolv.conf
```

---

### Step 24: Install Samba and dependencies

```bash
sudo apt install -y acl attr samba samba-dsdb-modules samba-vfs-modules \
  winbind libpam-winbind libnss-winbind libpam-krb5 krb5-config krb5-user \
  dnsutils ldap-utils
```

**Kerberos configuration:**
```
Default realm: AWSLAB.LAN
Kerberos servers: samba-server.awslab.lan
Administrative server: samba-server.awslab.lan
```

---

### Step 25: Stop default Samba services

```bash
sudo systemctl stop smbd nmbd winbind
sudo systemctl disable smbd nmbd winbind
```

**Back up smb.conf:**
```bash
sudo mv /etc/samba/smb.conf /etc/samba/smb.conf.bak 2>/dev/null || true
```

---

### Step 26: Domain provision

```bash
sudo samba-tool domain provision --use-rfc2307 --interactive
```

**Answers:**
```
Realm: AWSLAB.LAN (press Enter)
Domain: AWSLAB (press Enter)
Server Role: dc (press Enter)
DNS backend: SAMBA_INTERNAL (press Enter)
DNS forwarder: 8.8.8.8
Administrator password: Admin_21
Retype password: Admin_21
```

**Should say:**
```
Provision OK for domain DN DC=awslab,DC=lan
```

---

### Step 27: Copy krb5.conf and start Samba

```bash
# Copy Kerberos configuration
sudo cp /var/lib/samba/private/krb5.conf /etc/krb5.conf

# Start Samba AD DC
sudo systemctl unmask samba-ad-dc
sudo systemctl start samba-ad-dc
sudo systemctl enable samba-ad-dc

# Verify status
sudo systemctl status samba-ad-dc
```

Should be `active (running)`.

---

### Step 28: Verify DNS and Kerberos

```bash
# DNS
host awslab.lan
host samba-server.awslab.lan
host -t SRV _ldap._tcp.awslab.lan

# Kerberos
kinit Administrator
# Password: Admin_21
klist
```

✅ Everything should work correctly.

---

### Step 29: Create shared folders

```bash
# Create folders
sudo mkdir -p /srv/samba/FinanceDocs
sudo mkdir -p /srv/samba/HRDocs
sudo mkdir -p /srv/samba/Public

# Permissions
sudo chmod 777 /srv/samba/FinanceDocs
sudo chmod 777 /srv/samba/HRDocs
sudo chmod 755 /srv/samba/Public
```

---

### Step 30: Configure shared resources in smb.conf

```bash
sudo nano /etc/samba/smb.conf
```

**Add at the end:**
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

**Save and reload:**
```bash
sudo smbcontrol all reload-config
```

---

## 🌐 PART 7: Verify connectivity

### Step 31: From Ubuntu, ping Windows

```bash
# Ping using Windows private IP
ping -c 4 10.0.14.107
```

**Change `10.0.14.107` to your Windows private IP.**

✅ Should respond correctly.

---

### Step 32: From Windows, ping Ubuntu

**Open PowerShell in Windows (RDP):**

```powershell
# Ping using Ubuntu private IP
ping 10.0.1.226
```

**Change `10.0.1.226` to your Ubuntu private IP.**

✅ Should respond.

---

### Step 33: Test ports from Windows

```powershell
# Test SSH (port 22)
Test-NetConnection -ComputerName 10.0.1.226 -Port 22

# Test LDAP (port 389)
Test-NetConnection -ComputerName 10.0.1.226 -Port 389

# Test SMB (port 445)
Test-NetConnection -ComputerName 10.0.1.226 -Port 445
```

✅ All should show `TcpTestSucceeded: True`

---

## 🔐 PART 8: Join Windows to the domain

### Step 34: Configure DNS on Windows

**In Windows (RDP), open PowerShell as Administrator:**

```powershell
# Get network interface name
Get-NetAdapter
```

**Should show something like:**
```
Name                      InterfaceDescription
----                      --------------------
Ethernet                  AWS PV Network Device
```

**Configure DNS:**
```powershell
# Change DNS to the Ubuntu server's private IP
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses ("10.0.1.226","8.8.8.8")

# Verify
Get-DnsClientServerAddress -InterfaceAlias "Ethernet" -AddressFamily IPv4
```

**Change `10.0.1.226` to your Ubuntu private IP.**

---

### Step 35: Verify DNS resolution from Windows

```powershell
# Resolve the domain
nslookup awslab.lan

# Resolve the server
nslookup samba-server.awslab.lan
```

✅ Should resolve correctly.

---

### Step 36: Join Windows to the domain

**Method 1: From PowerShell (faster):**

```powershell
# Join the domain
Add-Computer -DomainName awslab.lan -Credential AWSLAB\Administrator -Restart
```

**Enter password:** `Admin_21`

---

**Method 2: From GUI:**

```
1. Start → Search: "This PC"
2. Right-click → Properties
3. Click on "Rename this PC (advanced)"
4. Click on "Change..."
5. Select "Domain"
6. Type: awslab.lan
7. OK
8. User: Administrator
9. Password: Admin_21
10. OK → Restart
```

**Windows will restart.**

---

### Step 37: Reconnect and verify domain join

**Reconnect via RDP:**

```bash
xfreerdp /v:54.221.100.222 \
         /u:Administrator \
         /p:'admin_21' \
         /cert:ignore \
         /dynamic-resolution \
         /clipboard
```

**Verify in PowerShell:**

```powershell
# View computer information
systeminfo | findstr /B /C:"Domain"
```

**Should show:**
```
Domain: awslab.lan
```

✅ Windows correctly joined to the domain.

---

## 📁 PART 9: Create users and test access

### Step 38: Create users on the Ubuntu server

**Return to the Ubuntu server SSH:**

```bash
# Create users
sudo samba-tool user create alice Admin_21 --given-name="Alice" --surname="Finance"
sudo samba-tool user create bob Admin_21 --given-name="Bob" --surname="HR"

# Create groups
sudo samba-tool group add Finance
sudo samba-tool group add HR

# Add users to groups
sudo samba-tool group addmembers Finance alice
sudo samba-tool group addmembers HR bob

# Verify
sudo samba-tool group listmembers Finance
sudo samba-tool group listmembers HR
```

---

### Step 39: Configure permissions in smb.conf

```bash
sudo nano /etc/samba/smb.conf
```

**Modify the sections:**
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

**Save and reload:**
```bash
sudo smbcontrol all reload-config
```

---

### Step 40: Log in as a domain user on Windows

**In Windows (RDP):**

```
1. Sign out (Start → User icon → Sign out)
2. On the login screen:
   - Click "Other user"
   - User: AWSLAB\alice
   - Password: Admin_21
3. Log in
```

**The first time will take 1-2 minutes (creates profile).**

---

### Step 41: Access shared resources

**Open File Explorer (Windows + E):**

**In the address bar, type:**
```
\\samba-server.awslab.lan
```

**Or using private IP:**
```
\\10.0.1.226
```

**Should show:**
```
FinanceDocs
Public
```

⚠️ **HRDocs does NOT appear** because alice is not in the HR group.

---

### Step 42: Test access

**Open FinanceDocs:**
```
1. Double-click on FinanceDocs
2. MUST open correctly
3. Right-click → New → Text Document
4. Name it: test_alice.txt
5. Open and write something
6. Save
```

✅ alice can access FinanceDocs.

---

**Try to access HRDocs:**
```
From Explorer, type in the address bar:
\\samba-server.awslab.lan\HRDocs
```

❌ **MUST deny access** (alice is not in HR).

---

**Open Public:**
```
\\samba-server.awslab.lan\Public
```

✅ MUST open (Public is for everyone).

---

### Step 43: Map network drive

```
1. File Explorer → This PC
2. Top menu → Computer → Map network drive
3. Drive: Z:
4. Folder: \\samba-server.awslab.lan\FinanceDocs
5. ✅ Reconnect at sign-in
6. Finish
```

**Drive Z: appears in This PC.**

---

## ✅ FINAL CHECKPOINT

### Verifications on Ubuntu server:

```bash
# 1. Samba running
sudo systemctl status samba-ad-dc | grep Active

# 2. Users created
sudo samba-tool user list

# 3. Groups created
sudo samba-tool group list

# 4. Finance members
sudo samba-tool group listmembers Finance

# 5. DNS works
host awslab.lan

# 6. Kerberos works
klist
```

---

### Verifications on Windows:

```powershell
# 1. Joined to the domain
systeminfo | findstr /B /C:"Domain"

# 2. Current user
whoami

# 3. User information
net user alice /domain

# 4. DNS resolution
nslookup samba-server.awslab.lan

# 5. Connectivity
ping 10.0.1.226
Test-NetConnection -ComputerName 10.0.1.226 -Port 445
```

---

### Access verifications:

| User | FinanceDocs | HRDocs | Public |
|------|-------------|--------|--------|
| alice | ✅ Access | ❌ Denied | ✅ Access |
| bob | ❌ Denied | ✅ Access | ✅ Access |
| Administrator | ✅ Access | ✅ Access | ✅ Access |

---

## 🛠️ TROUBLESHOOTING

### Cannot connect via RDP

**Check:**
```bash
# Security group has port 3389 open
# Elastic IP is correct
# Password copied without extra spaces
# Instance is "Running"
```

**Test connection:**
```bash
# Test if the port is open
telnet 54.221.100.222 3389

# If you don't have telnet:
nc -zv 54.221.100.222 3389
```

---

### Windows cannot join the domain

**In Windows, verify DNS:**
```powershell
Get-DnsClientServerAddress -InterfaceAlias "Ethernet" -AddressFamily IPv4
```

Should show the Ubuntu private IP as primary DNS.

**Verify resolution:**
```powershell
nslookup awslab.lan
```

Should resolve to the Ubuntu private IP.

**In Ubuntu, verify Samba:**
```bash
sudo systemctl status samba-ad-dc
host -t SRV _ldap._tcp.awslab.lan
```

---

### Cannot access shared folders

**Check in Windows:**
```powershell
# Ping the server
ping 10.0.1.226

# SMB port open
Test-NetConnection -ComputerName 10.0.1.226 -Port 445

# List resources
net view \\10.0.1.226
```

**Check in Ubuntu:**
```bash
# Shared resources configured
sudo smbclient -L localhost -U Administrator%Admin_21

# Folder permissions
ls -la /srv/samba/

# smb.conf configuration
sudo grep -A 5 "\[FinanceDocs\]" /etc/samba/smb.conf
```

---

### FreeRDP doesn't connect

**Install updated version:**
```bash
sudo apt update
sudo apt install -y freerdp2-x11 freerdp2-shadow-x11
```

**Test with minimum parameters:**
```bash
xfreerdp /v:54.221.100.222 /u:Administrator /p:'admin_21' /cert:ignore
```

**If it still fails, use Remmina:**
```bash
sudo apt install -y remmina remmina-plugin-rdp
remmina
```

---

### IPs changed after restart

**Elastic IPs do NOT change.** If they changed, they are not Elastic IPs.

**Check:**
```
EC2 → Elastic IPs → There should be 2 associated IPs
```

**Private IPs (10.0.X.X) DO persist even when you stop/start the instance.**

---

## 💰 COSTS AND LIMITS

**Available credits:** $50-100

**Approximate consumption:**
```
t3.micro Ubuntu:  ~$0.02/hour
t3.micro Windows: ~$0.02/hour
Elastic IPs:      $0 (while associated with running instances)

Total: ~$0.04/hour = ~$0.96 per 24 hours
```

**Tips:**
```
1. STOP (Stop) instances when not in use
2. Do NOT terminate (Terminate) if you want to keep them
3. Elastic IPs associated with stopped instances DO cost money
4. The lab shuts down automatically (varies by course)
```

---

## 🎯 FINAL SUMMARY

**You have completed:**
- ✅ Security Group configured with all AD ports
- ✅ Ubuntu server with Samba AD DC on AWS
- ✅ Windows Server on AWS
- ✅ Persistent Elastic IPs on both instances
- ✅ RDP connection from Linux with FreeRDP
- ✅ Spanish keyboard and simple password on Windows
- ✅ Bidirectional connectivity verified (ping, ports)
- ✅ Windows joined to the awslab.lan domain
- ✅ Users and groups created
- ✅ Shared folders with group permissions
- ✅ Access verified (alice → FinanceDocs ✅, HRDocs ❌)
- ✅ Network drives mapped

**Final architecture:**
```
Internet
    ↓
┌───────────────────────────────────────┐
│          AWS Lab VPC                  │
│   Security Group: LAB-SG              │
│                                       │
│  Ubuntu Server          Windows       │
│  ┌─────────────────┐   ┌──────────┐   │
│  │ Samba AD DC     │   │ Client   │   │
│  │ awslab.lan      │←──│ RDP      │   │
│  │ 10.0.1.226      │   │ 10.0.14. │   │
│  └─────────────────┘   └──────────┘   │
│    Elastic IP            Elastic IP   │
└───────────────────────────────────────┘
      ↑                      ↑
    SSH (22)             RDP (3389)
   from Linux          from Linux
                      (FreeRDP)
```
