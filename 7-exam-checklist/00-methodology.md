# 00 — Methodology & Setup

## Before the exam (day-of prep)

- [ ] Kali is snapshotted and up to date (`apt update && apt full-upgrade`).
- [ ] `~/.zshrc` aliases are loaded for `nxc`, `evil-winrm`, `impacket-*`.
- [ ] `/etc/hosts` file cleared of stale lab entries.
- [ ] Screenshot tool bound to a hotkey (Flameshot: `PrtSc`).
- [ ] Working directory scaffolded — one folder per host:

```bash
mkdir -p engagement/{recon,creds,loot,screenshots,report}
cd engagement
touch hosts.txt subdomains.txt urls.txt creds.txt users.txt passwords.txt
```

- [ ] Report template opened in a second monitor. See [Report Writing](../6-report-writing/).
- [ ] VPN pack downloaded, tested with `ping <gateway>`.
- [ ] Coffee / water / snacks within arm's reach — 10 days is a marathon, not a sprint.

## First 30 minutes of the exam

- [ ] Read the letter of engagement **twice**. Note: scope, restrictions (no DoS, no phishing), objectives, flag format.
- [ ] **Start `script -f` in the first terminal** — only ever runs one shell, so use it for the terminal where your primary attack chain runs. Every other terminal opens with its own `script -f` block:

  ```bash
  mkdir -p ~/AEN/logs
  script -f ~/AEN/logs/aen-$(date +%F-%H%M).log
  # ... work ...
  exit   # stops recording
  ```

- [ ] `nmap -sn <scope>` — enumerate live hosts, dump to `hosts.txt`.
- [ ] Kick off full-port scans on every live host in parallel (see [01-recon.md](./01-recon.md)).
- [ ] While scans run, re-read the objectives and outline the required findings.

## AEN-style day-one enum (the operational drill)

Distilled from the *Lessons from AEN* engagement. Do these in order the moment you have an initial host + domain name (typical case: `inlanefreight.local` + one IP).

- [ ] **AXFR first** — if any host runs DNS, try zone transfer before you fuzz. Zone transfer gives you every vhost for free:

  ```bash
  dig axfr inlanefreight.local @<dns-ip>
  ```

- [ ] **Then vhost fuzz to catch what AXFR missed** — long-running, dump it in the background. Baseline the fail-response length first so `-fs` filters cleanly:

  ```bash
  ## Length of a known-invalid vhost response
  curl -s -I http://inlanefreight.local -H "HOST: defnotvalid.inlanefreight.local" | grep "Content-Length:"

  ## FFUF with that length filtered out
  ffuf -u http://inlanefreight.local/ \
       -w /usr/share/wordlists/seclists/Discovery/DNS/namelist.txt:FUZZ \
       -H 'Host:FUZZ.inlanefreight.local' \
       -fs 15157 -t 100
  ```

- [ ] **On every login page with no other entry**, spray `admin` / `root` / product-default usernames against a small guessable password list before touching anything else. Cheap, high hit rate.
- [ ] **On every web app**, register any account you can. Registration typically 3× the accessible attack surface.
- [ ] **Test every input for XSS** — including obscure ones. Tracking numbers, search boxes, contact-form message fields. A tracking-number input that server-side generated PDFs (with no input validation) let this XSS payload read `/etc/passwd`:

  ```html
  <script>
    x = new XMLHttpRequest();
    x.onload = function(){ document.write(this.responseText) };
    x.open("GET","file:///etc/passwd");
    x.send();
  </script>
  ```

- [ ] **Every 403 during fuzzing gets verb-tampered before you skip it.** `server-status` is the classic — try `GET`, `POST`, `PUT`, `DELETE`, `TRACE`. See [Web Exploits](../3-exploitation/04-web-exploits.md).
- [ ] **Uploads directory found?** Track where uploads land: capture the request to `upload.php` in Burp → change the verb to one that echoes (`TRACE` is ideal) → add `X-Custom-IP-Authorization: 127.0.0.1` → send. When the response comes back with the path, request that URL in your browser to confirm the delivery location.
- [ ] **For every command executed on a website** (even ones behind a restricted shell), watch it in Burp — the underlying request is often more forgiving than the UI suggests.
- [ ] **Contact-us / message fields**: XSS them targeting an admin-viewer to steal the admin cookie.
- [ ] **Every visible DB → try to dump it** (SQLi + SQLMap → see [SQL Injection](../3-exploitation/04-web-exploits.md)).

The goal of this pass is one specific outcome: **find a web-facing machine whose `ipconfig` shows an internal-only subnet**. Once you find it, get a shell → SSH creds for persistence → escalate to root → set up the pivot. That's the pattern that unlocks the rest of the environment.

## When you're stuck (the 45-minute rule)

If you've spent 45 minutes on a single vector with zero progress:

- [ ] Screenshot what you've tried, note it in your report notes, **move on**.
- [ ] Return to the recon output for the current host — is there a port you haven't touched?
- [ ] Any new creds since you started? Try them everywhere (`nxc smb <all-hosts> -u creds -p creds`).
- [ ] Any hash you never cracked? Restart hashcat with a bigger ruleset.
- [ ] Try a different host — momentum on one target beats grinding on another.

## Cred hygiene (this saves you at flag 8)

- [ ] Every credential you find goes in `creds.txt` immediately.
- [ ] Format: `username:password` — one per line.
- [ ] Also append the username to `users.txt` and the password to `passwords.txt`.
- [ ] After every new cred: **spray across all hosts** — the box you're stuck on may take a cred from a box you already own.

```bash
nxc smb hosts.txt -u users.txt -p passwords.txt --continue-on-success
nxc winrm hosts.txt -u users.txt -p passwords.txt --continue-on-success
nxc ssh hosts.txt -u users.txt -p passwords.txt --continue-on-success
```

## Screenshot rules

- [ ] Every flag → screenshot with the command that produced it visible.
- [ ] Every shell → screenshot `whoami && hostname && ip a` in the shell.
- [ ] Every privesc → screenshot before + after (`id` as low user, then `id` as root/SYSTEM).
- [ ] Every exploit → screenshot the vulnerable request/response side-by-side in Burp.
- [ ] File naming convention: `<host>_<technique>_<step>.png`. Future-you will thank you.

## Flag submission

- [ ] Submit flags **immediately** as you find them — don't save them for a batch.
- [ ] Verify the flag format matches the exam brief before submitting.
- [ ] Keep a running scoreboard in `engagement/flags.md`:
  ```
  | Host           | Flag                                      | Time  |
  |----------------|-------------------------------------------|-------|
  | 10.10.10.5     | HTB{redacted}                             | 09:14 |
  ```

## Sleep & pacing (10 days is long — treat it like one)

- [ ] Sleep a full night every night. You will make critical errors on 4 hours of sleep and lose more time debugging bad decisions than you saved by staying up.
- [ ] Pick working hours and stick to them — the exam is a marathon, not a sprint. Burning out on day 3 leaves 7 days of degraded work ahead.
- [ ] Log off cleanly at end of day: commit notes, screenshots filed, `creds.txt` up to date. Coming back cold to messy state kills momentum.
- [ ] Take at least one full rest day if the schedule allows — a fresh look on day 6 catches things day-3 you missed.
- [ ] Reserve **at least the final 24 hours** for the report. Set a hard-stop alarm 30 minutes before the exam ends to submit final flags and lock the report.
