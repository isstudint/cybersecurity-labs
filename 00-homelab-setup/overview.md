# Homelab overview

The host is running on Windows 10 with all VMs connected on a NAT network so they can communicate with each other but are isolated from my home network.

---

## Machines

- **Kali Linux** - Used for both offense and defense. Runs Nmap, Metasploit, Wireshark, Snort, and Wazuh Manager.
- **Windows 7** - Unpatched target for exploitation (EternalBlue). Runs Wazuh agent for monitoring.
- **Metasploitable 2** - Intentionally vulnerable Linux machine. Multiple exploitable services (Samba, vsftpd, Telnet). Forwards syslog to Wazuh.

---

## Network design

```mermaid
flowchart TD
    Host["Windows 10 Host"]

    subgraph Lab["NAT Network 10.0.2.0/24"]
        Kali["Kali Linux<br/>10.0.2.4<br/>Attacker / Defender / SIEM"]
        Win7["Windows 7<br/>10.0.2.6<br/>Target + Wazuh Agent"]
        Meta["Metasploitable 2<br/>10.0.2.3<br/>Target + Syslog"]
    end

    Host --> Lab

    Kali -->|attacks & monitors| Win7
    Kali -->|attacks & monitors| Meta
    Win7 -->|logs| Kali
    Meta -->|syslog| Kali
```

---

## Tools used

**Offensive:** Nmap, Metasploit, Meterpreter, searchsploit, OpenVAS, theHarvester, Sherlock

**Defensive:** Snort IDS, Wazuh SIEM, iptables/UFW, Wireshark, auditd

**Other:** VirtualBox, Python, Bash, Git

---

## Skills developed

- Penetration testing workflow: planning, scanning, exploitation, post-exploitation, reporting
- Network traffic analysis and packet inspection
- IDS rule writing and tuning
- SIEM deployment and log correlation
- Firewall configuration and network hardening
- Incident investigation using NIST IR framework
- Security automation with Python