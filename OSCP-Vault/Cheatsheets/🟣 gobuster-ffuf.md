# Gobuster and ffuf - OSCP Web Fuzzing Cheatsheet

> **Methodology and Cheatsheet Cross-References**:
> - Web Attack Methodology: [[02-Web-Attack|02 - Web Attack Methodology]]
> - Burp Suite Proxy and Repeater: [[🟠 BurpSuite]]
> - SQL Injection Cheatsheet: [[💉 sql-injection]]

---

## Navigation Triage Matrix

| Operational Goal | Tool / Technique | Quick Link |
|---|---|---|
| Fuzz directories and files with extensions | Gobuster Directory Fuzzing | [[#1. Directory and File Fuzzing]] |
| High-performance directory and recursive fuzzing | ffuf Directory and File Fuzzing | [[#2. ffuf Directory and File Fuzzing]] |
| Discover virtual hosts and hidden subdomains | vHost Fuzzing | [[#3. Virtual Host (vHost) Fuzzing]] |
| Uncover hidden GET, POST, or JSON parameters | Parameter Fuzzing | [[#4. Parameter Fuzzing (GET and POST)]] |
| Fuzz for LFI and path traversal vulnerabilities | LFI and Path Traversal Fuzzing | [[#5. Fuzzing LFI and Path Traversal]] |
| Fuzz endpoints behind cookies, tokens, or basic auth | Authenticated Fuzzing | [[#6. Authenticated Fuzzing]] |
| Find the right wordlist on Kali for any scenario | Wordlist Master Reference | [[#7. Wordlist Master Reference]] |
| High-speed multi-threaded recursive discovery | Feroxbuster Scanning | [[#8. Feroxbuster (Fast Recursive Scanning)]] |
| Conceptual difference: vHost vs DNS subdomain | Architecture Comparison | [[#9. vHost Fuzzing vs DNS Subdomain Enumeration]] |
| Dump and extract source code from exposed `.git` | Git Repository Extraction | [[#10. Exposed Git Repository Exploitation]] |
| Copy-paste high-yield command cheat block | Quick Reference | [[#11. Quick Reference Cheat Sheet]] |

---

## 1. Directory and File Fuzzing

### Gobuster Directory Fuzzing

> ⚠️ **Critical Wordlist Rule**:
> **Never rely solely on `/usr/share/wordlists/dirb/common.txt`.** It contains only ~4,600 terms and misses modern frameworks, configuration dotfiles (`.git`, `.env`, `.htpasswd`), and administrative endpoints. Always use `raft-medium-directories.txt` or `directory-list-2.3-medium.txt` combined with the `-x` extension flag.

#### Standard OSCP Baseline Scan
Run this scan systematically on every newly discovered HTTP/HTTPS port:
```bash
gobuster dir -u http://[TARGET_IP]   -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt   -x php,txt,html,bak,zip,git,env   -t 50   -b 404,403
```

#### Fast Initial Reconnaissance Scan
Quick preliminary probe before launching heavier wordlists:
```bash
gobuster dir -u http://[TARGET_IP]   -w /usr/share/wordlists/dirb/common.txt   -x php,txt,html,git,env   -t 30
```

#### Exhaustive Secondary Scan
When standard wordlists reveal no attack surface:
```bash
gobuster dir -u http://[TARGET_IP]   -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt   -x php,txt,html,asp,aspx,bak,git,env,config   -t 50
```

#### HTTP Basic Authentication
```bash
gobuster dir -u http://[TARGET_IP]   -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt   -x php,txt   -U [USERNAME] -P [PASSWORD]
```

#### HTTPS with Self-Signed Certificates
```bash
gobuster dir -u https://[TARGET_IP]   -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt   -x php,txt   -k
```

#### Session Cookie Authentication
```bash
gobuster dir -u http://[TARGET_IP]   -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt   -c "PHPSESSID=[SESSION_ID]"   -x php,txt
```

---

## 2. ffuf Directory and File Fuzzing

`ffuf` is significantly faster than Gobuster and offers flexible response filtering (`-fs`, `-fc`, `-fw`, `-fl`).

### Basic ffuf Directory Scan
```bash
ffuf -u http://[TARGET_IP]/FUZZ   -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
```

### Scan with Extensions
```bash
ffuf -u http://[TARGET_IP]/FUZZ   -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt   -e .php,.txt,.html,.bak,.zip
```

### Response Calibration and Filtering
- Filter by HTTP Status Code:
  ```bash
  ffuf -u http://[TARGET_IP]/FUZZ -w wordlist.txt -fc 404,403
  ```
- Filter by Response Size (essential for filtering soft-404 pages):
  ```bash
  ffuf -u http://[TARGET_IP]/FUZZ -w wordlist.txt -fs [RESPONSE_SIZE]
  ```
- Filter by Word Count:
  ```bash
  ffuf -u http://[TARGET_IP]/FUZZ -w wordlist.txt -fw 42
  ```
- Filter by Line Count:
  ```bash
  ffuf -u http://[TARGET_IP]/FUZZ -w wordlist.txt -fl 15
  ```

### Recursive Directory Discovery
```bash
ffuf -u http://[TARGET_IP]/FUZZ   -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt   -recursion   -recursion-depth 2   -e .php,.txt
```

---

## 3. Virtual Host (vHost) Fuzzing

Virtual host fuzzing locates separate websites or admin consoles hosted on the same IP address routed via the `Host:` header.

### ffuf vHost Discovery (Recommended)
```bash
ffuf -u http://[TARGET_IP]   -H "Host: FUZZ.[DOMAIN]"   -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt   -fs [DEFAULT_PAGE_SIZE]
```

> ⚠️ **Calibration Step**:
> 1. Run the command initially without `-fs` for 5 seconds to observe the default response size (e.g., `Size: 1540`).
> 2. Press `Ctrl + C` and append `-fs 1540` to suppress all false-positive wildcard responses.

### Gobuster vHost Discovery
```bash
gobuster vhost -u http://[TARGET_IP]   -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt   --append-domain   --domain [DOMAIN]
```

> 💡 **Action Item**: When a valid vHost is discovered (e.g., `admin.[DOMAIN]`), immediately add it to `/etc/hosts`:
> ```bash
> echo "[TARGET_IP] [DOMAIN] admin.[DOMAIN]" | sudo tee -a /etc/hosts
> ```

---

## 4. Parameter Fuzzing (GET and POST)

Identify hidden or undocumented parameters that may be vulnerable to SQLi, LFI, SSRF, or command injection.

### GET Parameter Discovery
```bash
ffuf -u "http://[TARGET_IP]/index.php?FUZZ=test"   -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt   -fs [BASELINE_SIZE]
```

### POST Parameter Discovery (Form Encoded)
```bash
ffuf -u http://[TARGET_IP]/submit.php   -X POST   -d "FUZZ=test"   -H "Content-Type: application/x-www-form-urlencoded"   -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt   -fs [BASELINE_SIZE]
```

### JSON Body Parameter Fuzzing
```bash
ffuf -u http://[TARGET_IP]/api/v1/user   -X POST   -d '{"FUZZ":"test"}'   -H "Content-Type: application/json"   -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt   -fs [BASELINE_SIZE]
```

---

## 5. Fuzzing LFI and Path Traversal

Discover injectable parameter values capable of traversing the local filesystem:

### Basic LFI Fuzzing with Regex Match
Look for the standard root entry string `root:x:` inside Linux `/etc/passwd`:
```bash
ffuf -u "http://[TARGET_IP]/view.php?page=FUZZ"   -w /usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt   -mr "root:x:"
```

### Windows Path Traversal Fuzzing
Search for standard Windows INI header tags:
```bash
ffuf -u "http://[TARGET_IP]/view.php?file=FUZZ"   -w /usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt   -mr "\[extensions\]"
```

### Filter Evasion Traversal Wordlist
```bash
ffuf -u "http://[TARGET_IP]/download?file=FUZZ"   -w /usr/share/seclists/Fuzzing/LFI/LFI-graceful-filter-bypass.txt   -fs [BASELINE_SIZE]
```

---

## 6. Authenticated Fuzzing

### Session Cookie Fuzzing
```bash
ffuf -u http://[TARGET_IP]/admin/FUZZ   -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt   -b "PHPSESSID=[SESSION_ID]; admin_token=[TOKEN]"   -fc 404,403
```

### Authorization Bearer Token
```bash
ffuf -u http://[TARGET_IP]/api/v1/FUZZ   -w /usr/share/seclists/Discovery/Web-Content/api/objects.txt   -H "Authorization: Bearer [JWT_TOKEN]"
```

### HTTP Basic Auth with ffuf
```bash
ffuf -u http://[TARGET_IP]/protected/FUZZ   -w /usr/share/seclists/Discovery/Web-Content/raft-medium-files.txt   -H "Authorization: Basic [BASE64_CREDENTIALS]"
```

---

## 7. Wordlist Master Reference

### Primary SecLists Paths on Kali

| Fuzzing Objective | Wordlist File Path | Typical Entries |
|---|---|---|
| **Default Directories** | `/usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt` | ~30,000 |
| **Default Files** | `/usr/share/seclists/Discovery/Web-Content/raft-medium-files.txt` | ~17,000 |
| **Standard Comprehensive** | `/usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt` | ~220,000 |
| **Exhaustive Deep Scan** | `/usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt` | ~1,300,000 |
| **Quick Probe** | `/usr/share/wordlists/dirb/common.txt` | ~4,600 |
| **Virtual Hosts / Subdomains** | `/usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt` | ~5,000 |
| **Heavy Subdomain List** | `/usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt` | ~20,000 |
| **HTTP Parameters** | `/usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt` | ~6,000 |
| **API Endpoints** | `/usr/share/seclists/Discovery/Web-Content/api/objects.txt` | ~3,000 |
| **LFI / Path Traversal** | `/usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt` | ~900 |

### Mandatory File Extensions Matrix
Always provide relevant extensions via `-x` (Gobuster) or `-e` (ffuf):

- **Linux Targets**:
  `-x php,txt,html,bak,zip,git,env,conf,sh,py,json`
- **Windows Targets**:
  `-x asp,aspx,txt,html,bak,zip,git,env,config,xml,cs`
- **Critical Dotfiles to Always Check**:
  `.git`, `.env`, `.htpasswd`, `.DS_Store`, `.svn`, `.hg`, `.bash_history`

---

## 8. Feroxbuster (Fast Recursive Scanning)

`feroxbuster` is a multi-threaded, recursive content locator written in Rust. It automatically traverses directories as it finds them without requiring custom scripts.

### Baseline Recursive Scan
```bash
feroxbuster -u http://[TARGET_IP]   -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt   -C 404,403
```

### Scan with Extensions and Depth Limit
```bash
feroxbuster -u http://[TARGET_IP]   -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt   -x php,txt,html,bak   -d 2   -t 40
```

### Filtering False Positives
```bash
# Filter specific status codes
feroxbuster -u http://[TARGET_IP] -w wordlist.txt -C 404,403,500

# Filter by response size
feroxbuster -u http://[TARGET_IP] -w wordlist.txt -S [SIZE_IN_BYTES]

# Filter by word count
feroxbuster -u http://[TARGET_IP] -w wordlist.txt -W 42
```

### Tunneling Through Burp Proxy
Route Feroxbuster traffic through Burp Suite to populate the Target Site Map:
```bash
feroxbuster -u http://[TARGET_IP]   -w /usr/share/wordlists/dirb/common.txt   -k   --proxy http://127.0.0.1:8080
```

---

## 9. vHost Fuzzing vs DNS Subdomain Enumeration

Understanding the technical difference is critical to avoid wasted exam time:

| Feature | Virtual Host (vHost) Fuzzing | Subdomain Enumeration |
|---|---|---|
| **Mechanism** | Changes the HTTP `Host:` header to single IP address | Performs DNS queries (A, CNAME) against nameservers |
| **Server Target** | Target web server (Apache, Nginx, IIS) virtual routing | Target DNS server / external public DNS zones |
| **IP Outcome** | Points to the **exact same** target IP | May resolve to **different** external/internal IPs |
| **Primary Tools** | `ffuf -H "Host: FUZZ.[DOMAIN]"`, `gobuster vhost` | `dig`, `dnsenum`, `subfinder`, `nslookup` |
| **OSCP Exam Relevance** | ✅ High relevance (multi-tenant web services on single VM) | ⚠️ Low relevance (internal lab networks lack real public DNS) |

---

## 10. Exposed Git Repository Exploitation

Deploying web code directly from Git without removing the `.git` folder exposes entire commit histories, hardcoded passwords, API secrets, and deleted configuration files.

### 1. Verification of Exposed Repository
Check whether `.git` endpoints are accessible:
```bash
curl -s -i http://[TARGET_IP]/.git/HEAD
# Valid output contains: "ref: refs/heads/main" or "ref: refs/heads/master"

curl -s -i http://[TARGET_IP]/.git/config
```

### 2. Dumping the Repository
Use `git-dumper` to reconstruct the source tree locally:
```bash
# Install git-dumper if missing
pip3 install git-dumper

# Reconstruct entire repository
git-dumper http://[TARGET_IP]/.git ./git-dump
cd ./git-dump
```

### 3. Extracting Sensitive Data and Credentials
```bash
# View recent commit history
git log --oneline

# Search full commit diff history for credentials
git log -p | grep -i -E "password|passwd|pwd|secret|key|token|api_key|db_|mysql|credentials"

# Inspect diffs of recent commits
git diff HEAD~1
git diff HEAD~5

# Search all commits modifying a specific term
git log -p --all -S 'password'

# List all files ever tracked in repository history (including deleted ones)
git log --all --full-history -- "*.php"
git log --all --full-history -- "*.env"
git log --all --full-history -- "*.conf"

# Restore a deleted sensitive file from a specific commit
git show [COMMIT_HASH]:config.php
```

### 4. High-Value Configuration Files to Inspect
```bash
git show HEAD:config.php 2>/dev/null
git show HEAD:wp-config.php 2>/dev/null
git show HEAD:.env 2>/dev/null
git show HEAD:settings.py 2>/dev/null
git show HEAD:web.config 2>/dev/null
git show HEAD:appsettings.json 2>/dev/null
git show HEAD:.htpasswd 2>/dev/null
git show HEAD:.ssh/id_rsa 2>/dev/null
```

---

## 11. Quick Reference Cheat Sheet

```bash
# 1. Standard Gobuster baseline scan
gobuster dir -u http://[TARGET_IP] -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -x php,txt,html,bak,git,env -t 50 -b 404,403

# 2. Exhaustive Gobuster scan
gobuster dir -u http://[TARGET_IP] -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php,txt,asp,aspx,bak,git,env -t 50

# 3. Recursive Feroxbuster scan
feroxbuster -u http://[TARGET_IP] -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -x php,txt,html,git,env -d 3

# 4. vHost fuzzing with ffuf
ffuf -u http://[TARGET_IP] -H "Host: FUZZ.[DOMAIN]" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fs [RESPONSE_SIZE]

# 5. Parameter fuzzing with ffuf
ffuf -u "http://[TARGET_IP]/index.php?FUZZ=test" -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt -fs [RESPONSE_SIZE]

# 6. Exposed .git dump and credential search
git-dumper http://[TARGET_IP]/.git ./git-dump && cd git-dump && git log -p | grep -i -E "password|secret|key"
```
