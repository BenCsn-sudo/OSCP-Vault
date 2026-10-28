# Blaster - TryHackMe

> **Kill Chain Summary**:
> Port Scanning (Ports 80, 3389) -> Web Directory Discovery (`/retro/`) -> WordPress Blog Review -> OSINT Comment Extraction (Cleartext Password `parzival` for User `Wade`) -> Direct RDP Authentication -> Interactive Graphical Session -> Filesystem Investigation (Recycle Bin / `hhupd.exe`) -> UAC Elevation via Windows Certificate Dialog (CVE-2019-1388) -> Elevated Command Prompt (`Administrator`).

| Property | Value |
| :--- | :--- |
| **Target Name** | Blaster |
| **Platform** | TryHackMe |
| **OS** | Windows Server 2016 |
| **Difficulty** | Easy |
| **Status** | Rooted |
| **Key Techniques** | Web Enumeration, OSINT Comment Scraping, RDP Access, Windows Certificate Dialog UAC Bypass (CVE-2019-1388) |

---

## 1. Reconnaissance and Enumeration

### 1.1 Port Scanning

A full TCP port scan was performed using Nmap with `-Pn`:

```bash
sudo nmap -Pn -sC -sV -p- -T4 -oA nmap/full_tcp_10.130.182.142 10.130.182.142
```

**Open Ports:**

| Port | Service | Version | Description |
| :--- | :--- | :--- | :--- |
| 80/tcp | HTTP | Microsoft IIS httpd 10.0 | Default IIS landing page |
| 3389/tcp | RDP | Microsoft Terminal Services | Remote Desktop protocol |

```text
PORT     STATE SERVICE       VERSION
80/tcp   open  http          Microsoft IIS httpd 10.0
3389/tcp open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info: 
|   Target_Name: RETROWEB
|   NetBIOS_Domain_Name: RETROWEB
|   NetBIOS_Computer_Name: RETROWEB
|   DNS_Domain_Name: RetroWeb
|   DNS_Computer_Name: RetroWeb
|   Product_Version: 10.0.14393
```

### 1.2 Web Directory Enumeration

Directory brute-forcing with Gobuster using extensions identified a custom application path:

```bash
gobuster dir -u http://10.130.182.142/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html,bak,old
```

```text
/retro               (Status: 301) [--> http://10.130.182.142/retro/]
```

Navigating to `http://10.130.182.142/retro/` displays a personal blog based on retro gaming and "Ready Player One" themes, administered by author **Wade**.

---

## 2. Initial Access

### 2.1 OSINT Analysis and Credential Discovery

Reviewing the blog posts and user comments on the site revealed a public comment left by author Wade detailing his password:

![[Pasted image 20260709191452.png]]

> "Leaving myself a note so I don't forget my password again: parzival"

Discovered credentials:
- **Username:** `Wade`
- **Password:** `parzival`

### 2.2 Direct RDP Authentication

Because port 3389 (RDP) is exposed, authenticate directly to the host without deploying a reverse shell:

```bash
xfreerdp /v:10.130.182.142 /u:Wade /p:parzival /cert:ignore
```

An interactive desktop session is established as `RetroWeb\Wade`.

Retrieve the user flag on the desktop:

```cmd
type C:\Users\Wade\Desktop\user.txt
```

```text
[FLAG REDACTED]
```

---

## 3. Privilege Escalation

### 3.1 Local Enumeration: Executable Discovery

Exploring the user's desktop and Recycle Bin reveals an old installer binary: `hhupd.exe` (Microsoft HTML Help Update executable).

Checking the binary signature and version confirms it is an unpatched legacy utility vulnerable to **CVE-2019-1388**.

### 3.2 Exploitation: UAC Bypass via Certificate Dialog (CVE-2019-1388)

> [!NOTE] Concept: CVE-2019-1388 (Windows Error Reporting / UAC Privilege Escalation)
> When an executable requiring administrative elevation triggers the User Account Control (UAC) prompt, Windows inspects the digital signature. Clicking "Show information about the publisher's certificate" displays certificate details. Clicking the hyperlink inside the certificate launches the default browser (Internet Explorer) running elevated under `NT AUTHORITY\SYSTEM` or the elevated consent token. From this browser instance, an attacker can launch an administrative command prompt.

#### Step-by-Step GUI Exploitation
1. Right-click `hhupd.exe` and select **Run as administrator**.
2. When the User Account Control (UAC) consent dialog appears, click **Show more details** -> **Show information about the publisher's certificate**.
3. In the certificate dialog, locate the **Issued by** section and click the **VeriSign commercial hyperlink**.
4. Internet Explorer launches under the elevated token context.
5. In Internet Explorer, navigate to the menu bar: **File -> Save as**.
6. In the file explorer address bar, enter `C:\Windows\System32\cmd.exe` and press **Enter**.
7. An administrative command prompt opens with elevated privileges:

```cmd
whoami
retroweb\administrator
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
# Direct RDP Connection Bypassing Certificate Warnings
xfreerdp /v:<TARGET_IP> /u:<USER> /p:'<PASSWORD>' /cert:ignore

# Direct RDP Connection with Legacy TLS Configuration
xfreerdp /v:<TARGET_IP> /u:<USER> /p:'<PASSWORD>' /cert:ignore /tls-seclevel:0

# Gobuster Directory Search with File Extensions
gobuster dir -u http://<TARGET_IP>/ -w <WORDLIST> -x php,txt,html,bak,old
```

---

## 5. Key Takeaways and Lessons Learned

1. **Content Review Before Brute-Forcing:** Automated dictionary attacks should only be conducted after thoroughly reading the target site's published content. Comments, release notes, and personal posts frequently disclose passwords or account names in plain text.
2. **Direct Protocol Authentication:** When Remote Desktop (RDP) or WinRM are reachable externally alongside valid user credentials, connect directly through those protocols rather than attempting complex web shell uploads.
3. **GUI-Based UAC Vectors:** In graphical environments, privilege escalation is not limited to kernel exploits or service permissions. Flaws in administrative prompts and certificate inspection dialogues can allow subverting UAC to launch privileged subshells.

---

#tryhackme #windows #easy #wordpress #osint #rdp #cve-2019-1388 #uac-bypass #gui #privesc
