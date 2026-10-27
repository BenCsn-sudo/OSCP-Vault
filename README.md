# OSCP Vault and Starter Kit

[![Certification](https://img.shields.io/badge/Certification-OSCP-red?style=flat-square&logo=target)](https://www.offsec.com/courses/pen-200/)
[![Obsidian](https://img.shields.io/badge/Notes-Obsidian-7A3EE8?style=flat-square&logo=obsidian)](https://obsidian.md/)
[![Machines](https://img.shields.io/badge/Writeups-53%20Targets-brightgreen?style=flat-square)](./OSCP-Vault/Machines)
[![Methodology](https://img.shields.io/badge/Methodology-10%20Modules-blue?style=flat-square)](./OSCP-Vault/Methodology)
[![Cheatsheets](https://img.shields.io/badge/Cheatsheets-12%20Files-orange?style=flat-square)](./OSCP-Vault/Cheatsheets)

Personal study notes, 53 machine walkthroughs, and a clean Obsidian template for Offensive Security Certified Professional (OSCP / PEN-200) preparation.

---

## Overview

This repository contains the notes, checklists, and machine writeups I developed while preparing for the OSCP exam. Everything is structured natively for [Obsidian](https://obsidian.md/) with internal wikilinks, copy-pasteable commands, and zero conversational fluff.

To make this practical for others, the project is split into two directories:

| Directory | What It Contains | When to Use It |
|---|---|---|
| **[`OSCP-Vault/`](./OSCP-Vault)** | My complete personal notes: 53 machine writeups, 10 methodology modules, 12 cheatsheets, and screenshots | When you want to review full writeups, check attack paths, or reference complete methodologies |
| **[`OSCP-Starter-Kit/`](./OSCP-Starter-Kit)** | A clean, blank template vault with all methodologies and cheatsheets included, plus empty machine folders | When you want a pre-configured Obsidian vault to organize your own lab machines and study notes |

---

## Quick Start (How to Open in Obsidian)

Obsidian treats individual folders as separate vaults. 

1. **Clone the repository**:
   ```bash
   git clone https://github.com/BenCsn-sudo/OSCP-Vault.git
   cd OSCP-Vault
   ```
2. **Open in Obsidian**:
   - Open Obsidian and select **Open folder as vault**.
   - Choose **either** `OSCP-Vault` (to read the full notes) **or** `OSCP-Starter-Kit` (to start your own).

> ⚠️ **Note**: Do not open the top-level repository folder in Obsidian. Opening the root will load both folders into the same workspace and show duplicate search results.

---

## Repository Structure

```text
OSCP-Repo/
├── README.md                  <-- This guide
│
├── OSCP-Vault/                <-- COMPLETE PERSONAL REFERENCE
│   ├── Methodology/           <-- 10 step-by-step methodologies
│   ├── Cheatsheets/           <-- 12 command cheatsheets with triage tables
│   ├── Machines/              <-- 53 solved machine writeups
│   │   ├── THM/               <-- 33 TryHackMe targets
│   │   └── PG-Practice/       <-- 20 Proving Grounds Practice targets
│   ├── Exam/                  <-- Live exam scratchpad and report template
│   ├── attachments/           <-- Screenshots and evidence images
│   └── README.md              <-- Full documentation for the vault
│
└── OSCP-Starter-Kit/          <-- BLANK TEMPLATE FOR STUDENTS
    ├── Methodology/           <-- All 10 methodologies included
    ├── Cheatsheets/           <-- All 12 command cheatsheets included
    ├── Machines/              <-- Blank practice folders (THM, PG-Practice, HTB)
    │   └── 00-TEMPLATE-MACHINE.md  <-- Clean writeup template
    ├── Exam/                  <-- Live exam scratchpad and report template
    ├── attachments/           <-- Empty folder for your screenshots
    └── README.md              <-- Getting started guide for the starter kit
```

---

## What is Included

### 1. Methodology Modules (Methodology Directory)
Detailed step-by-step Standard Operating Procedures covering the attack chain:
- `01-Enumeration.md` : Port scanning, service enumeration (SMB, SNMP, NFS, RPC, LDAP, DNS, SMTP).
- `02-Web-Attack.md` : SQLi, LFI/RFI, Command Injection, File Upload Bypasses, SSRF, SSTI, JWT.
- `03-Linux-PrivEsc.md` : Sudo, SUID/SGID, Capabilities, Cron Jobs, Systemd Timers, NFS, Docker.
- `04-Windows-PrivEsc.md` : Token Impersonation, Service Paths, DLL Hijacking, AlwaysInstallElevated.
- `05-Active-Directory.md` : Domain enumeration, Kerberoasting, AS-REP, Pass-the-Hash, DCSync, AD CS, Delegation.
- `06-Pivoting-Directory.md` : Ligolo-ng TUN tunnels, Chisel SOCKS5 proxies, SSH port forwarding, Socat.
- `07-File-Transfer.md` : Inbound and outbound file staging (HTTP, SMB, Certutil, PowerShell, Base64).
- `08-Shells-Payloads.md` : Reverse shells, bind shells, webshells, PTY stabilization.
- `09-Password-Attacks.md` : Hashcat GPU modes, John the Ripper, Hydra, password spraying.
- `10-Post-Exploitation.md` : Dumping SAM, LSA secrets, NTDS.dit, Linux shadow, host looting.

### 2. Command Cheatsheets (Cheatsheets Directory)
High-density command references with triage navigation tables:
- `⚔️ ad-attacks.md` : Active Directory exploitation triage matrix.
- `🟠 BurpSuite.md` : Proxy, Repeater, Intruder limits, SOCKS proxy chaining.
- `🟣 gobuster-ffuf.md` : Directory fuzzing, vHosts, parameter discovery, .git dumps.
- `⛏️ hashcat-john.md` : Hash identification, common OSCP hash modes, rules, mask attacks.
- `🔧 impacket.md` : Remote execution, secretsdump, mssqlclient, Kerberos ticket tools.
- `🔀 ligolo-chisel.md` : Tunneling, reverse port forward listeners, double pivot.
- `🐧 linux-commands.md` : PTY shell stabilization, quick PrivEsc checks, stream processing.
- `🐢 msfvenom.md` : Stageless binaries, webshells, Buffer Overflow shellcode.
- `🔎 nmap.md` : 3-tier scanning strategy, open port extraction, NSE scripts.
- `🐢 reverse-shells.md` : Reverse shell one-liners (Bash, Python, PowerShell, Netcat, Socat).
- `💉 sql-injection.md` : Auth bypass, UNION extraction, Error-based, Blind, webshell RCE.
- `🪟 windows-commands.md` : Token enumeration, non-interactive user switching, AMSI bypass.

### 3. Machine Writeups (53 Targets)
53 complete writeups with Kill Chain summaries, commands summaries, and lessons learned:
- **TryHackMe (`Machines/THM/`)**: 33 targets covering Linux/Windows fundamentals, web exploits, and Active Directory initial footholds.
- **Proving Grounds Practice (`Machines/PG-Practice/`)**: 20 targets reflecting real OffSec exam difficulty.

---

## Practical Exam Tips

A few practical habits that made a difference during preparation and the exam:

1. **The 45-Minute Timebox**: If you spend 45 minutes on a service or port without finding any new information, stop. Note what you tried, step back, and switch to another port or machine. Rabbit holes are the primary reason people run out of time.
2. **Fast Navigation**: In Obsidian, press `Ctrl/Cmd + O` and type a keyword (like `reverse`, `ligolo`, or `impacket`) to open the exact cheatsheet in less than two seconds.
3. **Capture Proof Right Away**: As soon as you get a user or root shell, take an unedited, full-window screenshot showing `whoami`, `hostname`, `ip a` (or `ipconfig`), and `cat proof.txt`. It takes 15 seconds and saves you from losing points.

---

## Quality Standard

- **No Leaks**: All flags are redacted to `[FLAG REDACTED]` and private target IPs are replaced with standard placeholders (`[TARGET_IP]`, `[KALI_IP]`).
- **Clean Markdown**: Free of em-dash characters and free of slashes in headings, ensuring all Obsidian internal wikilinks work out of the box.
- **Technical English**: Written in concise, direct technical English.

---

## Disclaimer

This repository is for educational purposes and authorized penetration testing only. Do not test against any systems or networks without explicit, prior written permission.
