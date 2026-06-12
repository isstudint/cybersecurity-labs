# Firewall hardening with iptables

After exploiting Metasploitable 2 and Windows 7 from the offensive side, I wanted to go back and see if basic firewall rules could have stopped those attacks. I configured `iptables` on a Linux target and tested whether the same exploits still worked.

Short answer: a properly configured firewall blocks most of what I did in the exploitation labs.

---

## The problem

Metasploitable 2 ships with no firewall rules. Everything is open. That's why it was so easy to exploit - Samba, vsftpd, Telnet, all of it was just sitting there accepting connections from anyone.

```bash
sudo iptables -L
```

Output: all chains set to `ACCEPT` with no rules. Wide open.

---

## Building a basic ruleset

The approach: deny everything by default, then open only what's needed. This is the "default deny" model.

### Step 1 - Set default policies to DROP

```bash
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT
```

This drops all incoming traffic and forwarding by default. Outbound is allowed so the machine can still reach the internet for updates.

### Step 2 - Allow established connections

```bash
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
```

Without this, the machine can send requests but can't receive responses. This rule allows traffic that's part of an existing connection.

### Step 3 - Allow loopback

```bash
sudo iptables -A INPUT -i lo -j ACCEPT
```

Localhost traffic needs to work for local services.

### Step 4 - Allow only SSH (port 22)

```bash
sudo iptables -A INPUT -p tcp --dport 22 -s 10.0.2.4 -j ACCEPT
```

Only allow SSH from Kali's IP (`10.0.2.4`). No one else gets in.

### Step 5 - Log dropped traffic

```bash
sudo iptables -A INPUT -j LOG --log-prefix "IPTABLES-DROP: " --log-level 4
```

Everything that gets dropped is logged. This feeds into SIEM monitoring later - Wazuh can pick up these log entries and generate alerts.

---

## Testing: do the exploits still work?

Ran the same attacks from my exploitation writeups against the hardened machine:

| Attack | Port | Before firewall | After firewall |
|---|---|---|---|
| Samba usermap_script | 445 | Root shell | Connection refused |
| vsftpd backdoor | 21/6200 | Root shell | Connection refused |
| Telnet (no auth) | 23 | Root shell | Connection refused |
| Nmap SYN scan | All | 40+ open ports | Only port 22 visible |

Everything was blocked except SSH. The Nmap scan only showed port 22 open with the rest filtered.

```bash
nmap -sV 10.0.2.3
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 4.7p1
```

The attack surface went from 40+ services down to one.

---

## Making rules persistent

`iptables` rules don't survive a reboot by default. To save them:

```bash
sudo apt install iptables-persistent
sudo netfilter-persistent save
```

Or manually:

```bash
sudo iptables-save > /etc/iptables/rules.v4
```

---

## UFW - simpler alternative

`iptables` is powerful but verbose. UFW (Uncomplicated Firewall) is a frontend that makes the same thing easier:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 10.0.2.4 to any port 22
sudo ufw enable
```

Same result, fewer commands. Under the hood, UFW writes `iptables` rules.

---

## Takeaway

A firewall is the simplest defense that would have stopped every exploit I ran in this lab. None of the attacks required anything more sophisticated than connecting to an open port. Default deny with explicit allow rules eliminates the attack surface that made Metasploitable exploitable.

The tricky part in real environments isn't setting up the firewall - it's knowing what to allow. You need to understand what services are legitimately required and restrict everything else. That's where the offensive perspective helps: knowing *how* attackers use open ports makes it easier to decide which ones to close.
