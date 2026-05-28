# Lessons Learned

Honest notes from running this lab — what went wrong, what I fixed, and what I'd tell myself before starting.

---

## Kali Linux — Don't Full Upgrade

The biggest mistake early on was running `sudo apt full-upgrade` thinking it was just a normal update. It broke tools, messed up dependencies, and ended with a full reformat.

**What I learned:**

- Use `sudo apt update && sudo apt upgrade` — not `full-upgrade`
- Update weekly or bi-weekly, not constantly
- If a tool is working and you're mid-lab, don't update — updates can break things mid-session
- If it's not broken, leave it alone
- Snapshots before any system update, always

---

## VM Snapshots — Use Them Before Anything

Learned this after corrupting a VM state and having to rebuild from scratch. Now I take a snapshot before:

- Major updates
- Any exploit that touches system files
- Changing network configs

One snapshot saves hours of rebuilding.

---

## Storage — Allocate Properly From the Start

Undersized a VM disk early on and ran out of space mid-lab. Resizing after the fact is a pain.

- Allocate enough disk space when setting up the VM — don't lowball it
- Use dynamically allocated storage if you're tight on host space, but set a reasonable max

---

## General

- Read what a command does before running it, especially in Metasploit
- Nmap version detection isn't always accurate — verify with auxiliary scanners when it matters
- Root access on a box doesn't mean you understand it — document what you did and why it worked
