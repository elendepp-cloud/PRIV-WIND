# Network Enumeration & Exploitation Cheat Sheet
389	LDAP	Directory Services
443	HTTPS	Secure Web Server
445	SMB	Server Message Block
464	Kerberos	Password Change
636	LDAPS	Secure LDAP
1433	MSSQL	Microsoft SQL Server
3268	Global Catalog	Active Directory
3269	Global Catalog SSL	Secure Active Directory
3389	RDP	Remote Desktop
5985	WinRM	Windows Remote Management
5986	WinRM HTTPS	Secure Remote Management
9389	ADWS	Active Directory Web Services

A practical reference for network reconnaissance, service enumeration, authentication testing, and remote access in authorized penetration testing labs.

## 1. Network Scanning — Nmap

### Basic scan

```bash
nmap <IP>
```

### Service and version detection

```bash
nmap -sC -sV <IP>
```

### Scan all TCP ports

```bash
nmap -p- --min-rate 1000 <IP>
```

### Scan specific ports

```bash
nmap -p 21,22,53,80,88,135,139,389,443,445,1433,3389,5985,5986 <IP>
```

### UDP scan

```bash
sudo nmap -sU --top-ports 100 <IP>
```

### Save results

```bash
nmap -sC -sV -oN scan.txt <IP>
```

---

## 2. SMB — Server Message Block

**Ports:** 445, 139

SMB is used for file sharing, network communication, and access to shared resources.

### Nmap enumeration

```bash
nmap -p 445 --script smb-protocols <IP>
nmap -p 445 --script smb-os-discovery <IP>
nmap -p 445 --script smb-enum-shares <IP>
nmap -p 445 --script smb-enum-users <IP>
```

### smbclient

List shares without credentials:

```bash
smbclient -L //<IP>/ -N
```

Connect to a share:

```bash
smbclient //<IP>/<SHARE> -U <USER>
```

Useful commands inside smbclient:

```text
ls
cd <directory>
get <file>
put <file>
mget *
recurse ON
prompt OFF
exit
```

### NetExec (nxc)

Check SMB:

```bash
nxc smb <IP>
```

Authenticate:

```bash
nxc smb <IP> -u <USER> -p '<PASS>'
```

Enumerate users:

```bash
nxc smb <DC_IP> -u <USER> -p '<PASS>' --users
```

Enumerate groups:

```bash
nxc smb <DC_IP> -u <USER> -p '<PASS>' --groups
```

Enumerate password policy:

```bash
nxc smb <DC_IP> -u <USER> -p '<PASS>' --pass-pol
```

Combine enumeration:

```bash
nxc smb <DC_IP> -u <USER> -p '<PASS>' --users --groups --pass-pol
```

Retrieve SAM hashes (requires appropriate administrative privileges):

```bash
nxc smb <IP> -u <USER> -p '<PASS>' --sam
```

Retrieve LSA secrets (requires appropriate administrative privileges):

```bash
nxc smb <IP> -u <USER> -p '<PASS>' --lsa
```

### Impacket SMB Client

```bash
impacket-smbclient <DOMAIN>/<USER>:'<PASS>'@<IP>
```

### Impacket SMBExec

```bash
impacket-smbexec <DOMAIN>/<USER>:'<PASS>'@<IP>
```

Remote command execution generally requires administrative privileges.

### Impacket PsExec

```bash
impacket-psexec <DOMAIN>/<USER>:'<PASS>'@<IP>
```

---

## 3. FTP — File Transfer Protocol

**Port:** 21

FTP is used to transfer files between a client and a server.

### Enumeration

```bash
nmap -p 21 -sC -sV <IP>
```

Check anonymous access:

```bash
nmap -p 21 --script ftp-anon <IP>
```

Connect:

```bash
ftp <IP>
```

Anonymous login:

```text
Username: anonymous
Password: anonymous
```

### FTP commands

```text
ls
pwd
cd <directory>
get <file>
put <file>
mget *
mkdir <directory>
delete <file>
bye
```

Check FTP server information:

```bash
nmap -p 21 --script ftp-syst <IP>
```

**Note:** Standard FTP does not encrypt credentials or transferred files. Check whether FTPS is supported.

---

## 4. DNS — Domain Name System

**Port:** 53 TCP/UDP

DNS translates domain names into IP addresses and helps identify network infrastructure.

### Basic queries

```bash
nslookup <DOMAIN>
dig <DOMAIN>
```

### Record enumeration

```bash
dig A <DOMAIN>
dig AAAA <DOMAIN>
dig MX <DOMAIN>
dig NS <DOMAIN>
dig TXT <DOMAIN>
```

### Reverse lookup

```bash
dig -x <IP>
```

### Zone transfer

```bash
dig axfr <DOMAIN> @<DNS_SERVER>
```

A successful zone transfer may reveal internal hostnames, IP addresses, and infrastructure records.

### DNS enumeration tools

```bash
dnsenum <DOMAIN>
```

```bash
dnsrecon -d <DOMAIN>
```

---

## 5. RDP — Remote Desktop Protocol

**Port:** 3389

RDP provides remote graphical access to Windows systems.

### Scan

```bash
nmap -p 3389 -sV <IP>
```

### Connect with FreeRDP

```bash
xfreerdp3 /v:<IP> /u:<USER> /p:'<PASS>' /cert:ignore
```

Domain authentication:

```bash
xfreerdp3 /v:<IP> /u:<DOMAIN>\<USER> /p:'<PASS>' /cert:ignore
```

### Windows client

```powershell
mstsc
```

---

## 6. WinRM — Windows Remote Management

**Ports:** 5985 (HTTP), 5986 (HTTPS)

WinRM enables remote Windows administration through PowerShell.

### Scan

```bash
nmap -p 5985,5986 <IP>
```

### NetExec

```bash
nxc winrm <IP> -u <USER> -p '<PASS>'
```

### Evil-WinRM

```bash
evil-winrm -i <IP> -u <USER> -p '<PASS>'
```

---

## 7. SSH — Secure Shell

**Port:** 22

SSH provides encrypted remote command-line access.

### Scan

```bash
nmap -p 22 -sV <IP>
```

### Connect

```bash
ssh <USER>@<IP>
```

### Connect using a private key

```bash
ssh -i <PRIVATE_KEY> <USER>@<IP>
```

---

## 8. RPC — Remote Procedure Call

**Port:** 135 and dynamic RPC ports

RPC allows Windows services and applications to communicate remotely.

### Connect with rpcclient

```bash
rpcclient -U "" -N <IP>
```

Authenticated connection:

```bash
rpcclient -U '<DOMAIN>/<USER>' <IP>
```

### Useful commands

```text
enumdomusers
enumdomgroups
querydominfo
netshareenumall
```

### enum4linux-ng

```bash
enum4linux-ng -A <IP>
```

---

## 9. LDAP — Lightweight Directory Access Protocol

**Ports:** 389 (LDAP), 636 (LDAPS)

LDAP provides access to directory information, including Active Directory objects.

### Scan

```bash
nmap -p 389,636 <IP>
```

### RootDSE query

```bash
ldapsearch -x -H ldap://<IP> -s base
```

### Search directory information

```bash
ldapsearch -x -H ldap://<IP> -b '<BASE_DN>'
```

Example:

```text
DC=example,DC=local
```

### Authenticated LDAP query

```bash
ldapsearch -x -H ldap://<IP> -D '<USER>@<DOMAIN>' -W -b '<BASE_DN>'
```

Use LDAPS or another protected configuration when handling credentials.

---

## 10. Kerberos

**Ports:** 88, 464

Kerberos is an authentication protocol commonly used in Active Directory environments.

### Scan

```bash
nmap -p 88,464 <IP>
```

### View tickets on Windows

```powershell
klist
```

Purge tickets from the current logon session:

```powershell
klist purge
```

---

## 11. MSSQL — Microsoft SQL Server

**Port:** 1433

MSSQL is Microsoft's relational database management system.

### Scan

```bash
nmap -p 1433 -sV <IP>
```

### Connect with Impacket

```bash
impacket-mssqlclient <DOMAIN>/<USER>:'<PASS>'@<IP>
```

For SQL authentication:

```bash
impacket-mssqlclient <USER>:'<PASS>'@<IP>
```

---

## 12. HTTP / HTTPS

**Ports:** 80, 443

HTTP and HTTPS are used to access web applications and web services.

### Scan

```bash
nmap -p 80,443 -sC -sV <IP>
```

### Check response headers

```bash
curl -I http://<IP>
```

### Web directory enumeration

```bash
gobuster dir -u http://<IP> -w <WORDLIST>
```

Alternative:

```bash
ffuf -u http://<IP>/FUZZ -w <WORDLIST>
```

### Technology detection

```bash
whatweb http://<IP>
```

---

## 13. Common Network Enumeration Commands

### Local network information

```bash
ip a
```

### Routing table

```bash
ip route
```

### ARP / Neighbor table

```bash
ip neigh
```

### Active connections

```bash
ss -antup
```

### DNS lookup

```bash
nslookup <DOMAIN>
```

### Test connectivity

```bash
ping <IP>
```

### Trace network route

```bash
traceroute <IP>
```

---

## 14. Important Tools

| Tool | Purpose |
|---|---|
| Nmap | Port scanning and service detection |
| NetExec (nxc) | SMB, LDAP, WinRM and other network enumeration |
| Impacket | Network protocol and Active Directory utilities |
| smbclient | SMB share access |
| smbmap | SMB share enumeration |
| rpcclient | RPC enumeration |
| enum4linux-ng | SMB/RPC enumeration |
| Evil-WinRM | WinRM remote shell |
| FreeRDP | RDP connection |
| dig | DNS queries |
| dnsenum | DNS enumeration |
| dnsrecon | DNS reconnaissance |
| ldapsearch | LDAP queries |
| Gobuster | Web directory enumeration |
| FFUF | Web fuzzing |
| WhatWeb | Web technology detection |

---

## 15. Network Enumeration Workflow

1. Scan the target with Nmap.
2. Identify open ports and service versions.
3. Enumerate each exposed service.
4. Check for anonymous or guest access where applicable.
5. Test authorized credentials against relevant services.
6. Enumerate users, groups, shares, and domain information.
7. Identify accessible remote-management services.
8. Document findings and validate potential misconfigurations.

### Important Reminders

- Open ports do not automatically indicate vulnerabilities.
- Authentication and authorization are different.
- SMB share permissions and NTFS permissions are separate.
- SAM and LSA extraction require appropriate privileges.
- Remote execution tools require sufficient permissions.
- FTP without TLS can expose credentials and data.
- DNS zone transfers should only be tested against authorized systems.
- Never expose real passwords or tokens in public GitHub repositories.
