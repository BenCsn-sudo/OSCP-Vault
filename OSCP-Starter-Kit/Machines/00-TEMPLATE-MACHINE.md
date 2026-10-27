# [Machine Name] - [Platform]

> **Kill Chain Summary**:
> [Reconnaissance & Ports] -> [Initial Vulnerability] -> [Foothold User Context] -> [Internal PrivEsc Vector] -> [Root / SYSTEM Shell].

| Property | Value |
| :--- | :--- |
| **Target Name** | [Machine Name] |
| **Platform** | TryHackMe / Proving Grounds / HackTheBox |
| **OS** | Linux / Windows |
| **Difficulty** | Easy / Intermediate / Hard |
| **Status** | Completed / Rooted |
| **Key Techniques** | [e.g., SQLi, Kerberoasting, Sudo NOPASSWD, Token Impersonation] |

---

## 1. Reconnaissance and Enumeration

### 1.1 Port Scanning
```bash
sudo nmap -p- --min-rate 1000 -T4 -Pn [TARGET_IP] -oA nmap/tcp_all
sudo nmap -sC -sV -p [OPEN_PORTS] -Pn [TARGET_IP] -oA nmap/tcp_detailed
```

**Open Ports:**
- **Port [PORT]:** [Service Name] ([Version])
- **Port [PORT]:** [Service Name] ([Version])

### 1.2 Web Directory Enumeration
```bash
gobuster dir -u http://[TARGET_IP]/ -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -x php,txt,html,bak -t 40
```

**Discovered Endpoints & Notes:**
- `/[ENDPOINT]/` : [Description and functionality observed]

---

## 2. Initial Access

### 2.1 Vulnerability Discovery
*Explain the discovered vulnerability, relevant software versions, misconfigurations, or credentials.*

### 2.2 Exploitation Execution
*Document exact commands, payloads, or exploit scripts executed to attain remote code execution.*

```bash
# [Exploit execution command]
```

- **Local User Proof Flag (`local.txt` / `user.txt`)**: `[FLAG REDACTED]`

---

## 3. Privilege Escalation

### 3.1 Local Enumeration Findings
*Internal enumeration steps that revealed the privilege escalation path (e.g., Sudo NOPASSWD, SUID binary, unquoted service path, stored credentials).*

```bash
# [Enumeration command executed on target]
```

### 3.2 Privilege Escalation Execution
*Exact commands used to elevate to root (Linux) or NT AUTHORITY\SYSTEM (Windows).*

```bash
# [Privilege escalation command]
```

- **Administrative Proof Flag (`proof.txt` / `root.txt`)**: `[FLAG REDACTED]`

---

## 4. Useful Commands Summary

```bash
# Command 1: [Purpose]
[command]

# Command 2: [Purpose]
[command]

# Command 3: [Purpose]
[command]
```

---

## 5. Key Takeaways and Lessons Learned

1. **[Key Lesson 1]**: [What technique or nuance was critical to solving this target?]
2. **[Key Lesson 2]**: [What rabbit hole or mistake caused wasted time during the engagement?]
3. **[Key Lesson 3]**: [What reusable command or concept should be applied to future targets?]

---

#[platform] #[os] #[difficulty] #[techniques]
