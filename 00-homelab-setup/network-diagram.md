### Network Diagram
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