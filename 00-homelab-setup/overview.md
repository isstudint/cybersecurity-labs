## Home Lab Overview

The Host is running on Windows 10 with virtual networks connected on NAT network so I can access other host.


### Common diagram types

- `Kali Linux` - (Use for Defense and Offense practice / attack simulation / security testing)
- `Windows 7` - (Use for attack simulations)
- `Metasploitable 2` (intentionally vulnerable system)

---
### Network Design
```mermaid
flowchart TD

    Host[Windows 10 Host]

    subgraph Lab["NAT Network (Homelab)"]
        Kali[Kali Linux]
        Win7[Windows 7]
        Meta[Metasploitable 2]
    end

    Host --> Lab

    Kali --> Win7
    Kali --> Meta
```
---
### Tools Used
- Kali Linux
- Basic OSINT tools
- Nmap
- Metasploit
- Wireshark
- Virtualization platforms (VirtualBox)

---

### Key Skills Developed

- Learned basic fundamentals Planning &rarr; Scanning &rarr; Exploitation &rarr; Post-Exploitation &rarr; Reporting
- Awareness of common system vulnerabilities
- Learned offensive perspective