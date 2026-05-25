# CPTS Notes

A clean, sectioned reference for the **HTB Certified Penetration Testing Specialist (CPTS)** path, distilled from course material (CIS4201) and lab notes.

Each stage folder holds a set of `.rmd` files covering one focused topic; sections are ordered to mirror an engagement from the first packet to the final report.

---

## Master Directory

| # | Stage | What lives here |
|---|-------|-----------------|
| 1 | [Information Gathering](./1-information-gathering/) | OSINT, DNS, Nmap, service & web recon, AD enumeration |
| 2 | [Pre-Exploitation](./2-pre-exploitation/) | Vulnerability assessment concepts, web proxies, password / wordlist prep |
| 3 | [Exploitation](./3-exploitation/) | Shells, payloads, Metasploit, file transfers, web exploits, AD initial access |
| 4 | [Post-Exploitation](./4-post-exploitation/) | Linux & Windows privesc, credential dumping, persistence, shell upgrades |
| 5 | [Lateral Movement](./5-lateral-movement/) | AD overview, Kerberos attacks, ACL abuse, Windows pivoting |
| 6 | [Report Writing](./6-report-writing/) | Report structure, engagement types, deliverables & remediation |

---

## Full Section Index

### 1. Information Gathering
- [01 - OSINT](./1-information-gathering/01-osint.rmd)
- [02 - DNS Reconnaissance](./1-information-gathering/02-dns-recon.rmd)
- [03 - Nmap](./1-information-gathering/03-nmap.rmd)
- [04 - Service Enumeration](./1-information-gathering/04-service-enumeration.rmd)
- [05 - Web Reconnaissance](./1-information-gathering/05-web-recon.rmd)
- [06 - Active Directory Enumeration](./1-information-gathering/06-active-directory-enumeration.rmd)

### 2. Pre-Exploitation
- [01 - Vulnerability Assessment](./2-pre-exploitation/01-vulnerability-assessment.rmd)
- [02 - Web Proxies](./2-pre-exploitation/02-web-proxies.rmd)
- [03 - Password Attacks](./2-pre-exploitation/03-password-attacks.rmd)
- [04 - Wordlist Generation](./2-pre-exploitation/04-wordlist-generation.rmd)

### 3. Exploitation
- [01 - Shells and Payloads](./3-exploitation/01-shells-and-payloads.rmd)
- [02 - Metasploit](./3-exploitation/02-metasploit.rmd)
- [03 - File Transfers](./3-exploitation/03-file-transfers.rmd)
- [04 - Web Exploits](./3-exploitation/04-web-exploits.rmd)
- [05 - AD Initial Access](./3-exploitation/05-ad-initial-access.rmd)
- [06 - Container Escape](./3-exploitation/06-container-escape.rmd)

### 4. Post-Exploitation
- [01 - Linux Privilege Escalation](./4-post-exploitation/01-linux-privilege-escalation.rmd)
- [02 - Windows Privilege Escalation](./4-post-exploitation/02-windows-privilege-escalation.rmd)
- [03 - Credential Dumping](./4-post-exploitation/03-credential-dumping.rmd)
- [04 - Persistence](./4-post-exploitation/04-persistence.rmd)
- [05 - Shell Upgrades](./4-post-exploitation/05-shell-upgrades.rmd)

### 5. Lateral Movement
- [01 - Active Directory Overview](./5-lateral-movement/01-active-directory-overview.rmd)
- [02 - AD Protocols](./5-lateral-movement/02-ad-protocols.rmd)
- [03 - Kerberos Attacks](./5-lateral-movement/03-kerberos-attacks.rmd)
- [04 - ACL Abuse with bloodyAD](./5-lateral-movement/04-acl-abuse-bloodyad.rmd)
- [05 - Windows Pivoting](./5-lateral-movement/05-windows-pivoting.rmd)

### 6. Report Writing
- [01 - Report Fundamentals](./6-report-writing/01-report-fundamentals.rmd)
- [02 - Engagement Types & Compliance](./6-report-writing/02-engagement-types.rmd)
- [03 - Deliverables and Remediation](./6-report-writing/03-deliverables-remediation.rmd)

---

## Conventions

- Files use the `.rmd` extension so the same source can be rendered as Markdown on GitHub *or* knit through `rmarkdown` for a PDF / book build.
- Each file opens with a YAML header (`title`, `stage`, `tags`), followed by a short summary, then practical content organized by tool or technique.
- Code blocks are fenced with `bash`, `powershell`, `php`, `sql`, etc. so syntax highlighting works on GitHub.
- Placeholders use `<angle-brackets>` (e.g. `<dc-ip>`, `<domain>`, `<user>`) — replace with engagement values.

## Layout

```
notes_refurb/
├── README.md                       ← this file (master directory)
├── 1-information-gathering/
│   ├── README.md                   ← stage index
│   ├── 01-osint.rmd
│   ├── 02-dns-recon.rmd
│   └── ...
├── 2-pre-exploitation/
├── 3-exploitation/
├── 4-post-exploitation/
├── 5-lateral-movement/
├── 6-report-writing/
└── _archive/                       ← original PDF sources
```

## Acknowledgements

Structural inspiration from [sachinn403/HTB-CPTS-Notes](https://github.com/sachinn403/HTB-CPTS-Notes). Source content distilled from personal CIS4201 course notes and HTB CPTS path lab work.
