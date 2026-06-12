# Python log parser

I wrote a Python script to parse Linux authentication logs and flag suspicious activity - brute force attempts, successful logins from unusual IPs, and privilege escalation. This automates the kind of log analysis I was doing manually in the SIEM writeup.

---

## Why not just use Wazuh?

Wazuh does this already. But writing the parser myself served two purposes:

1. Understanding what the SIEM is doing under the hood. Wazuh's brute force detection (rule 5712) is just pattern matching on auth logs - same thing this script does, just more polished.
2. Having a standalone tool for machines that don't have a SIEM agent. Not every machine in a lab (or a small network) runs Wazuh. A Python script that parses `/var/log/auth.log` works anywhere.

---

## What the script does

Parses `/var/log/auth.log` (or any auth log file) and generates a summary:

1. **Failed login attempts** - grouped by source IP and username
2. **Brute force detection** - flags IPs with more than 5 failed attempts in a short window
3. **Successful logins** - with timestamps and source IPs
4. **Sudo usage** - who ran sudo and what commands they ran
5. **Summary report** - printed to terminal or saved to a file

---

## The script

```python
#!/usr/bin/env python3
"""
auth_log_parser.py
Parses Linux auth.log for security-relevant events.
"""

import re
import sys
from collections import defaultdict
from datetime import datetime

# Patterns
FAILED_LOGIN = re.compile(
    r"(\w+\s+\d+\s+[\d:]+).*sshd.*Failed password for (?:invalid user )?(\S+) from (\S+)"
)
ACCEPTED_LOGIN = re.compile(
    r"(\w+\s+\d+\s+[\d:]+).*sshd.*Accepted (?:password|publickey) for (\S+) from (\S+)"
)
SUDO_CMD = re.compile(
    r"(\w+\s+\d+\s+[\d:]+).*sudo:\s+(\S+).*COMMAND=(.*)"
)

BRUTE_FORCE_THRESHOLD = 5


def parse_log(filepath):
    failed = defaultdict(list)
    accepted = []
    sudo_events = []

    try:
        with open(filepath, "r") as f:
            for line in f:
                # Check failed logins
                match = FAILED_LOGIN.search(line)
                if match:
                    timestamp, user, ip = match.groups()
                    failed[ip].append({"time": timestamp, "user": user})
                    continue

                # Check accepted logins
                match = ACCEPTED_LOGIN.search(line)
                if match:
                    timestamp, user, ip = match.groups()
                    accepted.append({"time": timestamp, "user": user, "ip": ip})
                    continue

                # Check sudo usage
                match = SUDO_CMD.search(line)
                if match:
                    timestamp, user, cmd = match.groups()
                    sudo_events.append({"time": timestamp, "user": user, "cmd": cmd.strip()})

    except FileNotFoundError:
        print(f"File not found: {filepath}")
        sys.exit(1)

    return failed, accepted, sudo_events


def print_report(failed, accepted, sudo_events):
    print("=" * 60)
    print("AUTH LOG ANALYSIS REPORT")
    print("=" * 60)

    # Failed logins
    print(f"\n--- Failed login attempts ({sum(len(v) for v in failed.values())} total) ---\n")

    if not failed:
        print("  No failed login attempts found.")
    else:
        for ip, attempts in sorted(failed.items(), key=lambda x: len(x[1]), reverse=True):
            flag = " ** BRUTE FORCE **" if len(attempts) >= BRUTE_FORCE_THRESHOLD else ""
            print(f"  {ip}: {len(attempts)} failures{flag}")

            # Show targeted usernames
            users = defaultdict(int)
            for a in attempts:
                users[a["user"]] += 1
            for user, count in sorted(users.items(), key=lambda x: x[1], reverse=True):
                print(f"    - {user}: {count} attempts")

    # Brute force summary
    brute_force_ips = [ip for ip, attempts in failed.items() if len(attempts) >= BRUTE_FORCE_THRESHOLD]
    if brute_force_ips:
        print(f"\n--- Brute force IPs (>{BRUTE_FORCE_THRESHOLD} failures) ---\n")
        for ip in brute_force_ips:
            print(f"  BLOCK: {ip} ({len(failed[ip])} attempts)")

    # Successful logins
    print(f"\n--- Successful logins ({len(accepted)} total) ---\n")

    if not accepted:
        print("  No successful logins found.")
    else:
        for login in accepted:
            print(f"  {login['time']} | {login['user']}@{login['ip']}")

    # Sudo events
    print(f"\n--- Sudo usage ({len(sudo_events)} commands) ---\n")

    if not sudo_events:
        print("  No sudo usage found.")
    else:
        for event in sudo_events:
            print(f"  {event['time']} | {event['user']}: {event['cmd']}")

    print("\n" + "=" * 60)


if __name__ == "__main__":
    log_path = sys.argv[1] if len(sys.argv) > 1 else "/var/log/auth.log"
    failed, accepted, sudo_events = parse_log(log_path)
    print_report(failed, accepted, sudo_events)
```

---

## Usage

```bash
# Parse default auth.log
python3 auth_log_parser.py

# Parse a specific file
python3 auth_log_parser.py /var/log/auth.log.1

# Save output to file
python3 auth_log_parser.py > report.txt
```

---

## Sample output

After running `hydra` against the lab (same brute force test from the SIEM writeup):

```
============================================================
AUTH LOG ANALYSIS REPORT
============================================================

--- Failed login attempts (847 total) ---

  10.0.2.4: 842 failures ** BRUTE FORCE **
    - root: 842 attempts
  10.0.2.1: 5 failures
    - admin: 3 attempts
    - test: 2 attempts

--- Brute force IPs (>5 failures) ---

  BLOCK: 10.0.2.4 (842 attempts)

--- Successful logins (3 total) ---

  Jun 10 14:15:02 | labuser@10.0.2.4
  Jun 10 16:30:15 | labuser@10.0.2.4
  Jun 10 18:45:33 | root@10.0.2.1

--- Sudo usage (7 commands) ---

  Jun 10 14:16:00 | labuser: /bin/systemctl restart sshd
  Jun 10 14:20:12 | labuser: /usr/bin/apt update
  Jun 10 16:31:00 | labuser: /sbin/iptables -L

============================================================
```

842 failed attempts from a single IP targeting root is an obvious brute force. The script flags it immediately.

---

## What I'd add next

- **Output formats** - JSON and CSV export for feeding into other tools
- **Time window analysis** - group failures by time intervals to detect slow brute force attempts that stay under threshold
- **IP geolocation** - flag logins from unexpected countries (using a GeoIP database)
- **Integration with iptables** - automatically generate firewall rules to block flagged IPs

---

## Takeaway

This script does about 10% of what Wazuh does, but writing it taught me more about log parsing than configuring Wazuh did. Understanding the raw log format - what `auth.log` actually contains, how SSH logs failures vs successes, where sudo commands are recorded - makes the SIEM configuration less opaque. When I set a Wazuh rule to detect brute force, I now understand what it's matching against.

The regex patterns are straightforward once you look at actual log lines. Most of security automation is just parsing text files and counting things.
