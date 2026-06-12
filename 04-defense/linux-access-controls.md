# Linux access controls and hardening

This covers the basics of securing a Linux system - user management, file permissions, SSH hardening, and audit logging. I went through these on Kali and tested them against the kind of access I gained in my exploitation labs.

The question I wanted to answer: if an attacker gets a low-privilege shell, how much can proper access controls slow them down?

---

## User and group management

### Don't run everything as root

Metasploitable 2 runs most services as root. That's why every exploit gave me root access immediately - there was no privilege separation. A properly configured system runs services under dedicated accounts with limited permissions.

```bash
# Create a service account with no login shell
sudo useradd -r -s /usr/sbin/nologin samba_svc

# Create a regular user
sudo useradd -m -s /bin/bash labuser
sudo passwd labuser
```

The `-r` flag creates a system account. The `-s /usr/sbin/nologin` prevents interactive login - the account can run a service but nobody can SSH in as it.

### Sudo configuration

Instead of giving users root access, use `sudo` with specific permissions:

```bash
sudo visudo
```

```
# Allow labuser to restart services only
labuser ALL=(ALL) NOPASSWD: /bin/systemctl restart samba, /bin/systemctl restart ssh
```

This lets `labuser` restart Samba and SSH without a password, but nothing else. If an attacker compromises this account, they can't `sudo su` to root.

---

## File permissions

### Understanding permission bits

```bash
ls -la /etc/shadow
# -rw-r----- 1 root shadow 1.2K Jun 10 14:00 /etc/shadow
```

| Position | Meaning | Example |
|---|---|---|
| Owner (rw-) | Read/write for root | root can read and modify password hashes |
| Group (r--) | Read for shadow group | Members of shadow group can read (needed for auth) |
| Other (---) | Nothing for everyone else | Regular users can't access password hashes |

### Finding world-writable files

World-writable files are security risks - any user can modify them.

```bash
find / -type f -perm -o+w 2>/dev/null
```

On Metasploitable 2, this returns a concerning number of results. On a hardened system, this list should be very short (mostly just `/tmp` contents).

### Setting proper permissions on sensitive files

```bash
# SSH config - owner read/write only
sudo chmod 600 /etc/ssh/sshd_config

# SSH keys - owner read only
chmod 400 ~/.ssh/id_rsa

# Web content - owner read/write, group/other read only
sudo chmod 644 /var/www/html/index.html

# Scripts - owner execute, no one else
chmod 700 /home/labuser/scripts/*.sh
```

---

## SSH hardening

SSH is usually the one service you keep open (I left port 22 open in my [firewall writeup](firewall-hardening.md)). That means it needs to be locked down.

### `/etc/ssh/sshd_config` changes

```bash
# Disable root login
PermitRootLogin no

# Use SSH key authentication only
PasswordAuthentication no
PubkeyAuthentication yes

# Limit SSH to specific users
AllowUsers labuser

# Change default port (optional, security through obscurity)
Port 2222

# Disable empty passwords
PermitEmptyPasswords no

# Set idle timeout (disconnect after 5 minutes of inactivity)
ClientAliveInterval 300
ClientAliveCountMax 0
```

### Setting up key-based authentication

```bash
# Generate key pair on Kali (client)
ssh-keygen -t ed25519 -C "lab-key"

# Copy public key to target
ssh-copy-id -i ~/.ssh/id_ed25519.pub labuser@10.0.2.3

# Test login
ssh -i ~/.ssh/id_ed25519 labuser@10.0.2.3
```

After confirming key-based login works, disable password authentication in `sshd_config` and restart SSH:

```bash
sudo systemctl restart sshd
```

Now even if someone knows the password, they can't SSH in without the private key.

---

## Audit logging

### Setting up auditd

`auditd` tracks system calls and file access. Useful for catching unauthorized activity after the fact.

```bash
sudo apt install auditd
sudo systemctl enable auditd
sudo systemctl start auditd
```

### Adding audit rules

```bash
# Watch /etc/passwd for changes (user account modifications)
sudo auditctl -w /etc/passwd -p wa -k passwd_changes

# Watch /etc/shadow for reads (potential credential harvesting)
sudo auditctl -w /etc/shadow -p r -k shadow_reads

# Watch SSH config for changes
sudo auditctl -w /etc/ssh/sshd_config -p wa -k ssh_config

# Watch for privilege escalation attempts
sudo auditctl -a always,exit -F arch=b64 -S execve -F euid=0 -F auid!=0 -k priv_esc
```

### Checking audit logs

```bash
# Search for password file changes
sudo ausearch -k passwd_changes

# Search for privilege escalation
sudo ausearch -k priv_esc

# Generate a report
sudo aureport --summary
```

These logs can be forwarded to Wazuh for centralized monitoring - covered in my [SIEM writeup](../05-siem-monitoring/wazuh-siem-setup.md).

---

## Testing: what changes with hardening?

| Scenario | Before hardening | After hardening |
|---|---|---|
| Attacker gets reverse shell | Root access immediately | Low-privilege user, no sudo |
| Attacker tries SSH brute force | Password auth, root login allowed | Key-only auth, root disabled |
| Attacker modifies /etc/passwd | No monitoring | auditd alert generated |
| Attacker reads /etc/shadow | World-readable (on Metasploitable) | Root and shadow group only |

---

## Takeaway

Access controls won't prevent initial exploitation - if there's a vulnerable service, an attacker can still exploit it. But they limit what happens after. The difference between getting root immediately and getting a restricted user account is massive. With proper permissions, no root SSH, and audit logging, an attacker has to work a lot harder to escalate privileges and stay hidden.

The audit logs are especially useful when paired with SIEM. Instead of checking logs manually on each machine, Wazuh aggregates them and alerts on suspicious patterns across the whole network.
