# Nmap - OSCP Network Scanning Cheatsheet

> **Methodology and Cheatsheet Cross-References**:
> - Reconnaissance and Enumeration: [[01-Enumeration|01 - Enumeration Methodology]]
> - Pivoting and Tunneling: [[06-Pivoting-Directory|06 - Pivoting Methodology]] | [[🔀 ligolo-chisel]]
> - Web Attack Methodology: [[02-Web-Attack|02 - Web Attack Methodology]]
> - Active Directory Methodology: [[05-Active-Directory|05 - Active Directory Methodology]]

---

## Navigation Triage Matrix

| Operational Goal | Scanning Strategy / Technique | Quick Link |
|---|---|---|
| Run the standardized 3-stage host scan | Standard 3-Tier Scan Strategy | [[#1. Standard 3-Tier Host Scanning Strategy]] |
| Discover live hosts on local or internal subnets | Host Discovery (Ping Sweeps) | [[#2. Network Host Discovery (Ping Sweeps)]] |
| Scan through SOCKS proxies or Ligolo TUN tunnels | Pivoted Scanning | [[#3. Scanning Through Tunnels (Pivoting)]] |
| TCP SYN vs TCP Connect vs UDP capabilities | Scan Types Reference | [[#4. Nmap Scan Types Comparison]] |
| Tuning timing, ports, evasion, and output formats | Essential Flags and Options | [[#5. Essential Command Flags and Performance Tuning]] |
| High-yield vulnerability and enumeration NSE scripts | NSE Scripts Master Reference | [[#6. High-Yield Nmap Scripting Engine (NSE) Modules]] |
| Copy-paste workflow and terminal cheat sheet | Quick Reference | [[#7. Quick Reference Cheat Sheet]] |

---

## 1. Standard 3-Tier Host Scanning Strategy

Every OSCP exam machine and lab target begins with this standardized 3-phase scanning sequence:

### Phase 1: Fast Full TCP Port Sweep (~1 minute)
Identifies every open port across the entire 65,535 port spectrum without wasting time on service probes:
```bash
sudo nmap -p- --min-rate 1000 -T4 -Pn [TARGET_IP] -oA nmap/allports
```

#### Automated Open Port Extraction One-Liner
Parse the grepable `.gnmap` output file to extract a clean, comma-delimited port list:
```bash
PORTS=$(grep -oP '\d+/open' nmap/allports.gnmap | cut -d/ -f1 | tr '\n' ',' | sed 's/,$//') && echo "Discovered Ports: $PORTS"
```

### Phase 2: Targeted Deep Service and Script Scan
Launches default NSE enumeration scripts (`-sC`) and service banner fingerprinting (`-sV`) strictly on confirmed open ports:
```bash
sudo nmap -sC -sV -p $PORTS [TARGET_IP] -oA nmap/detailed
```

### Phase 3: Top UDP Port Scan
Probes the most common UDP services (DNS, SNMP, TFTP, DHCP, NTP, NFS) in the background:
```bash
sudo nmap -sU --top-ports 20 -Pn [TARGET_IP] -oA nmap/udp
```

> 💡 **High-Yield UDP Ports**:
> - Port 53 (DNS)
> - Port 69 (TFTP)
> - Port 161 (SNMP - check default community strings like `public` or `private`)
> - Port 500 / 4500 (IKE VPN)
> - Port 2049 (NFS)

---

## 2. Network Host Discovery (Ping Sweeps)

### Subnet Sweep from Kali
```bash
# Ping sweep to discover live hosts across a /24 subnet
sudo nmap -sn [SUBNET]/24 -oA nmap/sweep

# Extract list of live target IPs
grep "Up" nmap/sweep.gnmap | awk '{print $2}' > live_hosts.txt
```

### Ping Sweep from Compromised Linux Pivot (No Nmap)
```bash
# Bash one-liner ping sweep
for i in $(seq 1 254); do (ping -c 1 -W 1 [SUBNET_PREFIX].$i | grep "bytes from" &); done | awk '{print $4}' | tr -d ':'
```

### Ping Sweep from Compromised Windows Pivot (PowerShell)
```powershell
1..254 | ForEach-Object {
    $ip = "[SUBNET_PREFIX].$_"
    if (Test-Connection -ComputerName $ip -Count 1 -Quiet) {
        Write-Output "$ip is UP"
    }
}
```

---

## 3. Scanning Through Tunnels (Pivoting)

> ⚠️ **Critical Rules for Pivoted Scanning**:
> - **Must use TCP Connect Scan (`-sT`)**: Raw SYN packets (`-sS`) cannot be forged across userland SOCKS proxies.
> - **Must use No Ping (`-Pn`)**: ICMP ping packets cannot traverse SOCKS proxies.
> - **Must disable DNS (`-n`)**: Avoids long timeout hangs on unresolvable internal hostnames.
> - **No UDP Scanning**: Standard SOCKS4/5 proxies do not support UDP packets.
> - **Tune Scan Volume**: SOCKS proxies create socket overhead; scan specific high-value ports rather than full ranges.

### Scanning via Proxychains (Chisel and SSH Dynamic)
```bash
# Fast triage scan of primary corporate ports
proxychains -q nmap -sT -Pn -n -p 21,22,80,135,139,445,1433,3306,3389,5985,8080 [INTERNAL_IP]

# Scan top 100 ports
proxychains -q nmap -sT -Pn -n -F [INTERNAL_IP]

# Internal subnet sweep
proxychains -q nmap -sT -Pn -n -p 445,3389,5985 [INTERNAL_SUBNET]/24
```

### Scanning via Ligolo-ng (Transparent TUN Interface)
Ligolo routes packets through a kernel TUN adapter (`ligolo`), allowing native Nmap scanning without `proxychains`:
```bash
# Direct scan against internal IP
sudo nmap -sT -Pn -n -p- --min-rate 1000 [INTERNAL_IP] -oA nmap/internal
```
*Note: Using `-sT` is still recommended across Ligolo tunnels for maximum connection reliability.*

### Native Port Probing Without Nmap
When neither Nmap nor proxychains can be deployed:

```bash
# Linux Bash TCP Port Sweep:
for port in 21 22 80 135 139 443 445 1433 3306 3389 5985 8080; do
    (echo > /dev/tcp/[INTERNAL_IP]/$port) 2>/dev/null && echo "Port $port is OPEN"
done

# Windows PowerShell Socket Probe:
1..1024 | ForEach-Object {
    $sock = New-Object System.Net.Sockets.TcpClient
    $conn = $sock.BeginConnect("[INTERNAL_IP]", $_, $null, $null)
    $wait = $conn.AsyncWaitHandle.WaitOne(100, $false)
    if ($sock.Connected) { Write-Output "Port $_ is OPEN" }
    $sock.Close()
}
```

---

## 4. Nmap Scan Types Comparison

| Scan Type | Flag | Requires Root? | SOCKS Proxy Compatible? | Primary Characteristics |
|---|---|---|---|---|
| **TCP SYN (Half-Open)** | `-sS` | ✅ Yes | ❌ No | Default scan; sends SYN, receives SYN-ACK, drops RST; fast and stealthy |
| **TCP Connect** | `-sT` | ❌ No | ✅ Yes | Completes 3-way handshake via OS socket; required across tunnels |
| **UDP Scan** | `-sU` | ✅ Yes | ❌ No | Probes UDP ports (DNS, SNMP, NFS); slower; relies on ICMP port unreachable |
| **Ping Sweep (No Port Scan)** | `-sn` | ❌ No | ❌ No | Host discovery only; sends ICMP echo, TCP ACK (80), TCP SYN (443) |

---

## 5. Essential Command Flags and Performance Tuning

### Port Selection Options
```bash
-p 22,80,443       # Specific comma-delimited ports
-p 1-1024          # Port range
-p-                # All 65,535 ports (mandatory for OSCP)
-F                 # Fast scan: Top 100 most common ports
--top-ports 20     # Scan top N most common ports
```

### Timing and Performance Options
```bash
-T4                # Aggressive timing (OSCP standard for reliable local networks)
-T3                # Normal timing (use if target network drops packets or throttles)
--min-rate 1000    # Minimum packets per second (massively accelerates full port scans)
--max-retries 1    # Limit packet retransmission attempts (accelerates UDP scans)
```

### Detection and Fingerprinting Options
```bash
-sV                # Probe open ports to determine service names and version details
-sC                # Execute default safe NSE scripts
-O                 # Enable TCP/IP stack OS fingerprinting
-A                 # Comprehensive scan: -sV + -sC + -O + traceroute
```

### Firewall and IDS Evasion Options
```bash
-Pn                # Treat all hosts as online; skip ICMP host discovery probe
-n                 # Never perform reverse DNS resolution (accelerates scans)
--disable-arp-ping # Disable ARP discovery on local subnets (forces TCP/IP probes)
```

### Output Formats
```bash
-oA nmap/target    # Save scan results in all 3 formats (.nmap, .gnmap, and .xml)
-oN nmap/scan.txt  # Human-readable format
-oG nmap/scan.gnmap# Grepable format for shell pipelines
-v                 # Increase verbosity (prints open ports as soon as detected)
```

---

## 6. High-Yield Nmap Scripting Engine (NSE) Modules

### Default and Vulnerability Scans
```bash
# Run default safe NSE script suite
nmap -sC [TARGET_IP]

# Run general vulnerability auditing scripts (intrusive)
nmap -p 445 --script vuln [TARGET_IP]
```

### SMB and Windows Enumeration
```bash
# Enumerate accessible SMB shares
nmap -p 445 --script smb-enum-shares [TARGET_IP]

# Enumerate local and domain users via MSRPC/SAM
nmap -p 445 --script smb-enum-users [TARGET_IP]

# Check for critical SMB vulnerabilities (MS17-010 EternalBlue)
nmap -p 445 --script "smb-vuln-ms17-010" [TARGET_IP]
nmap -p 445 --script "smb-vuln*" [TARGET_IP]

# Extract OS build, computer name, and domain from SMB
nmap -p 445 --script smb-os-discovery [TARGET_IP]
```

### Web Services (HTTP)
```bash
# Enumerate interesting web paths and directories
nmap -p 80,443 --script http-enum [TARGET_IP]

# Inspect HTTP server headers and technologies
nmap -p 80,443 --script http-headers [TARGET_IP]

# Check for Shellshock (CVE-2014-6271) on CGI scripts
nmap -p 80 --script http-shellshock --script-args uri=/cgi-bin/test.sh [TARGET_IP]
```

### Network Services (FTP, SMTP, DNS, RDP)
```bash
# Check FTP anonymous authentication
nmap -p 21 --script ftp-anon [TARGET_IP]

# Enumerate valid SMTP user accounts
nmap -p 25 --script smtp-enum-users [TARGET_IP]

# Attempt DNS zone transfer
nmap -p 53 --script dns-zone-transfer --script-args dns-zone-transfer.domain=[DOMAIN] [TARGET_IP]

# Extract Windows domain name and NetBIOS hostname from RDP
nmap -p 3389 --script rdp-ntlm-info [TARGET_IP]
```

### Script Discovery and Documentation
```bash
# Search available NSE scripts by service keyword
ls /usr/share/nmap/scripts/ | grep -i smb
ls /usr/share/nmap/scripts/ | grep -i http

# View script documentation and argument requirements
nmap --script-help smb-enum-shares
```

---

## 7. Quick Reference Cheat Sheet

```bash
# 1. Full TCP port sweep
sudo nmap -p- --min-rate 1000 -T4 -Pn [TARGET_IP] -oA nmap/allports

# 2. Extract open ports
PORTS=$(grep -oP '\d+/open' nmap/allports.gnmap | cut -d/ -f1 | tr '\n' ',' | sed 's/,$//')

# 3. Targeted deep service scan
sudo nmap -sC -sV -p $PORTS [TARGET_IP] -oA nmap/detailed

# 4. Top UDP scan
sudo nmap -sU --top-ports 20 -Pn [TARGET_IP] -oA nmap/udp

# 5. SOCKS Proxy scan via Proxychains
proxychains -q nmap -sT -Pn -n -p 21,22,80,135,139,445,3389,5985 [INTERNAL_IP]

# 6. SMB vulnerability verification
nmap -p 445 --script "smb-vuln-ms17-010" [TARGET_IP]

# 7. Check service version in Searchsploit
searchsploit [SERVICE_NAME] [VERSION]
```
