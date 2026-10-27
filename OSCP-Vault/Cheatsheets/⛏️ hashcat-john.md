# Hashcat and John the Ripper - OSCP Cracking Cheatsheet

> **Methodology and Cheatsheet Cross-References**:
> - Password Attacks Methodology: [[09-Password-Attacks|09 - Password Attacks Methodology]]
> - Active Directory Kerberoasting & AS-REP: [[05-Active-Directory|05 - Active Directory Methodology]] | [[⚔️ ad-attacks]]
> - Linux Shadow Cracking: [[03-Linux-PrivEsc|03 - Linux Privilege Escalation]]
> - Windows SAM / LSA Cracking: [[04-Windows-PrivEsc|04 - Windows Privilege Escalation]]

---

## Navigation Triage Matrix

| Operational Goal | Technique / Tool | Quick Link |
|---|---|---|
| Visually recognize or automatically identify hash types | Hash Identification | [[#1. Hash Identification Reference]] |
| Fast lookup via online rainbow tables and OSINT | Pre-Cracking Hash Lookup | [[#2. Pre-Cracking Hash Lookup (OSINT)]] |
| High-performance GPU cracking with Hashcat | Hashcat Dictionary Attacks | [[#3. Hashcat GPU Cracking (Dictionary Attacks)]] |
| 10 essential Hashcat modes for OSCP exams | Common Hashcat Modes | [[#4. Essential OSCP Hashcat Modes]] |
| Apply mutation rules to wordlists | Hashcat Rule Attacks | [[#5. Hashcat Rule-Based Mutations]] |
| Targeted brute-force with password patterns | Hashcat Mask Attacks | [[#6. Hashcat Mask Attacks (Pattern Brute-Force)]] |
| CPU cracking and file hash extraction | John the Ripper Basics | [[#7. John the Ripper (CPU Cracking)]] |
| Convert protected files to crackable hashes | 2john Conversion Tools | [[#8. 2john File Hash Extraction Suite]] |
| Advanced mutations in John the Ripper | John the Ripper Rules | [[#9. John the Ripper Rule-Based Attacks]] |
| Wordlist optimization and policy pruning | Wordlist Filtering | [[#10. Wordlist Filtering for Password Policies]] |
| High-yield copy-paste command block | Quick Reference | [[#11. Quick Reference Cheat Sheet]] |

---

## 1. Hash Identification Reference

### Visual Identification Matrix

| Hash Format / Signature | Length / Encoding | Algorithm | Hashcat Mode | John Format |
|---|---|---|---|---|
| `d41d8cd98f00b204e9800998ecf8427e` | 32 Hex characters | **MD5** | `-m 0` | `raw-md5` |
| `da39a3ee5e6b4b0d3255bfef95601890afd80709` | 40 Hex characters | **SHA-1** | `-m 100` | `raw-sha1` |
| `e3b0c44298fc1c149afbf4c8996fb92427ae41e4...` | 64 Hex characters | **SHA-256** | `-m 1400` | `raw-sha256` |
| `cf83e1357eefb8bdf1542850d66d8007d620e405...` | 128 Hex characters | **SHA-512** | `-m 1700` | `raw-sha512` |
| `31d6cfe0d16ae931b73c59d7e0c089c0` | 32 Hex characters | **NTLM** | `-m 1000` | `NT` |
| `admin:500:aad3b435b51404ee...:31d6cfe0d16a...:::` | Impacket pwdump format | **NTLM (SAM/NTDS)** | `-m 1000` | `NT` |
| `user::DOMAIN:1122334455667788:B1B2...:0101...` | NetNTLMv2 challenge string | **NetNTLMv2** | `-m 5600` | `netntlmv2` |
| Starts with `$1$` | Base64-like | **MD5crypt (Linux)** | `-m 500` | `md5crypt` |
| Starts with `$5$` | Base64-like | **SHA-256crypt (Linux)** | `-m 7400` | `sha256crypt` |
| Starts with `$6$` | Base64-like | **SHA-512crypt (Linux)** | `-m 1800` | `sha512crypt` |
| Starts with `$y$` or `$7$` | Base64-like | **yescrypt (Modern Linux)**| `-m 25600` | `yescrypt` |
| Starts with `$2a$`, `$2b$`, `$2y$` | 60 characters | **bcrypt** (Slow) | `-m 3200` | `bcrypt` |
| Starts with `$apr1$` | Apache MD5 variant | **Apache MD5 (.htpasswd)**| `-m 1600` | `md5apr1` |
| Starts with `$krb5asrep$23$` | Kerberos ticket | **Kerberos AS-REP** | `-m 18200` | `krb5asrep` |
| Starts with `$krb5tgs$23$` | Kerberos ticket | **Kerberos TGS-REP** | `-m 13100` | `krb5tgs` |

### Automated Hash Identification
```bash
# name-that-hash (recommended modern identifier)
nth --text '5d41402abc4b2a76b9719d911017c592'

# hashid (classic identification tool)
hashid -m -j '5d41402abc4b2a76b9719d911017c592'

# Built-in Kali identifier
hash-identifier
```

> 💡 **Official Hash Reference**: Check the complete list of formats on the [Hashcat Example Hashes](https://hashcat.net/wiki/doku.php?id=example_hashes) documentation page.

---

## 2. Pre-Cracking Hash Lookup (OSINT)

> 💡 **Time-Saver Protocol**: Always verify standard unsalted hashes (MD5, SHA1, NTLM) against lookup services before launching local cracking jobs. A 5-second lookup can save 20 minutes of compute time.

| Database / Service | URL | Supported Formats |
|---|---|---|
| **CrackStation** | `https://crackstation.net` | MD5, SHA-1, SHA-256, NTLM, LM |
| **Hashes.com** | `https://hashes.com/en/decrypt/hash` | Multiple algorithms and custom salts |
| **Google Search** | Search raw hash surrounded by quotes | Public CTF hashes, default vendor credentials |

---

## 3. Hashcat GPU Cracking (Dictionary Attacks)

Run Hashcat on your host OS GPU whenever possible to achieve orders of magnitude higher hash rates than inside a virtualized Kali guest.

### Standard Hashcat Invocation
```bash
hashcat -m [MODE] -a 0 [HASH_FILE] [WORDLIST]
```

### Essential Command Flags

| Flag | Purpose | Recommended Setting |
|---|---|---|
| `-m [MODE]` | Defines the cryptographic target hash mode | e.g., `-m 1000` for NTLM |
| `-a 0` | Straight dictionary attack mode | Default wordlist mode |
| `-a 3` | Mask attack mode (patterned brute-force) | Custom character masks |
| `-O` | Enable optimized GPU kernels | Limits password length (typically <= 32) but doubles speed |
| `-w 3` | Workload profile (1=Low, 2=Default, 3=High, 4=Nightmare) | Use `-w 3` for dedicated host cracking |
| `-d 1` | Force specific GPU device | Selects discrete GPU over integrated graphics |
| `--show` | Display previously recovered credentials | Reads from `hashcat.potfile` |
| `--left` | Show only remaining uncracked hashes | Useful for multi-hash files |
| `-r [RULE_FILE]` | Apply mutation rules to wordlist | e.g., `-r /usr/share/hashcat/rules/best64.rule` |

---

## 4. Essential OSCP Hashcat Modes

```bash
# 1. MD5 (Common in web CMS and CTF challenges)
hashcat -m 0 -a 0 -O hashes.txt /usr/share/wordlists/rockyou.txt

# 2. SHA-1 (Legacy web applications and Git hashes)
hashcat -m 100 -a 0 -O hashes.txt /usr/share/wordlists/rockyou.txt

# 3. SHA-256 (Modern applications and database dumps)
hashcat -m 1400 -a 0 -O hashes.txt /usr/share/wordlists/rockyou.txt

# 4. SHA-512 (High-security web hashes)
hashcat -m 1700 -a 0 -O hashes.txt /usr/share/wordlists/rockyou.txt

# 5. NTLM (Windows SAM / NTDS.dit - highest OSCP yield)
hashcat -m 1000 -a 0 -O ntlm_hashes.txt /usr/share/wordlists/rockyou.txt

# 6. NetNTLMv2 (Captured via Responder or Inveigh SMB relays)
hashcat -m 5600 -a 0 netntlmv2.txt /usr/share/wordlists/rockyou.txt

# 7. SHA-512crypt (Linux /etc/shadow standard $6$ format)
hashcat -m 1800 -a 0 shadow_hashes.txt /usr/share/wordlists/rockyou.txt

# 8. MD5crypt (Legacy Linux /etc/shadow $1$ format)
hashcat -m 500 -a 0 md5crypt_hashes.txt /usr/share/wordlists/rockyou.txt

# 9. Kerberos AS-REP (Active Directory AS-REProasting with Impacket GetNPUsers)
hashcat -m 18200 -a 0 asrep_hashes.txt /usr/share/wordlists/rockyou.txt

# 10. Kerberos TGS-REP (Active Directory Kerberoasting with GetUserSPNs)
hashcat -m 13100 -a 0 -O tgs_hashes.txt /usr/share/wordlists/rockyou.txt
```

---

## 5. Hashcat Rule-Based Mutations

Rules take base words from your dictionary and systematically generate variations (capitalization, leet speak, symbol appending, prepending numbers).

### Common Hashcat Rule Syntax

| Function | Action | Transformation (Base: `password`) |
|---|---|---|
| `c` | Capitalize first character | `Password` |
| `u` | Convert all characters to uppercase | `PASSWORD` |
| `l` | Convert all characters to lowercase | `password` |
| `$X` | Append character `X` to the end | `$!` yields `password!` |
| `^X` | Prepend character `X` to the beginning | `^1` yields `1password` |
| `r` | Reverse string order | `drowssap` |
| `d` | Duplicate entire string | `passwordpassword` |
| `t` | Toggle casing across entire string | `PaSsWoRd` |

### Custom Rule Construction
Create a custom rule file `custom.rule`:
```text
# Capitalize and append digits/symbols: password -> Password1!
c $1 $!

# Capitalize and append current year: password -> Password2026!
c $2 $0 $2 $6 $!

# Prepend company code: password -> DevPassword
^v ^e ^D c
```

### Executing Rule Attacks
```bash
# Preview mutated candidates without launching cracking
hashcat -r custom.rule --stdout wordlist.txt | head -n 20

# Run best64 mutation rule set (fast, high success rate)
hashcat -m 1000 -a 0 -O ntlm.txt rockyou.txt -r /usr/share/hashcat/rules/best64.rule

# Run custom mutation rule file
hashcat -m 1000 -a 0 -O ntlm.txt rockyou.txt -r custom.rule
```

---

## 6. Hashcat Mask Attacks (Pattern Brute-Force)

When a target password policy is known (e.g., minimum 8 characters, at least one uppercase letter, ending in 2 digits), a mask attack (`-a 3`) is dramatically faster than pure brute-force.

### Built-in Character Sets

| Mask Identifier | Character Set | Expanded Definition |
|---|---|---|
| `?l` | Lowercase letters | `abcdefghijklmnopqrstuvwxyz` |
| `?u` | Uppercase letters | `ABCDEFGHIJKLMNOPQRSTUVWXYZ` |
| `?d` | Decimal digits | `0123456789` |
| `?s` | Special symbols | `!"#$%&'()*+,-./:;<=>?@[\]^_\`{\|}~` |
| `?a` | All printable characters | `?l?u?d?s` |
| `?b` | Binary octets | `0x00 - 0xff` |

### Practical Mask Profiles
```bash
# 8 characters: 1 uppercase, 5 lowercase, 2 digits (e.g., Password12)
hashcat -m 1000 -a 3 ntlm.txt '?u?l?l?l?l?l?d?d'

# 6-digit numeric PIN
hashcat -m 0 -a 3 hash.txt '?d?d?d?d?d?d'

# Base root + 4 digits (e.g., Spring2026, Admin1234)
hashcat -m 1000 -a 3 -1 '?l?d' ntlm.txt 'Admin?d?d?d?d'

# Corporate standard: Uppercase + lowercase + 2 digits + special symbol (8 chars)
hashcat -m 1000 -a 3 ntlm.txt '?u?l?l?l?l?d?d?s'

# Incrementing length dynamically (e.g., all alphanumeric between 6 and 8 chars)
hashcat -m 1000 -a 3 ntlm.txt '?1?1?1?1?1?1?1?1' -1 '?l?d' --increment --increment-min 6 --increment-max 8
```

---

## 7. John the Ripper (CPU Cracking)

John the Ripper is ideal inside virtual machines for CPU-bound cracking, quick format conversion testing, and handling formats that Hashcat does not natively ingest.

### Basic Invocations
```bash
# Automatic format detection with rockyou
john --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt

# Specify format explicitly
john --format=NT --wordlist=/usr/share/wordlists/rockyou.txt ntlm.txt
john --format=netntlmv2 --wordlist=/usr/share/wordlists/rockyou.txt netntlmv2.txt
john --format=raw-sha256 --wordlist=/usr/share/wordlists/rockyou.txt sha256.txt

# View cracked passwords
john --show hashes.txt
john --show --format=NT ntlm.txt
```

---

## 8. 2john File Hash Extraction Suite

Extract encrypted hashes from password-protected archives, private keys, and documents using the `*2john` suite:

### SSH Private Key Passphrase
```bash
# Extract passphrase hash from SSH private key
ssh2john id_rsa > id_rsa.hash

# Crack with John
john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa.hash
```

### Password-Protected ZIP Archive
```bash
# Extract ZIP encryption hash
zip2john secure.zip > zip.hash

# Crack with John
john --wordlist=/usr/share/wordlists/rockyou.txt zip.hash
```

### Password-Protected RAR Archive
```bash
rar2john backup.rar > rar.hash
john --wordlist=/usr/share/wordlists/rockyou.txt rar.hash
```

### Encrypted PDF Document
```bash
pdf2john document.pdf > pdf.hash
john --wordlist=/usr/share/wordlists/rockyou.txt pdf.hash
```

### Microsoft Office Documents (Word, Excel, PowerPoint)
```bash
office2john finances.xlsx > office.hash
john --wordlist=/usr/share/wordlists/rockyou.txt office.hash
```

### KeePass Password Database
```bash
keepass2john Database.kdbx > keepass.hash
john --wordlist=/usr/share/wordlists/rockyou.txt keepass.hash
```

### Linux Shadow and Passwd Unshadowing
Combine `/etc/passwd` and `/etc/shadow` into a unified crackable format:
```bash
unshadow /etc/passwd /etc/shadow > unshadowed.txt
john --wordlist=/usr/share/wordlists/rockyou.txt --format=sha512crypt unshadowed.txt
```

---

## 9. John the Ripper Rule-Based Attacks

John provides powerful built-in rulesets configured in `/etc/john/john.conf`:

### Built-in Rulesets
```bash
# best64: The top 64 most productive mutation rules
john --wordlist=rockyou.txt --rules=best64 hashes.txt

# Jumbo: Exhaustive comprehensive mutation ruleset
john --wordlist=rockyou.txt --rules=jumbo hashes.txt

# KoreLogic: Massive ruleset designed for cracking competitions
john --wordlist=rockyou.txt --rules=KoreLogic hashes.txt

# List all available rules in configuration
john --list=rules
```

### Custom John Rules in john.conf
Add a custom section to `/etc/john/john.conf`:
```ini
[List.Rules:CustomOSCP]
# Capitalize first letter and append digit plus special character
cAz"[0-9]" Az"[!@#$%]"
# Capitalize and prepend current year (2020-2029)
cA0"[2][0][2][0-9]"
# Replace letters with leet speak equivalents (a->@, e->3, o->0, s->$)
sa@ se3 so0 ss$
```
Run with custom rules:
```bash
john --wordlist=rockyou.txt --rules=CustomOSCP hashes.txt
```

---

## 10. Wordlist Filtering for Password Policies

Target environments often enforce strict password complexity rules. Filtering `rockyou.txt` reduces cracking duration significantly:

```bash
# Remove all passwords shorter than 8 characters
sed -i '/^.\{1,7\}$/d' targeted_wordlist.txt

# Keep only passwords between 8 and 16 characters
grep -E '^.{8,16}$' /usr/share/wordlists/rockyou.txt > filtered_rockyou.txt

# Strip lines starting with numbers
sed -i '/^[0-9]/d' targeted_wordlist.txt

# Deduplicate and sort list in place
sort -u targeted_wordlist.txt -o unique_wordlist.txt
```

---

## 11. Quick Reference Cheat Sheet

```bash
# 1. Automatic Hash Identification
nth --text '[HASH_STRING]'

# 2. Hashcat NTLM Cracking (Windows SAM / Active Directory)
hashcat -m 1000 -a 0 -O hashes.txt /usr/share/wordlists/rockyou.txt

# 3. Hashcat NetNTLMv2 Cracking (Responder captures)
hashcat -m 5600 -a 0 hashes.txt /usr/share/wordlists/rockyou.txt

# 4. Hashcat Kerberoasting (TGS tickets)
hashcat -m 13100 -a 0 -O tgs.txt /usr/share/wordlists/rockyou.txt

# 5. Hashcat AS-REProasting (AS-REP tickets)
hashcat -m 18200 -a 0 asrep.txt /usr/share/wordlists/rockyou.txt

# 6. Hashcat with best64 mutation rules
hashcat -m 1000 -a 0 -O hashes.txt rockyou.txt -r /usr/share/hashcat/rules/best64.rule

# 7. Hashcat Mask Attack (e.g., Password12!)
hashcat -m 1000 -a 3 hashes.txt '?u?l?l?l?l?l?d?d?s'

# 8. John the Ripper standard cracking
john --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt

# 9. Extract and crack SSH private key passphrase
ssh2john id_rsa > id_rsa.hash && john --wordlist=rockyou.txt id_rsa.hash

# 10. Display cracked credentials
hashcat -m 1000 --show hashes.txt
john --show hashes.txt
```
