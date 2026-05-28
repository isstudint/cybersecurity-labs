## OSINT

Tools used to practice passive reconnaissance in a homelab environment. 
The goal was to understand how publicly available information can expose 
a target's digital footprint.

**Tools:** `Whois` `NSLookup` `theHarvester` `RedHawk` `Sherlock`

---

### What I Did

**DNS & Infrastructure Mapping**  
Used Whois and NSLookup to pull domain registration data and DNS records 
(A, MX, TXT). Learned how these records can reveal mail servers and 
hosting providers.

**Passive Email & Subdomain Harvesting**  
Ran theHarvester to understand how search engines can surface subdomains 
and corporate email formats without touching the target directly.

**Username Enumeration**  
Used Sherlock to trace a username across platforms. Learned how a single 
identity leak can let attackers build a broader profile of a target.