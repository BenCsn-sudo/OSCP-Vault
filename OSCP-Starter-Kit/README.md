# OSCP Starter Kit - Turnkey Obsidian Exam Vault

> **A pre-configured, modular Obsidian knowledge vault engineered for penetration testing students preparing for the Offensive Security Certified Professional (OSCP / PEN-200) certification.**

---

## Welcome to Your Operational Arsenal

Preparing for the OSCP requires more than solving laboratory machines; it demands an **organized, sub-second accessible knowledge retrieval system**. Under the pressure of a 24-hour examination, you cannot afford to waste time searching for syntax, re-researching privilege escalation vectors, or re-inventing note-taking templates.

This **OSCP Starter Kit** provides you with a fully configured, production-grade Obsidian vault out-of-the-box:
- **Pre-Built Methodologies**: 10 comprehensive Standard Operating Procedures (SOPs) covering the entire attack chain.
- **Battle-Tested Cheatsheets**: 12 high-density command cheat sheets equipped with internal navigation triage matrices.
- **Standardized Machine Logging**: Blank practice directories (`THM/`, `PG-Practice/`, `HTB/`) and a battle-proven machine documentation template (`00-TEMPLATE-MACHINE.md`).
- **Exam Readiness Suite**: 24-hour live scratchpad (`00-TEMPLATE-OSCP.md`) and the official OffSec Markdown report template (`TEMPLATE.md`).
- **Pre-Tuned Workspace**: Configured `.obsidian/` settings for instant graph visualizer connections, sub-second search, and clean Markdown rendering.

---

## Vault Layout & Contents

```text
OSCP-Starter-Kit/
├── Methodology/       --> 10 Standard Operating Procedures (Pre-populated)
├── Cheatsheets/       --> 12 High-yield command sheets with triage tables (Pre-populated)
├── Machines/          --> Practice directories with standardized writeup template
│   ├── 00-TEMPLATE-MACHINE.md  --> Standardized machine writeup template
│   ├── THM/           --> Practice logging directory for TryHackMe targets
│   ├── PG-Practice/   --> Practice logging directory for Proving Grounds Practice
│   └── HTB/           --> Practice logging directory for HackTheBox targets
├── Exam/              --> Live exam documentation and official report templates
│   ├── 00-TEMPLATE-OSCP.md     --> Real-time 24-hour exam machine scratchpad
│   └── TEMPLATE.md    --> Official OffSec Markdown exam report template
├── attachments/       --> Dedicated directory for all screenshots and proof images
└── README.md          --> This onboarding guide
```

---

## 1. Pre-Populated Resources Included in Your Kit

You do **not** need to write your methodologies or cheatsheets from scratch. They are already complete, audited, and certified:

### Methodology Modules (Methodology Directory)
- `01-Enumeration.md` : Port scanning, network service enumeration (SMB, SNMP, NFS, RPC, LDAP, DNS, SMTP).
- `02-Web-Attack.md` : Web vulnerability exploitation (SQLi, LFI/RFI, Command Injection, File Upload Bypasses, SSRF, SSTI, JWT).
- `03-Linux-PrivEsc.md` : Linux privilege escalation triage (Sudo, SUID/SGID, Capabilities, Cron Jobs, Systemd, NFS, Docker, Kernel).
- `04-Windows-PrivEsc.md` : Windows privilege escalation triage (Token Impersonation, Service Paths, DLL Hijacking, AlwaysInstallElevated).
- `05-Active-Directory.md` : Domain enumeration, Kerberoasting, AS-REP Roasting, Pass-the-Hash, DCSync, AD CS, Delegation, Golden Tickets.
- `06-Pivoting-Directory.md` : Network pivoting (Ligolo-ng TUN routing, Chisel SOCKS5 proxies, SSH port forwarding, Socat, Netsh).
- `07-File-Transfer.md` : File transfer protocols across Linux and Windows (HTTP, SMB, Certutil, PowerShell, Base64).
- `08-Shells-Payloads.md` : Reverse shells, bind shells, webshells, PTY shell stabilization, Socat TTY.
- `09-Password-Attacks.md` : Password cracking (Hashcat GPU, John the Ripper, Hydra, Medusa, password spraying).
- `10-Post-Exploitation.md` : Credential dumping (SAM/LSA/NTDS, Linux shadow), host looting, defense evasion.

### Rapid Cheatsheets (Cheatsheets Directory)
- `⚔️ ad-attacks.md` : Active Directory exploitation triage matrix.
- `🟠 BurpSuite.md` : Proxy, Repeater, Intruder, Match and Replace, SOCKS proxy chaining.
- `🟣 gobuster-ffuf.md` : Web directory and parameter fuzzing, vHosts, .git dumps.
- `⛏️ hashcat-john.md` : Hash identification, modes, mutation rules, mask attacks.
- `🔧 impacket.md` : Remote execution, credential dumping, MSSQL client, Kerberos tools.
- `🔀 ligolo-chisel.md` : Pivoting and tunneling reference.
- `🐧 linux-commands.md` : Linux shell stabilization, triage commands, stream processing.
- `🐢 msfvenom.md` : Stageless binaries, webshells, Buffer Overflow shellcode.
- `🔎 nmap.md` : 3-tier scanning strategy, port extraction, NSE scripts.
- `🐢 reverse-shells.md` : High-density reverse shell one-liners and bind shells.
- `💉 sql-injection.md` : SQL injection payloads, UNION extraction, error-based, RCE.
- `🪟 windows-commands.md` : Windows enumeration, non-interactive user switching, AMSI bypass.

---

## 2. Daily Practice Workflow

Follow this procedure for every lab machine you compromise during your preparation:

### Step 1: Initialize Your Target Note
1. Navigate to `Machines/00-TEMPLATE-MACHINE.md`.
2. Duplicate the file into the appropriate practice folder (e.g., `Machines/PG-Practice/`).
3. Rename the file using a clean sequential or target identifier:
   ```text
   Machines/PG-Practice/01-TargetName.md
   ```

### Step 2: Document Real-Time Progress
- Populate the **Metadata Table** (Target Name, Platform, OS, Difficulty).
- Log your initial Nmap scans in **Section 1 (Reconnaissance and Enumeration)**.
- Document the exploitation path and initial user compromise in **Section 2 (Initial Access)**.
- Record internal enumeration and root/SYSTEM elevation in **Section 3 (Privilege Escalation)**.
- Save screenshots directly into `attachments/` and embed them using Obsidian syntax:
  ```markdown
  ![[target-proof.png]]
  ```

### Step 3: Extract One-Liners and Key Takeaways
- Consolidate key commands used in **Section 4 (Useful Commands Summary)**.
- **MANDATORY**: Complete **Section 5 (Key Takeaways and Lessons Learned)**:
  - What was the core vulnerability?
  - What rabbit hole or false assumption caused wasted time?
  - What reusable command or concept can you apply to future machines?

---

## 3. Obsidian Keyboard Shortcuts for Speed

Master these core shortcuts to achieve sub-second information retrieval:

| Shortcut | Function | Practical Exam Application |
|---|---|---|
| `Ctrl/Cmd + O` | **Quick Switcher** | Type `reverse`, `ligolo`, or `ad-attacks` to open cheatsheets instantly |
| `Ctrl/Cmd + Shift + F` | **Global Search** | Search for specific privileges (e.g., `SeImpersonatePrivilege`) across all notes |
| `Ctrl/Cmd + Click` | **Open Link in New Tab** | Open a methodology SOP without losing your active machine note |
| `Ctrl/Cmd + E` | **Toggle View / Edit** | Switch between editing raw Markdown and viewing rendered tables and code blocks |
| `Ctrl/Cmd + F` | **In-Note Search** | Search within the active document for specific command flags |

---

## 4. Exam Day Protocol (24-Hour Execution)

When starting your official OSCP exam:

1. **Launch Scratchpads**:
   - Duplicate `Exam/00-TEMPLATE-OSCP.md` for each assigned exam machine.
   - Record active IP addresses, listening ports, and credential discoveries in real-time.
2. **Apply the 45-Minute Timeboxing Rule**:
   - If you make zero measurable progress on a service within **45 minutes**, stop immediately.
   - Review your notes, check corresponding methodology SOPs, and rotate to another target.
3. **Capture Evidence Immediately Upon Flag Recovery**:
   - Every flag must be captured in an unedited, full-window terminal containing:
     1. IP address (`ip a` or `ipconfig`).
     2. Current user (`whoami`).
     3. Hostname (`hostname`).
     4. Proof flag (`cat local.txt` / `type proof.txt`).
   - Store the image immediately in `attachments/`.
4. **Compile the Final Report**:
   - Use `Exam/TEMPLATE.md` to construct your comprehensive exam penetration test report.

---

## Standards and Best Practices

- **Zero-Leak Policy**: When backing up or publishing your vault, ensure you never commit live exam flags or active personal credentials. Use `[FLAG REDACTED]` and standard placeholders.
- **Documentation Discipline**: Treat your vault as an operational combat kit, not a personal journal. Keep entries concise, commands copyable, and lessons learned honest.
