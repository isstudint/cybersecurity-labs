# Snort IDS - detecting attacks in real time

After running exploits and analyzing traffic in Wireshark manually, I set up Snort as an IDS (Intrusion Detection System) to automate that detection. The idea: if I know what EternalBlue looks like on the wire, I should be able to write a rule that catches it without me staring at Wireshark.

---

## What Snort does

Snort sits on the network and inspects packets in real time. When traffic matches a rule, it generates an alert. It doesn't block anything in IDS mode (that's IPS mode) - it just tells you something happened.

Think of it as automated Wireshark with pattern matching.

---

## Installation

Installed Snort on Kali:

```bash
sudo apt update
sudo apt install snort
```

During setup, it asks for the home network. I set it to `10.0.2.0/24` (my homelab NAT network).

Verify it's working:

```bash
sudo snort -V
```

---

## Configuration

Main config file: `/etc/snort/snort.conf`

Key settings I changed:

```bash
# Set the home network
var HOME_NET 10.0.2.0/24

# Set the external network (everything else)
var EXTERNAL_NET !$HOME_NET

# Rule paths
var RULE_PATH /etc/snort/rules
```

---

## Writing custom rules

Snort's community rules cover a lot, but I wanted to write my own rules for the specific attacks I ran in this lab.

### Rule format

```
action protocol source_ip source_port -> dest_ip dest_port (options)
```

### Rule 1: detect Nmap SYN scan

```
alert tcp any any -> $HOME_NET any (msg:"Possible Nmap SYN scan detected"; flags:S; threshold:type threshold, track by_src, count 20, seconds 3; sid:1000001; rev:1;)
```

This fires when a single source sends more than 20 SYN packets in 3 seconds. Normal traffic doesn't do that. Nmap does.

### Rule 2: detect EternalBlue (MS17-010)

```
alert tcp any any -> $HOME_NET 445 (msg:"Possible EternalBlue exploit attempt"; content:"|ff|SMB"; content:"|23 00 00 00 07 00|"; sid:1000002; rev:1;)
```

Looks for SMB traffic on port 445 with byte patterns associated with the Trans2 SESSION_SETUP requests that EternalBlue uses. This is a simplified version - production rules from Snort's community set (SID 41978) are more specific.

### Rule 3: detect reverse shell on common Meterpreter port

```
alert tcp $HOME_NET any -> any 4444 (msg:"Possible reverse shell connection on port 4444"; flags:S; sid:1000003; rev:1;)
```

Catches outbound SYN connections to port 4444. Meterpreter uses this as the default callback port. A machine in your network initiating a connection to an external host on port 4444 is suspicious.

### Rule 4: detect vsftpd backdoor trigger

```
alert tcp any any -> $HOME_NET 6200 (msg:"Possible vsftpd 2.3.4 backdoor connection"; flags:S; sid:1000004; rev:1;)
```

Port 6200 is where the vsftpd backdoor opens a shell. Any connection to this port is worth investigating.

---

## Saving rules

Added my custom rules to a new file:

```bash
sudo nano /etc/snort/rules/local.rules
```

Then made sure `snort.conf` includes it:

```
include $RULE_PATH/local.rules
```

---

## Running Snort

Started Snort in IDS mode on the homelab network interface:

```bash
sudo snort -A console -q -c /etc/snort/snort.conf -i eth0
```

- `-A console` - print alerts to terminal
- `-q` - quiet mode (no startup banner)
- `-c` - config file path
- `-i eth0` - network interface

---

## Testing the rules

Ran each attack from Kali while Snort was running and checked if alerts fired:

| Attack | Rule triggered | Alert message |
|---|---|---|
| `nmap -sS 10.0.2.3` | SID 1000001 | Possible Nmap SYN scan detected |
| EternalBlue exploit | SID 1000002 | Possible EternalBlue exploit attempt |
| Meterpreter callback | SID 1000003 | Possible reverse shell connection on port 4444 |
| vsftpd backdoor | SID 1000004 | Possible vsftpd 2.3.4 backdoor connection |

All four rules fired correctly. The SYN scan rule fired first (during recon), then the exploit-specific rules fired during exploitation.

### Alert output example

```
[**] [1:1000001:1] Possible Nmap SYN scan detected [**]
[Priority: 0]
06/10-14:23:17.443221 10.0.2.4:54832 -> 10.0.2.3:445
TCP TTL:64 TOS:0x0 ID:12345 IpLen:20 DgmLen:44
******S* Seq: 0xABCDEF01  Ack: 0x0  Win: 0x0400  TcpLen: 24
```

---

## IDS vs IPS

Snort ran in IDS mode here - it detected and logged but didn't block anything. In IPS (inline) mode, Snort can drop packets that match rules. I kept it in IDS mode for the lab because I wanted the attacks to succeed so I could see the full traffic pattern and correlate with my Wireshark captures.

In a production environment, you'd run it inline or pair it with a firewall that acts on Snort alerts.

---

## Takeaway

Writing Snort rules for attacks I already performed was a good exercise. Knowing the offensive side made the rules easier to write - I already knew what the traffic looked like from my Wireshark analysis. The SYN scan rule was the most useful because port scanning is usually the first thing that happens before an exploit.

The limitation is that Snort only catches what you write rules for (plus the community ruleset). If an attacker uses non-standard ports or encrypted traffic, basic rules won't catch it. That's where SIEM correlation and anomaly detection come in - covered in my [Wazuh writeup](../05-siem-monitoring/wazuh-siem-setup.md).
