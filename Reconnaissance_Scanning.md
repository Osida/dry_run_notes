# 🗓️ 4/6/2026 — SSH Multiplexing & Pivoting

## 🔍 Concept Overview
SSH multiplexing allows multiple SSH sessions to reuse a single authenticated connection, dramatically speeding up pivoting, port forwarding, and enumeration inside a target network.

This workflow is foundational for internal recon because it:
- Reduces repeated authentication
- Enables persistent tunnels
- Supports layered pivoting (t1 → t2 → deeper hosts)
- Integrates cleanly with tools like proxychains and nmap

## ⚙️ Methodology

### 1. Create the Master Control Connection
```bash
ssh -M -S /tmp/<name> <user>@<jump_ip>

ssh -M -S /tmp/web student@10.50.15.247
# jSmJnXaaKOs7
```
- `/tmp/<name>` is the file path for a `ControlSocket`. It is used by SSH's `ControlMaster` feature to allow multiple SSH sessions to share a single network connection
- `-M`: Tells SSH to become the "Master" of the connection for multiplexing.
- `-S /tmp/jump`: Specifies the location of the control socket. You can name this file anything and put it anywhere you have write permissions, but /tmp/ is common for temporary session data
- Once the master connection is active, running `ssh -S /tmp/<name> <user>@<jump_ip>` in another terminal grants you instant access, bypassing the usual password prompts and handshakes.

### 2. Enumerate Internal Network
```bash
for i in {1..254}; do (ping -c 1 x.x.x.$i | grep "bytes from" &) 2>/dev/null; done
```
- Ping sweep to identify live hosts

### 3. Start SOCKS Proxy for Proxychains
```bash
ssh -S /tmp/<name> <Host Alias> -O forward -D 9050 dummy_host

ssh -S /tmp/<name> <Host Alias> -O forward -D 9050

# Cancel an active forwarding.
ssh -S /tmp/<name> <Host Alias> -O cancel -KD 9050

```
- `dummy_host` is required syntactically but ignored.
- This adds a `dynamic port forward` to the existing master connection.

### 4. Service Enumeration via Proxy
```bash
proxychains nmap -Pn -sV -n -T4 <target_ip>
```

### 5. Local Port Forwarding (On Demand)
```bash
ssh -S /tmp/jump jump -O forward -D 9050 -L <local_port>:<target_ip>:<target_port> dummy_host


ssh -S /tmp/jump jump -O forward -D 9050 -L <local_port>:<target_ip>:<target_port>
```

### 6. Build a New Pivot Layer (t1)
```bash
ssh -MS /tmp/<t1> <user>@localhost -p <local_port>
```
> “Use your first pivot to SSH into the next internal machine, and create a new master socket there so you can pivot even deeper.”
- `-M`: Creates a new master connection
- `-S /tmp/t1`: Stores the new control socket at `/tmp/t1`
- `<user>@localhost`: You are SSH’ing to localhost, not the real internal host.
    - Why? Because you already forwarded a local port (e.g., 1234) to the internal host’s SSH port.
- `-p <local_port>`: This is the port you forwarded earlier:
    ```bash
    -L 1234:10.208.50.42:22
    # so
    ssh user@localhost -p 1234
    ```
- You are creating a second master SSH connection (t1), but this time THROUGH your first pivot.
```text
Up to this point:
1. /tmp/jump = your first master socket
2. It connects you to the jump host
3. You’ve forwarded a port from your machine → through the jump host → to an internal machine

Now you want to pivot again — meaning:
1. “Use the internal machine as a new jump host.”
2. To do that, you create a new master socket that connects through the first one.
```

### 7. Forward Ports Through t1
```bash
ssh -S /tmp/t1 -O forward -D 9050 -L <local_port>:<target_ip>:<target_port>
```

### 8. Repeat for Deeper Pivoting
- Continue chaining as needed.

### 💡Tips
- Stale sockets may require manual cleanup if SSH crashes.