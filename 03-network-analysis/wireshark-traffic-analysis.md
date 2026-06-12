# Wireshark traffic analysis

I wanted to see what exploitation actually looks like on the wire. Ran Wireshark on Kali during several of the attacks I performed against Metasploitable 2 and Windows 7, then went back through the captures to understand the traffic patterns.

This is the defensive side of what I already did offensively. If I know what an attack looks like in a packet capture, I know what to look for when monitoring a network.

---

## Setup

Started Wireshark on the `eth0` interface on Kali before launching each exploit. Saved each capture as a `.pcap` file for later analysis.

```bash
sudo wireshark &
```

Selected the `eth0` interface and started capturing before running each attack from a separate terminal.

---

## Capture 1: EternalBlue (MS17-010)

### What I saw

The EternalBlue exploit generates a recognizable traffic pattern on port 445 (SMB):

1. **SMB negotiation** - Normal-looking protocol negotiation between Kali and Windows 7
2. **Trans2 requests** - A burst of SMB `Trans2` session setup requests. This is the actual exploitation phase. Normally you'd see one or two of these, not dozens in rapid succession.
3. **Reverse shell connection** - After exploitation, a new TCP connection opens on the Meterpreter port (4444 by default). Traffic on this port is the C2 channel.

### Filter used

```
smb || tcp.port == 4444
```

### What stands out to a defender

- Multiple `Trans2 SESSION_SETUP` requests in a short burst. Normal SMB traffic doesn't look like this.
- New outbound connection on a high port (4444) immediately after the SMB burst. That's the reverse shell calling home.
- The Meterpreter traffic on port 4444 is encrypted (TLS), so you can't read the content, but the timing and port usage are suspicious.

---

## Capture 2: Samba exploitation (usermap_script)

### What I saw

The Samba exploit (CVE-2007-2447) injects a command through the username field during SMB authentication.

1. **SMB session setup** - Normal initial connection on port 445
2. **Malicious username** - In the session setup request, the username field contains a backtick command injection: `` `nohup /bin/sh -c ...` ``
3. **Reverse connection** - New TCP connection back to Kali on the listener port

### Filter used

```
smb || tcp.port == 4444
```

### What stands out

- The username field in the SMB session setup contains shell commands. That's obviously not a real username. Any content inspection or IDS rule checking for backticks or shell metacharacters in SMB auth fields would catch this.
- Same pattern as EternalBlue after exploitation: new connection on a non-standard port.

---

## Capture 3: Nmap SYN scan

### What I saw

An Nmap SYN scan (`nmap -sS`) is fast but noisy:

1. **SYN packets to sequential ports** - Kali sends SYN packets to hundreds of ports in rapid succession
2. **SYN-ACK responses** - Open ports respond with SYN-ACK
3. **RST from Kali** - Instead of completing the handshake, Kali sends RST (this is why it's called a "half-open" scan)
4. **RST-ACK from closed ports** - Closed ports respond with RST-ACK

### Filter used

```
tcp.flags.syn == 1 && tcp.flags.ack == 0
```

### What stands out

- Hundreds of SYN packets from a single source in seconds. Normal clients don't connect to 200 ports on the same host.
- No completed TCP handshakes. Every connection is half-open (SYN, SYN-ACK, RST). A port scan, not legitimate traffic.
- Sequential or semi-sequential port targeting. Nmap's default scan order has a recognizable pattern.

---

## Common Wireshark filters I use

| Filter | Purpose |
|---|---|
| `ip.addr == 10.0.2.3` | All traffic to/from Metasploitable |
| `tcp.port == 445` | SMB traffic |
| `tcp.port == 4444` | Default Meterpreter port |
| `tcp.flags.syn == 1 && tcp.flags.ack == 0` | SYN packets only (scan detection) |
| `smb` | All SMB protocol traffic |
| `ftp` | FTP traffic |
| `http` | HTTP traffic |
| `tcp.analysis.retransmission` | Retransmissions (network issues) |

---

## Takeaway

Running Wireshark during my own attacks gave me a better understanding of what defenders see. The patterns are pretty distinct once you know what to look for: bursts of unusual protocol requests, reverse connections on non-standard ports, and scanning traffic that doesn't match normal user behavior.

The harder part is scale. In a real network with hundreds of hosts, you can't sit there watching Wireshark. That's where IDS (Snort) and SIEM (Wazuh) come in - they automate the detection of these patterns. But understanding the raw packets helps you write better rules and investigate alerts that automated tools flag.
