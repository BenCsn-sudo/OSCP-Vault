# OSCP Knowledge Vault - Operational Penetration Testing Framework

> **A battle-tested, structured Obsidian knowledge vault engineered for the Offensive Security Certified Professional (OSCP / PEN-200) certification and practical network penetration testing.**

---

## Executive Overview

This repository represents an end-to-end, production-grade penetration testing knowledge vault designed for sub-second information retrieval during time-critical 24-hour examinations and real-world infrastructure assessments.

Every module, command snippet, and writeup in this repository adheres to a strict standard:
- **Zero-Friction Retrieval**: Equipped with native Obsidian wikilink matrices enabling direct access to relevant sections without manual scrolling.
- **Copy-Paste Verifiability**: All command one-liners feature explicit syntax with standardized placeholders (`[TARGET_IP]`, `[KALI_IP]`, `[LPORT]`, `[DOMAIN]`).
- **100% Technical English**: Fully written in standardized technical English with zero decorative noise or conversational filler.
- **Strict Quality Compliance**: Certified free of em-dashes, free of broken wikilinks, and sanitized against private credential and live flag leaks (`[FLAG REDACTED]`).

---

## Vault Architecture

The vault is structured into four primary pillars, accompanied by centralized visual attachments:

```text
OSCP-Vault/
├── Methodology/       --> 10 Standard Operating Procedures (SOP) across the attack chain
├── Cheatsheets/       --> 12 High-density command references with triage navigation tables
├── Machines/          --> 53 Certified target writeups (TryHackMe & Proving Grounds Practice)
│   ├── THM/           --> 33 Linux and Windows TryHackMe room writeups
│   └── PG-Practice/   --> 20 Proving Grounds Practice exam-like target writeups
├── Exam/              --> Operational templates for 24-hour exam execution and reporting
└── attachments/       --> Centralized screenshots, diagrams, and execution proof archives
```

---

## 1. Methodology Directory (Standard Operating Procedures)

The `Methodology/` folder provides comprehensive, deep-dive methodologies organized sequentially across the penetration testing lifecycle:

| Module | File | Core Focus & Techniques |
|:---:|---|---|
| **01** | [[Methodology/01-Enumeration|01 - Enumeration]] | TCP/UDP port scanning, service identification, SMB, SNMP, NFS, DNS, LDAP, RPC, SMTP |
| **02** | [[Methodology/02-Web-Attack|02 - Web Attack]] | SQLi, LFI/RFI, Command Injection, File Upload Bypasses, SSRF, SSTI, XSS, XXE, JWT, API |
| **03** | [[Methodology/03-Linux-PrivEsc|03 - Linux PrivEsc]] | Sudo NOPASSWD, SUID/SGID, Capabilities, Cron Jobs, Systemd Timers, NFS, Docker, Kernel Exploits |
| **04** | [[Methodology/04-Windows-PrivEsc|04 - Windows PrivEsc]] | Token Impersonation, Unquoted Service Paths, DLL Hijacking, AlwaysInstallElevated, Registry |
| **05** | [[Methodology/05-Active-Directory|05 - Active Directory]] | Domain Recon, Kerberoasting, AS-REP Roasting, Pass-the-Hash, DCSync, AD CS, Delegation, Golden Tickets |
| **06** | [[Methodology/06-Pivoting-Directory|06 - Pivoting]] | Ligolo-ng TUN tunnels, Chisel SOCKS5 proxies, SSH port forwarding (-L/-R/-D), Socat, Netsh |
| **07** | [[Methodology/07-File-Transfer|07 - File Transfer]] | Python HTTP, SMB shares, Certutil, PowerShell IWR/WebClient, bitsadmin, Base64, FTP |
| **08** | [[Methodology/08-Shells-Payloads|08 - Shells & Payloads]] | Reverse shells, bind shells, webshells (PHP/ASP/ASPX/JSP), PTY stabilization, Socat TTY |
| **09** | [[Methodology/09-Password-Attacks|09 - Password Attacks]] | Hash identification, Hashcat GPU cracking, John the Ripper, Hydra, Medusa, Password Spraying |
| **10** | [[Methodology/10-Post-Exploitation|10 - Post Exploitation]] | SAM/SYSTEM dumping, LSA secrets, Linux shadow extraction, credential looting, defense evasion |

---

## 2. Technical Cheatsheets (Rapid Command Reference)

The `Cheatsheets/` folder contains rapid-reference command sheets equipped with dedicated **Navigation Triage Matrices** for immediate access:

| Cheatsheet | Target Domain / Tool | Key Capabilities |
|---|---|---|
| [[Cheatsheets/⚔️ ad-attacks|ad-attacks]] | Active Directory Exploitation | 28 attack techniques: BloodHound, Roasting, DCSync, Coercion, AD CS, Delegation |
| [[Cheatsheets/🟠 BurpSuite|BurpSuite]] | Web Proxy & Manual Testing | Scope setup, Repeater, Intruder limits, Match & Replace, SOCKS proxy chaining, OOB testing |
| [[Cheatsheets/🟣 gobuster-ffuf|gobuster-ffuf]] | Web Directory & Parameter Fuzzing | Recursive discovery, response size filtering (-fs), vHost fuzzing, LFI search, .git dumps |
| [[Cheatsheets/⛏️ hashcat-john|hashcat-john]] | Hash Identification & Cracking | Visual recognition, 10 OSCP hash modes (NTLM, NetNTLMv2, Kerberos), rules, mask attacks |
| [[Cheatsheets/🔧 impacket|impacket]] | Windows Network Protocol Suite | Remote code execution (psexec/wmiexec/smbexec), secretsdump, mssqlclient, Kerberos tickets |
| [[Cheatsheets/🔀 ligolo-chisel|ligolo-chisel]] | Pivoting and Port Forwarding | Ligolo-ng TUN routing, reverse listeners, double pivot, SSH tunnels, Chisel fallback |
| [[Cheatsheets/🐧 linux-commands|linux-commands]] | Linux Operations & PrivEsc | TTY stabilization, fast PrivEsc triage (sudo/SUID/caps/cron), pspy64, stream processing |
| [[Cheatsheets/🐢 msfvenom|msfvenom]] | Standalone Payload Generation | Stageless executables (EXE/ELF), webshells (PHP/JSP/WAR/ASPX), Buffer Overflow shellcode |
| [[Cheatsheets/🔎 nmap|nmap]] | Network Reconnaissance | 3-tier scanning protocol (-p-, -sC -sV, -sU), port extraction, pivoted scanning, NSE scripts |
| [[Cheatsheets/🐢 reverse-shells|reverse-shells]] | Shell Delivery and Listeners | One-liners (Bash, Python, PowerShell, Netcat, Socat, AWK), webshells, encrypted listeners |
| [[Cheatsheets/💉 sql-injection|sql-injection]] | SQL Injection Exploitation | Auth bypass, UNION extraction, Error-based, Blind boolean/time, file operations, RCE |
| [[Cheatsheets/🪟 windows-commands|windows-commands]] | Windows Operations & PrivEsc | Token enumeration, non-interactive user switching (PSCredential), Defender/AMSI bypass |

---

## 3. Certified Target Writeups (Machines)

The vault contains **53 complete, certified machine writeups** providing concrete, real-world attack chains:

### Platform Breakdown
- **TryHackMe (`Machines/THM/`)**: 33 Machines (01-RootMe to 38-TomGhost)
  - Covers fundamental CTF patterns, Linux privilege escalation vectors, Active Directory initial footholds, and web vulnerabilities.
- **Proving Grounds Practice (`Machines/PG-Practice/`)**: 20 Machines (04-Potato to 53-Zino)
  - Hardened, exam-like standalone targets reflecting actual OffSec certification environments.

### Standardized Writeup Anatomy
Every single machine writeup strictly implements this six-section structure:
1. **Kill Chain Summary**: High-level attack path summary (e.g., Port 80 LFI -> Log Poisoning -> User Foothold -> Sudo NOPASSWD -> Root).
2. **Metadata Table**: Target Name, Platform, OS, Difficulty, Status, and Key Exploitation Techniques.
3. **Section 1: Reconnaissance and Enumeration**: Port scans, service versions, discovered web endpoints, and initial observations.
4. **Section 2: Initial Access**: Detailed vulnerability discovery, exact commands, and local user compromise.
5. **Section 3: Privilege Escalation**: Internal enumeration, privilege escalation vector execution, and root/SYSTEM compromise.
6. **Section 4: Useful Commands Summary**: Copy-pasteable list of every key command executed during exploitation.
7. **Section 5: Key Takeaways and Lessons Learned**: Critical post-mortem analysis (what worked, what failed, time sinks to avoid).

---

## 4. The 24-Hour Exam Operational Framework

### Navigation and Sub-Second Retrieval in Obsidian
- **Quick Switcher (`Ctrl/Cmd + O`)**: Type tool or technique names (e.g., `reverse`, `ligolo`, `ad-attacks`) to jump immediately to the target file.
- **Global Search (`Ctrl/Cmd + Shift + F`)**: Search specific commands, error messages, or privilege names (e.g., `SeImpersonatePrivilege`, `no_root_squash`).
- **Graph View and Internal Links**: Navigate fluidly between machine writeups and methodology SOPs via native wikilinks.

### The 45-Minute Timeboxing Rule
- If no measurable progress or new technical evidence is uncovered on a service within **45 minutes**:
  1. Halt active testing immediately.
  2. Document all attempted vectors and negative results in your scratchpad.
  3. Step back, review the open port list, and switch to an alternate service or target machine.
  4. Consult the corresponding methodology SOP to identify missed enumeration angles.

### Parallel Execution Protocol
Always structure your workflow to prevent idle waiting:
1. **Minute 0-2**: Launch fast TCP all-ports sweep (`nmap -p- --min-rate 1000 -T4 -Pn [TARGET_IP]`) and top 20 UDP scan.
2. **Minute 2-5**: Launch deep script/version scan (`-sC -sV`) on discovered open ports.
3. **Minute 5-10**: Start background directory fuzzing (`feroxbuster` or `ffuf`) while simultaneously performing manual web reconnaissance.
4. **Minute 10+**: Investigate secondary non-web services (SMB shares, SNMP strings, RPC endpoints) while web fuzzing completes.

### Evidence and Proof Preservation Standard
To satisfy OffSec exam reporting requirements without ambiguity:
- Every proof screenshot must capture an **unedited, full-window terminal** displaying:
  1. The target IP address (`ip a` or `ipconfig`).
  2. The active username (`whoami`).
  3. The machine hostname (`hostname`).
  4. The contents of the proof flag (`cat local.txt` / `type proof.txt`).
- Use the operational template in `Exam/00-TEMPLATE-OSCP.md` to track flags, credentials, and network paths live during the exam.

---

## 5. Getting Started with This Vault

### Opening as an Obsidian Vault
1. Clone this repository to your local system:
   ```bash
   git clone https://github.com/BenCsn-sudo/OSCP-Vault.git
   ```
2. Open Obsidian, click **Open folder as vault**, and select the `OSCP-Vault/` directory.
3. All internal links, triage matrices, and visual attachments will resolve natively out-of-the-box.

### Terminal Usage (CLI-First)
The vault is constructed using clean standard GitHub-Flavored Markdown and can be navigated directly via CLI tools:
```bash
# Search for specific privilege escalation techniques
grep -rn "AlwaysInstallElevated" Methodology/

# Search for specific one-liners across cheatsheets
grep -rn "impacket-secretsdump" Cheatsheets/
```

---

## Disclaimer and Ethics

This repository is provided strictly for educational purposes, professional skill development, and authorized penetration testing engagements. All techniques, methodologies, and writeups must only be executed against systems where explicit, written authorization has been granted. Unauthorized testing against target networks without prior consent is illegal.
