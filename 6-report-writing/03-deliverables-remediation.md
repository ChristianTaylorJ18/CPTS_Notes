## Deliverables and Remediation

A report isn't finished when the PDF is signed. The full deliverable set drives the client through *fix → verify → close*, and that's what generates the next engagement.

---

### Standard Deliverables

| Artifact | Purpose | Audience |
|----------|---------|----------|
| **Report (PDF)** | Primary deliverable. | Both |
| **Executive briefing slides** | 10-15 slides; talked through with leadership. | Execs |
| **Findings spreadsheet (XLSX)** | One row per finding, sortable / filterable / importable into ticketing. | Engineers |
| **Raw evidence pack** | Screenshots, command outputs, capture files. Often a zip. | Engineers / auditors |
| **Scanner output** | Nessus / OpenVAS / Burp scans in native format. | Engineers |
| **Retest report** | Delta report after remediation. | Both |

### Risk Scoring

#### CVSS v3.1 (the current default)

Score = base × temporal × environmental modifiers.

Severity bands:

| Range | Severity |
|-------|----------|
| 9.0 – 10.0 | Critical |
| 7.0 – 8.9 | High |
| 4.0 – 6.9 | Medium |
| 0.1 – 3.9 | Low |
| 0.0 | Informational |

Always include the **CVSS vector string** alongside the number — others should be able to recompute.

```bash
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H   = 9.8 Critical
```

CVSS v4.0 is gaining traction; if the client requests it, use the matching calculator. Pure CVSS misses business context, so almost every mature program augments with their own **Likelihood × Impact** matrix.

#### Likelihood × Impact (the contextual fix)

| Likelihood | Impact | Risk |
|-----------|--------|------|
| Low | Low | Low |
| Low | Medium | Low |
| Low | High | Medium |
| Medium | Low | Low |
| Medium | Medium | Medium |
| Medium | High | High |
| High | Low | Medium |
| High | Medium | High |
| High | High | Critical |

Likelihood factors: required privilege, exploit availability, accessibility, detection presence.
Impact factors: data sensitivity, blast radius, regulatory exposure, recovery cost.

### Writing Remediation Guidance

Bad: *"Apply patches."*
Better: *"Apply Microsoft security update KB5005565 (released Sept 2021) to all Server 2016 Domain Controllers to address CVE-2021-36934."*
Best: *"Apply KB5005565 to all 2016 DCs by 2026-06-30; in the interim, restrict access to volume shadow copy backups by removing READ for `BUILTIN\Users` on `C:\Windows\System32\config`."*

A good remediation block answers:

1. **What to change** (specific config / patch / code).
2. **Where to change it** (file path, console, GPO).
3. **How to verify** (the command or check that proves the fix).
4. **By when** (deadline tied to severity SLAs).
5. **Compensating controls** if a fast fix isn't possible.

### Evidence

For each finding:

- **Timestamped** screenshot (full window, no cropping that hides context).
- The exact command(s) run, copy-pastable.
- Output excerpts long enough to be conclusive but redacted of unrelated noise.
- The IP/hostname of every system involved.

Store the evidence pack alongside the report — auditors and retest teams need it.

### Retest Process

1. Client signals "ready for retest" on specific findings.
2. Test only those findings (re-running adjacent tests is scope creep unless paid for).
3. Outcomes:
   - **Closed** — issue no longer reproducible.
   - **Partially mitigated** — risk reduced but exploit still possible.
   - **Open** — fix didn't take.
4. Issue a short retest report — usually 2–5 pages.

### SLAs to Suggest

Default starter SLAs (client can negotiate):

| Severity | Triage | Fix |
|----------|--------|-----|
| Critical | 24h | 7 days |
| High | 72h | 30 days |
| Medium | 7 days | 90 days |
| Low | 30 days | 180 days |
| Informational | — | next planning cycle |

### Presentation / Readout

After delivery, run a 30–60 min walkthrough:
- Slides 1–3: executive posture & top risks.
- Slides 4–8: attack narrative (the chain).
- Slides 9–12: remediation roadmap.
- Q&A.

Goals: shared understanding, clear ownership of each remediation, agreement on the retest window. Send the slides to attendees afterward — they'll be circulated up the chain.

### Common Pitfalls

| Pitfall | Cost |
|---------|------|
| Findings without reproduction steps | Engineers ignore them; "we can't verify" → no fix |
| Vague severity ("seems bad") | Triage process can't prioritize |
| No evidence of impact | Easy for stakeholders to dismiss |
| Mixed exec/engineer language | Both audiences disengage |
| Big screenshots, tiny font | Print versions illegible — use real text where you can |
| No retest plan | Findings linger for years |

### Tooling

| Tool | Use |
|------|-----|
| **SysReptor** | Open-source pentest reporting platform. |
| **Dradis** | Findings DB + report generator. |
| **PwnDoc** | Open-source pentest report tool. |
| **Markdown + Pandoc** | DIY; convert your `.md` notes into a polished PDF. |
| **LaTeX template (Eisvogel etc.)** | High-quality PDF rendering for Markdown sources. |

For a Markdown-based workflow:

```bash
pandoc report.md -o report.pdf --template=eisvogel --toc
```
