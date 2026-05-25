## Service Enumeration

Per-service playbooks for every common port Nmap is likely to find. Each block follows the same shape: where the config lives, dangerous settings to look for, and the tools / commands to interact with it.

---

### FTP (21)

```bash
ftp anonymous@<ip>              # try anonymous first
ftp user@<ip>                   # or named user
nc -nv <ip> 21                  # raw banner
telnet <ip> 21                  # alternative banner check
openssl s_client -connect <ip>:21 -starttls ftp   # if TLS/SSL wrapped
wget -m --no-passive ftp://anonymous:anonymous@<ip>   # mirror all files
```

vsFTPd config (`/etc/vsftpd.conf`) — danger flags:

| Setting | Why it matters |
|---------|----------------|
| `anonymous_enable=YES` | Anonymous login |
| `anon_upload_enable=YES` | Anonymous can upload |
| `anon_mkdir_write_enable=YES` | Anonymous can create dirs |
| `no_anon_password=YES` | No anon password prompt |
| `anon_root=/path` | Where anon lands |
| `write_enable=YES` | STOR/DELE/RNFR/RNTO/MKD/RMD/APPE/SITE allowed |
| `chown_uploads=YES` + `chown_username=user` | Uploaded files re-owned |
| `chroot_local_user=YES` | Local users chrooted to home |
| `ls_recurse_enable=YES` | Recursive listings allowed |

Blacklist file: `/etc/ftpusers`.

### SMB (137-139, 445)

Config: `/etc/samba/smb.conf` — danger flags:

| Setting | Why |
|---------|-----|
| `browseable = yes` | Shares listed |
| `read only = no` / `writable = yes` | Writeable shares |
| `guest ok = yes` | No password needed |
| `enable privileges = yes` | Honors SID privs |
| `create mask = 0777` / `directory mask = 0777` | World-writeable files/dirs |
| `logon script = script.sh` | Runs on logon |
| `magic script = script.sh` | Runs when closed |

```bash
## Listing & null sessions
smbclient -L //<ip> -N                              # list shares (null)
smbmap -H <ip>                                       # recon shares
nmap -p445 --script smb-enum-shares <ip>

## Connect
smbclient //<ip>/<share> -N                          # anonymous
smbclient //<ip>/<share> -U 'user%pass'              # creds
smbclient //<ip>/<share> -W <DOMAIN> -U 'user%pass'  # domain-joined

## Mount
sudo mount -t cifs //<ip>/<share> /mnt/smb -o user=<user>
```

#### rpcclient (null sessions)

```bash
rpcclient -U "" <dc-ip> -N
## Once in:
srvinfo
enumdomains
querydominfo
netshareenumall
netsharegetinfo <share>
enumdomusers
queryuser <RID>

## Build users.txt from null session
rpcclient -U "" -N <dc-ip> -c "enumdomusers" \
  | awk -F'[][]' '{print $2}' | cut -d' ' -f1 > users.txt

## If enumdomusers blocked, brute-force RIDs
for i in $(seq 500 2000); do
  echo "queryuser $i" | rpcclient -U "" -N <dc-ip> 2>/dev/null | grep -i "User Name"
done
```

#### netexec (formerly crackmapexec) — SMB

```bash
nxc smb <ip> -u '' -p '' --pass-pol                  # password policy
nxc smb <ip> -u 'user' -p 'pass' --shares            # share perms
nxc smb <ip> -u '' -p '' --rid-brute                 # dump RIDs/users

## Generate users.txt from RID brute
nxc smb <ip> -u <user> -p <pass> --rid-brute \
  | grep 'SidTypeUser' | cut -d '\' -f2 | cut -d '(' -f1 > users.txt

## Test "username == password" before spraying
nxc smb <ip> -d <DOMAIN> -u usernames.txt -p usernames.txt --no-brute --continue-on-success
```

If you land valid creds, kerberoast SPNs immediately:

```bash
impacket-GetUserSPNs <DOMAIN>/<user>:<pass> -dc-ip <dc-ip> -request -outputfile hashes.txt
```

#### enum4linux-ng (SMB + LDAP one-shot)

```bash
enum4linux-ng -A -u 'user' -p 'pass' <ip>
```

### NFS (111, 2049)

Config: `/etc/exports`.

```bash
showmount -e <ip>                                    # available shares
mkdir target-NFS
sudo mount -t nfs <ip>:/ ./target-NFS/ -o nolock
```

⚠ Check `no_root_squash` in `/etc/exports` — see [Linux Privesc → NFS root squashing](../4-post-exploitation/01-linux-privilege-escalation.md).

### DNS (53)

See [DNS Reconnaissance](./02-dns-recon.md).

```bash
dig ns <domain> @<dc-ip>
dig CH TXT version.bind <ip>
dig any <domain> @<dc-ip>
dig axfr <domain> @<dc-ip>                           # zone transfer
```

Server settings worth flagging: `allow-query`, `allow-recursion`, `allow-transfer`.

### SMTP (25)

```bash
sudo nmap <ip> -p25 --script smtp-open-relay -v

## Metasploit user enumeration
## search smtp_enum → use 0 → set RHOSTS <ip> → set USER_FILE <wordlist> → run
```

### IMAP (143/993) and POP3 (110/995)

```bash
openssl s_client -connect <ip>:imaps                 # IMAP over TLS
openssl s_client -connect <ip>:pop3s                 # POP3 over TLS
```

IMAP command form: `a<n> <command>` (where `<n>` increments per command). POP3 commands run plain.

| IMAP | POP3 |
|------|------|
| `LOGIN user pass` | `USER user` / `PASS pass` |
| `LIST "" *` | `STAT` |
| `SELECT INBOX` | `LIST` |
| `FETCH <id> all` | `RETR <id>` |
| `FETCH <id> BODY[TEXT]` | — |
| `CLOSE` | `DELE <id>` |
| `LOGOUT` | `QUIT` |

Dangerous Dovecot settings: `auth_debug`, `auth_debug_passwords`, `auth_verbose`, `auth_verbose_passwords`, `auth_anonymous_username`.

### SNMP (161/162 UDP)

```bash
sudo nmap -sU --top-ports 100 <ip>
snmpwalk -v2c -c public <ip>
snmpwalk -v2c -c public <ip> | grep 'STRING:*'        # the useful filter
onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt <ip>   # brute community strings
braa <community>@<ip>:.1.3.6.*                        # brute OIDs
```

Dangerous settings in `snmpd.conf`:

| Setting | Why |
|---------|-----|
| `rwuser noauth` | Read/write to full OID tree, no auth |
| `rwcommunity <str> <ipv4>` | Same, but by community string |
| `rwcommunity6 <str> <ipv6>` | Same over IPv6 |

### MySQL (3306)

```bash
sudo nmap <ip> -sV -sC -p3306 --script mysql*
mysql -u <user> -p<pass> -h <ip>           # no space after -p

## Inside MySQL
show databases;
use <db>;
show tables;
show columns from <table>;
select * from <table>;
select * from <table> where <col> = "<string>";
```

Dangerous my.cnf settings: `user`, `password`, `admin_address`, `debug`, `sql_warnings`, `secure_file_priv`.

### MSSQL (1433)

```bash
## Auth modes
impacket-mssqlclient user:pass@<ip>
impacket-mssqlclient -windows-auth DOM/user:pass@<ip>
impacket-mssqlclient -hashes [LM:]NT DOM/user@<ip>

## 95% script coverage
sudo nmap --script ms-sql-* \
  --script-args mssql.instance-port=1433,mssql.username=sa,mssql.password=,mssql.instance-name=MSSQLSERVER \
  -sV -p 1433 <ip>
```

Goal: basic enum, then if the SQL server is privileged → run `xp_cmdshell`. See [Windows Privesc → SQL → SeImpersonate](../4-post-exploitation/02-windows-privilege-escalation.md).

Dangerous settings: unencrypted client connections, self-signed certs, named-pipes auth, weak/default `sa` creds.

### TNS — Oracle (1521)

```bash
sudo nmap -p1521 -sV <ip> --open
sudo nmap -p1521 -sV <ip> --open --script oracle-sid-brute
./odat.py all -s <ip>                                # easy-pass / SID enum
```

#### sqlplus

```bash
## Install on Parrot/Kali
sudo apt update && sudo apt upgrade parrot-core
sudo apt install oracle-instantclient-sqlplus
## if libsqlplus missing:
sudo sh -c "echo /usr/lib/oracle/12.2/client64/lib > /etc/ld.so.conf.d/oracle-instantclient.conf"
sudo ldconfig

sqlplus -v
sqlplus scott/tiger@<ip>/XE
sqlplus scott/tiger@<ip>/XE as <other_account>       # if delegation rights
```

Inside sqlplus:

```sql
select * from user_role_privs;
select name, password from sys.user$;   -- if logged in as sysdba
```

If Oracle is fronting a web app, upload via UTL_FILE:

```bash
echo "Oracle File Upload Test" > testing.txt
./odat.py utlfile -s <ip> -d XE -U scott -P tiger --sysdba \
    --putFile 'C:\inetpub\wwwroot' testing.txt ./testing.txt
curl http://<ip>/testing.txt
```

### IPMI (623 UDP)

```bash
sudo nmap -sU --script ipmi-version -p 623 <ip>

## Dump and crack with Metasploit
use auxiliary/scanner/ipmi/ipmi_dumphashes
set RHOSTS <ip>
run

hashcat -m 7300 hash.txt -a 3 ?1?1?1?1?1?1?1?1 -1 ?d?u
```

### SSH (22)

```bash
nmap -sC -sV -p22 <ip>
ssh -v user@<ip>                                # show offered auth methods
ssh -v user@<ip> -o PreferredAuthentications=password
```

Dangerous `sshd_config` flags:

| Setting | Why |
|---------|-----|
| `PasswordAuthentication yes` | Allows password auth (sprayable) |
| `PermitEmptyPasswords yes` | Empty passwords accepted |
| `PermitRootLogin yes` | Direct root login |
| `Protocol 1` | Outdated crypto |
| `X11Forwarding yes` | X11 forwarding |
| `AllowTcpForwarding yes` | TCP forwarding |
| `PermitTunnel` | Layer-3 tunneling allowed |

Don't forget to read `/etc/passwd` for service accounts (mysql, etc.) you can target.

### Rsync (873)

```bash
sudo nmap -sV -p873 <ip>
nc -nv <ip> 873                                # banner
rsync -av --list-only rsync://<ip>/<share>
```

### R-services (512/513/514)

```bash
sudo nmap -sV -p 512,513,514 <ip>
rlogin <ip> -l htb-student
rwho
rusers -al <ip>
```

### RDP (3389)

```bash
nmap -sV -sC -p3389 --script rdp* <ip>

## Security check
git clone https://github.com/CiscoCXSecurity/rdp-sec-check.git
./rdp-sec-check.pl <ip>

## Clients
xfreerdp /u:<user> /p:'<pass>' /v:<ip>
rdesktop -u <user> -p - <ip>
remmina -c rdp://<user>@<ip>
remmina -c rdp://<DOMAIN>\\<user>@<ip>
```

### WinRM (5985/5986)

```bash
nmap -sV -sC -p5985,5986 --disable-arp-ping -n <ip>
evil-winrm -i <ip> -u <user> -p <pass>
evil-winrm -i <ip> -u <user> -H <NTLM>
```

### WMI (135)

```bash
/usr/share/doc/python3-impacket/examples/wmiexec.py user:'pass'@<ip> "hostname"
```

---

### Server-Profile Cheats

| Server type | Expect to see |
|-------------|---------------|
| Mail / management | SSH, POP3, IMAPS, SNMP |
| Windows server | WinRM, RDP, WMI, SMB |
| DNS server | DNS, FTP, SSH |
