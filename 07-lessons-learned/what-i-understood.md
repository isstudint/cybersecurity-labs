# Lessons learned

Notes from running this lab - what went wrong, what I fixed, and what I'd tell myself before starting.

---

## Kali Linux - don't full-upgrade

The biggest mistake early on was running `sudo apt full-upgrade` thinking it was just a normal update. It broke tools, messed up dependencies, and ended with a full reformat.

What I learned:

- Use `sudo apt update && sudo apt upgrade`, not `full-upgrade`
- Update weekly or bi-weekly, not constantly
- If a tool is working and you're mid-lab, don't update. Updates can break things mid-session.
- If it's not broken, leave it alone
- Snapshots before any system update, always

---

## VM snapshots - use them before anything

Learned this after corrupting a VM state and having to rebuild from scratch. Now I take a snapshot before:

- Major updates
- Any exploit that touches system files
- Changing network configs
- Installing new tools (Snort, Wazuh, etc.)

One snapshot saves hours of rebuilding.

---

## Storage - allocate properly from the start

Undersized a VM disk early on and ran out of space mid-lab. Resizing after the fact is a pain.

- Allocate enough disk space when setting up the VM - don't lowball it
- Use dynamically allocated storage if you're tight on host space, but set a reasonable max
- Wazuh in particular uses more disk than expected because of log indexing

---

## Snort rules - test before trusting

My first custom Snort rules were too broad. The SYN scan detection rule fired on normal traffic because the threshold was too low. Had to tune it from 10 SYN packets in 5 seconds to 20 in 3 seconds to reduce false positives.

Writing rules for attacks you've performed yourself makes tuning easier - you know exactly what the traffic looks like.

---

## Wazuh - check the logs, not just the dashboard

The dashboard is convenient but it's a summary. When investigating the simulated incident, I found more detail in the raw logs (`/var/ossec/logs/alerts/alerts.json`) than the dashboard showed. The dashboard aggregates and sometimes hides individual events that matter.

---

## Offense informs defense

This is the biggest takeaway from the whole lab. Writing Snort rules was straightforward because I'd already analyzed the attack traffic in Wireshark. Configuring the firewall was easy because I knew which ports the exploits used. Investigating in the SIEM made sense because I knew what I was looking for.

Running attacks first, then defending against them, is a better learning order than trying to understand defense in the abstract.

---

## General

- Read what a command does before running it, especially in Metasploit
- Nmap version detection isn't always accurate - verify with auxiliary scanners when it matters
- Root access on a box doesn't mean you understand it - document what you did and why it worked
- Default ports for tools (Meterpreter 4444, etc.) are easy to detect. Real attackers change them. Keep that in mind when writing IDS rules.
- Automated scanners (OpenVAS) find things you'd miss manually, but manual testing gives you depth. Use both.
