# Nmap reconnaissance

Nmap was used to scan both target machines - Metasploitable 2 and Windows 7. The goal was to find open ports, running services, and potential attack paths before moving to exploitation.

**MITRE ATT&CK:** T1046 (Network Service Discovery)

---

## Scanning Metasploitable 2

### Service scan

```bash
nmap -sV 10.0.2.0/24
```

Ran a subnet scan first to find live hosts. Metasploitable showed up at `10.0.2.3` with a lot of open ports - FTP, SSH, Telnet, HTTP, Samba, and more. Most of these are intentionally misconfigured or outdated, which is the whole point of the machine.

### Findings

| Port | Service | Version | Notes |
|---|---|---|---|
| 21 | FTP | vsftpd 2.3.4 | Known backdoor (CVE-2011-2523) |
| 23 | Telnet | - | Open, no auth needed |
| 139/445 | Samba | 3.x - 4.x | Version too vague, needed further enumeration |
| 6200 | - | - | Related to vsftpd backdoor trigger |

---

## Samba version - Nmap wasn't enough

Nmap returned a vague Samba version range (`3.x - 4.x`), which wasn't specific enough to find an exploit. I didn't want to guess and run through every 3.x to 4.x exploit, so I used a Metasploit auxiliary scanner to get the exact version.

```bash
msfconsole
use auxiliary/scanner/smb/smb_version
set RHOSTS 10.0.2.3
run
```

Result: Samba `3.0.20`. That confirmed it was exploitable via `usermap_script` (CVE-2007-2447).

---

## Scanning Windows 7

```bash
nmap -sV --script vuln 10.0.2.6
```

Nmap's vuln scripts flagged the machine as vulnerable to MS17-010 (EternalBlue). SMB port 445 was open and the system was running an unpatched version of Windows 7.

This confirmed the attack path before touching Metasploit.

---

## Takeaway

Nmap is a network scanner that discovers hosts and services by sending packets and analyzing responses. It gives you the map, but it won't always give you exact versions or confirm whether something is actually exploitable. The Samba case was a good example - Nmap got me close, but I needed the auxiliary scanner to get a specific enough version to actually find the right exploit. Knowing when to go deeper matters more than the initial scan.
