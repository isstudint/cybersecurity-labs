# Nmap Reconnaissance

I Nmap was used to scan both target machines  Metasploitable 2 and Windows 7. The goal was to identify open ports, running services, and potential attack vectors before moving to exploitation.

In general I use Nmap to 

---

## Scanning Metasploitable 2

### Basic Service Scan

```bash
nmap -sV 10.0.2.0/24 and found the metasolpitable on 10.0.2.3
```

Identified a wide range of open ports typical of Metasploitable — FTP, SSH, Telnet, HTTP, Samba, and more.

### Key Findings

| Port | Service | Version (Nmap) | Notes |
|---|---|---|---|
| 21 | FTP | vsftpd 2.3.4 | Known backdoor |
| 23 | Telnet | — | Open, no auth needed |
| 139/445 | Samba | 3.x - 4.x | Version too vague  needed further enumeration |
| 6200 | — | — | Related to vsftpd backdoor trigger |

---

## Samba Version — Nmap Wasn't Enough

Nmap returned a vague Samba version range (`3.x - 4.x`), which wasn't specific enough to find an exploit directly. Used a Metasploit auxiliary scanner to get the exact version.

```bash
msfconsole
use auxiliary/scanner/smb/smb_version
set RHOSTS 192.168.x.x
run
```

**Result:** Samba `3.0.20` — confirmed exploitable via `trans2open`.

---

## Scanning Windows 7

```bash
nmap -sV --script vuln 10.0.2.3
```

Nmap flagged the machine as potentially vulnerable to **MS17-010 (EternalBlue)** — SMB Remote Code Execution.

This confirmed the attack path before touching Metasploit.

---

## Takeaway
In general its a network scanner that discovers hosts and services on a computer network by sending packets and analyzing responses

Nmap gives you the map. It won't always give you exact versions or confirm exploitability  that's what auxiliary scanners and searchsploit are for. The Samba case was a good example of knowing when to go deeper.
