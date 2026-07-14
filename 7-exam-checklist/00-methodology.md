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
- [ ] Coffee / water / snacks within arm's reach — 35 hours is long.

## First 30 minutes of the exam

- [ ] Read the letter of engagement **twice**. Note: scope, restrictions (no DoS, no phishing), objectives, flag format.
- [ ] `nmap -sn <scope>` — enumerate live hosts, dump to `hosts.txt`.
- [ ] Kick off full-port scans on every live host in parallel (see [01-recon.md](./01-recon.md)).
- [ ] While scans run, re-read the objectives and outline the required findings.

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

## Sleep

- [ ] Non-negotiable — sleep at least 6 hours in the middle. You will make critical errors after hour 20.
- [ ] Set an alarm for 30 minutes before the exam ends to guarantee time to submit final flags and start the report.
