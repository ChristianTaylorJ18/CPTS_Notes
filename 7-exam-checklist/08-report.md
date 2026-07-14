# 08 — Report

Trigger: you have flags/root/DA — start assembling the report **while there's still exam time on the clock**.
Reference: [Report Fundamentals](../6-report-writing/01-report-fundamentals.md) · [Engagement Types](../6-report-writing/02-engagement-types.md) · [Deliverables & Remediation](../6-report-writing/03-deliverables-remediation.md).

## Timing rule

- [ ] Reserve **at least the final 24 hours** of the 10-day exam for the report, even if you're mid-attack. Assemble as you go — don't leave it all to the end.
- [ ] The exam is passed on the *report*, not on shell count. Bad screenshots → failed pass.

## Structure your report will need

- [ ] **Executive Summary** — 1 page, plain-English, business impact. No jargon.
- [ ] **Scope & Rules of Engagement** — quoted directly from the exam brief.
- [ ] **Methodology** — brief walkthrough of the phases you followed (mirrors sections 1-5 of these notes).
- [ ] **Findings** — one per vulnerability, structured (see below).
- [ ] **Attack Chain / Storyline** — how findings chained to compromise.
- [ ] **Remediation** — one action per finding.
- [ ] **Appendix** — full command output, hash dumps, screenshots.

## Per-finding template (use for every vuln, no exceptions)

- [ ] **Title** — plain English, not the CVE ID.
- [ ] **Severity** — CVSS 3.1 base score + rating (Critical/High/Medium/Low).
- [ ] **Affected asset(s)** — hostname + IP + URL/port.
- [ ] **Description** — what the vulnerability is, in one paragraph.
- [ ] **Impact** — what an attacker can do with it, in business terms.
- [ ] **Steps to reproduce** — numbered, with the exact commands used.
- [ ] **Evidence** — screenshot(s), with red boxes around the important text.
- [ ] **Remediation** — what to change, with vendor doc links where possible.
- [ ] **References** — CVE, CWE, vendor advisory, OWASP.

## Screenshot audit (do this first — it's the most-failed step)

For every flag you submitted:

- [ ] Screenshot of the exploit command executing.
- [ ] Screenshot of the shell / access gained.
- [ ] Screenshot of `whoami && hostname && ip a` proving where you are.
- [ ] Screenshot of `cat flag.txt` / `type flag.txt` with the flag visible.
- [ ] Every screenshot cropped, with an obvious red-box on the payload / output.
- [ ] File-name convention: `<host>_<finding>_<step>.png` — future-you writing at 3am will thank you.

## Findings you should always include (assuming you found them)

- [ ] **Missing patches** — every unpatched CVE you exploited, with the KB number.
- [ ] **Weak passwords** — with the cracked hashes as evidence (redact except last 4 chars in the exec summary, full in appendix).
- [ ] **Password reuse** — same hash appearing on multiple accounts / hosts.
- [ ] **SMB signing off / SMBv1 enabled** — even if you didn't exploit it.
- [ ] **Kerberoastable / AS-REP-roastable accounts** — always include, even without a crack.
- [ ] **GPP `cpassword`** — if found, this is a Critical.
- [ ] **Excessive share permissions** — Domain Users with write to sensitive shares.
- [ ] **Cleartext creds in scripts / configs** — with file paths, redacted values.
- [ ] **Local admin sprawl** — one local admin password across all workstations.

## Remediation quality (this separates a pass from a fail)

- [ ] Every remediation is **specific and actionable**. Not "harden the server" — "disable SMBv1 via `Set-SmbServerConfiguration -EnableSMB1Protocol $false`, then reboot".
- [ ] Every remediation links to vendor documentation.
- [ ] Group findings by root cause where it makes sense (many findings often share one fix).
- [ ] Include a **strategic** section — patch management, tiering, LAPS, PAWs, EDR gaps.

## Executive summary rules

- [ ] Under one page.
- [ ] Zero jargon. Assume the reader is a CFO.
- [ ] Business-impact language: "an unauthenticated attacker could impersonate any employee, including executives, and access all customer records."
- [ ] Two graphics max — a severity pie chart, and the attack chain diagram.

## Final pass (30 minutes before submission)

- [ ] Spell check.
- [ ] Every screenshot legible (zoom to 100% and check).
- [ ] Table of contents regenerated, page numbers correct.
- [ ] All hostnames/IPs match reality (search-and-replace any test values).
- [ ] Client name is correct everywhere (this catches everyone at least once).
- [ ] File exported as PDF, opened, re-verified.
- [ ] Submitted with 15 minutes to spare.
