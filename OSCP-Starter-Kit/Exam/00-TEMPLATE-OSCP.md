# Target Machine Exam Scratchpad - [TARGET_NAME]

> **Operational Golden Rule**: Launch TCP all-ports sweep and UDP top-ports scans immediately upon target assignment.

---

## 1. Reconnaissance and Enumeration

### 1.1 Full TCP Port Scan
```bash
sudo nmap -p- --min-rate 1000 -T4 -Pn [TARGET_IP] -oA nmap/tcp_all
```

### 1.2 Targeted Service and Script Scan
```bash
sudo nmap -sC -sV -p [OPEN_PORTS] -Pn [TARGET_IP] -oA nmap/tcp_detailed
```

### 1.3 Top UDP Port Scan
```bash
sudo nmap -sU --top-ports 20 -Pn [TARGET_IP] -oA nmap/udp_top20
```

### 1.4 Web Application Enumeration
```bash
# Directory and file fuzzing (background process)
feroxbuster -u http://[TARGET_IP]:[PORT] -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -C 404,403,400

# Virtual host discovery (if domain name identified)
ffuf -u http://[TARGET_IP] -H "Host: FUZZ.[DOMAIN]" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fs [DEFAULT_SIZE]
```

### 1.5 Network Services Probing
```bash
# SMB enumeration
nxc smb [TARGET_IP] -u '' -p '' --shares

# SNMP enumeration (UDP 161)
snmpwalk -v2c -c public [TARGET_IP] . > snmp_full.txt
```

---

## 2. Initial Access (Foothold)

### 2.1 Vulnerability Identification
*Document identified vulnerability, affected software version, and exploitation hypothesis.*

### 2.2 Exploitation Execution
*Exact commands, scripts, or payloads used to attain remote code execution.*

- **Local User Proof Flag (`local.txt`)**: `[FLAG REDACTED]`
- **Command Output Verification (`whoami`, `hostname`, `ip a` / `ipconfig`)**:
  ```text
  [Insert terminal screenshot or terminal log]
  ```

---

## 3. Privilege Escalation

### 3.1 Local Enumeration Findings
*Identified privilege escalation vectors (e.g., Sudo NOPASSWD, SUID binaries, SeImpersonatePrivilege, unquoted service paths, stored credentials).*

### 3.2 Privilege Escalation Execution
*Exact commands used to elevate to root (Linux) or NT AUTHORITY\SYSTEM (Windows).*

- **Root / Administrative Proof Flag (`proof.txt`)**: `[FLAG REDACTED]`
- **Root Command Output Verification (`whoami`, `hostname`, `ip a` / `ipconfig`, `type proof.txt` / `cat proof.txt`)**:
  ```text
  [Insert terminal screenshot or terminal log]
  ```

---

## 4. Post-Exploitation and Pivoting Checklist

> ⚠️ **Mandatory Checklist Before Leaving Host**:
> - [ ] **Credential Dumping**: SAM/SYSTEM/SECURITY or NTDS.dit dumped (Windows) OR `/etc/shadow` dumped (Linux).
> - [ ] **Command History**: `ConsoleHost_history.txt` OR `~/.bash_history` inspected.
> - [ ] **SSH Keys**: Private keys (`id_rsa`) searched and preserved.
> - [ ] **Network Interfaces**: Inspected for secondary/internal subnets (`ip a` or `ipconfig /all`).
> - [ ] **Routing and ARP Cache**: `arp -a` and `netstat -ano` reviewed for pivot discovery.
> - [ ] **File Looting**: User desktops, Downloads, and backup directories inspected.
> - [ ] **Screenshots Verified**: Full-window screenshot containing IP, whoami, hostname, and proof flag captured.

---

## 5. Loot and Recovered Credentials

| Account / Username | Plaintext / Hash | Service / Host | Privilege Level | Reusability Validated? |
|---|---|---|---|---|
| | | | | |
